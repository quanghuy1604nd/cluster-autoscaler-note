## **Tổng Quan về Cluster Autoscaler và Cluster API**

Trước khi đi vào chi tiết, hãy hiểu rõ cơ chế hoạt động cốt lõi:

  * **Cluster Autoscaler (CA):** Là một thành phần của Kubernetes, có nhiệm vụ tự động điều chỉnh số lượng node trong một cluster. Nó theo dõi các Pod đang ở trạng thái `Pending` vì không đủ tài nguyên (CPU, memory) và quyết định có cần thêm node mới hay không. Ngược lại, nó cũng tìm kiếm các node đang sử dụng dưới một ngưỡng nhất định trong một khoảng thời gian để loại bỏ chúng nhằm tiết kiệm chi phí.
  * **Cluster API (CAPI):** Là một dự án của Kubernetes cho phép quản lý các cluster Kubernetes một cách khai báo (declarative) bằng các tài nguyên của chính Kubernetes. Thay vì dùng script để tạo máy ảo trên AWS, GCP, v.v., bạn định nghĩa một tài nguyên `MachineDeployment` trong cluster quản lý, và CAPI sẽ lo việc tạo ra các máy ảo tương ứng ở cluster con.
  * **Sự kết hợp:** Khi CA hoạt động với CAPI, nó không trực tiếp tương tác với cloud provider (AWS, Azure, GCP...). Thay vào đó, **CA sẽ điều chỉnh trường `replicas` của tài nguyên `MachineDeployment` hoặc `MachineSet`** trong cluster quản lý. CAPI sẽ theo dõi sự thay đổi này và tự động tạo hoặc xóa các `Machine` (tương ứng là các máy ảo) ở hạ tầng bên dưới.

-----

## **Phân Tích Chi Tiết Các Tham Số và Đề Xuất**

Việc cấu hình Cluster Autoscaler cho CAPI chủ yếu được thực hiện qua hai cách:

1.  **Các cờ (flags)** khi khởi chạy pod Cluster Autoscaler.
2.  **Các annotations** trên tài nguyên `MachineDeployment` (MD) hoặc `MachineSet` (MS). **Đây là cách tiếp cận hiện đại và linh hoạt hơn, được khuyến khích sử dụng.**

Dưới đây là phân tích các tham số quan trọng nhất.

### **1. Cấu hình Ngưỡng Scale và Kích Thước Node Group**

Đây là nhóm tham số cơ bản và quan trọng nhất, quyết định giới hạn hoạt động của CA.

