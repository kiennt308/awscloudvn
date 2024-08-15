---
title : "Giới thiệu"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1. </b> "
---
### **Kubernetes là gì**
{{% notice info %}}
Kubernetes is an open-source system for automating deployment, scaling, and management of containerized applications.
{{% /notice %}}

"Kubernetes là một hệ thống mã nguồn mở dành cho việc tự động triển khai, thu phóng cũng như quản lý các ứng dụng đã được contanier"

Từ Kubernetes bắt nguồn từ κυβερνήτης trong tiếng Hi Lạp, có nghĩ là người lái tàu. Với sự liên tưởng trong tâm trí của mình, ta có thể nghĩ Kubernetes giống như người thuyền trưởng trên một con tàu chứa đầy containers.

Kubernetes đôi khi được viết tắt là k8s (đọc là Kate's), bởi có 8 kí tự giữa k và s. Nó lấy cảm hứng từ hệ thống Borg của Google, một bộ điều phối container và workload cho quá trình vận hành toàn cầu trong hơn một thập kỷ công ty này.

Nó là một dự án mã nguồn mở được viết bằng ngôn ngữ Go và được phân phối dưới giấy phép Apache License, phiên bản 2.0.

### **Kiến trúc của Kubernetes** 

- Một nút **master** hoặc nhiều hơn, đây là một bộ phận của **control plane**
- Một nút **worker** hoặc nhiền hơn
  
![ConnectPrivate](/images/001.png)

#### **Master (Control-plane) Node**

Nút master cung cấp môi trường cho control plane chịu trách nhiệm quản lý trạng thái của các cụm Kubernetes và nó là đầu nào đằng sau mọi quá trình vận hành trong cụm.

Thành phần control plane với vai trò vô cùng đặc thù trong bộ quản lý cụm. Để có thể giao tiếp với cụm Kubernetes, người sử dụng gửi các requests đến control plane thông qua các công cụ Command Line Interface (CLI), một Dashboard dưới dạng giao diện người dùng web (Web UI) , hoặc các API (Application Programming Interface).

Việc giữ control plane chạy bằng mọi giá là tối quan trọng. Việc control plane ngừng hoạt động có thể dẫn đến downtime, thứ mà gây ra việc các dịch vụ gián đoạn và không thể cung cấp cho người dùng và có thể làm kinh doanh thua lỗ. 

Để đảm bảo cho khả năng chịu lỗi của control plane, các bản sao của nút master được thêm vào cụm, được cấu hình ở chế độ HA (High-Availability, độ khả dụng cao). Trong khi chỉ một trong nút master được chuyên biệt để quản lý chủ động cụm, các thành phần control plane giữ trạng thái đồng bộ giữa các bản sao của nút master. Chính loại cấu hình này đã cung cấp khả năng tự phục hồi cho control plane của cụm, thứ có thể cho phép nút master đang ở trạng thái active xảy ra lỗi ở mức độ nào đó.

Để duy trình trạng thái của cụm, tất cả các dữ liệu về cấu hình của cụm đều được lưu trong etcd. etcd là một bộ lưu trữ dạng key-value phân tán, thứ chỉ giữ các dữ liệu liên quan đến trạng thái của cụm chứ không bao gồm dữ liệu workload của client. etcd có thể được cấu hình trên nút master (stacked topology) hoặc trên một host được chuyên biệt (external topology) để có thể giảm thiểu khả nằng mất mát dữ liệu được lưu trữ bằng cách tách riêng chúng khỏi các tác tử control plane khác.

Với ở dạng stacked topology, các bản sao HA của nút master đảm bảo khả năng tự phục hồi của dữ liệu lưu trữ trong etcd. Tuy nhiên, đây không giống như trường hợp dùng etcd ở dạng external topology, khi mà host của etcd bắt buộc phải được tạo các nhân bản và tách rời để phục vụ cho chế đồ HA, cấu hình mà dẫn đến nhu cầu về các phần cứng bổ sung.

Một nút master có thể có những thành phần control plane như sau:

##### **1. API Server (máy chủ API)**

Tất cả các tác vụ quản trị đều được phối hợp triển khai bởi kube-apiserver, thành phần control plane trung tâm được chạy trên nút master. API Server tiếp nhận các yếu cầu RESTful từ người dùng, tác tử hoạc các tác tử từ phía bên ngoài, sau đó xác nhận và xử lý chúng.

Trong khi xử lý, API Server đọc trạng thái hiện tại của cụm Kubernetes từ bộ lưu trữ dữ liệu etcd và sau khi thực thi xong yêu cầu, trạng thái kết quả của cụm Kubernetes được lưu vào bộ lưu trữ dữ liệu dạng key-value phân tán để đảm bảo tính bền bỉ.

API Server là thành phần master plane duy nhất có thể giao tiếp với bộ lưu trữ dữ liệu etcd. bao gồm cả đọc và ghi các thông tin trạng thái của cụm Kubernetes - hoạt động như một giao diện trung gian cho bất kì một tác tử control plane khác truy cập đến trạng thái của cụm.

API Server có thể cấu hình và tùy biến một cách dễ dàng. Nó có thể mở rộng quy mô theo chiều ngang, nhưng nó cũng hỗ trợ việc bổ sung API Server phụ tùy chỉnh, một cấu hình biến API Server chính thành proxy cho tất cả API Server tùy chỉnh, thứ cấp và định tuyến tất cả các lệnh gọi RESTful đến chúng dựa trên các quy tắc được xác định tùy chỉnh.

##### **2. Scheduler (Bộ lập lịch)**

Vai trò của kube-scheduler là giao các các đối tượng workload mới, chẳng hạn như các pod cho các nút. Trong suốt quá trình lập lịch, các quyết định sẽ được tạo dựa trên trạng thái hiện tại của cụm Kubernetes và yêu cầu của các đối tượng mới. 

Bộ lập lịch thu thập từ bộ lưu trữ dữ liệu, thông qua API server, dữ liệu sử dụng tài nguyên của từng nút worker trong cụm. Bộ lập lịch cũng có thể nhận từ API Server các requirements của các đối tượng mới, thứ là một phần của dữ liệu cấu hình của chúng.

Bộ lập lịch có khả năng cấu hình và tùy chỉnh cực kì cao dựa trên các scheduling policies, plugins, and profiles.

Một bộ lập lịch thường cực kì quan trọng và phức tạp trong một cụm Kubernetes có nhiều nút. Tuy nhiên trong các cụm chỉ có duy nhất một nút, chẳng hạn như cụm được sử dụng làm ví dụ trong khóa học này, công việc của bộ lập lịch về cơ bản khá đơn giản.

##### **3. Controller Managers (Trình quản lý các controller)**

**controller managers** là các thành phần của control plane trên nút master, thứ mà chạy các controller để điều tiết trạng thái của cụm Kubernetes.

**kube-controller-manager** chạy các controllers chịu trách nhiệm phản ứng khi các nút không khả dụng, để đảm bảo số pod như mong đợi, để tạo endpoints, service accounts, và API access tokens.

**cloud-controller-manager** chạy các controllers chịu trách nhiệm tương tác với các hệ sinh thái cơ bản của bên cung cấp dịch vụ đám mây khi có nút trở nên không khả dụng cùng với đó là quản lý các container dữ liệu được cung cấp bởi các dịch vụ đám mấy và quản lý quá trình cân bằng tải và điều hướng.

##### **4. etcd Data Store (Kho dữ liệu)**

**etcd** là kho dữ liệu dạng key/value phân tán và có có tính nhất quán cao được sử dụng để đảm bảo tính bền bỉ của trạng thái của cụm Kubernetes. Dữ liệu mới được ghi vào kho lưu trữ chỉ bằng cách thêm vào cuối nó bởi vậy nên dữ liệu sẽ không bao giờ bị thay đổi trong đây. Dữ liệu lỗi thời được nén định kỳ để giảm thiểu dung lượng của kho dữ liệu.

Công cụ quản lý dưới dạng CLI cuar etcd - **etcdctl** cung cấp khả năng backup, snapshot, and restore, những thứ đặc biệt tiện dụng đối với một cụm Kubernetes có một etcd duy nhất - thường thấy trong trong môi trường Phát triển.

Một số công cụ khởi động cụm Kubernetes, chẳng hạn như **kubeadm**, mặc định sẽ cung cấp nút master với etcd dạng stacked, bơi ,à kho dữ liệu chạy song song và chia sẻ các tài nguyên với các thành phần control plane khác trên nút master đó.

![etcdstack](/images/002.png)

Để cách ly kho dữ liệu khỏi các thành phần control plane, quy trình khởi động có thể được cấu hình cho cấu trúc liên kết etcd bên ngoài (external), nơi lưu trữ dữ liệu được cung cấp trên một máy chủ riêng biệt chuyên dụng, do đó giảm nguy cơ xảy ra lỗi etcd.

![etcdexternal](/images/003.png)

Cả cách cấu hình etcd dạng stacked hoặc external đều hỗ trợ cấu hình HA. etcd dựa trên thuật toán Raft Consensus điều này cho phép một tập hợp các máy hoạt động như một Group thống nhất có thể tồn tại sau những sự cố của một số thành viên của nó. 

Tại bất kỳ thời điểm nào, một trong các nút trong Group sẽ là nút chính (master) và phần còn lại sẽ là nút theo dõi (follower). etcd xử lý khéo léo các cuộc bầu cử chính và có thể chịu được lỗi của nút, bao gồm cả lỗi của nút chính. Bất kỳ nút nào cũng có thể được coi là nút chính.

![etcd](/images/004.png)

etcd được viết bằng ngôn ngữ lập trình Go. Trong Kubernetes, bên cạnh việc lưu trữ trang thái cụm, etcd cũng được sử dụng để lưu trữ đặc tả các dữ liệu cấu hình chẳng hạn như subnets, ConfigMaps, Secrets,...

Ngoài ra, nút master có thể có:

Container Runtime
Node Agent
Proxy.

#### **Worker Node**

**worker node** cung cấp môi trường chạy cho các ứng dụng client.

Trên một cụm Kubernetes có nhiều worker, lưu lượng mạng giữa các client users và các ứng dụng được triển khai trên các Pods được xử lý trực tiếp bởi các nút worker và nó không được điều hướng thông qua nút master.

Một nút worker có các thành phần như sau:

##### **1. Container Runtime (môi trường chạy của container)**

Mặc dù Kubernetes được thiết kế như một engine có khả năng điều phối container, nó lại không có khả năng để có thể xử lý trực tiếp các container.

Để quản lý vòng đời của một container, Kubernetes cần có container runtime trên nuts mà một Pod và các container của nó được lập lịch. Kubernetes hỗ trợ các môi trường runtime sau:

**Docker** - container runtime phổ biến nhất được sử dụng với Kubernetes (nhưng k8s đã ngừng hỗ trợ tại thời điểm gần đây)
**CRI-O** - container runtime cho Kubernetes, nó cũng hỗ trợ Docker image registries
**containerd** - container runtime đơn giản và di động cung cấp sự tự động đáng kể
**frakti** - container runtime dựa trên hypervisor cho Kubernetes

##### **2. kubelet**

**kubelet** là tác tử chạy trên mỗi nút và giao tiếp với các thành phần của control plane từ nút master. Nó nhận thông tin định nghĩa Pod, chủ yếu từ API Server và tương tác với container runtime ở trên nút để chạy các container liên quan đến Pod đó. 

Nó cũng theo dõi tình trạng và tài nguyên của các Pod đang chạy các containers.

##### **3. Proxy - kube-proxy**

**kube-proxy** là tác nhân mạng chạy trên mỗi nút chịu trách nhiệm cập nhật và bảo trì động của các luật điều phối mạng trên nút đó. Nó trừu tượng hóa chi tiết của quá trình hoạt động mạng của Pod và chuyển tiếp các kết nối đến Pods.

Nguồn: [Tổng quan về kiến trúc của Kubernetes](https://viblo.asia/p/tong-quan-ve-kien-truc-cua-kubernetes-E375zVpq5GW)