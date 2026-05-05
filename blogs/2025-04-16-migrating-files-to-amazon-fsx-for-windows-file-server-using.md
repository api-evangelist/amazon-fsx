---
title: "Migrating files to Amazon FSx for Windows File Server using Robocopy"
url: "https://aws.amazon.com/blogs/storage/migrating-files-to-fsx-for-windows-file-server-using-robocopy/"
date: "Wed, 16 Apr 2025 16:46:57 +0000"
author: "Dante Ventura"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
<p>When migrating file shares to AWS, users need a direct and efficient solution for transferring their data. <a href="https://aws.amazon.com/fsx/windows/" rel="noopener" target="_blank">Amazon FSx for Windows File Server</a> provides fully managed shared storage built on Windows Server, and delivers a wide range of data access, data management, and administrative capabilities.</p> 
<p>There are a few different choices for migration. <a href="https://docs.aws.amazon.com/fsx/latest/WindowsGuide/migrate-files-to-fsx-datasync.html" rel="noopener" target="_blank">AWS DataSync</a> is the most common tool for the job, however setting it up can introduce added complexity, especially for smaller migrations. In such cases, Robocopy (short for “Robust File Copy”) can be a powerful alternative. Robocopy is a command-line utility built into Windows, designed for copying and synchronizing files and directories, which offers control, scalability, and file verification.</p> 
<p>In this post, we walk you through the process of migrating data from an existing Windows File Server to Amazon FSx for Windows File Server using Robocopy. We do recommend using DataSync for moving data to or from AWS storage services (such as <a href="https://aws.amazon.com/s3/" rel="noopener" target="_blank">Amazon S3</a>, <a href="https://aws.amazon.com/efs/" rel="noopener" target="_blank">Amazon Elastic File System (Amazon EFS),</a> FSx for Windows File Server) in most cases. This is because DataSync is a cross-platform service that can handle NFS and SMB protocols, while Robocopy is a Windows-only tool that works specifically with NTFS. Robocopy also does not have built-in encryption or compression, thus it can be detrimental for larger migrations and isn’t recommended when moving data across unsecured channels such as the internet.</p> 
<h2><strong>Solution overview</strong></h2> 
<p>Our objective is to migrate existing files to the cloud quickly and reliably. To do this, we use three key components: 1/source file server, 2/FSx for Windows file system, and 3/EC2 intermediary instance that acts as the bridge between the source file server and FSx for Windows file system. The solution is shown in <em>figure 1</em>.</p> 
<p>We begin by creating the Amazon FSx for Windows file system. This acts as the final destination for our files. Then, we create an EC2 instance and mount the source file share (on-premises or otherwise) and the Amazon FSx for Windows file system target file shares to it. When the EC2 instance has access to both the source file server and FSx for Windows File Server, we can use Robocopy to migrate the data from one share to the other efficiently.</p> 
<p><img alt="Storage architecture with names" class=" wp-image-25077 aligncenter" height="390" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2025/04/28/Storage-architecture-with-names.png" width="1042" /></p> 
<p style="text-align: center;"><em>Figure 1: Solution overview</em></p> 
<p>Although a t2.medium instance may suffice for testing, we recommend choosing a larger instance, especially with higher throughput SSD volumes such as gp2 or gp3 volumes for large-scale or time-sensitive migrations. Lower-tier instances could act as a performance bottleneck during high-data transfers.</p> 
<h2><span style="color: #000000;"><strong>Prerequisites</strong></span></h2> 
<p>Amazon FSx for Windows File Server needs domain integration for authentication and access control. You can join FSx for Windows File Server to an external domain controller or use AWS Managed Microsoft AD. <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html" rel="noopener" target="_blank">AWS Managed Microsoft AD</a> is a simple, managed Active Directory service that can be set up with just a few clicks in the AWS console. You can follow this post on how to set it up. You also need an existing file server and established connectivity from it to AWS through a private tunnel, Direct Connect, or the internet.</p> 
<h2><span style="color: #000000;"><strong>Walkthrough</strong></span></h2> 
<p>We will walk through the following steps to implement this solution.</p> 
<ol> 
 <li><strong>Set up an Amazon FSx for Windows File Server file system.</strong></li> 
