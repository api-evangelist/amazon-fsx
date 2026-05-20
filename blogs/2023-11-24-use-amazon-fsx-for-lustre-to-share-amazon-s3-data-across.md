---
title: "Use Amazon FSx for Lustre to share Amazon S3 data across accounts"
url: "https://aws.amazon.com/blogs/storage/using-amazon-fsx-for-lustre-to-share-amazon-s3-data-across-accounts/"
date: "Fri, 24 Nov 2023 20:45:08 +0000"
author: "Justin Leto"
feed_url: "https://aws.amazon.com/blogs/storage/category/storage/amazon-fsx/feed/"
---
Update 4/9/2025: The cross-account bucket policy in the blog has been updated. It was missing a required principal: “arn:aws:iam::accountID:role/AWS-Signed-In-Console-Role.” This omission causes an access denied error. As enterprises evolve their cloud governance practices, multiple teams working in separate accounts may need to share data.
