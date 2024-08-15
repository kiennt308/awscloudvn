---
title : "Creating a Single-Master Cluster with Kubeadm"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 3.1. </b> "
---

Chào mừng các bạn đến bài lab triển khai K8s trên EC2 AWS dùng kubeadm.

Trong bài lab này mình sẽ hướng dẫn các bạn triển khai cụm K8s.

**Mục tiêu**: 

- Hiểu được cách triển khai K8s dùng kubeadm trên EC2 AWS
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

+ [VPC Cloudformation Stack](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/create?stackName=lab1&templateURL=https://workshopk8scfnstoragres.s3.ap-southeast-1.amazonaws.com/HOL21_Workshop.yaml)

Link sẽ tự động chuyển hướng về Cloudformation Console, các bạn thay đổi một số thông tin trước khi deploy:

| Parameters | Value |
|-------|-------|
| Stack name | [yourname]-lab1 |
| EC2KeyPairName | Select Keypair EC2 |
| **MemberHOL** | **your_name** |

{{% notice info %}}
Lưu ý: Các bạn phải chọn đúng tên mình không được chọn thay tên người khác!!!.
{{% /notice %}}

![Cfn](/images/006.png)

![Cfn](/images/007.png)

![Cfn](/images/008.png)

Các bạn click **Next**

![Cfn](/images/009.png)

Sau khi đã kiểm tra kỹ càng các bạn click "Submit" để hệ thống tiến hành triển khai

![Cfn](/images/010.png)

Các bạn đợi một vài phút để triển khai các tài nguyên cần thiết cho việc triển khai.

![Cfn](/images/011.png)

Sau khi Cloudformation đã chạy xong, chúng ta vào EC2 Console, lúc này sẽ có 4 EC2 đang **Running**

![Cfn](/images/012.png)

Chúng ta tiến hành remote vào từng EC2 để cấu hình:

Có 2 cách để remote:

1. Các bạn có thể remote trực tiếp trên trình duyệt.
2. Các bạn có thể sử dụng một số công cụ terminal để login với keypair đã khai báo trước đó.

Ở đây mình sẽ sử dụng remote bằng trình duyệt web.

![Cfn](/images/013.png)

![Cfn](/images/014.png)

#### **Cấu hình Control-plane node**

Các bạn login vào EC2 Control-plane Node

Chuyển sang mode root

```bash
sudo su
```

![Cfn](/images/015.png)

Trong quá trình triển khai Cloudformation đã giúp chúng ta cài đặt một số thư viện có sẵn bạn có thể tham khảo script bên dưới

```bash
#!/bin/bash

apt update
apt install unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Load required kernel modules
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Setup required sysctl params
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# Install containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz -P /tmp/
tar -C /usr/local -xzf /tmp/containerd-1.7.13-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd

# Install runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 -P /tmp/
install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc

# Install CNI plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz -P /tmp/
mkdir -p /opt/cni/bin
tar -C /opt/cni/bin -xzf /tmp/cni-plugins-linux-amd64-v1.4.0.tgz

# Configure containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# Disable swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Install dependencies
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repository
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

# Update apt and install Kubernetes components
apt-get update
apt-get install -y kubelet=1.29.1-1.1 kubeadm=1.29.1-1.1 kubectl=1.29.1-1.1
apt-mark hold kubelet kubeadm kubectl
```

Bây giờ chúng ta sẽ tiến hành khởi chạy Kubernetes bằng lệnh như sau:

```bash
# Initialize the Kubernetes control plane
kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.29.1
```

![Cfn](/images/016.png)

Sau khi init không lỗi nó sẽ thông báo successfully

![Cfn](/images/017.png)

**Output**
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.1.168:6443 --token bip3vm.s1h46hf1lczg623k \
        --discovery-token-ca-cert-hash sha256:432ae9d8d1f9174c0ffea6c3f015036b1b27e5f32506c4825daa147657a8ca5a 

```
Hiện tại Kubernetes đang run với root user, nếu chúng ta muốn thêm user để có thể sử dụng truy vấn đến Kubernetes thì ta thực hiện một số câu lệnh bên dưới:

```bash
# Add kube config to ubuntu user.
mkdir -p /home/ubuntu/.kube;
cp -i /etc/kubernetes/admin.conf /home/ubuntu/.kube/config;
chmod 755 /home/ubuntu/.kube/config

# Set up kubeconfig for the root user
export KUBECONFIG=/etc/kubernetes/admin.conf
```

{{% notice info %}}
Lưu ý hiện tại user chạy với full quyền admin K8s, để giới hạn một số role ta sẽ thực hiện ở bài Lab 3.4
{{% /notice %}}

![Cfn](/images/018.png)

Tiếp theo chúng ta sẽ cài đặt Calico CNI cho K8s

```bash
# Install Calico CNI
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml

kubectl apply -f custom-resources.yaml
```

![Cfn](/images/019.png)

Kiểm tra kết quả sau khi cài đặt

![Cfn](/images/020.png)

Ta dùng lệnh:

```bash
kubectl get node
```
![Cfn](/images/021.png)

Ta thấy hiện tại chỉ có một control-plane node đã join vào cụm K8s, lúc này ta sẽ join các Worker node vào cụm K8s sau bằng lênh:

```bash
# Generate kubeadm join command and save to a file
kubeadm token create --print-join-command
```

![Cfn](/images/022.png)

Kết quả hiển thị 1 dòng lệnh, lúc này ta login vào các Worker Node để tiến hành cấu hình Join vào cụm K8s trên

**Output**
```bash
kubeadm join 10.0.1.168:6443 --token s34g2r.rgpyp5ksuoyh29r9 --discovery-token-ca-cert-hash sha256:432ae9d8d1f9174c0ffea6c3f015036b1b27e5f32506c4825daa147657a8ca5a
```

#### **Cấu hình các Worker node**

Các bạn login vào EC2 Workers Node 1

Chuyển sang mode root

```bash
sudo su
```

![Cfn](/images/023.png)

Trong quá trình triển khai Cloudformation đã giúp chúng ta cài đặt một số thư viện có sẵn bạn có thể tham khảo script bên dưới:

```bash
#!/bin/bash

apt update
apt install unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Load required kernel modules
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Setup required sysctl params
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# Install containerd
wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz -P /tmp/
tar -C /usr/local -xzf /tmp/containerd-1.7.13-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now containerd

# Install runc
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 -P /tmp/
install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc

# Install CNI plugins
wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz -P /tmp/
mkdir -p /opt/cni/bin
tar -C /opt/cni/bin -xzf /tmp/cni-plugins-linux-amd64-v1.4.0.tgz

# Configure containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# Disable swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Install dependencies
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repository
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list

# Update apt and install Kubernetes components
apt-get update
apt-get install -y kubelet=1.29.1-1.1 kubeadm=1.29.1-1.1 kubectl=1.29.1-1.1
apt-mark hold kubelet kubeadm kubectl
```

Chúng ta tiến hành join cụm Worker node vào K8s ở trên bạn lênh:

```bash
kubeadm join 10.0.1.168:6443 --token s34g2r.rgpyp5ksuoyh29r9 --discovery-token-ca-cert-hash sha256:432ae9d8d1f9174c0ffea6c3f015036b1b27e5f32506c4825daa147657a8ca5a
```
{{% notice note %}}
Lệnh này được sinh ra từ Control-plane mà ta đã chạy trước đó
{{% /notice %}}

![Cfn](/images/024.png)


Chúng login lại control plane node và tiến hành kiểm tra xem worker node đã join vào cụm K8s chưa

```bash
kubectl get node
```
![Cfn](/images/025.png)

Ta thấy Worker 1 đã join vào cụm K8s

{{% notice note %}}
Các bạn làm tương tự cho Worker 2 Node nhé
{{% /notice %}}

Kết quả đạt được

![Cfn](/images/026.png)

Như vậy bạn qua bài Lab trên đã giúp bạn cài đặt K8s sử dụng kubeadm.