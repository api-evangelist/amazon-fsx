---
title: "Managing NTFS permissions at scale on Amazon FSx for NetApp ONTAP"
url: "https://aws.amazon.com/blogs/storage/manage-ntfs-permissions-at-scale-on-amazon-fsx-for-netapp-ontap/"
date: "Tue, 07 Apr 2026 15:49:28 +0000"
author: "Aravindan A"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
<p>Organizations migrating large file systems to the cloud often encounter New Technology File System (NTFS) permission inconsistencies where folders lack proper access control list (ACL) entries, preventing inheritance to new files and subfolders. Without proper inheritance, newly created content doesn’t automatically receive the correct permissions, leading to access denied errors for legitimate users, hours of manual permission fixes, and application failures during and after migrations. Traditional client-side approaches to managing NTFS permissions struggle at scale. They operate slowly over the Server Message Block (SMB) protocol, become error-prone when processing thousands of files, and lack atomic operations for bulk changes. For enterprises managing millions of files, these approaches can turn what should be a straightforward permission update into an 8-12 hour manual remediation effort.</p> 
<p><a href="https://aws.amazon.com/fsx/netapp-ontap/" rel="noopener noreferrer" target="_blank">Amazon FSx for NetApp ONTAP</a> (FSx for ONTAP) provides enterprise-grade file storage with ONTAP capabilities so administrators can manage NTFS permissions at the storage layer rather than through client-side tools. By using ONTAP security descriptors and policies, you can apply permissions recursively across millions of files with operations that are significantly faster than Windows-based approaches. These operations run as single, all-or-nothing transactions at the storage level, delivering consistent results without partial failures. This storage-layer approach provides consistent ACL coverage, proper inheritance flags, and the ability to reapply permissions after data migrations, all without the performance bottlenecks and reliability issues of SMB-based tools.</p> 
<p>In this post, we demonstrate how to use ONTAP security descriptors and policies to apply consistent NTFS permissions at scale on FSx for ONTAP. You will learn how to create reusable permission templates, configure inheritance flags to make sure new content automatically receives correct permissions, apply policies recursively across existing files and folders, and validate that permissions propagate correctly throughout your volume hierarchy. This approach reduces permission management time from hours to minutes while providing reliable, repeatable results across your entire file system.</p> 
<h2>Solution overview</h2> 
<p>The solution uses the following ONTAP components to manage NTFS permissions at scale:</p> 
<ul> 
 <li><strong>Security descriptor</strong> – A reusable template that defines ownership and discretionary access control list (DACL) entries, specific permissions such as full-control, modify, read-and-execute, read, and write assigned to users or groups, with inheritance flags.</li> 
 <li><strong>Security policy</strong> – A container that links security descriptors to target volume paths.</li> 
 <li><strong>Apply operation</strong> – A job-based execution that recursively propagates permissions across files and subfolders.</li> 
</ul> 
<p><em>Figure 1</em> illustrates the complete workflow for applying NTFS permissions at scale using FSx for ONTAP.</p> 
<p><img alt="Diagram showing the workflow for applying NTFS permissions using ONTAP security descriptors and policies on Amazon FSx for NetApp ONTAP, including security descriptor creation, DACL configuration, policy linking, and recursive apply operation" class="wp-image-28643 size-full aligncenter" height="1001" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2026/04/07/FSx-ONTAP-NTFS-Permissions-Implementation-Architecture-Blog.drawio.png" width="1531" /></p> 
<p style="text-align: center;"><em>Figure 1: Applying NTFS permissions at scale using FSx for ONTAP</em></p> 
<p>The workflow follows a simple pattern: create a security descriptor template, add DACL entries with appropriate permissions and inheritance flags, create a policy that links the descriptor to your target path, and apply the policy recursively. ONTAP’s job engine handles the execution, updating permissions on millions of files while maintaining consistency through atomic operations.The solution uses the following key inheritance flags:</p> 
<ul> 
 <li><strong>OI (Object Inherit)</strong> – Files inherit this permission.</li> 
 <li><strong>CI (Container Inherit)</strong> – Subfolders inherit this permission.</li> 
 <li><strong>OI|CI</strong> – Both files and subfolders inherit, achieving complete top-down inheritance.</li> 