| Tên Tham Số/Annotation | Ý Nghĩa | Đề Xuất & Lý Do |
| :--- | :--- | :--- |
| **`cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size`** | (Annotation trên MD/MS) Số lượng node tối thiểu cho node group này. CA sẽ không bao giờ scale down thấp hơn con số này. | **Đặt \>= 1**. \<br\> **Lý do:** Đảm bảo luôn có ít nhất một node sẵn sàng cho các workload hệ thống hoặc các workload quan trọng, giảm độ trễ khi có yêu cầu đột ngột. Đối với môi trường production, có thể đặt là `2` hoặc `3` để đảm bảo tính sẵn sàng cao (HA). |
| **`cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size`** | (Annotation trên MD/MS) Số lượng node tối đa cho node group này. CA sẽ không bao giờ scale up vượt quá con số này. | **Đặt theo ngân sách và giới hạn của cloud provider.** \<br\> **Lý do:** Đây là tham số kiểm soát chi phí quan trọng nhất. Cần tính toán cẩn thận dựa trên tải dự kiến và ngân sách cho phép để tránh chi phí phát sinh ngoài ý muốn. |
| **`--scale-down-utilization-threshold`** | (Cờ) Ngưỡng sử dụng tài nguyên (CPU và Memory) của một node. Nếu một node có mức sử dụng dưới ngưỡng này, nó sẽ được xem là một ứng viên để scale down. Giá trị là một số float từ 0.0 đến 1.0. | **Đề xuất: `0.5` - `0.6` (mặc định là `0.5`)**. \<br\> **Lý do:** Giá trị `0.5` (50%) là một điểm khởi đầu an toàn. Nó chừa lại một khoảng trống tài nguyên đủ để hấp thụ các biến động nhỏ về tải mà không cần scale up ngay lập tức. Tăng giá trị này (ví dụ `0.7`) sẽ làm CA scale down mạnh mẽ hơn (tiết kiệm chi phí hơn) nhưng có thể khiến cluster chậm phản ứng với tải tăng đột ngột. |
| **`--scale-down-unneeded-time`** | (Cờ) Khoảng thời gian một node phải ở trạng thái "không cần thiết" (unneeded) trước khi bị CA xóa đi. | **Đề xuất: `10m` - `15m` (mặc định là `10m`)**. \<br\> **Lý do:** Giá trị mặc định `10m` là hợp lý. Nó giúp tránh tình trạng "đập-xây" (thrashing) khi tải biến động nhanh. Nếu tải của bạn rất ổn định, có thể giảm xuống `5m` để tiết kiệm chi phí nhanh hơn. Nếu tải biến động lớn, hãy tăng lên `15m` hoặc `20m` để CA "kiên nhẫn" hơn. |
| **`--max-node-provision-time`** | (Cờ) Thời gian tối đa CA chờ một node mới được thêm vào cluster và chuyển sang trạng thái `Ready`. | **Đề xuất: `20m` - `25m` (mặc định là `15m`)**. \<br\> **Lý do:** Thời gian khởi tạo node phụ thuộc vào cloud provider và quá trình bootstrap (cài đặt CNI, kube-proxy...). `15m` đôi khi hơi ngắn. Tăng lên `20m` hoặc `25m` giúp CA có đủ thời gian chờ, đặc biệt khi kéo các image lớn hoặc cấu hình phức tạp, tránh việc CA từ bỏ một node đang khởi tạo hợp lệ. |

### **2. Kiểm Soát Việc Scale Down**

Nhóm tham số này giúp ngăn chặn CA xóa các node quan trọng hoặc gây gián đoạn dịch vụ.

| Tên Tham Số/Annotation | Ý Nghĩa | Đề Xuất & Lý Do |
| :--- | :--- | :--- |
| **`--skip-nodes-with-local-storage`** | (Cờ) Nếu là `true`, CA sẽ không xóa các node đang chạy các pod sử dụng `emptyDir` hoặc `hostPath` volume. | **Đề xuất: `true` (mặc định là `true`)**. \<br\> **Lý do:** Cực kỳ quan trọng để bảo vệ dữ liệu. Các pod stateful sử dụng local storage sẽ mất dữ liệu nếu node bị xóa. Luôn bật cờ này trừ khi bạn chắc chắn 100% rằng các pod dùng local storage là tạm thời và có thể bị xóa an toàn. |
| **`--skip-nodes-with-system-pods`** | (Cờ) Nếu là `true`, CA sẽ không xóa các node đang chạy các pod trong namespace `kube-system` nếu các pod đó không được quản lý bởi `DaemonSet`. | **Đề xuất: `true` (mặc định là `true`)**. \<br\> **Lý do:** Bảo vệ các thành phần cốt lõi của cluster. Một số pod hệ thống quan trọng (ví dụ: `metrics-server`, `coredns` nếu không chạy dạng deployment) cần được giữ lại. Bật cờ này là một biện pháp an toàn tốt. |
| **`--scale-down-delay-after-add`** | (Cờ) CA sẽ không thực hiện scale down trong khoảng thời gian này sau khi vừa thực hiện scale up. | **Đề xuất: `15m` (mặc định là `10m`)**. \<br\> **Lý do:** Sau khi thêm node mới, tải có thể cần thời gian để cân bằng lại trên toàn cluster. Tăng giá trị này lên một chút (ví dụ `15m`) cho phép cluster ổn định hoàn toàn trước khi CA đưa ra quyết định scale down tiếp theo, tránh thrashing. |

### **3. Chiến Lược Mở Rộng Node Group (Expander)**

Khi có nhiều node group (nhiều `MachineDeployment`) có thể đáp ứng yêu cầu của một pod đang `Pending`, `expander` sẽ quyết định chọn node group nào để scale up.

