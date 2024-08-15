---
title : "Upgrading a Single-Master Cluster Version with Kubeadm"
date :  "`r Sys.Date()`" 
weight : 2 
chapter : false
pre : " <b> 3.2. </b> "
---

### Tại sao phải cần cập nhật cho cluster cụm K8s

Dưới đây sẽ là vài lý do lý giải vì sao chúng ta cần phải cập nhật cụm Kubernetes:

- Đảm bảo tính ổn định của cụm Cluster
- Trải nghiệm và sử dụng các tính năng mới
- Ngăn ngừa các vấn đề bảo mật, lỗ hổng phiên bản cũ
- etc.

Do đó việc upgrade K8s là điều cần thiết, trong bài Lab này sẽ hướng dẫn các bạn nâng cấp cluster K8s.

**Mục tiêu**: 

- Hiểu được cách upgrade K8s cluster dùng kubeadm trên EC2 AWS
- Xử lý lỗi nếu có

**Thiết lập môi trường**

Một số thông số yêu cầu trước khi cài đặt

| Item | Version | Link |
|-------|-------|-------|
| UBUNTU SERVER | 22.04.3 | https://ubuntu.com/download/server |
| KUBERNETES | 1.29.1 | https://kubernetes.io/releases/ |
| CONTAINERD | 1.7.13 | https://containerd.io/releases/ |
| RUNC | 1.1.12 | https://github.com/opencontainers/runc/releases |
| CNI PLUGINS | 1.4.0 | https://github.com/containernetworking/plugins/releases |
| CALICO CNI | 3.27.2 | https://docs.tigera.io/calico/3.27/getting-started/kubernetes/quickstart |

Trong bài Lab này mình sử dụng 4 EC2:

| EC2 | Qty | AMI | vCPU | Memory |
|-------|-------|-------|-------|-------|
| ec2-cluster | 1 | Ubuntu 22.04 | 1 | 1GB | 
| ec2-control-plane | 1 | Ubuntu 22.04 | 2 | 4GB |
| ec2-worker1 | 2 | Ubuntu 22.04 | 1 | 1GB |

**Các bạn truy cập vào link bên dưới, để dùng Cloudformation thiết lập môi trường nhé:**

+ [VPC Cloudformation Stack](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/create?stackName=lab2&templateURL=https://workshopk8scfnstoragres.s3.ap-southeast-1.amazonaws.com/HOL2234_Workshop.yaml)

Link sẽ tự động chuyển hướng về Cloudformation Console, các bạn thay đổi một số thông tin trước khi deploy:

| Parameters | Value |
|-------|-------|
| Stack name | [yourname]-lab2 |
| EC2KeyPairName | Select Keypair EC2 |
| **MemberHOL** | **your_name** |

{{% notice info %}}
Lưu ý: Các bạn phải chọn đúng tên mình không được chọn thay tên người khác!!!.
{{% /notice %}}

![Cfn](/images/027.png)

![Cfn](/images/028.png)

![Cfn](/images/029.png)

Các bạn click **Next**

![Cfn](/images/030.png)

Sau khi đã kiểm tra kỹ càng các bạn click "Submit" để hệ thống tiến hành triển khai

![Cfn](/images/031.png)

Các bạn đợi một vài phút để triển khai các tài nguyên cần thiết cho việc triển khai.

![Cfn](/images/033.png)

Sau khi Cloudformation đã chạy xong, chúng ta vào EC2 Console, lúc này sẽ có 4 EC2 đang **Running**

![Cfn](/images/032.png)

Các bạn login vào EC2 Cluster với user là ubuntu, kiểm tra K8s bằng lệnh sau:
```bash
kubectl get node
```

![Cfn](/images/034.png)


Như vậy ta đã triển khai thành công

Phiên bản hiện tại của cụm K8s mà chúng ta cài đặt là v1.29.1 do đó bây giờ ta sẽ tiến hành nâng cấp:

1. Nâng cấp các control-plane node trước
2. Sau đó nâng cấp các Worker node

#### **Cấu hình Control-plane node**

Các bạn login vào EC2 Control-plane Node


![Cfn](/images/035.png)

Bây giờ chúng ta sẽ bắt đầu upgrade k8s ở control plane trước

1. Tìm kiếm phiên bản hỗ trợ kubeadm

```bash
sudo apt update
sudo apt-cache madison kubeadm
```

![Cfn](/images/036.png)

![Cfn](/images/037.png)

2. Kiểm tra phiên bản hiện tại

```bash
kubeadm version
```

![Cfn](/images/038.png)

3. Chọn phiên bản cần nâng cấp ở đây mình chọn phiên bản cần nâng câp là **v1.29.8-1.1**

```bash
sudo apt-mark unhold kubeadm
sudo apt-get update 
sudo apt-get install -y kubeadm=1.29.8-1.1
sudo apt-mark hold kubeadm
```
![Cfn](/images/039.png)

4. Sau khi đã upgrade kiểm tra phiên bản mới đã được cập nhật hay chưa
```bash
kubeadm version
```
![Cfn](/images/040.png)

5. Sau đó ta tiến hành verify upgrade plan
```bash
sudo kubeadm upgrade plan
```
![Cfn](/images/041.png)

6. Nếu tiến trình không phát sinh lỗi ta tiến hành upgrade kubeadm
```bash
sudo kubeadm upgrade apply v1.29.8
```

![Cfn](/images/042.png)

7. Sau khi upgrade kubeadm thành công ta tiến hành upgrade kubelet và kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet=1.29.8-1.1 kubectl=1.29.8-1.1
sudo apt-mark hold kubelet kubectl
```
![Cfn](/images/043.png)

8. Restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

![Cfn](/images/044.png)

9. Kiểm tra cụm K8s
```bash
kubectl get node
```

![Cfn](/images/045.png)

#### **Cấu hình Các Worker node**

1. Chúng ta cấu hình tuần tự cho từng worker node

2. Các bạn login vào Worker Node 1 và tiến hành upgrade

![Cfn](/images/046.png)

3. Drain Worker Node
```bash
# Control-plane or cluster set:
kubectl drain ip-10-0-1-232 --ignore-daemonsets

kubectl get node
```
![Cfn](/images/047.png)

4. Upgrade Worker Node

```bash
sudo kubeadm upgrade node
```
![Cfn](/images/048.png)

5. Upgrade kubelet và kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get update
sudo apt-get install -y kubelet=1.29.8-1.1 kubectl=1.29.8-1.1
sudo apt-mark hold kubelet kubectl
```

![Cfn](/images/049.png)

6. Restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
![Cfn](/images/050.png)

7. uncordon cho Worker Node
```bash
# Control-plane or cluster set:
kubectl uncordon ip-10-0-1-232
```
![Cfn](/images/051.png)

{{% notice note %}}
Các bạn làm tương tự cho Worker 2 Node nhé
{{% /notice %}}

Kết quả đạt được

![Cfn](/images/052.png)

Như vậy trong bài Lab này đã giúp các bạn upgrade cụm K8s thành công.