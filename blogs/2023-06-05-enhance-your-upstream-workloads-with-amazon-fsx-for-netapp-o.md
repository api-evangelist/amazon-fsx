---
title: "Enhance your upstream workloads with Amazon FSx for NetApp ONTAP"
url: "https://aws.amazon.com/blogs/storage/enhance-your-upstream-workloads-with-amazon-fsx-for-netapp-ontap/"
date: "Mon, 05 Jun 2023 18:02:33 +0000"
author: "Tom McDonald"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
<p>Geological and Geophysical (G&amp;G) workloads in Upstream Energy have different workflows associated with them, including Reservoir Simulation, Subsurface Interpretation, and Drilling and Completions. Due to the diverse performance and client requirements of these workflows, organizations often face a heavy operational burden of copying their data to multiple solutions for different protocols. Until recently, they faced the challenge of developing their own solutions to achieve multi-protocol access, which provides the ability to access the same file at the same time, from either Windows or Linux. This process was not only time-consuming but also added complexity and maintenance overhead.</p> 
<p>Now, organizations can take advantage of <a href="https://aws.amazon.com/fsx/netapp-ontap/" rel="noopener" target="_blank">Amazon FSx for NetApp ONTAP</a>. This fully managed service provides a seamless solution for multi-protocol access, allowing the organization to focus on their core business activities instead of managing complex infrastructure.</p> 
<p>In this post, I examine FSx for ONTAP as a fully managed service and look at additional capabilities that are available. These include advanced performance techniques and active monitoring methods. By exploring these aspects, readers will gain deeper insight into leveraging the full potential of FSx for ONTAP.</p> 
<h2>Solution overview</h2> 
<p>As seen in <em>Figure 1</em>, an <a href="https://aws.amazon.com/ec2/" rel="noopener" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> Network-Attached Storage (NAS) instance is a common deployment as organizations begin their journey to AWS. Two pillars of the <a href="https://aws.amazon.com/architecture/well-architected/?wa-lens-whitepapers.sort-by=item.additionalFields.sortDate&amp;wa-lens-whitepapers.sort-order=desc&amp;wa-guidance-whitepapers.sort-by=item.additionalFields.sortDate&amp;wa-guidance-whitepapers.sort-order=desc" rel="noopener" target="_blank">AWS Well Architected Framework</a> are Operational Excellence and Reliability. When running EC2 NAS instances, an organization must patch instances, architect its instances for resiliency when failover is required, and self-manage their services. All of this introduces operational overhead.</p> 
<p><em> <img alt="Figure_1_EC2_NAS_Instance_Design" class="aligncenter size-full wp-image-16986" height="543" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_1_EC2_NAS_Instance_Design.png" width="531" /></em></p> 
<p style="text-align: center;"><em>Figure 1: EC2 NAS instance design</em></p> 
<p>FSx for ONTAP, as seen in <em>Figure 2</em>, provides a fully managed file system offering multiprotocol access. You do not need to provision individual instances, architect for failover, or manage upgrades. NetApp ONTAP has had presence in the Upstream Energy industry for decades. Data is automatically mirrored between the active and standby files servers in the FSx for ONTAP file system. This solution provides high availability to data access with two file servers in an active-standby configuration in a Single Availability Zone configuration.</p> 
<p><img alt="Figure_2_Single_Availability_Zone_Design" class="aligncenter size-full wp-image-16988" height="625" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_2_Single_Availability_Zone_Design.png" width="824" /></p> 
<p style="text-align: center;"><em>Figure 2: Single Availability Zone design</em></p> 
<p>Along with the ability deploy a Single-AZ solution, Multi-Availability Zone (AZ) file systems, as seen in <em>Figure 3</em>, are available and provide cross-AZ resiliency. Classic ONTAP technologies, such as SnapMirror and SnapVault, are available for additional Disaster Recovery and Business Continuity requirements. In addition to <a href="https://aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">Amazon CloudWatch</a> and <a href="https://aws.amazon.com/cloudtrail/" rel="noopener" target="_blank">Amazon CloudTrail</a> for monitoring, NetApp tools in the NetApp ecosystem, such as <a href="https://bluexp.netapp.com/" rel="noopener" target="_blank">BlueXP</a> and <a href="https://nabox.org/" rel="noopener" target="_blank">NAbox</a>, are available to be leveraged for the operational excellence and telemetry data of the FSx for ONTAP file system.</p> 
<p>The FSx for ONTAP service is available with <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/limits.html#limits-ontap-resources-file-system" rel="noopener" target="_blank">various throughput and IOPS configuration</a>. FSx for ONTAP can be used for an individual project – allowing organizations to only pay for what is needed – or as a persistent file system.</p> 
<p><img alt="Figure 3: Multi-Availability Zone Design" class="aligncenter wp-image-16989 size-full" height="625" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_3_Multi-Availability_Zone_Design.png" width="824" /></p> 
<p style="text-align: center;"><em>Figure 3: Multi-Availability Zone Design</em></p> 
<h2>Optimizing FSx for ONTAP for Upstream Workloads</h2> 
<p>Upstream workloads’ I/O pattern often consists of low concurrency and sequential reads. For workloads with these behaviors, FSx for ONTAP solutions include NVMe read cache to accelerate reads. In addition to NVMe read cache, we can create FlexGroups.</p> 
<h3><strong>Creation of a FlexGroup</strong></h3> 
<p>A FlexGroup is a volume created from other constituent volumes.&nbsp;There is automatic load distribution and scalability occurring on the ONTAP side that adds benefits to some workloads.&nbsp;One advantage of FlexGroups is that they can grow to be much larger, and contain many more files, than a regular FlexVol. A standard FlexVol has a 100TB limit, where FlexGroups have virtually no limit (NetApp recommends a maximum of 20 PB):</p> 
<pre>FSxId0::&gt; vol create -vserver svm0 -volume flexg -aggr-list aggr1 -size 400T
Notice: The FlexGroup volume "flexg" will be created with the following number of constituents of size 100TB: 4.
Do you want to continue? {y|n}: y</pre> 
<p>In the preceding example, ONTAP creates four 100TB constituent volumes of the FlexGroup. Four constituent volumes are the ONTAP defaults, which can be tuned based on workload requirements.</p> 
<p>The <a href="https://fio.readthedocs.io/en/latest/fio_doc.html" rel="noopener" target="_blank">Flexible IO Tester</a> (FIO) is a common tool to evaluate and profile storage solutions. Clients exist for Linux and Windows operating systems. To explore the benefits of FlexGroups, I created a 512MB throughput file system. Eight FIO runs per row were averaged in the following chart. The FG/VOL and NFS Version columns show various volume types and protocol versions respectively.</p> 
<p>In G&amp;G workloads, applications have varied block sizes and numerous clients. Furthermore, most ISV applications still require older operating systems which limit new capabilities, such as nconnect for NFS. There are nuances to every workload, and variations should be profiled based on the application and supporting operating system. Presented in <em>Figure 4</em>, I ran a common workload size of 64KB blocks, with both random and sequential I/O to show these differences from a single client. The read percentage is 70%, while the concurrent write percentage is 30%, for a mixed workload. The following table represents the impact of FlexGroups, where nconnect can assist, and where protocol versions introduce varied overhead.</p> 
<p><img alt="Figure 4: FlexGroup table" class="aligncenter wp-image-17041 size-full" height="526" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/02/Figure_4_FlexGroup_Table.png" width="1676" /></p> 
<p style="text-align: center;"><em>Figure 4: FlexGroup table</em></p> 
<p>One current limitation of FlexGroups today is they are not supported by FSx Backups or <a href="https://aws.amazon.com/backup/" rel="noopener" target="_blank">AWS Backup.</a> Other techniques, such as <a href="https://aws.amazon.com/fsx/netapp-ontap/features/?refid=cr_card#Availability_and_Data_Protection" rel="noopener" target="_blank">SnapMirror</a>, offer a solution to backup and disaster recovery.</p> 
<h3><strong>SMB-specific optimizations for performance in interpretation applications</strong></h3> 
<p>Many subsurface applications on Windows read and write a large amount of data for interpretation, analysis, and visualization. To explore some tunings, I start with a read Robocopy of SEGY data from an FSx for ONTAP with 2GB/sec of throughput and default SSD IOPS to a g4ad.4xlarge instance. The g4ad.4xlarge provides up to 10Gb networking with burst benefits, which are seen in the following charts. Afterward, I continued with writes of SEGY and Robocopy to understand the implications of directional I/O.</p> 
<p>Two options I chose to explore are the SMB Maximum Transition Unit (MTU) setting on the FSx for ONTAP side, which defaults to Large MTU enabled, and the client-side bandwidth throttling, which is also enabled from a client perspective by default. As seen in <em>Figure 5</em>, client-side bandwidth throttling, enabled or disabled, added no measurable performance increases to these profiles.</p> 
<p>Although this is the simple movement of a specific data types, there was enough performance differences to dive deeper on SMB MTU size impacts.</p> 
<p><img alt="Figure 5: SMB Throttle performance table" class="aligncenter size-full wp-image-16991" height="285" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_5_SMB_Throttle_performance_table.png" width="977" /></p> 
<p style="text-align: center;"><em>Figure 5: SMB Throttle performance table</em></p> 
<p>Large MTU provides SMB blocks to be transferred in up to 1MB in size, and disabling this feature results in a smaller block size of 64KB. This is an application specific requirement, and should be evaluated based on organizational needs. Interpretation applications often have mixes of small and large file sizes yet have low concurrency for data. Large SMB MTU can impact performance regardless of file size. To explore these block sizes, I chose to use an FIO profile of 70% Read and 30% Write, with 64KB as the block size and an I/O depth of 16, along with a corresponding profile with large MTU block sizes of 1MB. This was done to have more than a single process running against the file system. As a load is added to the file system, the results were also in favor of disabling the large MTU for sequential workloads, regardless of block size. For random I/O, the results are mixed. The results are displayed in <em>Figure 6</em>.</p> 
<p>For these applications which stream large files into memory for visualization of results, it is worth validating the performance results when disabling SMB large MTU within your application and environment for additional performance gains.</p> 
<p><img alt="Figure 6: SMB Large MTU comparison performance table" class="aligncenter size-full wp-image-16992" height="181" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_6_SMB_Large_MTU_comparison_performance_table.png" width="977" /></p> 
<p style="text-align: center;"><em>Figure 6: SMB Large MTU comparison performance table</em></p> 
<h3><strong><br /> Commands to disable SMB large MTU</strong></h3> 
<p>To disable large MTUs, follow these steps:</p> 
<p style="padding-left: 23px;"><strong>1. Go into Advanced mode:<br /> </strong></p> 
<pre>FSxId::&gt; set -privilege advanced