| Tên Tham Số/Annotation | Ý Nghĩa | Đề Xuất & Lý Do |
| :--- | :--- | :--- |
| **`--expander`** | (Cờ) Xác định chiến lược lựa chọn node group. Các giá trị phổ biến: `random`, `most-pods`, `least-waste`, `priority`. | **Đề xuất: `least-waste` cho môi trường đồng nhất, `priority` cho môi trường không đồng nhất.** \<br\> **Lý do:** \<br\> - **`least-waste`**: Lựa chọn node group mà sau khi thêm node mới, lượng tài nguyên không sử dụng (lãng phí) là ít nhất. Đây là lựa chọn tốt nhất cho việc tối ưu chi phí trong hầu hết các trường hợp. \<br\> - **`priority`**: Cho phép bạn gán một mức độ ưu tiên cho mỗi node group. CA sẽ luôn thử scale up node group có độ ưu tiên cao nhất trước. Rất hữu ích khi bạn có các loại máy khác nhau (ví dụ: máy thường và máy có GPU) và muốn ưu tiên sử dụng loại máy rẻ hơn trước. \<br\> - **`random`**: Chọn ngẫu nhiên, thường chỉ dùng để gỡ lỗi. \<br\> - **`most-pods`**: Ưu tiên node group có khả năng chứa nhiều pod nhất, hữu ích khi muốn tối đa hóa mật độ pod. |
| **`cluster-autoscaler.kubernetes.io/priority`** | (Annotation trên CRD `ClusterAutoscalerNodeGroup` hoặc đôi khi là MD) Gán mức độ ưu tiên cho một node group khi sử dụng `expander=priority`. | **Gán số nguyên, số càng cao ưu tiên càng thấp.** \<br\> **Lý do:** Khi dùng `expander=priority`, bạn phải tạo các CRD `ClusterAutoscalerNodeGroup` tương ứng và đặt annotation này. Ví dụ: gán `priority: 10` cho node group máy thường và `priority: 20` cho node group máy có GPU. CA sẽ luôn chọn node group có `priority: 10` trước. |

### **4. Tham Số Đặc Thù cho Provider 'ClusterAPI'**

| Tên Tham Số/Annotation | Ý Nghĩa | Đề Xuất & Lý Do |
| :--- | :--- | :--- |
| **`--cloud-provider`** | (Cờ) Chỉ định cloud provider mà CA sẽ tương tác. | **Bắt buộc: `clusterapi`**. \<br\> **Lý do:** Đây là cờ để kích hoạt chế độ hoạt động với CAPI. |
| **`--clusterapi-mode`** | (Cờ) Chế độ hoạt động của CA. Có hai giá trị: `MachineDeployment` (mặc định) hoặc `MachineSet`. | **Đề xuất: `MachineDeployment` (mặc định)**. \<br\> **Lý do:** `MachineDeployment` cung cấp cơ chế cập nhật và rollback tương tự như `Deployment` cho Pods, giúp quản lý vòng đời của các node dễ dàng hơn. Sử dụng `MachineSet` chỉ khi bạn cần kiểm soát trực tiếp và không muốn có logic cập nhật tự động của `MachineDeployment`. |

-----
## Tương thích version giữa Kubernetes, Cluster API và Cluster Autoscaler
- **Từ Kubernetes 1.12, các version Kubernetes và Cluster Autoscaler tương thích sẽ trùng với nhau**
- Mỗi một cluster API minor được release sẽ support:
  - 4 Version của Kubernetes cho cụm management (N - N-3)
  - 6 Version của Kubernetes cho cụm workload (N - N-5)
- Nếu một version của Kubernetes được phát hành, Cluster API sẽ hỗ trọ thêm version đó, đồng nghĩa với việc nó sẽ support
  - 5 Version của Kubernetes cho cụm management (N - N-4)
  - 7 Version của Kubernetes cho cụm workload (N - N-6)
- Bảng tương thích version Kubernetes và Cluster API

