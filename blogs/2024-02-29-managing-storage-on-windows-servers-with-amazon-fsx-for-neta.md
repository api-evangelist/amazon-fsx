---
title: "Managing storage on Windows servers with Amazon FSx for NetApp ONTAP"
url: "https://aws.amazon.com/blogs/storage/managing-storage-on-windows-servers-with-amazon-fsx-for-netapp-ontap/"
date: "Thu, 29 Feb 2024 16:35:44 +0000"
author: "Baris Furtinalar"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
<p>Users who choose to migrate workloads to the cloud prefer to do so without modifying application code and without being required to learn new methods for managing data. Ideally they are seeking a cloud service with like-for-like functionality, and management similar to their on-premises infrastructure. The goal is to accelerate migration and deployment in the cloud without having to re-learn operational and management skills.</p> 
<p>Many organizations have successfully deployed <a href="https://aws.amazon.com/fsx/netapp-ontap/" rel="noopener" target="_blank">Amazon FSx for NetApp ONTAP</a> storage for various use cases since its initial release in September 2021, and it enables them to migrate workloads to AWS seamlessly while delivering a familiar and consistent experience for data management. We frequently work with enterprise organizations who use Amazon FSx for NetApp ONTAP as a foundational building block for their clustered workloads, especially when they are deploying a Windows Server Failover Cluster (WSFC). One of the main benefits of using Amazon FSx for NetApp ONTAP in this way is that you can configure it to provide shared block storage via iSCSI. You can then configure this shared block storage as traditional WSFC cluster storage or as Cluster Shared Volumes (CSV).</p> 
<p>In this blog post, we discuss various operations that you can perform on FSx for NetApp ONTAP volumes and LUNs. In working with users, we have observed that there are three commonly encountered scenarios where guidance is most often sought: 1/extending a disk; 2/creating a new disk or volume; and 3/moving a LUN to a different volume. In this blog, we will walk through each of these three scenarios.</p> 
<h2>Prerequisites</h2> 
<p>To follow these walkthroughs, you must have access to at least one Windows server that has been configured to connect to FSx for NetApp ONTAP via iSCSI. If you do not have such a setup, or are looking for an easy way to deploy a fresh test configuration, you can follow this previous blog post to configure a <a href="https://aws.amazon.com/blogs/modernizing-with-aws/sql-server-high-availability-amazon-fsx-for-netapp-ontap/" rel="noopener" target="_blank">SQL Server Always On Failover Cluster Instance using Amazon FSx for NetApp ONTAP</a> as shared storage via iSCSI. This blog described and walked through the steps needed to deploy a single ONTAP volume containing three LUNs – one each for 1/Quorum, 2/Data and 3/Logs.</p> 
<p>Throughout this blog, we use a number of scripts and commands referring to existing objects (FSx for ONTAP storage virtual machine, initiator Group, SQL Server cluster resource) that have been deployed using the <a href="https://aws.amazon.com/blogs/modernizing-with-aws/sql-server-high-availability-amazon-fsx-for-netapp-ontap/" rel="noopener" target="_blank">previously mentioned blog</a>. The parameters that you will need to customize for your own custom environment are indicated&nbsp;<strong><em>in bold italics</em></strong> in the following commands and scripts.</p> 
<h2>Scenario 1 – Extend an existing disk</h2> 
<p>In this scenario, we walk you through how to extend an existing volume, LUN and disk as in<em> Figure 1</em>. This can be useful in situations where you are running out of disk space. The maximum LUN size for FSx for ONTAP systems running ONTAP 9.12.1P2 and later is 128 TB. Previous ONTAP versions support a maximum LUN size of 16 TB.</p> 
<p><img alt="Extend existing LUN/OS disk" class="size-full wp-image-19197 aligncenter" height="525" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/Extend-existing-LUNOS-disk.png" width="768" /></p> 
<p style="text-align: center;"><em>Figure 1: Extending the existing LUN/OS disk when running out of disk space<br /> </em></p> 
<p>First, we will connect to the <a href="https://aws.amazon.com/ec2/" rel="noopener" target="_blank">Amazon EC2</a> Windows server to identify the correct disk(s). Then we will switch over to the FSx for NetApp ONTAP CLI via SSH to execute the required storage commands. Finally, we will go back to the EC2 Windows server to complete this walkthrough.</p> 
<h3><strong>Steps: Extend an existing disk<br /> </strong></h3> 
<p>1. <a href="https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html" rel="noopener" target="_blank">Connect to your EC2 Windows</a> instance through RDP.</p> 
<p>2. Run the following PowerShell script (run as Administrator) to list drive letters and corresponding disk serial numbers for connected NetApp LUNs as in <em>Figure 2</em>:</p> 
<pre><code class="lang-powershell">$disklist=@{}
get-disk | ForEach-Object{$disklist.Add($_.DiskNumber, $_.SerialNumber)}
$partitions=get-disk | Where-Object{$_.FriendlyName -eq 'NETAPP LUN C-MODE'} | foreach{Get-Partition -disknumber $_.Number |?{$_.Type -eq 'Basic' -OR $_.type -eq 'IFS'}} | select DiskNumber, Driveletter
foreach($partition in $partitions){$disklist.GetEnumerator() | ForEach-Object{if($partition.DiskNumber -eq $_.Key){Write-Output "Drive Letter $($partition.Driveletter): Disk Serial Number: $($_.value)"}}}</code></pre> 
<p><img alt="PowerShell command output" class="size-full wp-image-19196 aligncenter" height="106" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/PowerShell-command-output.jpg" width="754" /></p> 
<p style="text-align: center;"><em>Figure 2: PowerShell command output to list drive letters<br /> </em><em> and corresponding disk serial numbers for connected NetApp LUNs<br /> </em></p> 
<p>Note the serial number of the disk you want to extend.</p> 
<p>3. Connect to your <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/managing-resources-ontap-apps.html" rel="noopener" target="_blank">FSx for ONTAP system using SSH</a> (e.g. ssh fsxadmin@x.x.x.x from PowerShell). If you cannot access your file system, then follow <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/unable-to-access.html" rel="noopener" target="_blank">these Amazon FSx instructions</a>.</p> 
<p>4. The size to which you can increase your LUN <a href="https://docs.netapp.com/us-en/ontap/san-admin/resize-lun-task.html#increase-the-size-of-a-lun" rel="noopener" target="_blank">varies</a> depending upon your version of ONTAP. If you need to extend the LUN beyond 16TB size, check if you are running ONTAP 9.12.1P2 and later. From your FSx for ONTAP SSH session, use the following command to display the current version of your FSx for ONTAP system.</p> 
<pre><code class="lang-bash">version</code></pre> 
<p>5. List the existing LUN as in <em>Figure 3</em> with a matching serial number that you noted in step 2</p> 
<pre><code class="lang-bash">lun show -serial lWB0k$TDzEXX -fields lun,volume,size</code></pre> 
<p><img alt="“lun show” command output" class="size-full wp-image-19195 aligncenter" height="116" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/lun-show-command-output.jpg" width="886" /></p> 
<p style="text-align: center;"><em>Figure 3: “lun show” command output to display the current version of your FSx for ONTAP system </em></p> 
<p>6. Make sure to modify the command to use the serial number from your environment. Extend the LUN you have identified in Step 5 by 10GB as well:</p> 
<pre><code class="lang-bash">lun resize -vserver sql-svm01 -path /vol/SQLCluster01/sqldata1 -size +10g</code></pre> 
<p>7. <a href="https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html" rel="noopener" target="_blank">Connect back to your EC2 instance</a> through RDP.</p> 
<p>8. Extend the OS disk on Windows (S: drive in this example) running the following PowerShell script (run as Administrator):</p> 
<pre><code class="lang-powershell">Resize-Partition -Driveletter S -Size (Get-PartitionSupportedSize -Driveletter S).SizeMax</code></pre> 
<p><em>Figure 4</em> shows how it would look on the Window Server in the Disk Manager GUI before extending S: drive:</p> 
<p><img alt="Disk Management before disk extension" class="size-full wp-image-19194 aligncenter" height="651" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/Disk-Management-before-disk-extension.png" width="1008" /></p> 
<p style="text-align: center;"><em>Figure 4: Windows server “Disk Manager” GUI before disk extension</em></p> 
<p>And after extending, the S: drive size was increased by 10 GB as in <em>Figure 5</em>:</p> 
<p><img alt="Disk Management after disk extension" class="size-full wp-image-19192 aligncenter" height="651" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/Disk-Management-after-disk-extension.png" width="1008" /></p> 
<p style="text-align: center;"><em>Figure 5: Windows server “Disk Manager” GUI after disk extension</em></p> 
<h2>Scenario 2: Create a new disk/volume</h2> 
<p>In the following scenario, we walk through how to create a new volume and LUN on the FSx for ONTAP file system as in<em> Figure 6</em>, and then add the newly created LUN as a new disk to your Windows server. Before you proceed with this operation make sure you have enough space in your FSx for ONTAP file system.</p> 
<p>Using a single ONTAP volume containing multiple LUNs is a valid design choice that works well across most of Windows Server Failover Cluster (WSFC) deployment scenarios that leverage shared iSCSI storage. However, it it rare for a single design to be the absolute best choice under all circumstances, and there are a variety of use cases in which users can benefit from deploying multiple ONTAP volumes as part of their design.</p> 
<p>The primary reason to deploy multiple volumes when using FSx for ONTAP is that the majority of NetApp storage operations, such as deduplication, snapshots, cloning, replication, and intelligent tiering, are done on the volume level and cannot be set at a LUN level. For a standard SQL Server FCI deployment scenario, this is unlikely to matter – but in more complex scenarios, the additional flexibility provided by using multiple ONTAP volumes can be crucial.</p> 
<p>For example, a more complex scenario is one where you intend to use a single SQL Server FCI to support multiple tenants or multiple applications. Providing each tenant/application database with its own set of ONTAP volumes and LUNs enables a level of granular control that is not possible with a single ONTAP volume. For example, each tenant/application could have a different storage tiering policy or replication schedule configured on their ONTAP volume(s), depending on individual requirements.</p> 
<p>Further examples of reasons to go beyond a single ONTAP volume include:</p> 
<ul> 
 <li>Isolating databases with I/O-intensive queries into different volumes</li> 
 <li>Placing large databases or databases that have minimal RTO requirements into separate volumes for faster recovery</li> 
 <li>Placing SQL transaction log files (.ldf) on a separate volume to the SQL data files so that independent backup schedules can be created for each</li> 
