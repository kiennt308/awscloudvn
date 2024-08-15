---
title : "Regulating Access to API Resources with RBAC"
date :  "`r Sys.Date()`" 
weight : 4 
chapter : false
pre : " <b> 3.4. </b> "
---
### Tại sao lại cần phải phân quyền quản trị trong K8s

Hãy tưởng tượng ta gặp trường hợp như sau. Công ty ta có một Kubernetes Cluster với hai môi trường pro và dev. Với pro cho môi trường sản phẩm thực tế và dev dùng để phát triển sản phẩm. 

Trước giờ bạn chỉ làm một mình và có toàn bộ quyền để quản lý Kubernetes Cluster, mỗi lần Developer muốn làm gì đều phải thông qua bạn, kể cả xem logs. 

Sếp bạn thấy vậy hơi bất tiện nên kêu bạn tìm cách làm thế nào để mấy bạn Developer có thể tự liệt kê ứng dụng và xem poda ở dưới môi trường dev.

**Giải pháp**
Ta dùng Role-based access control (RBAC) để giải quyết vấn đề trên.


**Mục tiêu**: 

- Hiểu được phương pháp phân quyền quản trị tài nguyên K8s thông qua user
- Xử lý lỗi nếu có

**Thiết lập môi trường**

- Sử dụng lại môi trường trong Lab3.2
{{% notice info %}}
Lưu ý: Các bạn xem lại Lab 3.2 để thiết lập môi trường.
{{% /notice %}}

Các bạn login vào EC2 Cluster Node với user là ubuntu, kiểm tra K8s bằng lệnh sau:
```bash
kubectl get node
```

![Cfn](/images/064.png)

### **Tiếp theo ta tiến hành phân quyền cho K8s sử dụng RBAC**

1. Trên EC2 Cluster tạo 2 user, user "devuser"
```bash
sudo su
sudo adduser devuser #password: admin123
sudo usermod -aG sudo devuser
```

![Cfn](/images/065.png)

2. Tạo certification cho user devuser:

```bash
sudo su - ubuntu
openssl genrsa -out devuser.pem

```

![Cfn](/images/066.png)

Tạo Create a Certificate Signing Request (CSR) cho devuser:

```bash
sudo su - ubuntu
openssl req -new -key devuser.pem -out devuser.csr -subj "/CN=devuser"
```
![Cfn](/images/067.png)

3. Encode base64 CSR user devuser:

```bash
cat devuser.csr | base64 | tr -d "\n" # output csr encode-base64 text
```
![Cfn](/images/068.png)

**Output**
```bash
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1Z6Q0NBVDhDQVFBd0VqRVFNQTRHQTFVRUF3d0haR1YyZFhObGNqQ0NBU0l3RFFZSktvWklodmNOQVFFQgpCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFQZ3dYb3J0V3BCemsxSHg3eVYyM1NkSW1rT1p3Ui90QzNja0ZMOWRLcDhTCk1HU3dSQy9VeS9lMlY5OVlSS3A1Qk5IeXhsekNYQ3QwRCs1UHZqR2hwWnlURjlBYkFtQXduN01RRWtWT29sbXoKRC9aTEdtQ2lhOFh0N1RTYnUxbFMweUtGWkcwRDNYdEtZYVA1Rnk0SWJpWEsxd0luSnk1Vk9ycnBvYkk4UG96WgoxOWlmek9HcGFGRythUEg2d0ZPa0NveWU5cXJBaFg5dUpjVzVJSlFPRnNBQ2RUWjVCcFZBOE5jUkdDZzF3K01DClFWSjdXZGtVNjc5bEllTjlZSFZ6UVBOK1gxdkhnSElaTG4wSXcxcWVNUUFWMG5odlBnNTA3MXZEL1lhQkxFcGMKL0VNKzlVSUxFQ2RSMTB1MWtwMnV3WmxGQ3QraXg0UlJ0R1htbHlBWlEzMENBd0VBQWFBQU1BMEdDU3FHU0liMwpEUUVCQ3dVQUE0SUJBUUFHa3lDMUxKMk45QUNoRDk1L1VOdGxCQ1ZpbDE4MVRSc3pTOTFhbzJYM1BudlFVOUFrCmJNekJaY3lKSGo0cnY4ZkFEalV6WnJQb205cjJCZnIvbXRVdmduZzRBTk90U0ViUWY0UHN6SEdYdVRianJ1aU8KcFo5VWxBdE9iOUpWQm1ya1hIdjFFVXRDdyt5azZlMkJjUVZ1T01SdXdBWGFYSDBKR0tlcENsemlqM21CSnhzOQorYTFNaVdCSmpLWGMzc3VCaVovNWFMZXp6aG1uR3JBU0tTaEFBMjVOcGtoR0FTc3NvMEVWQUpmclBXRVNEK2NkCm81bVFxT1QvdlhvTDh0cFVyMUF1ZTNiaFA3RWI0TGdqaVp3U29mVjh6ZDZnbFBDVG1QQjB5blBnYXBGcXFmOWwKOWhnZEl4dXhRdFZWa2s5UlZ6ZTB6UUhwT0s4WGtwSjRIcEVSCi0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
```