Warning: These advanced commands are potentially dangerous; use them only when directed to do so by NetApp personnel.
Do you want to continue? {y|n}: y</pre> 
<p style="padding-left: 23px;"><strong>2. Verify the defaults are set to true:</strong></p> 
<pre>FSxId::*&gt; cifs options show -vserver svm0 -fields is-large-mtu-enabled
vserver is-large-mtu-enabled
------- --------------------
svm0 true</pre> 
<p style="padding-left: 23px;"><strong>3. Set large MTU size to false:</strong></p> 
<pre>FSxId::*&gt; cifs options modify -vserver svm0 -is-large-mtu-enabled false</pre> 
<p style="padding-left: 23px;"><strong>4. Verify that large MTU is set to false:</strong></p> 
<pre>FSxId::*&gt; cifs options show -vserver svm0 -fields is-large-mtu-enabled
vserver is-large-mtu-enabled
------- --------------------
svm0 false</pre> 
<p>For the changes to take effect, disconnect and reconnect your SMB shares on the Windows hosts.</p> 
<h3><strong>Active monitoring</strong></h3> 
<p>Active monitoring is running a workload and monitoring the real-time statistics of the workload. This is often used to understand immediately what is occurring on a file system. Although quality of service is a technique to throttle storage objects, in FSx for ONTAP this also includes performance monitoring of volumes with the <a href="https://docs.netapp.com/us-en/ontap-cli-96/qos-statistics-workload-performance-show.html#description" rel="noopener" target="_blank">QoS&nbsp;statistics commands</a> The following QoS commands are only available with the&nbsp;fsxadmin&nbsp;file system administrator account, not the vsadmin SVM account.&nbsp;This is at the file system level.</p> 
<pre>FSxId::&gt; qos statistics workload latency show</pre> 
<p>The statistics command is great for active benchmarking and latency – you’ll see Network, Data, Disk, etc. (as seen in an example in <em>Figure 7). </em>This can help pinpoint a potential problem with latency. Moreover, it is possible to send that data into a timeseries database for graphing purposes.</p> 
<p><img alt="Figure 7: QOS statistics workload latency output example" class="aligncenter size-full wp-image-16993" height="127" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_7_QOS_statistics_workload_latency_output_example.png" width="977" /></p> 
<p style="text-align: center;"><em>Figure 7: QOS statistics workload latency output example</em></p> 
<p><strong>Enable refresh display with:</strong></p> 
<pre>FSxId::&gt; qos statistics workload latency show -refresh-display true</pre> 
<p><strong>For throughput and aggregate latency, the command below would produce results as seen in <em>Figure 8</em>:</strong></p> 
<pre>FSxId::&gt; qos statistics workload performance show</pre> 
<p><img alt="Figure 8: QOS statistics workload performance output example" class="aligncenter size-full wp-image-16994" height="304" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/06/01/Figure_8_QOS_statistics_workload_performance_output_example.png" width="791" /></p> 
<p style="text-align: center;"><em>Figure 8: QOS statistics workload performance output example</em></p> 
<p>IOPS, Throughput, and Latency are all important for assessing overall performance. IOPS are Input/Output operations per second, impacted concurrently by applications. Throughput is the measure of bytes transmitted per second, and is a function of IOPS times block size, where block size is determined by an individual application. Latency is the amount of time it takes to fulfill a request, is influenced by the caching and infrastructure associated with the service, and has a direct impact on the number of IOPS driven. All of these are important to dive deep into when exploring performance bottlenecks.</p> 
<p>With SVM-level credentials, the amount of commands at your disposal are more limited (as SVMs are isolated virtual file servers, this is done to limit the level of access each SVM has to the overall file system). Available commands include the statistics command. The performance statistics you can measure from an SVM administrator account are documented as follows. For this command, you specify the <code>-iterations</code> (the number of times you want the statistics to be reported), and the <code>-interval</code> (the time between each report being displayed).</p> 
<pre>svm0::&gt; statistics volume show -iterations 10 -interval 5