</ul> 
<p>This approach operates at the storage layer, bypassing SMB protocol overhead and delivering significantly faster performance than Windows-based tools.</p> 
<h2>Prerequisites</h2> 
<p>Before implementing this solution, ensure you have the following:</p> 
<ul> 
 <li>An FSx for ONTAP file system with at least one Storage Virtual Machine (SVM).</li> 
 <li>A volume configured with NTFS security style.</li> 
 <li>Active Directory integration configured for your SVM.</li> 
 <li>SSH access to the FSx for ONTAP management endpoint within the virtual private cloud (VPC) using <strong>fsxadmin</strong> credentials.</li> 
 <li>Basic understanding of NTFS permissions and Active Directory groups.</li> 
</ul> 
<p>To verify your prerequisites, use the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># SSH to your FSx ONTAP management endpoint
ssh fsxadmin@<span style="color: red;">&lt;management-ip-address&gt;</span>

# Verify SVM and volume configuration
vserver show
volume show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -fields security-style</code></pre> 
</div> 
<h2>Implement the solution</h2> 
<p>This section walks you through the complete implementation, from verifying current permissions to validating the final results.</p> 
<h3>Step 1: Verify current permissions</h3> 
<p>Before making changes, examine the current permission state on your target folder to understand what’s missing:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -path <span style="color: red;">&lt;folder-path&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory show -vserver svm01 -path /data_volume</code></pre> 
</div> 
<p>We get the following output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">                  Vserver: svm01
                File Path: /data_volume
           Security Style: ntfs
          Effective Style: ntfs
                     ACLs: NTFS Security Descriptor
                           Owner:DOMAIN\Storage_Admins
                           Group:BUILTIN\Administrators
                           DACL - ACEs
                             ALLOW-DOMAIN\Storage_Admins-0x1f01ff-OI|CI (Inherited)</code></pre> 
</div> 
<p>Look for missing groups such as DOMAIN\FileShare_Users in the DACL to identify which permission entries need to be added. Note whether inheritance flags (OI|CI) are present on existing entries.</p> 
<h3>Step 2: Create NTFS security descriptor</h3> 
<p>Create a security descriptor that serves as a template for your permissions. This descriptor will define ownership and contain the DACL entries you will add in the next step.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs create -vserver <span style="color: red;">&lt;svm-name&gt;</span> -ntfs-sd <span style="color: red;">&lt;descriptor-name&gt;</span> -owner "<span style="color: red;">&lt;DOMAIN\Group&gt;</span>"</code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs create -vserver svm01 -ntfs-sd data_volume_sd -owner "DOMAIN\Storage_Admins"</code></pre> 
</div> 
<p>Verify security descriptor creation:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs show -vserver svm01 -ntfs-sd data_volume_sd</code></pre> 
</div> 
<p>This displays the security descriptor with its owner and confirms successful creation.</p> 
<p>This code uses the following parameters:</p> 
<ul> 
 <li><strong>-vserver</strong> – Your SVM name</li> 
 <li><strong>-ntfs-sd</strong> – A descriptive name for your security descriptor</li> 
 <li><strong>-owner</strong> – Optional owner specification in DOMAIN\Group or DOMAIN\User format</li> 
</ul> 
<p>ONTAP automatically adds four default Windows security groups to new security descriptors:</p> 
<ul> 
 <li>BUILTIN\Administrators</li> 
 <li>BUILTIN\Users</li> 
 <li>CREATOR OWNER</li> 
 <li>NT AUTHORITY\SYSTEM</li> 