</ul> 
<p>For many reasons, such as the preceding examples, you must inevitably either: provision your storage in specific ways during deployment; or make changes to an existing storage system and the Windows servers that are connected to it.</p> 
<p><img alt="Creating new ONTAP volume/LUN/OS disk " class="size-full wp-image-19191 aligncenter" height="494" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/Creating-new-ONTAP-volumeLUNOS-disk.jpg" width="717" /></p> 
<p style="text-align: center;"><em>Figure 6: Creating a new ONTAP volume/LUN/OS disk&nbsp;</em></p> 
<p>We connect to the FSx for ONTAP CLI through SSH to execute the necessary storage commands. Then we will switch over to the active SQL server cluster node to complete this walkthrough.</p> 
<h3><strong>Steps: Create a new disk/volume<br /> </strong></h3> 
<p>1. Connect to your <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/managing-resources-ontap-apps.html" rel="noopener" target="_blank">FSx for ONTAP system using SSH</a> (such as ssh fsxadmin@x.x.x.x from PowerShell). If you cannot access your file system, then follow these <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/unable-to-access.html" rel="noopener" target="_blank">Amazon FSx instructions</a>.</p> 
<p>2. From your FSx for ONTAP SSH session, use the following commands to create a new volume (in this example 50GiB in size). We are using the same commands as in our previous post.</p> 
<pre><code class="lang-bash">volume create -vserver sql-svm01 -volume newvolume -aggregate aggr1 -size 50G -state online -tiering-policy snapshot-only -percent-snapshot-space 0 -autosize-mode grow -snapshot-policy none
volume modify -vserver sql-svm01 -volume newvolume -fractional-reserve 0
volume modify -vserver sql-svm01 -volume newvolume -space-mgmt-try-first snap_delete
volume snapshot autodelete modify -vserver sql-svm01 -volume newvolume -delete-order oldest_first -enabled true</code></pre> 
<p>3. Create a new LUN (50GB in size as well) within the newly created volume and map the LUN to the existing iGroup. If you do not know the name of your existing iGroup u<em>s</em>e the<em>igroup show</em> command to retrieve a list of your iGroups.</p> 
<pre><code class="lang-bash">lun create -vserver sql-svm01 -volume newvolume -lun newdisk -size 50G -ostype windows_2008
lun map -vserver sql-svm01 -volume newvolume -lun newdisk -igroup SQLCluster01-IG</code></pre> 
<p>4. <a href="https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html" rel="noopener" target="_blank">Connect to your Windows EC2 instance</a> through RDP.</p> 
<p>5. Go to the Windows disk management tool (by running diskmgmt.msc), initialize and format the new iSCSI disk, or run the following PowerShell script (run as Administrator) to initialize the newly added disk, format it and assign drive letter “N”:</p> 
<pre><code class="lang-powershell">#Retrieve a list of added LUNs from FSxN
$disklist=Get-Disk | Where-Object{$_.FriendlyName -eq 'NETAPP LUN C-MODE' -and ($_.OperationalStatus -eq 'Offline')} | Sort-Object -Property Size 
#Create a list of drive letters
$driveletters=@("N")
#Initiate, create and format volumes from the list of available FSx for ONTAP disks