</ol> 
<p style="padding-left: 40px;">An FSx for Windows File Server file system is a specific folder (and its subfolders) within FSx for Windows File Server that is made accessible to compute instances over the network using the SMB protocol. Start by creating a new FSx for Windows File Server file system that you can access.</p> 
<p style="padding-left: 80px;">1.1. From the <a href="https://aws.amazon.com/console/" rel="noopener" target="_blank">AWS Management Console</a>, navigate to FSx and choose Create file system.</p> 
<p style="padding-left: 80px;">1.2. Choose <strong>Amazon FSx for Windows file server with a Multi-AZ deployment</strong> for high availability.</p> 
<p style="padding-left: 80px;">1.3. Use SSD storage for higher throughput and scalability. The file system size can be increased but not decreased after creation.</p> 
<p style="padding-left: 80px;">1.4. Configure VPC settings and Windows authentication through AWS Managed Microsoft AD or Self-managed AD. To create an AWS Managed Microsoft AD from scratch, you can follow this <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_getting_started.html" rel="noopener" target="_blank">documentation</a>. Make sure that the VPC selected is the VPC in which you want this file server to exist in the future. Depending on your scenario, fill out the information regarding your domain.</p> 
<p style="padding-left: 80px;">1.5. Complete the set up and save your DNS name and network configuration details when available. On the <strong>Review and create</strong> page, you can now see a summary of your selections for the FSx for Windows File Server file system. Note the settings that can and can’t be modified after the file system has been created, then choose the <strong>Create file system</strong> button.</p> 
<p style="padding-left: 80px;">1.6. When the file system has been created and shows as <strong>Available</strong> on the <strong>File systems</strong> overview page, choose <strong>File system ID</strong> and note the <strong>DNS name</strong> for the file system under <strong>Network and security.</strong></p> 
<p><img alt="Review and create" class=" wp-image-24914 aligncenter" height="358" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2025/04/14/Figure-2-1.png" width="1143" /></p> 
<p style="text-align: center;"><em>Figure 2: Create a file system</em></p> 
<p style="padding-left: 40px;"><strong>2. Set up an EC2 Windows instance</strong></p> 
<p style="padding-left: 40px;">Now that you have a FSx for Windows File Server running, you can spin up a client in Amazon EC2 to view and access the shares. For our example, we are using a t2.medium EC2 instance running Microsoft Windows Server 2022 Base.</p> 
<p style="padding-left: 80px;">2.1. From the search bar on the top of the console, type in “EC2” and choose <strong>EC2</strong> from the Services drop-down.</p> 
<p style="padding-left: 80px;">2.2. On the <strong>EC2 Dashboard</strong> page, choose the <strong>Launch instance</strong> button.</p> 
<p style="padding-left: 80px;">2.3. From the <strong>Launch an instance</strong> page, find the <strong>Name</strong> field and enter a name for the new EC2 instance.</p> 
<p style="padding-left: 120px;">2.3.1. Under <strong>Quick Start</strong> choose <strong>Windows</strong>, then under <strong>Amazon Machine Image (AMI)</strong> choose <strong>Microsoft Server 2022 Base</strong>.</p> 
<p style="padding-left: 120px;">2.3.2. Under <strong>Instance type</strong> choose <strong>t2.medium</strong></p> 
<p style="padding-left: 120px;">2.3.3. Under <strong>Key pair (login)</strong> there is a subsection called <strong>Key pair name</strong>. Choose <strong>Create new key pair</strong> on the right. Input a memorable key pair name, and make sure the <strong>Private key file format</strong> section has <strong>.pem</strong> chose, then choose the <strong>Create key pair</strong> button. You are prompted to download the Private Key. Save the Private Key on the local computer. This Private Key file is not used in this tutorial, but it is the only way to retrieve your local administrator password for this EC2 instance.</p> 
<p style="padding-left: 120px;">2.3.4. Under <strong>Network Settings</strong>, scroll down to <strong>Firewall (security groups)</strong> and choose a security group that allows incoming RDP connectivity. Make sure the Network VPC and Subnet are the same as the FSx for Windows File Server Share.</p> 
<p style="padding-left: 120px;">2.3.5. Under <strong>Configure storage</strong> you leave leave the default of 30 GiB of gp2 storage.</p> 
<p style="padding-left: 120px;">2.3.6. Expand the <strong>Advanced details</strong> section and locate the <strong>Domain join directory</strong> section. Expand the drop-down and choose your preferred domain.</p> 
<p style="padding-left: 120px;">2.3.7. Locate the <strong>IAM instance profile</strong> drop-down box and choose the <a href="https://aws.amazon.com/iam/" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> role that can join a domain.</p> 
<p style="padding-left: 120px;">2.3.8. Choose the <strong>Launch Instance</strong> button on the right side of the page. The creation and launch of the EC2 instance take a few minutes. Expand the Hamburger Menu icon on the left and then choose <strong>Instances</strong>. You may need to press the refresh button to see the newly created instance.</p> 
<p style="padding-left: 80px;">2.4. In the <strong>Instances</strong> page, choose the newly created <strong>Instance ID</strong> from Step 5.3.8. On the <strong>Instance summary</strong> page, note the <strong>Public IPv4 DNS</strong> address.</p> 
<p style="padding-left: 40px;"><strong>3. RDP into the EC2 instance</strong></p> 
<p style="padding-left: 40px;">The creation of the necessary resources is complete. You are now ready to connect to the EC2 instance and mount the FSx for Windows File Server file system as shown in <em>figure 3</em>.</p> 
<p style="padding-left: 80px;">3.1. Enter the <strong>Public IPv4 DNS</strong> address from Step 2.4 into the Microsoft Remote Desktop client and choose <strong>Connect</strong>.</p> 
<p style="padding-left: 80px;">3.2. You are prompted to log in to the EC2 instance. Enter the Domain Name, a backslash, and then a valid username. For testing purposes, we can use the Domain Admin username. Reference step 1.4. where the Active Directory settings were entered during the FSx for Windows File Server creation process. The full username should look something like the following: <strong>corp.example.com\admin.</strong></p> 
<p style="padding-left: 80px;">3.3. Input the password and choose <strong>OK</strong>.</p> 
<p style="padding-left: 80px;">3.4. If you are prompted for a security dialog box about an untrusted certificate, then this can be ignored. Choose <strong>Yes</strong> to continue.</p> 
<p style="padding-left: 80px;">3.5. During the first log in, the initial log in process takes a few minutes. After the desktop has appeared, you can mount your file system created in Step 1.</p> 
<p><img alt="Enter your credentials" class="size-full wp-image-24915 aligncenter" height="796" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2025/04/14/Figure-3.png" width="673" /></p> 
<p style="text-align: center;"><em>Figure 3: Connect to EC2 instance</em></p> 
<p style="padding-left: 80px;"><strong>4. Mount the FSx for Windows File Server file system</strong></p> 
<p style="padding-left: 80px;">Next, you mount the file shares on the EC2 instance. This instance acts as the intermediary between source file server and AWS.</p> 
<p style="padding-left: 80px;">4.1. After you’re connected, open File Explorer as in <em>figure 4</em>.</p> 
<p style="padding-left: 80px;">4.2. In the navigation panel, open the context (right-click) menu for Network, and choose <strong>Map Network Drive</strong>.</p> 
<p style="padding-left: 80px;">4.3. For Drive, choose a drive letter.</p> 
<p style="padding-left: 80px;">4.4. For Folder, enter either the file system’s DNS name or a DNS alias associated with the file system, and the share name.</p> 
<p style="padding-left: 80px;">Using an IP address instead of the DNS name could result in unavailability during the failover process of the Multi-Availability Zone (AZ) file system. Furthermore, DNS names or associated DNS aliases are necessary for Kerberos-based authentication in Multi-AZ and Single- AZ file systems. You can find the file system’s DNS name and any associated DNS aliases on the Amazon FSx console by choosing <strong>Windows File Server, Network &amp; security</strong>.</p> 
<p style="padding-left: 80px;">4.5. To use a DNS alias associated with the file system, enter the following for <strong>Folder</strong>.</p> 
<p style="padding-left: 120px;">\\&lt;fqdn-dns-alias&gt;\share</p> 
<p><img alt="map to network folder" class="size-full wp-image-24919 aligncenter" height="535" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2025/04/14/Figure-4.png" width="702" /></p> 
<p style="text-align: center;"><em>Figure 4: Map network drive</em></p> 
<p style="padding-left: 40px;"><strong>5. Mount the source file share</strong></p> 
<p style="padding-left: 80px;">5.1. After you’re connected, open File Explorer.</p> 
<p style="padding-left: 80px;">5.2. In the navigation panel, open the context (right-click) menu for Network, and choose <strong>Map Network Drive</strong>.</p> 
<p style="padding-left: 80px;">5.3. For Drive, choose a drive letter.</p> 
<p style="padding-left: 80px;">5.4. For Folder, enter either the file system’s DNS name or IP associated with the source file system, and the share name.</p> 
<p style="padding-left: 40px;"><strong>6. Using Robocopy</strong></p> 
<p style="padding-left: 40px;">Here you use Robocopy to migrate the files from source to AWS using the created EC2 instance as the bridge between both environments.</p> 
<p style="padding-left: 80px;">6.1. Open Command Prompt or Windows PowerShell as an administrator, and run the following Robocopy command to copy the files from the source share to the target share as shown in <em>figure 5</em>.</p> 
<pre><code class="lang-bash">robocopy
Y:\source-folder\ Z:\target-FSx-folder\ /copy:DATSOU /secfix /e /b /MT:8
/LOG:C:\migration.log /v /tee</code></pre> 
<p>This command uses the following elements and options:</p> 
<ul> 
 <li>Y:\source-folder\ – Refers to the source share.</li> 
 <li>Z:\target-FSx-folder\ – Refers to the target file system on FSx for Windows File Server. Ex: \\amznfsxabcdef1.mydata.com\share</li> 
 <li>/copy – Specifies the following file properties to be copied: 
  <ul> 
   <li>D – data</li> 
   <li>A – attributes</li> 
   <li>T – timestamps</li> 
   <li>S – NTFS ACLs</li> 
   <li>O – owner information</li> 
   <li>U – auditing information.</li> 
  </ul> </li> 
 <li>/secfix – Makes sure that file security is updated for all files, even if the file itself wasn’t copied during the operation.</li> 
 <li>/e – Copies subdirectories, including empty ones.</li> 
 <li>/b – Uses the backup and restore privilege in Windows to copy files even if their NTFS ACLs deny permissions to the current user.</li> 
 <li>/MT:8 – Specifies how many threads to use for performing multithreaded copies.</li> 
 <li>/LOG:C:\migration.log – Writes the status output to “migration” log file in the C disk.</li> 
 <li>/v – Produces verbose output, and shows all skipped files.</li> 
 <li>/tee – Writes the status output to the console window, and to the log file.</li> 