</ul> 
<p>If you want a minimal DACL with only specific entries, you can remove these defaults using the vserver security file-directory ntfs dacl remove command before adding your custom entries.</p> 
<h3>Step 3: Configure DACL entries</h3> 
<p>Add specific permission entries to your security descriptor. This is where you define which Active Directory groups or users have what level of access. Using groups (like DOMAIN\FileShare_Users) is recommended for easier management at scale.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs dacl add -vserver <span style="color: red;">&lt;svm-name&gt;</span> -ntfs-sd <span style="color: red;">&lt;descriptor-name&gt;</span> -access-type allow -account "<span style="color: red;">&lt;account-name&gt;</span>" -rights <span style="color: red;">&lt;permission-level&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs dacl add -vserver svm01 -ntfs-sd data_volume_sd -access-type allow -account "DOMAIN\FileShare_Users" -rights full-control</code></pre> 
</div> 
<p>Verify the DACL entry was added:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs dacl show -vserver svm01 -ntfs-sd data_volume_sd</code></pre> 
</div> 
<p>This confirms the permission entry was added with the correct access rights and inheritance flags. In this example, DOMAIN\FileShare_Users is an Active Directory security group. Using groups instead of individual user accounts simplifies permission management, and you can control access by adding or removing users from the group.</p> 
<p>The following permission levels are available:</p> 
<ul> 
 <li><strong>full-control</strong> – Read, write, modify, delete, change permissions (0x1f01ff)</li> 
 <li><strong>modify</strong> – Read, write, modify without delete or permission changes (0x1301bf)</li> 
 <li><strong>read-and-execute</strong> – Read and execute only (0x1200a9)</li> 
 <li><strong>read</strong> – Read-only access (0x120089)</li> 
 <li><strong>write</strong> – Write access (0x100116)</li> 
</ul> 
<p>Start with the minimum required permissions and expand as needed. For most migration scenarios, full-control for DOMAIN\FileShare_Users makes sure authenticated users can access their files, and NTFS permissions on individual files and folders provide granular control.</p> 
<h3>Step 4: Verify security descriptor configuration</h3> 
<p>Before applying your security descriptor, confirm it contains the correct permissions and inheritance flags:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs dacl show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -ntfs-sd <span style="color: red;">&lt;descriptor-name&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory ntfs dacl show -vserver svm01 -ntfs-sd data_volume_sd</code></pre> 
</div> 
<p>We get the following output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">Vserver: svm01
  NTFS Security Descriptor Name: data_volume_sd

    Account Name              Access   Access             Apply To
                              Type     Rights
    -----------------------   -------  -------            -----------
    DOMAIN\FileShare_Users    allow    full-control       this-folder, sub-folders, files</code></pre> 
</div> 
<p>Verify the output shows the correct account name, access type is allow, permission level matches your intent, and Apply To includes this-folder, sub-folders, files (indicating OI|CI flags are set).</p> 
<h3>Step 5: Create security policy</h3> 
<p>Create a policy that will act as a container for linking your security descriptor to specific paths:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy create -vserver <span style="color: red;">&lt;svm-name&gt;</span> -policy-name <span style="color: red;">&lt;policy-name&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy create -vserver svm01 -policy-name data_volume_policy</code></pre> 
</div> 
<p>Verify creation:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy show -vserver svm01</code></pre> 
</div> 
<p>A single policy can contain multiple tasks, so you can apply different security descriptors to different paths within the same policy execution.</p> 
<h3>Step 6: Add task to policy</h3> 
<p>Link your security descriptor to the target path by adding a task to the policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy task add -vserver <span style="color: red;">&lt;svm-name&gt;</span> -policy-name <span style="color: red;">&lt;policy-name&gt;</span> -path <span style="color: red;">&lt;folder-path&gt;</span> -ntfs-sd <span style="color: red;">&lt;descriptor-name&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy task add -vserver svm01 -policy-name data_volume_policy -path /data_volume -ntfs-sd data_volume_sd</code></pre> 
</div> 
<p>This code uses the following parameters:</p> 
<ul> 
 <li><strong>-path</strong> – Junction path to your target folder (the volume’s mount point path, must start with /)</li> 
 <li><strong>-ntfs-sd</strong> – The security descriptor name you created in Step 2</li> 