svm0 : 10/25/2022 14:52:35

               *Total Read Write Other&nbsp;&nbsp;&nbsp;&nbsp; Read Write Latency

Volume Vserver&nbsp;&nbsp;&nbsp; Ops&nbsp; Ops&nbsp;&nbsp; Ops&nbsp;&nbsp; Ops&nbsp;&nbsp;&nbsp; (Bps) (Bps)&nbsp;&nbsp;&nbsp; (us)

------ ------- ------ ---- ----- ----- -------- ----- -------

   vol5&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1575 1575&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 95835136&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 524
   vol7&nbsp; &nbsp;&nbsp;svm0&nbsp;&nbsp; 1545 1545&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 93981696&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 518
   vol8&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1516 1516&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 92125184&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 543
   vol1&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1506 1506&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 91600896&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 548
   vol6&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1487 1487&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 90232832&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 485
   vol4&nbsp;&nbsp;&nbsp; svm0&nbsp; &nbsp;1435 1435&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 87359488&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 546
   vol2&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1412 1412&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 85709824&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 512
   vol3&nbsp;&nbsp;&nbsp; svm0&nbsp;&nbsp; 1353 1353&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 0 82245632&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp; 514

svm0_root

           svm0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;  0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;    0&nbsp;&nbsp;&nbsp;&nbsp; 0&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 0</pre> 
