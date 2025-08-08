# Lab: Cluster Autoscaler
## I. Cài đặt môi trường
### **⭐ Phiên bản Tương thích được Sử dụng trong Lab**

Để đảm bảo mọi thứ hoạt động trơn tru, chúng ta sẽ thống nhất sử dụng các phiên bản sau:

  * **Kubernetes (cho cả management & workload cluster):** `v1.33.0`
  * **Cluster API (CAPI):** 
  ```sh
  $ clusterctl version
clusterctl version: &version.Info{Major:"1", Minor:"10", GitVersion:"v1.10.4", GitCommit:"737edc88faed28b0bd81f6768ebe2ea276fdbe3a", GitTreeState:"clean", BuildDate:"2025-07-15T16:15:17Z", GoVersion:"go1.23.10", Compiler:"gc", Platform:"linux/amd64"}
  ```
  * **Infrastructure Provider (Docker - CAPD)**
  * **Cluster Autoscaler (CA):** `v1.33.0` (tương thích với Kubernetes v1.33.x)

-----

### **Pha 0: Chuẩn bị Môi trường**

1.  **Docker:** Cần thiết để chạy các node Kubernetes dưới dạng container. Hãy cài đặt Docker Desktop hoặc Docker Engine.
2.  **kubectl:** Công cụ dòng lệnh để tương tác với cluster Kubernetes.
3.  **Kind (Kubernetes in Docker):** Dùng để tạo management cluster một cách nhanh chóng.
    ```bash
    # For AMD64 / x86_64
    [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
    # For ARM64
    [ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-arm64
    chmod +x ./kind
    sudo mv ./kind /usr/local/bin/kind
    ```
4.  **clusterctl:** Công cụ dòng lệnh của Cluster API.
    ```sh
    curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.10.4/clusterctl-linux-amd64 -o clusterctl

    sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
    ```

-----

### **Pha 1: Khởi tạo Management Cluster (Cụm Quản lý)**

Cụm này sẽ không chạy workload của bạn, mà chỉ dùng để quản lý các cluster con khác.

1.  **Tạo Kind Cluster:**
  ```bash
  cat > kind-cluster-with-extramounts.yaml <<EOF
  kind: Cluster
  apiVersion: kind.x-k8s.io/v1alpha4
  networking:
    ipFamily: dual
  nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
  EOF

  kind create cluster --name autoscaler --config kind-cluster-with-extramounts.yaml
  ```

2.  **Xác nhận Cluster đã sẵn sàng:**
  ```bash
  kubectl cluster-info --context kind-capi-management
  ```
  Bạn sẽ thấy thông tin về control plane của cluster vừa tạo.

-----

### **Pha 2: Cài đặt Cluster API vào Management Cluster**

1.  **Thực hiện `clusterctl init`:**
    Lệnh này sẽ cài đặt các CRD (Custom Resource Definitions) và các controller cần thiết cho CAPI và provider Docker.
    ```bash
    # Enable the experimental Cluster topology feature.
    export CLUSTER_TOPOLOGY=true

    # Initialize the management cluster
    clusterctl init --infrastructure docker
    ```
- **Logs sẽ có dạng như sau:**
    ```bash
    Installing cert-manager version="v1.18.1"
    Waiting for cert-manager to be available...
    [KubeAPIWarningLogger] spec.privateKey.rotationPolicy: In cert-manager >= v1.18.0, the default value changed from `Never` to `Always`.
    Installing provider="cluster-api" version="v1.10.4" targetNamespace="capi-system"
    [KubeAPIWarningLogger] spec.privateKey.rotationPolicy: In cert-manager >= v1.18.0, the default value changed from `Never` to `Always`.
    Installing provider="bootstrap-kubeadm" version="v1.10.4" targetNamespace="capi-kubeadm-bootstrap-system"
    [KubeAPIWarningLogger] spec.privateKey.rotationPolicy: In cert-manager >= v1.18.0, the default value changed from `Never` to `Always`.
    Installing provider="control-plane-kubeadm" version="v1.10.4" targetNamespace="capi-kubeadm-control-plane-system"
    [KubeAPIWarningLogger] spec.privateKey.rotationPolicy: In cert-manager >= v1.18.0, the default value changed from `Never` to `Always`.
    Installing provider="infrastructure-docker" version="v1.10.4" targetNamespace="capd-system"
    [KubeAPIWarningLogger] spec.privateKey.rotationPolicy: In cert-manager >= v1.18.0, the default value changed from `Never` to `Always`.

    Your management cluster has been initialized successfully!

    You can now create your first workload cluster by running the following:

      clusterctl generate cluster [name] --kubernetes-version [version] | kubectl apply -f -
    ```
-----

### **Pha 3: Tạo Manifest Cluster (Cụm con)**

Đây là cluster mà ứng dụng sẽ chạy và cũng là nơi Cluster Autoscaler sẽ thực hiện co giãn.

1.  **Tạo tệp Manifest cho Workload Cluster:**
  Sử dụng `clusterctl generate` để tạo một tệp manifest mẫu.

    ```bash
    clusterctl generate cluster capi-quickstart --flavor development \
    --kubernetes-version v1.33.0 \
    --control-plane-machine-count=1 \
    --worker-machine-count=1 \
    > capi-quickstart.yaml

    ```