</ul> 
<p>Verify task creation:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory policy task show -vserver svm01 -policy-name data_volume_policy</code></pre> 
</div> 
<h3>Step 7: Apply policy recursively</h3> 
<p>Execute the policy to apply permissions recursively to existing files and folders:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory apply -vserver <span style="color: red;">&lt;svm-name&gt;</span> -policy-name <span style="color: red;">&lt;policy-name&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory apply -vserver svm01 -policy-name data_volume_policy</code></pre> 
</div> 
<p>We get the following output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">[Job 1234] Job is queued: Fsecurity Apply. Use the "job show -id 1234" command to view the status of this operation.</code></pre> 
</div> 
<p>ONTAP queues a background job that recursively updates permissions on existing content. The job processes folders first, then files, making sure inheritance propagates correctly throughout the directory tree. Note the job ID for monitoring in the next step.</p> 
<h3>Step 8: Monitor job progress</h3> 
<p>Track the apply operation status to confirm successful completion:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">job show -id <span style="color: red;">&lt;job-id&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">job show -id 1234</code></pre> 
</div> 
<p>We get the following output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">                            Owning
Job ID Name                 Vserver    Node           State
------ -------------------- ---------- -------------- ----------
1234   Fsecurity Apply      svm01      FsxId01        Running
       Description: File Directory Security Apply Job</code></pre> 
</div> 
<p>The job states are as follows:</p> 
<ul> 
 <li><strong>Queued</strong> – Job waiting to start.</li> 
 <li><strong>Running</strong> – Job in progress (check periodically).</li> 
 <li><strong>Success</strong> – Job completed successfully.</li> 
 <li><strong>Failed</strong> – Job encountered errors (check job details).</li> 
</ul> 
<p>For detailed job information, use the following code:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">job show -id 1234 -instance</code></pre> 
</div> 
<p>Monitor the job periodically until the state shows Success. Large directories might take considerable time to complete.</p> 
<h3>Step 9: Validate permissions and inheritance</h3> 
<p>Confirm permissions were applied correctly to folders, subfolders, and files:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># Verify root folder
vserver security file-directory show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -path <span style="color: red;">&lt;folder-path&gt;</span>

# Verify subfolder
vserver security file-directory show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -path <span style="color: red;">&lt;folder-path&gt;</span>/<span style="color: red;">&lt;subfolder&gt;</span>

# Verify file
vserver security file-directory show -vserver <span style="color: red;">&lt;svm-name&gt;</span> -path <span style="color: red;">&lt;folder-path&gt;</span>/<span style="color: red;">&lt;subfolder&gt;</span>/<span style="color: red;">&lt;filename&gt;</span></code></pre> 
</div> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory show -vserver svm01 -path /data_volume
vserver security file-directory show -vserver svm01 -path /data_volume/project_files
vserver security file-directory show -vserver svm01 -path /data_volume/project_files/document.txt</code></pre> 
</div> 
<p>We get the following output:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">                  Vserver: svm01
                File Path: /data_volume
           Security Style: ntfs
          Effective Style: ntfs
                     ACLs: NTFS Security Descriptor
                           Control:0x8014
                           Owner:DOMAIN\Storage_Admins
                           Group:BUILTIN\Administrators
                           DACL - ACEs
                             ALLOW-DOMAIN\FileShare_Users-0x1f01ff-OI|CI</code></pre> 
</div> 
<p>This code uses the following inheritance flags:</p> 
<ul> 
 <li><strong>OI (Object Inherit)</strong> – Files inherit this permission</li> 
 <li><strong>CI (Container Inherit)</strong> – Subfolders inherit this permission</li> 
 <li><strong>0x1f01ff</strong> – Hexadecimal representation of full control permissions</li> 
</ul> 
<p>Verify DOMAIN\FileShare_Users appears in the DACL with correct permissions, inheritance flags (OI|CI) are present, and the same permissions appear on subfolders and files.Now you can test inheritance on new content. From a Windows or Linux client, create a test file in the folder and verify it automatically inherits the Everyone permission:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># From Linux client
touch /mnt/data_volume/test_file.txt