<p><strong>&nbsp;</strong></p> 
<h3><strong>NetApp Harvest</strong></h3> 
<p>Although CloudWatch provides high-level metrics, the deeper performance metrics we showed in the ONTAP Command Line Interface are not exposed in CloudWatch. However, tools like NetApp Harvest and Grafana, and NetApp Cloud Insights, offer many more metrics. See the <a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/monitoring_overview.html" rel="noopener" target="_blank">Monitoring file systems</a> section of the FSx for ONTAP user guide for more information.</p> 
<p><a href="https://docs.aws.amazon.com/fsx/latest/ONTAPGuide/monitoring-harvest-grafana.html" rel="noopener" target="_blank">Monitoring FSx for ONTAP file systems with Harvest and Grafana</a></p> 
<p><a href="https://github.com/NetApp-Automation/harvest_install" rel="noopener" target="_blank">NetApp Harvest Github</a></p> 
<h2><strong>Cleaning up</strong></h2> 
<p>To avoid incurring unwanted AWS costs after performing these steps, delete any AWS resources created like Amazon EC2 instances and FSx for ONTAP resources.</p> 
<h2><strong>Conclusion</strong></h2> 
<p>In this post, I showed the durability and resiliency benefits of <a href="https://aws.amazon.com/fsx/netapp-ontap/">Amazon FSx for NetApp ONTAP</a> over existing <a href="https://aws.amazon.com/ec2/">Amazon EC2</a> NAS implementations while benefiting from the advanced features of FSx for ONTAP. Along with the multi-protocol capabilities of FSx for ONTAP, the feature rich tool set and simplify of a managed service, FSx for ONTAP enables existing applications and workloads to run seamlessly in AWS.</p> 
<p>Upstream, Mid-stream, and Downstream workloads with multi-protocol requirements (leveraging the benefits of a fully managed service) can access these capabilities today in AWS with FSx for ONTAP. Included are techniques using FlexGroups and SMB tunings for specific SMB workloads. Monitoring techniques both at the Command Line Interface and using alternative methods from the NetApp ecosystem provide robust telemetry data. FSx for ONTAP can be highly tuned to deliver the best performance for your workloads.</p> 
<p><a href="https://aws.amazon.com/fsx/netapp-ontap/" rel="noopener" target="_blank">Amazon FSx for NetApp ONTAP</a> is available in most AWS Regions. If you have any comments or questions, don’t hesitate to leave them in the comments section.</p>
