---
title : "Backing Up and Restoring etcd"
date :  "`r Sys.Date()`" 
weight : 3 
chapter : false
pre : " <b> 3.3. </b> "
---
### Tại sao phải cần Backup & Restore cho ETCD

Trong kiến trúc Kubernetes, etcd là một phần không thể thiếu của cụm. Tất cả các đối tượng cụm và trạng thái của chúng được lưu trữ trong etcd. Một số điều bạn nên biết về etcd từ góc độ Kubernetes.

- Nó là một kho lưu trữ key-value nhất quán, phân tán và an toàn.
- Hỗ trợ kiến trúc có tính khả dụng cao với stack etcd.
- Nó lưu trữ cấu hình cụm kubernetes, tất cả các đối tượng API, trạng thái đối tượng và service discovery.
- etc.

Do đó việc sao lưu và backup etcd theo định kỳ là điều tất yếu, trong bài Lab này sẽ hướng dẫn các bạn thực hiện việc sao lưu và restore etcd.

**Mục tiêu**: 

- Hiểu được phương pháp backup và restore etcd.
- Xử lý lỗi nếu có

**Thiết lập môi trường**

- Sử dụng lại môi trường trong Lab3.2
{{% notice info %}}
Lưu ý: Các bạn xem lại Lab 3.2 để thiết lập môi trường.
{{% /notice %}}

Các bạn login vào EC2 Control-plane Node với user là ubuntu, kiểm tra K8s bằng lệnh sau:
```bash
kubectl get node
```

![Cfn](/images/053.png)

Để giúp các bạn hiểu cách backup và restore etcd. Mình sẽ hướng dẫn các bạn deploy một số pod trên cụm K8s. Các bạn thực hiện các lệnh sau:

```bash
kubectl run nginx-before-backup --image=nginx
```
![Cfn](/images/054.png)

Như vậy ta đã tạo xong 1 pod nginx trước khi backup.

### **Tiếp theo ta tiến hành backup etcd**

1. Các bạn truy cập vào file:

```bash
cd /etc/kubernetes/manifests/
```

![Cfn](/images/055.png)

2. Ta nhận thấy điều kiện để để tiến hành sao lưu etcd thì một số thông tin gồm phải có:

| Parameter | Value |
|-------|-------|
| --endpoints | ? | 
| --cert | ? |
| --key | ? |
| --cacert | ? |

Nhưng giá trị này sẽ được tìm thấy trong file: **etcd.yaml** và file **kube-apiserver.yaml** 

![Cfn](/images/056.png)

![Cfn](/images/057.png)

Từ đó ta có những giá trị:
| Parameter | Value | Location |
|-------|-------|-------|
| --endpoints | 127.0.0.1:2379 | kube-apiserver.yaml |
| --cert | /etc/kubernetes/pki/etcd/server.crt  | etcd.yaml | 
| --key | /etc/kubernetes/pki/etcd/server.key | etcd.yaml |
| --cacert | /etc/kubernetes/pki/etcd/ca.crt | etcd.yaml|

Sau khi đã có các giá trị tương ứng ta thực hiện backup cho etcd bằng lệnh sau:

```bash
ETCDCTL_API=3 sudo etcdctl --endpoints 127.0.0.1:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot save /opt/cluster_backup.db
```

Nếu trong quá trình run lệnh báo lỗi ta tiến hành cài đặt bổ sung etcd-client packages

![Cfn](/images/058.png)

Ta tiến hành run lại command trên

![Cfn](/images/059.png)

Như vậy ta đã backup etcd thành công

Tiếp theo ta tạo thêm 1 Pods nginx sau khi đã backup etcd.

```bash
kubectl run nginx-after-backup --image=nginx
```

![Cfn](/images/060.png)

Lúc này ta thấy đã tạo 2 con pod 1 con trước khi backup và 1 con sau khi backup.

#### **Bây giờ ta sẽ tiến hành recovery etcd db.**

Ta cũng làm tương tự một số thông số yêu cầu:

| Parameter | Value |
|-------|-------|
| --data-dir | ? | 
| --endpoints | ? | 
| --cert | ? |
| --key | ? |
| --cacert | ? |

Nhưng giá trị này sẽ được tìm thấy trong file: **etcd.yaml** và file **kube-apiserver.yaml** 

![Cfn](/images/056.png)

![Cfn](/images/057.png)

Từ đó ta có những giá trị:
| Parameter | Value | Location |
|-------|-------|-------|
| --data-dir | /var/lib/etcd | etcd.yaml |
| --endpoints | 127.0.0.1:2379 | kube-apiserver.yaml |
| --cert | /etc/kubernetes/pki/etcd/server.crt  | etcd.yaml | 
| --key | /etc/kubernetes/pki/etcd/server.key | etcd.yaml |
| --cacert | /etc/kubernetes/pki/etcd/ca.crt | etcd.yaml|

```bash
ETCDCTL_API=3 sudo etcdctl --data-dir /var/lib/etcd \
  --endpoints 127.0.0.1:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  snapshot restore /opt/cluster_backup.db > restore.txt 2>&1
```
![Cfn](/images/061.png)

Ta thấy báo lỗi trong quá trình restore

{{% notice info %}}
Error: data-dir "/var/lib/etcd" exists
{{% /notice %}}


Chúng ta tiến hành remove file 
```bash
sudo rm -rf /var/lib/etcd
```

Sau đó tiến hành recovery lại

![Cfn](/images/062.png)

Tiến hành kiểm tra lại

```bash
kubectl get pod

```
![Cfn](/images/063.png)

Lúc này ta chỉ còn thấy 1 pod nginx before backup

Như vậy là đã thành công trong việc sao lưu và restore etcd.