# Check permissions from ONTAP CLI
vserver security file-directory show -vserver svm01 -path /data_volume/test_file.txt</code></pre> 
</div> 
<p>The new file should show DOMAIN\FileShare_Users permission inherited from the parent folder, confirming inheritance is working correctly for future content.</p> 
<h2>Managing permissions after data migrations</h2> 
<p>When migrating data using <a href="https://aws.amazon.com/datasync/" rel="noopener noreferrer" target="_blank">AWS DataSync</a> or other tools after your initial policy application, the migrated files might have different permissions from the source system. You can update permissions by reapplying your existing policy.</p> 
<p>For example: You’ve applied NTFS permissions to your FSx for ONTAP volume, then later migrate additional data from an on-premises file server using DataSync. The newly migrated files retain their source permissions, which might not match your target environment.To fix this, reapply the existing policy without recreating security descriptors or policies:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">vserver security file-directory apply -vserver svm01 -policy-name data_volume_policy</code></pre> 
</div> 
<p>The policy recursively updates existing files and folders, including newly migrated content, maintaining consistent permissions across your entire volume. This approach simplifies post-migration permission management and makes sure all content, whether originally created on FSx for ONTAP or migrated from external sources, has consistent ACLs.</p> 
<h2>Applying different permissions to subdirectories</h2> 
<p>In many environments, not all subdirectories within a volume should have the same level of access. For example, a finance team might need restricted access to sensitive reports, whereas a broader group needs read access to shared resources. You can apply different permissions to subdirectories within the same volume by creating separate security descriptors for each path.For example, within your /data_volume, you might need different access levels for subdirectories, one for confidential data with restricted access, and another for shared data with broader read access.The following is an example permission structure:</p> 
<ul> 
 <li><strong>Parent folder (/data_volume/)</strong> – Everyone: Full control (explicit).</li> 
 <li><strong>Confidential subdirectory (/data_volume/confidential/)</strong> – DOMAIN\Confidential_Users: Modify (explicit) + Everyone: Full control (inherited).</li> 
 <li><strong>Shared subdirectory (/data_volume/shared/) </strong>– DOMAIN\Shared_Users: Read-and-execute (explicit) + Everyone: Full control (inherited).</li> 
 <li><strong>Files in subdirectories</strong> – Inherit all permissions from their immediate parent folder.</li> 
</ul> 
<p>Each subdirectory has its own explicit permissions plus inherited permissions from parent. They coexist.We use the following approach:</p> 
<ul> 
 <li>Create separate security descriptors for each subdirectory.</li> 
 <li>Define appropriate permissions for each subdirectory’s user group.</li> 
 <li>Apply policies to specific subdirectory paths.</li> 
 <li>Parent directory permissions remain independent of subdirectory permissions.</li> 
</ul> 
<p>For example:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># Confidential subdirectory - restricted access
vserver security file-directory ntfs create -vserver svm01 -ntfs-sd confidential_sd -owner "DOMAIN\Data_Admins"
vserver security file-directory ntfs dacl add -vserver svm01 -ntfs-sd confidential_sd -access-type allow -account "DOMAIN\Confidential_Users" -rights modify
vserver security file-directory policy create -vserver svm01 -policy-name confidential_policy
vserver security file-directory policy task add -vserver svm01 -policy-name confidential_policy -path /data_volume/confidential -ntfs-sd confidential_sd
vserver security file-directory apply -vserver svm01 -policy-name confidential_policy

# Shared subdirectory - broader read access
vserver security file-directory ntfs create -vserver svm01 -ntfs-sd shared_sd -owner "DOMAIN\Data_Admins"
vserver security file-directory ntfs dacl add -vserver svm01 -ntfs-sd shared_sd -access-type allow -account "DOMAIN\Shared_Users" -rights read-and-execute
vserver security file-directory policy create -vserver svm01 -policy-name shared_policy
vserver security file-directory policy task add -vserver svm01 -policy-name shared_policy -path /data_volume/shared -ntfs-sd shared_sd
vserver security file-directory apply -vserver svm01 -policy-name shared_policy</code></pre> 
</div> 
<p>When you apply policies to both parent and subdirectories, the permissions work together as follows:</p> 
<ul> 
 <li>Subdirectories keep their explicit permissions even if you later apply a policy to the parent.</li> 
 <li>Subdirectories also inherit permissions from the parent.</li> 
 <li>Both explicit and inherited permissions coexist on the subdirectory.</li> 
 <li>Files inherit all permissions from their immediate parent folder.</li> 
