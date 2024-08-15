---
title : "Chuẩn bị VPC và Security Group"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 2.1 </b> "
---

Trong bước này chúng ta sẽ tạo VPC và Security Group để triển khai cụm K8s.

Sau khi đăng nhập vào AWS Console các bạn bấm vào link sau:

+ [VPC Cloudformation Stack](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/create?stackName=k8s-vpc&templateURL=https://workshopk8scfnstoragres.s3.ap-southeast-1.amazonaws.com/HOL0_Create_VPC.yaml)

Các bạn bấm next để tiến hành deploy, quá trình sẽ mất vài giây. Sau khi deploy xong kết quả sẽ như hình bên dưới:

![VPC](/images/005.png)

Như vậy bạn đã triển khai VPC K8s thành công.

  - [Tạo Stack Cloudformation](2.2-createcfn/)