4. "Đăng ký ""CertificateSigningRequest"" trên cụm K8s
   
Tạo file ""devuserSigningRequest.yaml"":
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: devuser
spec:
  request: <base64_encoded_csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - digital signature
  - key encipherment
  - client auth
```
Trong đó **<base64_encoded_csr>** chính là output của bước 3.

![Cfn](/images/069.png)

Sau đó chạy lệnh đăng ký trên K8s:
```bash
kubectl apply -f devuserSigningRequest.yaml
```
Kiểm tra trạng thái csr:
```bash
kubectl get csr
```

![Cfn](/images/070.png)

5. Ta thấy trong cột điều kiện đang thể hiện trạng thái pending chờ được approve từ admin:

Approve cho user devuser
```bash
kubectl certificate approve devuser
```
Sau khi appove tiến hành kiểm tra lại csr:
```bash
kubectl get csr/devuser
```
![Cfn](/images/071.png)

6. Tạo xác thực cho user devuser:
```bash
kubectl get csr/devuser -o jsonpath=""{.status.certificate}"" | base64 -d > devuser.crt
```
![Cfn](/images/072.png)

7. Tạo config file cho user **devuser**:
   
Lấy server url:
```bash
kubectl config view
kubectl config view -o jsonpath="{.clusters[0].cluster.server}" #output is url kubernetes api
```
![Cfn](/images/073.png)

**Output**
```bash
https://10.0.1.45:6443
```

Cấu hình config cho user **devuser**

```bash
sudo kubectl --kubeconfig ~/.kube/config-devuser config set-cluster devuser-cluster --insecure-skip-tls-verify=true --server=https://10.0.1.45:6443
```
![Cfn](/images/074.png)



8. Tạo credentials cho user **devuser**:

```bash
sudo kubectl --kubeconfig ~/.kube/config-devuser config set-credentials devuser --client-certificate=devuser.crt --client-key=devuser.pem --embed-certs=true
```
![Cfn](/images/075.png)

9. Tạo context cho user "devuser":

```bash
sudo kubectl --kubeconfig ~/.kube/config-devuser config set-context devuser-context --cluster=devuser-cluster --user=devuser
```
![Cfn](/images/076.png)

10. Đăng ký context cho cụm K8s:

```bash
sudo kubectl --kubeconfig ~/.kube/config-devuser config use-context devuser-context
```
![Cfn](/images/077.png)

11. Thêm quyền xr cho file **config-devuser**:

```bash
sudo chmod +xr config-devuser
```
![Cfn](/images/078.png)


12. Tạo roles và rolebindings cho user **devuser**:

Writing RBAC Rules

Tạo file **role-devuser.yaml**:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
   namespace: default
   name: pod-reader
rules:
- apiGroups: [""] # indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
Create role:
```bash
kubectl apply -f role-devuser.yaml
```
![Cfn](/images/079.png)

13. Tạo rolebinding cho role đã tạo trước đó:

tạo file **devuser-pod-reader-rolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: devuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

```
Create role:
```bash
kubectl apply -f devuser-pod-reader-rolebinding.yaml
```
![Cfn](/images/080.png)

14. Dùng quyền admin trong cụm ubuntu tạo 1 pod trên namespace default:
    
```bash
kubectl run nginx --image=nginx
```

Sử dụng config-devuser kiểm tra pods trên namspace default và kube-system
```bash
kubectl --kubeconfig ~/.kube/config-devuser get pod -n default
kubectl --kubeconfig ~/.kube/config-devuser get pod -n kube-system
```
![Cfn](/images/081.png)

15. Chuyển mode user devuser trong EC2 Cluster copy file **config-devuser** trong folder **.kube**

Sau khi copy đổi tên file config-devuser sang config.

```bash
sudo su - devuser
mkdir .kube
cd .kube/
sudo cp /home/ubuntu/.kube/config-devuser /home/devuser/.kube/
sudo mv config-devuser config

```

![Cfn](/images/082.png)

16. Tiến hành kiểm tra với user devuser:
```bash
kubectl get pods -n default
kubectl get pods -n kube-system
```

![Cfn](/images/083.png)

Như vậy thông qua bài Lab trên bạn đã phân quyền thành công.