</ul> 
<p>This makes it possible to set specific permissions on subdirectories while still benefiting from parent-level permissions.</p> 
<h2>Managing and modifying permissions</h2> 
<p>Understanding how ONTAP handles permission updates is critical for maintaining accurate access control without unintended consequences.When you apply a security policy to a path, ONTAP replaces existing explicit permissions on that path with the permissions defined in the security descriptor. Inherited permissions from parent directories remain unchanged.<br /> For example, you might apply a policy granting read access to Group A on /data_volume/reports. Later, you apply a different policy granting write access to Group B on the same path. As a result, only Group B has explicit permissions (write). Group A’s read permission is removed.<br /> If you need to add permissions to a folder that already has explicit permissions, update the existing security descriptor rather than creating a new policy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># Add a new permission to existing security descriptor
vserver security file-directory ntfs dacl add -vserver svm01 -ntfs-sd existing_sd -access-type allow -account "DOMAIN\New_Group" -rights modify

# Reapply the same policy to update the folder
vserver security file-directory apply -vserver svm01 -policy-name existing_policy</code></pre> 
</div> 
<p>This approach preserves existing permissions in the security descriptor while adding the new permission. Note the following key points:</p> 
<ul> 
 <li>Applying a new policy replaces explicit permissions on that path.</li> 
 <li>Inherited permissions from parent folders are not affected.</li> 
 <li>To add permissions, update the security descriptor and reapply the same policy.</li> 
 <li>Verify changes with vserver security file-directory show to see explicit vs. inherited permissions.</li> 
</ul> 
<h2>Performance considerations for large volumes</h2> 
<p>ONTAP security policies handle large-scale permission updates efficiently through storage-layer operations that run as background jobs, independent of client connections.We recommend the following planning steps:</p> 
<ul> 
 <li>Schedule operations during maintenance windows to minimize user impact.</li> 
 <li>Monitor job progress with the job show command.</li> 
</ul> 
<p>The following factors affect runtime:</p> 
<ul> 
 <li>File system throughput capacity and provisioned IOPS.</li> 
 <li>Total number of files and directory structure complexity.</li> 
 <li>Concurrent workload on the file system.</li> 
</ul> 
<p>ONTAP’s job-based execution provides reliable completion even for volumes with millions of files, alleviating the timeout and failure issues common with Windows-based permission tools.</p> 
<h2>Clean up</h2> 
<p>To avoid incurring future charges, delete the resources created during this walkthrough.Remove policies and security descriptors:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code"># Remove policy task
vserver security file-directory policy task remove -vserver svm01 -policy-name data_volume_policy -path /data_volume

# Delete policy
vserver security file-directory policy delete -vserver svm01 -policy-name data_volume_policy

# Delete security descriptor
vserver security file-directory ntfs delete -vserver svm01 -ntfs-sd data_volume_sd</code></pre> 
</div> 
<p>Deleting policies and security descriptors only removes the templates, permissions already applied to files and folders remain unchanged. If you created a test FSx for ONTAP file system specifically for this walkthrough, delete it through the <a href="https://console.aws.amazon.com/fsx/" rel="noopener noreferrer" target="_blank">Amazon FSx console</a> to stop incurring storage charges.</p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to use ONTAP security descriptors and policies to manage NTFS permissions at scale on FSx for ONTAP. This storage-layer approach delivers significant advantages over traditional Windows-based tools: faster performance, reliable operations, proper inheritance configuration, and the ability to reapply permissions after migrations.By following the implementation walkthrough presented in this post, you can apply consistent NTFS permissions to millions of files in minutes rather than hours, maintaining proper inheritance for future content and simplifying post-migration permission management. The advanced use cases demonstrate how this approach scales to support DataSync migrations, multi-tenant environments, and large-scale operations involving millions of files.</p> 
<p>To learn more about FSx for ONTAP, refer to the <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/what-is-fsx-ontap.html" rel="noopener noreferrer" target="_blank">Amazon FSx for NetApp ONTAP User Guide</a>. If you have questions or comments about this post, please leave them in the comments section.</p>