| Cluster API Version | Status                 | Kubernetes Versions (Management Cluster) | Kubernetes Versions (Workload Cluster) | Cluster Autoscaler Version | Notes |
|---------------------|------------------------|------------------------------------------|----------------------------------------|----------------------------|-------|
| v1.11.x             | Under development      | TBD                                      | TBD                                    | TBD                        | Kubernetes and CA support to be determined upon release. |
| v1.10.x             | Standard support       | v1.29.x - v1.33.x                        | v1.27.x - v1.33.x                      | 1.29.x - 1.33.x            | CA version must match management cluster Kubernetes version (e.g., CA 1.33.x for Kubernetes 1.33.x). Supports Kubernetes v1.33.x from v1.10.1. Maintenance mode starts when v1.12.0 is released; EOL when v1.13.0 is released. |
| v1.9.x              | Standard support       | v1.28.x - v1.32.x                        | v1.26.x - v1.32.x                      | 1.28.x - 1.32.x            | CA version must match management cluster Kubernetes version (e.g., CA 1.32.x for Kubernetes 1.32.x). Supports Kubernetes v1.32.x from v1.9.1. Maintenance mode starts when v1.11.0 is released; EOL when v1.12.0 is released. |
| v1.8.x              | Maintenance mode       | v1.27.x - v1.31.x                        | v1.25.x - v1.31.x                      | 1.27.x - 1.31.x            | CA version must match management cluster Kubernetes version (e.g., CA 1.31.x for Kubernetes 1.31.x). Supports Kubernetes v1.31.x from v1.8.1. Maintenance mode since 2025-04-22 (v1.10.0 release); EOL when v1.11.0 is released. |
| v1.7.x              | EOL                    | v1.26.x - v1.30.x                        | v1.24.x - v1.30.x                      | 1.26.x - 1.30.x            | CA version must match management cluster Kubernetes version. EOL since 2025-04-22 (v1.10.0 release). |
| v1.6.x              | EOL                    | v1.25.x - v1.29.x                        | v1.23.x - v1.29.x                      | 1.25.x - 1.29.x            | CA version must match management cluster Kubernetes version. EOL since 2024-12-10 (v1.9.0 release). |
| v1.5.x              | EOL                    | v1.24.x - v1.28.x                        | v1.22.x - v1.28.x                      | 1.24.x - 1.28.x            | CA version must match management cluster Kubernetes version. EOL since 2024-08-12 (v1.8.0 release). |
| v1.4.x              | EOL                    | v1.23.x - v1.27.x                        | v1.21.x - v1.27.x                      | 1.23.x - 1.27.x            | CA version must match management cluster Kubernetes version. EOL since 2024-04-16 (v1.7.0 release). |
| v1.3.x              | EOL                    | v1.22.x - v1.26.x                        | v1.20.x - v1.26.x                      | 1.22.x - 1.26.x            | CA version must match management cluster Kubernetes version. EOL since 2023-12-05 (v1.6.0 release). |
| v1.2.x              | EOL                    | v1.21.x - v1.25.x                        | v1.19.x - v1.25.x                      | 1.21.x - 1.25.x            | CA version must match management cluster Kubernetes version. EOL since 2023-07-25 (v1.5.0 release). |
| v1.1.x              | EOL                    | v1.20.x - v1.24.x                        | v1.18.x - v1.24.x                      | 1.20.x - 1.24.x            | CA version must match management cluster Kubernetes version. EOL since 2023-03-28 (v1.4.0 release). |
| v1.0.x              | EOL                    | v1.19.x - v1.23.x                        | v1.17.x - v1.23.x                      | 1.19.x - 1.23.x            | CA version must match management cluster Kubernetes version. EOL since 2022-12-01 (v1.3.0 release). |


---

## **Thiết Kế Các Bài Lab Thực Tế**

Để chứng minh các đề xuất trên, chúng ta sẽ thiết kế 3 bài lab với độ khó tăng dần.

**Yêu Cầu Chuẩn Bị Chung:**

1.  Một management cluster (có thể dùng `kind` hoặc `minikube`).
2.  Đã cài đặt `clusterctl`, `kubectl`.
3.  Một cluster con (workload cluster) được tạo bởi CAPI (ví dụ: sử dụng provider Docker - `clusterctl init --infrastructure docker`).
4.  Cluster Autoscaler đã được triển khai và trỏ tới workload cluster.

