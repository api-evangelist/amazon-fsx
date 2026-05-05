---
title: "Use Amazon FSx for Lustre to share Amazon S3 data across accounts"
url: "https://aws.amazon.com/blogs/storage/using-amazon-fsx-for-lustre-to-share-amazon-s3-data-across-accounts/"
date: "Fri, 24 Nov 2023 20:45:08 +0000"
author: "Justin Leto"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
<p><em><strong>Update 4/9/2025</strong>: The cross-account bucket policy in the blog has been updated. It was missing a required principal: “arn:aws:iam::accountID:role/AWS-Signed-In-Console-Role.” This omission causes an access denied error.<br /> </em></p> 
<hr /> 
<p>As enterprises evolve their cloud governance practices, multiple teams working in separate accounts may need to share data. One team may oversee an enterprise data lake in one account, while a data science team develops a high-performance computing (HPC) use case in another account. Customers want to take advantage of low-cost object storage and be able to quickly consume this data from a high-performance file system to support HPC use cases without creating additional copies of the data.</p> 
<p><a href="https://aws.amazon.com/fsx/lustre/" rel="noopener" target="_blank">Amazon FSx for Lustre</a> has become a critical building block for customers accelerating machine learning (ML) and HPC use cases on AWS. Amazon FSx for Lustre offers a fully POSIX-compliant, high performance file system that delivers sub-millisecond latencies, up to hundreds of gigabytes per second throughput, and millions of IOPS. It seamlessly integrates with <a href="https://aws.amazon.com/s3/" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3)</a>, offering cloud practitioners seamless access to their S3 datasets and cost-efficiency for colder data sets.</p> 
<p>In this blog post, we guide you through the process of seamlessly integrating an Amazon FSx for Lustre file system with an Amazon S3 data lake, where the Amazon FSx file system and Amazon S3 bucket reside in different AWS accounts in the same AWS Region. This solution will help you scale your AWS environment by allowing data to be shared from a centralized enterprise data lake to specialized team accounts consuming that data for ML and HPC use cases.</p> 
<h2>Solution overview</h2> 
<p>The following solution architecture consists of addressing two primary permissions issues. The first is authorizing Amazon FSx for Lustre to read from an Amazon S3 bucket in another account for the initial load. The second is authorizing the file system to receive bucket put notifications to replicate ongoing changes to keep the data synched.</p> 
<p><img alt="Solution architecture" class="size-full wp-image-19252 aligncenter" height="374" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-1-2.png" width="752" /></p> 
<h2>Prerequisites</h2> 
<p>To deploy the solution described in this blog, you will need the following:</p> 
<ul> 
 <li>Two (2) AWS accounts. You can <a href="https://portal.aws.amazon.com/gp/aws/developer/registration/index.html" rel="noopener" target="_blank">create an AWS account</a> if you do not already have two accounts available.</li> 