2. **Apply manifest vào cụm**
  ```bash
  kubectl apply -f capi-quickstart.yaml
  ```
  - Logs sẽ có dạng như sau:
  ```bash
  dockerclustertemplate.infrastructure.cluster.x-k8s.io/quick-start-cluster created
  kubeadmcontrolplanetemplate.controlplane.cluster.x-k8s.io/quick-start-control-plane unchanged
  dockermachinetemplate.infrastructure.cluster.x-k8s.io/quick-start-control-plane created
  dockermachinetemplate.infrastructure.cluster.x-k8s.io/quick-start-default-worker-machinetemplate created
  dockermachinepooltemplate.infrastructure.cluster.x-k8s.io/quick-start-default-worker-machinepooltemplate unchanged
  kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/quick-start-default-worker-bootstraptemplate unchanged
  cluster.cluster.x-k8s.io/capi-quickstart configured
  ```

3. ** Kiểm tra cluster được tạo trong CRDs**
  ```bash
  kubectl get cluster
  ```
  - Kiểm tra các resource được tạo cùng cluster
  ```bash
  $ clusterctl describe cluster capi-quickstart
  NAME                                                           READY  SEVERITY  REASON                  
         SINCE  MESSAGE
Cluster/capi-quickstart                                        False  Warning   ScalingUp               
         19s    Scaling up control plane to 1 replicas (actual 0)
├─ClusterInfrastructure - DockerCluster/capi-quickstart-68qlr  True                                     
         19s
├─ControlPlane - KubeadmControlPlane/capi-quickstart-xztkh     False  Warning   ScalingUp               
         19s    Scaling up control plane to 1 replicas (actual 0)
│ └─Machine/capi-quickstart-xztkh-brdkf                        False  Info      Bootstrapping           
         16s    1 of 2 completed
└─Workers                                                                                        

  └─MachineDeployment/capi-quickstart-md-0-m6ncw               False  Warning   WaitingForAvailableMachines      19s    Minimum availability requires 1 replicas, current 0 available
    └─Machine/capi-quickstart-md-0-m6ncw-2lbm7-hlxb9           False  Info      WaitingForControlPlaneAvailable  3s     0 of 2 completed
  ```
  - Export kubeconfig
  ```bash
  kind get kubeconfig --name capi-quickstart > capi-quickstart.kubeconfig
  ```
  Khi này, các Machine sẽ ở trạng thái `Running` nhưng MachineSet và MachineDeployment vẫn sẽ không `Ready` cho đến khi cài CNI.

4. ** Cài CNI Calico**
  ```bash
  kubectl --kubeconfig=./capi-quickstart.kubeconfig \
  apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
  ```
  - Check các pod ready
  ```
  kubectl --kubeconfig=./capi-quickstart.kubeconfig get pod -n kube-system
  ```

## II. Thử nghiệm các tham số Cluster Autoscaler với CAPI
### 1. Annotation `cluster.x.k8s/cluster-api-autoscaler-node-group-min-size`
- (Annotation trên MD/MS) Số lượng node tối thiểu cho node group này. CA sẽ không bao giờ scale down thấp hơn con số này. Phụ thuộc vào số lượng node người dùng định nghĩa => Đề xuất không Lab
### 2. Annotation `cluster.x.k8s/cluster-api-autoscaler-node-group-max-size`
- (Annotation trên MD/MS) Số lượng node tối đa cho node group này. CA sẽ không bao giờ scale up vượt quá con số này. Phụ thuộc vào số lượng node người dùng định nghĩa => Đề xuất không Lab.
### 3. Parameter `scale-down-utilization-threshold`

### 4. Parameter `scaler-down-unneed-time`

### 5. Parameter `max-node-provision-time`
- Thời gian 
### 6. Parameter `skip-nodes-with-local-storage`

### 7. Parameter `skip-nodes-with-system-pod`

### 8. Parameter `scale-down-delay-after-add`

### 9. Parameter `expander`

### 10. Annotation `cluster.autoscaler.kubernetes.io/priority`

### 11. Parameter `cloud-provider`

### 12. Parameter `clusterapi-mode`

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace default \
  --set cloudProvider=clusterapi \
  --set autoDiscovery.clusterName=capi-quickstart \
  --set extraArgs.expander=least-waste \
  --set rbac.create=true \
  --set extraArgs.kubeconfig=/etc/kubernetes/value \
  --set volumeMounts[0].name=kubeconfig \
  --set volumeMounts[0].mountPath=/etc/kubernetes \
  --set volumes[0].name=kubeconfig \
  --set volumes[0].secret.secretName=capi-quickstart-kubeconfig

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace default \
  --set cloudProvider=clusterapi \
  --set autoDiscovery.clusterName=capi-quickstart \
  --set extraArgs.expander=least-waste \
  --set rbac.create=true \
  --set extraArgs.kubeconfig=/etc/workload/value \
  --set volumeMounts[0].name=kubeconfig \
  --set volumeMounts[0].mountPath=/etc/workload \
  --set volumes[0].name=kubeconfig \
  --set volumes[0].secret.secretName=capi-quickstart-kubeconfig 