### **Lab 1: Chứng minh Cấu hình Min/Max Size và Scale-Up/Down Cơ Bản**

**Mục tiêu:** Chứng minh CA tôn trọng annotation `min-size`, `max-size` và phản ứng với tải bằng cách scale-up và scale-down.

**Các bước thực hiện:**

1.  **Chuẩn bị `MachineDeployment`:**
    Tạo một `MachineDeployment` và thêm các annotations sau:

    ```yaml
    apiVersion: cluster.x-k8s.io/v1beta1
    kind: MachineDeployment
    metadata:
      name: md-worker-pool-1
      annotations:
        "cluster.x-k8s.io/cluster-api-autoscaler-node-group-min-size": "1"
        "cluster.x-k8s.io/cluster-api-autoscaler-node-group-max-size": "4"
    spec:
      replicas: 1
      # ... phần còn lại của spec
    ```

    Áp dụng file này. Cluster của bạn sẽ bắt đầu với 1 worker node.

2.  **Tạo Tải để Scale-Up:**
    Tạo một file `deployment-test.yaml` để tạo các pod yêu cầu tài nguyên, buộc CA phải hành động.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: cpu-eater
    spec:
      replicas: 10 # Số replicas lớn để chắc chắn cần nhiều hơn 1 node
      selector:
        matchLabels:
          app: cpu-eater
      template:
        metadata:
          labels:
            app: cpu-eater
        spec:
          containers:
          - name: cpu-eater
            image: nginx
            resources:
              requests:
                cpu: "500m" # Yêu cầu 0.5 core CPU
    ```

    Áp dụng file này lên workload cluster.

3.  **Quan sát Scale-Up:**

      * Kiểm tra các pod: `kubectl get pods -o wide`. Bạn sẽ thấy nhiều pod ở trạng thái `Pending`.
      * Kiểm tra log của pod Cluster Autoscaler: `kubectl logs -f <cluster-autoscaler-pod-name> -n kube-system`. Bạn sẽ thấy các dòng log chỉ ra có các pod không thể được lập lịch và quyết định scale up.
      * Quan sát `MachineDeployment` trên management cluster: `kubectl get machinedeployment md-worker-pool-1`. Bạn sẽ thấy trường `replicas` tăng dần từ 1 lên (ví dụ) 3 hoặc 4.
      * Kết quả: Cluster đã scale up lên số node cần thiết (nhưng không vượt quá 4).

4.  **Dọn dẹp Tải để Scale-Down:**
    Xóa deployment vừa tạo: `kubectl delete deployment cpu-eater`.

5.  **Quan sát Scale-Down:**

      * Chờ khoảng thời gian bằng giá trị của `--scale-down-unneeded-time` (ví dụ: 10 phút).
      * Quan sát log của CA, bạn sẽ thấy nó xác định các node là "unneeded".
      * Kiểm tra lại `MachineDeployment`: `kubectl get machinedeployment md-worker-pool-1`. Bạn sẽ thấy trường `replicas` giảm về `1` (giá trị `min-size`).
      * **Kết luận Lab 1:** CA hoạt động đúng với các giới hạn `min/max` và tự động scale theo tải.

### **Lab 2: Chứng minh Tham Số Bảo Vệ Scale-Down (`--skip-nodes-with-local-storage`)**

**Mục tiêu:** Chứng minh CA sẽ không xóa một node nếu trên đó có pod sử dụng local storage.

**Các bước thực hiện:**

1.  **Tạo môi trường:** Thực hiện lại các bước 1, 2, 3 của Lab 1 để scale cluster lên 2 hoặc 3 node.

2.  **Tạo pod với Local Storage:**
    Khi đã có nhiều node, hãy tạo một pod sử dụng `hostPath` và ghim nó vào một trong các node mới được tạo (không phải node đầu tiên).

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-with-local-storage
    spec:
      containers:
      - name: shell
        image: nginx
        volumeMounts:
        - name: local-data
          mountPath: /data
      volumes:
      - name: local-data
        hostPath:
          path: /tmp/my-data # Sử dụng một thư mục trên host
    ```

    Áp dụng file này.