foreach($dk in $disklist)
{   
    if(($dk).IsOffline -eq $True){
       Set-Disk -Number ($dk).Number -IsOffline $False

    }
    if(($dk).PartitionStyle -eq 'RAW'){
        Initialize-Disk -Number ($dk).Number -PartitionStyle GPT -ErrorAction SilentlyContinue
    }
    if($dk.IsReadOnly -eq $True){ 
        Set-Disk -Number ($dk).Number -IsReadOnly $False
    }
} 

New-Partition -DiskNumber ($disklist[0]).Number -UseMaximumSize -DriveLetter $driveletters[0] | Format-Volume -FileSystem NTFS -Force -NewFileSystemLabel NewDisk</code></pre> 
<p>If you are using a Windows Server Failover Cluster, then you must perform these 3 additional steps:</p> 
<p style="padding-left: 40px;">1. Go to Failover Cluster Manager and add the disk as cluster disk or run the following PowerShell script (run as Administrator):</p> 
<pre style="padding-left: 40px;"><code class="lang-powershell">Get-ClusterAvailableDisk | Add-ClusterDisk; start-sleep 05; Get-ClusterResource | ?{$.ResourceType -eq "Physical Disk" -and $.Name -like "Cluster*"}| %{$_.Name = "newdisk"}</code></pre> 
<p style="padding-left: 40px;">2. In the Failover Cluster Manager, assign the cluster disk to the SQL Server role or run the following PowerShell script (run as Administrator):</p> 
<pre style="padding-left: 40px;"><code class="lang-powershell">Move-ClusterResource -Name "newdisk" -Group "SQL Server (MSSQLSERVER)"</code></pre> 
<p style="padding-left: 40px;">3. As a last step you add additional dependency for newly added cluster disk within the SQL Server resource. This makes sure that the disk is brought online before the SQL server resource. This can be done in the Failover Cluster Manager or throught the following PowerShell script (run as Administrator):</p> 
<pre><code class="lang-powershell">Add-ClusterResourceDependency -Resource "SQL Server" -Provider "newdisk"</code></pre> 
<p>If you followed these three additional steps, it is good practice to test the cluster failover to ensure everything has been properly configured and you can failover to another node.</p> 
<h2>Scenario 3 – Moving an existing LUN to a different NetAPP ONTAP volume</h2> 
<p>Scenario 3 provides a walkthrough of how to move an existing LUN to a new ONTAP volume as in <em>Figure 7</em>. This is an operation that you would perform in order to move existing LUNs on to their own ONTAP volumes to use the various ONTAP volume-level settings for specific LUNs, as detailed in the previous scenario.</p> 
<p><img alt="Moving LUN to a different ONTAP volume" class="size-full wp-image-19190 aligncenter" height="511" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/Moving-LUN-to-a-different-ONTAP-volume.jpg" width="731" /></p> 
<p style="text-align: center;"><em>Figure 7: Operation for moving a LUN to a different ONTAP volume</em></p> 
<p><a href="https://docs.netapp.com/us-en/ontap-cli-9131/lun-move-start.html#description" rel="noopener" target="_blank">LUNs moved between ONTAP volumes</a> within a storage virtual machine (SVM) are moved immediately and without loss of connectivity. The destination volume must be larger than the size of the LUN that you want to move. We will connect to the FSx ONTAP CLI through SSH to execute the needed storage steps. There are no additional steps needed on the Windows server.</p> 
<h3><strong>Steps: Moving an existing LUN to a different NetAPP ONTAP volume<br /> </strong></h3> 
<p>1. Connect to your <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/managing-resources-ontap-apps.html" rel="noopener" target="_blank">FSx for ONTAP system using SSH</a> (such as SSH fsxadmin@x.x.x.x from PowerShell). If you cannot access your file system, then follow the <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/unable-to-access.html" rel="noopener" target="_blank">these Amazon FSx instructions</a>.</p> 
<p>2. If you haven’t created a new volume yet and you do not have the existing volume that serves as the destination volume for the LUN, then refer to Step 2 in the previous scenario for how to create a new ONTAP volume.</p> 
<p>3. Identify the source LUN and its size by using the following command as in<em> Figure 8</em>:</p> 
<pre><code class="lang-bash">lun show
</code></pre> 
<p><img alt="Figure 8 – “lun show” command output" class="size-full wp-image-20781 aligncenter" height="179" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2024/02/28/Figure-8-1.jpg" width="860" /></p> 
<p style="text-align: center;"><em>Figure 8: “lun show” command output</em></p> 
<p>4. Move the desired source LUN to another volume:</p> 
<pre><code class="lang-bash">lun move start -vserver sql-svm01 -source-path /vol/SQLCluster01/sqllog -destination-path /vol/newvolume/sqllog</code></pre> 
<p>5. Track the progress of the LUN move as in <em>Figure 9</em>:</p> 
<pre><code class="lang-bash">lun move show</code></pre> 
<p><img alt="&quot;lun move show“command output after disk extension" class="size-full wp-image-19188 aligncenter" height="98" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/03/lun-move-showcommand-output-after-disk-extension.jpg" width="744" /></p> 
<p style="text-align: center;"><em>Figure 9: “lun move show“command output</em> <em>after disk extension</em></p> 
<h2>Cleaning up</h2> 
<p>If you provisioned any resources while following this walkthrough, it is a best practice to delete the resources that you are no longer using so that you do not incur unintended charges. After you have finished this exercise, you should follow these steps to <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/getting-started-step3.html" rel="noopener" target="_blank">clean up your Amazon FSx for NetApp ONTAP resources and protect your AWS account</a>. You can clean up additional resources you created for this tutorial by following the steps in these links.</p> 
<ul> 
 <li><a href="https://aws.amazon.com/premiumsupport/knowledge-center/delete-terminate-ec2/" rel="noopener" target="_blank">Delete or terminate Amazon EC2 Instances</a></li> 
 <li><a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-deleting-volume.html" rel="noopener" target="_blank">Delete an Amazon EBS Volume</a></li> 
</ul> 
<h2>Conclusion</h2> 
<p>In this post, we walked through how to perform three common storage management tasks for an Amazon EC2 Windows Server connected to an Amazon FSx for NetApp ONTAP file system through iSCSI storage; 1/extending a disk; 2/creating a new disk or volume; and 3/moving a LUN to a different volume. These three scenarios are the most common tasks where guidance is most often sought by users who have newly migrated to Amazon FSx for NetApp ONTAP. To learn more about Amazon FSx for NetApp ONTAP please refer to our “<a href="https://aws.amazon.com/fsx/netapp-ontap/resources/?blog-posts-cards.sort-by=item.additionalFields.createdDate&amp;blog-posts-cards.sort-order=desc&amp;fsx-new.sort-by=item.additionalFields.postDateTime&amp;fsx-new.sort-order=desc" rel="noopener" target="_blank">Resources</a>” page.</p>
