---
title: "Route traffic to an Amazon FSx for NetApp ONTAP Multi-AZ file system using Network Load Balancer"
url: "https://aws.amazon.com/blogs/storage/route-traffic-to-an-amazon-fsx-for-ontap-multi-az-file-system-using-network-load-balancer/"
date: "Fri, 09 Dec 2022 23:31:48 +0000"
author: "Jay Horne"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
Update (3/13/2023): When creating a target group, the Network Load Balancer currently verifies that the target IP is part of a subnet in the specified VPC, which means the approach outlined in this blog cannot be employed anymore. A core architecture principle of building highly available applications on AWS is to use a multi-Availability Zone […]