3.  **Giảm tải chính:**
    Scale deployment `cpu-eater` về 0: `kubectl scale deployment cpu-eater --replicas=0`.
    Lúc này, các node được tạo thêm không còn cần thiết cho `cpu-eater` nữa. Tuy nhiên, một trong số chúng đang chạy `pod-with-local-storage`.

4.  **Quan sát:**

      * Chờ qua thời gian `--scale-down-unneeded-time`.
      * Kiểm tra log của CA. Bạn sẽ thấy log báo rằng nó đã bỏ qua việc scale down một node vì có pod với local storage: `skip-nodes-with-local-storage is enabled and node has pods with local storage`.
      * Kiểm tra `MachineDeployment`: `kubectl get machinedeployment md-worker-pool-1`. Bạn sẽ thấy số lượng `replicas` giảm, nhưng sẽ còn lại 2 node: node ban đầu và node đang chạy `pod-with-local-storage`. Các node "thừa" khác sẽ bị xóa.
      * **Kết luận Lab 2:** Tham số `--skip-nodes-with-local-storage=true` đã bảo vệ thành công node chứa dữ liệu stateful.

### **Lab 3: Chứng minh Chiến Lược Expander (`priority`)**

**Mục tiêu:** Chứng minh CA sẽ ưu tiên scale-up node group có `priority` cao hơn (giá trị số nhỏ hơn).

**Các bước thực hiện:**

1.  **Chuẩn bị 2 `MachineDeployment`:**

      * `md-general-purpose`: Dùng loại máy thường, rẻ tiền.
      * `md-gpu-intensive`: Dùng loại máy có GPU, đắt tiền.
      * Thêm annotation `min-size=0`, `max-size=3` cho cả hai.

2.  **Cấu hình `priority`:**

      * Trong file deployment của Cluster Autoscaler, thêm cờ: `--expander=priority`.
      * Tạo 2 file CRD `ClusterAutoscalerNodeGroup` (nếu provider của bạn yêu cầu) hoặc thêm annotation trực tiếp vào MD nếu được hỗ trợ, để gán priority:
          * Gán `priority: 10` cho `md-general-purpose`.
          * Gán `priority: 20` cho `md-gpu-intensive`.

3.  **Tạo Tải thông thường:**
    Tạo một deployment yêu cầu CPU/Mem bình thường, có thể chạy trên cả hai loại node.
    `kubectl apply -f deployment-test.yaml` (dùng lại từ Lab 1).

4.  **Quan sát:**

      * Kiểm tra log CA và các `MachineDeployment`. Bạn sẽ thấy **chỉ có `md-general-purpose` được scale up**. CA đã chọn node group có priority cao hơn (`10`).

5.  **Tạo Tải yêu cầu GPU:**
    Tạo một deployment mới yêu cầu tài nguyên GPU (`nvidia.com/gpu: 1`). Pod này chỉ có thể chạy trên các node từ `md-gpu-intensive`.

    ```yaml
    # deployment-gpu.yaml
    # ...
    spec:
      containers:
      - name: cuda-test
        image: "nvidia/cuda:11.0.3-base-ubuntu20.04"
        resources:
          limits:
            nvidia.com/gpu: 1
    ```

    Áp dụng file này.

6.  **Quan sát:**

      * Lần này, dù `md-general-purpose` có priority cao hơn, nó không thể đáp ứng yêu cầu.
      * Kiểm tra log CA. Nó sẽ báo rằng không có lựa chọn nào trong node group ưu tiên, và sau đó sẽ chọn node group phù hợp duy nhất.
      * Kiểm tra các `MachineDeployment`. Bạn sẽ thấy **`md-gpu-intensive` được scale up**.
      * **Kết luận Lab 3:** Chiến lược `priority` hoạt động chính xác, giúp tối ưu chi phí bằng cách ưu tiên dùng loại tài nguyên rẻ hơn trước, nhưng vẫn linh hoạt scale up loại tài nguyên đắt tiền khi thực sự cần thiết.

Hy vọng bài nghiên cứu và các kịch bản lab này sẽ giúp bạn hiểu sâu và vận dụng thành công Cluster Autoscaler với Cluster API. Chúc bạn thành công\!