</ul> 
<p>The following sections describe how to integrate an Amazon FSx for Lustre file system in <em>ACCOUNT-A</em> with a Data Repository Association (DRA) Amazon S3 bucket in <em>ACCOUNT-B</em>.</p> 
<h2>Implement the solution</h2> 
<p>The following sections walk through integrating an Amazon FSx for Lustre file system in <em>ACCOUNT-A</em> with a Data Repository Association (DRA) Amazon S3 bucket in <em>ACCOUNT-B</em>.</p> 
<p><strong>Step 1:</strong> Create Amazon FSx file system<br /> <strong>Step 2:</strong> Create source bucket<br /> <strong>Step 3:</strong> Create data repository association<br /> <strong>Step 4:</strong> Configure bucket policy</p> 
<h3>Step 1: Create Amazon FSx file system</h3> 
<p>In <em>ACCOUNT-A</em>, confirm you are in the US East (N. Virginia) Region and navigate to the Amazon FSx console.</p> 
<p><img alt="Confirm you are in US East (N.Virgina)" class="wp-image-19250 aligncenter" height="262" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figures-2.png" width="1069" /></p> 
<p style="padding-left: 23px;">1. Click <strong>Create file system</strong>. On the next screen, you will be presented with different types of Amazon FSx file systems. Select <strong>Amazon Fsx for Lustre</strong> and then click <strong>Next</strong>.</p> 
<p><img alt="Select File System Type" class=" wp-image-19239 aligncenter" height="782" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-3.png" width="805" /></p> 
<p style="padding-left: 23px;">2. Enter the <strong>File system name</strong>, <strong>Storage capacity</strong> and set <strong>Data compression type</strong> to <strong>LZ4</strong> to enable compression as shown in the following image.</p> 
<p><img alt="Enter file system name" class=" wp-image-19240 aligncenter" height="881" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-4.png" width="849" /></p> 
<p style="padding-left: 23px;">3. In the <strong>Network &amp; security</strong> section choose the <strong>Virtual Private Cloud (VPC)</strong>, <strong>VPC Security Groups</strong>, and a <strong>Subnet</strong> for our new file system.</p> 
<p><img alt="Network and security" class=" wp-image-19241 aligncenter" height="477" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-5.png" width="909" /></p> 
<p>The selected security group must allow inbound access for Amazon FSx for Lustre traffic (TCP ports 988, 1018-1023) to enable Amazon EC2 instances in the same VPC to mount the Amazon FSx file system. For more information, see the documentation on <a href="https://docs.aws.amazon.com/fsx/latest/LustreGuide/limit-access-security-groups.html" rel="noopener" target="_blank">file system access control with Amazon VPC</a> in the <a href="https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html" rel="noopener" target="_blank">FSx for Lustre User Guide</a>.</p> 
<p>Amazon FSx does not support backups on file systems linked to an Amazon S3 bucket so we need to disable backups for our new file system.</p> 
<p style="padding-left: 23px;">4. Under the <strong>Backup and maintenance</strong> section, choose <strong>Disabled</strong> and then click <strong>Next</strong>.</p> 
<p><img alt="Backup and maintenance" class="size-full wp-image-19242 aligncenter" height="204" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-6.png" width="934" /></p> 
<p style="padding-left: 23px;">5. Review options for accuracy and click <strong>Create file system</strong>. It will take a few minutes to initialize. When the file system is ready the status will show <strong>Available</strong>.</p> 
<h3>Step 2: Create source bucket</h3> 
<p>Create an Amazon S3 bucket in <em>ACCOUNT-B</em>. You can find the detailed instructions for <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html" rel="noopener" target="_blank">creating a bucket</a> in the <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html" rel="noopener" target="_blank">Amazon S3 User Guide</a>. In our example, we choose the US East (N. Virginia) Region and name the bucket “new-lustre-file-system.” After we create the data repository association in the next section, we will return to update the bucket policy.</p> 
<p>Create an Amazon S3 bucket in <em>ACCOUNT-B</em>. The detailed instructions for <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html">Creating a bucket</a> can be found in the <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html">Amazon Simple Storage Service User Guide</a>.</p> 
<p>1. In our example, we choose the US East (N. Virginia) Region and name the bucket “new-lustre-file-system”. After we create the data repository association in the next section, we will return to further lock down the bucket policy.</p> 
<p>2. In <em>ACCOUNT-B</em>, navigate to the Amazon S3 console and choose the bucket you created. Click on the <strong>Permissions</strong> tab and in in the <strong>Bucket policy</strong> section choose <strong>Edit</strong>. Replace the current policy with the policy below.</p> 
<pre><code class="lang-bash">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:Get*",
                "s3:List*",
                "s3:PutBucketNotification"
            ],
            "Resource": [
                "arn:aws:s3:::new-lustre-file-system",
                "arn:aws:s3:::new-lustre-file-system/*"
            ],
            "Condition": {
                "StringLike": {
                    "aws:PrincipalArn": "arn:aws:iam::accountAID:role/aws-service-role/s3.data-source.lustre.fsx.amazonaws.com/AWSServiceRoleForFSxS3Access_fs-*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": [
                "s3:PutObject",
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": [
                "arn:aws:s3:::new-lustre-file-system",
                "arn:aws:s3:::new-lustre-file-system/*"
            ],
            "Condition": {
                "StringLike": {
                    "aws:PrincipalArn": "arn:aws:iam::accountAID:role/Your-AWS-Signed-In-Console-Role"
                }
            }
        }
    ]
}
</code></pre> 
<p>3. Replace the value for the AWS principal with the ARN of <em>ACCOUNT-A</em> ID and the role used to sign into <em>ACCOUNT-A</em></p> 
<p>4. Replace “new-lustre-file-system” with your own bucket name. Click on <strong>Save changes</strong>.</p> 
<p>If you want Amazon FSx to encrypt data when writing to your S3 bucket, you need to set the default encryption on your S3 bucket to either SSE-S3 or SSE-KMS. For more information, refer to <a href="https://docs.aws.amazon.com/fsx/latest/LustreGuide/create-dra-linked-data-repo.html#s3-server-side-encryption-support" rel="noopener" target="_blank">Working with server-side encrypted Amazon S3 buckets</a> in the Amazon <a href="https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html" rel="noopener" target="_blank">FSx for Lustre User Guide</a>.</p> 
<h3>Step 3: Create data repository association</h3> 
<p>Now we will create a data repository association (DRA) to link the Amazon FSx for Lustre file system to our Amazon S3 bucket.</p> 
<p style="padding-left: 23px;">1. In <em>ACCOUNT-A</em>, navigate to the Amazon FSx console and select the file system we created. Select the <strong>Data repository</strong> tab and then choose <strong>Create data repository association</strong>.</p> 
<p><img alt="Data Repository Association" class="wp-image-19243 aligncenter" height="199" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-7.png" width="1047" /></p> 
<p style="padding-left: 23px;">2. Enter the <strong>File system path</strong> and the path to the Amazon S3 bucket. Note that for our example we used the entire bucket, but we could instead restrict the DRA to a specific prefix.</p> 
<p><img alt="Data repository association information" class="wp-image-19244 aligncenter" height="402" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-8.png" width="965" /></p> 
<p style="padding-left: 23px;">3. Click <strong>Create</strong>. It will take a few minutes to initialize before the status shows <strong>Available</strong>.</p> 
<p><img alt="File system &quot;available&quot;" class="wp-image-19245 aligncenter" height="149" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-9.png" width="1099" /></p> 
<p style="padding-left: 23px;">4. When the DRA was created, an Amazon FSx service-linked role for Amazon S3 access was created. Navigate to the <a href="https://aws.amazon.com/iam/" rel="noopener" target="_blank">AWS Identity and Access Management (AWS IAM)</a> console and search for the service role created for our new file system.</p> 
<p><img alt="Identity, Access, Management" class="wp-image-19246 aligncenter" height="186" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-10.png" width="1047" /></p> 
<p style="padding-left: 23px;">5. Find the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener" target="_blank">Amazon Resource Name (ARN)</a> for the Amazon FSx for Lustre service-linked role and save this somewhere. We’ll need it for the bucket policy in the next section.</p> 
<p><img alt="Amazon Resource Name" class="wp-image-19247 aligncenter" height="197" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-11.png" width="1095" /></p> 
<h3></h3> 
<h3>Step 4: Lock down bucket policy</h3> 
<p>With the ARN from the previous section we’ll apply a bucket policy to our Amazon S3 bucket.</p> 
<pre><code class="lang-bash">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Example permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::accountAID:role/Your-AWS-Signed-In-Console-Role",
                    "arn:aws:iam::accountAID:role/aws-service-role/s3.data-source.lustre.fsx.amazonaws.com/AWSServiceRoleForFSxS3Access_fs-XXXXXXX"
                ]
            },
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:Get*",
                "s3:List*",
                "s3:PutBucketNotification"
            ],
            "Resource": [
                "arn:aws:s3:::new-lustre-file-system",
                "arn:aws:s3:::new-lustre-file-system/*"
            ]
        }
    ]
}
</code></pre> 
<p>In <em>ACCOUNT-B</em>, navigate to the Amazon S3 console and choose the bucket you created. Click on the <strong>Permissions</strong> tab and in in the <strong>Bucket policy</strong> section choose <strong>Edit</strong>. Replace the current policy with the policy below.</p> 
<p>Replace the value for the AWS principal with the ARN from <em>ACCOUNT-A</em> and the service-linked role (found in IAM) which was created by the data repository association the previous section.</p> 
<h2></h2> 
<h2>Testing the solution</h2> 
<p>Now we have an Amazon FSx for Lustre file system that is syncing with an Amazon S3 bucket in a different AWS account.</p> 
<h3>Step 1. Create the Amazon EC2 instance</h3> 
<p>To test the syncing, we need an Amazon EC2 instance so we can mount the file system.</p> 
<p>In <em>ACCOUNT-A</em>, navigate to the Amazon EC2 console. Launch an instance using an Amazon Linux 2 AMI in the same VPC as your Amazon FSx file system. For instructions on how to launch an instance, refer to the documentation on <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html" rel="noopener" target="_blank">launching your instance</a>.</p> 
<h3>Step 2. Mount the file system</h3> 
<p><a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html" rel="noopener" target="_blank">Connect to your Linux instance</a> using one of several methods described in the documentation.</p> 
<p>From the terminal window, mount the Amazon FSx file system. You can find instructions for how to mount your file system from the Amazon FSx console. Select the file system and then choose <strong>Attach</strong>.</p> 
<h3>Step 3. Create test files</h3> 
<p>After successfully mounting the file system, create a test file in the mounted directory /fsx/ns1/. We’ll call the file “file1.txt.”</p> 
<p><img alt="test file in mounted directory" class="wp-image-19248 aligncenter" height="275" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-12.png" width="1048" /></p> 
<p style="text-align: left;">Switch to <em>ACCOUNT-B</em>, and check the Amazon S3 bucket you created. You should find file1.txt.</p> 
<p><img alt="S3 bucket" class="wp-image-19249 aligncenter" height="373" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/11/06/Figure-13.png" width="1025" /></p> 
<p>Now upload another file directly to your Amazon S3 bucket. Let’s call it “file2.txt.”</p> 
<p>Go back to the EC2 terminal and type <code>ls -l</code>. you should see file2.txt in /fsx/ns1/.</p> 
<p><img alt="EC2 Terminal" class="wp-image-19065 aligncenter" height="261" src="https://d2908q01vomqb2.cloudfront.net/e1822db470e60d090affd0956d743cb0e7cdf113/2023/10/25/Fig14-1.png" width="782" /></p> 
<p>You can repeat the testing process with delete and update.</p> 
<h2>Cleaning up</h2> 
<p>Now that we tested the solution, execute the following four steps to delete the provisioned resources to avoid incurring unnecessary charges.</p> 
<ol> 
 <li>Terminate the Amazon EC2 instance you used to mount and test the file system.</li> 
 <li>Delete the Amazon FSx for Lustre file system you created in <em>ACCOUNT-A</em>.</li> 
 <li>Delete the sample data and the Amazon S3 bucket you created in <em>ACCOUNT-B</em>.</li> 
 <li>Delete the IAM service-linked role you created to provide Amazon S3 access to the Amazon FSx for Lustre file system.</li> 
</ol> 
<h2>Conclusion</h2> 
<p>Amazon FSx for Lustre’s native integration with S3 provides a proven, easy to deploy solution that leverages the high performance of a scale-out Lustre file system with the benefits of a data lake built on Amazon S3. In this post, we demonstrated how to deploy a solution to keep an Amazon FSx file system in sync with changes made to source data in an Amazon S3 bucket in a different AWS account. This solution helps enterprises scale their AWS environment by allowing data to be shared from a centralized enterprise data lake to specialized team accounts consuming that data for ML and HPC use cases.</p> 
<p>Do you have other challenges serving data from an enterprise data lake to ML and HPC teams? Let us know in the comments if this approach improves your delivery times!</p>