</ul> 
<p><img alt="Command prompt" class="size-full wp-image-24920 aligncenter" height="924" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2025/04/14/Figure-6.png" width="1472" /></p> 
<p style="text-align: center;"><em>Figure 5: Windows command prompt</em></p> 
<p>If you are copying large files over a slow or unreliable connection, then you can enable restartable mode by using the&nbsp;/zb&nbsp;option in place of the /b&nbsp;option. With restartable mode, if the transfer of a large file is interrupted, a subsequent Robocopy operation can pick up in the middle of the transfer instead of having to re-copy the entire file from the beginning. Enabling restartable mode can reduce the data transfer speed.</p> 
<p>After completing the initial copy, you can maintain synchronization between directories using Robocopy with the /MIR switch. This command mirrors the source directory to the destination, ensuring that any updates, additions, or deletions are consistently reflected. This approach is particularly useful for directories requiring continuous updates until the final cutoff time.</p> 
<h2>Cleaning up</h2> 
<p>If you’re running this as a lab, then you may need to delete the resources created using this post. To do so, go to the Amazon FSx console and choose the FSx for Windows File Server file system that was created, then choose <strong>Delete file system</strong> on the <strong>Actions</strong> button drop-down on the top-right.</p> 
<p>To clean up the EC2 instance, go to the Amazon EC2 console, choose <strong>Instances</strong> on the left, choose the instance that was created on the right side, then choose <strong>Terminate (delete) instance</strong> on the <strong>Instance state</strong> button drop-down on the top right.</p> 
<h2>Conclusion</h2> 
<p>In this post, we walk you through the process of migrating data from an existing Windows File Server to Amazon FSx for Windows File Server using Robocopy. Migrating files to Amazon FSx for Windows File Server using Robocopy offers a practical and efficient approach for smaller-scale migrations where AWS DataSync might be too complex. This method uses an intermediary EC2 instance to bridge the source file server and FSx for Windows File Server file system to provide flexibility and control over the migration process, making sure that file attributes, permissions, and other metadata are preserved.</p> 
<h2>Additional resources</h2> 
<p><a href="https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy" rel="noopener" target="_blank">Robocopy</a></p> 
<p><a href="https://docs.aws.amazon.com/fsx/latest/WindowsGuide/migrate-files-to-fsx-datasync.html" rel="noopener" target="_blank">Migrating files to Amazon Fsx with Datasync</a></p> 
<p><a href="https://aws.amazon.com/ec2/" rel="noopener" target="_blank">Amazon EC2</a></p> 
<p><a href="https://aws.amazon.com/fsx/windows/" rel="noopener" target="_blank">Amazon FSx for Windows File Server</a></p> 
<p><a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_getting_started.html" rel="noopener" target="_blank">Getting started with AWS Managed Microsoft AD</a></p>
