### 1. Giới thiệu

- Etcd: là 1 cơ sở dữ liệu key-value phân tán, tạo bởi CoreO, nay được quản lý bởi Cloud Native Computing Foundation. Nó đóng vai trò là xương sống của nhiều hệ thống phân tán quy mô lớn, cung cấp một cách đáng tin cậy để lưu trữ dữ liệu trên 1 cụm máy chủ.

- `Etcd` hoạt động trên nhiều hệ điều hành: Linux, BSD và OS X.

- `Etcd` có các thuộc tính sau:

	* Fully replicated: đảm bảo việc lưu trữ dữ liệu đồng nhất trên tất cả các node trong cụm cluster.
	* Đảm bảo tính sẵn sàng cao
	* Nhất quán
	* Đơn giản
	* Bảo mật
	* Nhanh: Benchmark đạt 10000 writes per second.
	* Độ tin cậy cao
	
### 2. Cơ chế hoạt động của Etcd:

- Để hiểu cách `etcd` hoạt động, ta cần xác định 3 khái niệm chính trong `etcd`: `leaders`, `elections` và `terms`. Dựa trên thuật toán Raft, trong 1 cụm cluster etcd, các node sẽ thực hiện bầu cử để chọn ra 1 `leaders`.

- `Leaders` sẽ chịu trách nhiệm xử lý tất cả các request từ client và cần sự đồng thuận của các node trong cụm cluster. Các request không yêu cầu sự đồng thuận như việc đọc dữ liệu, có thể sẽ được xử lý bởi bất kì node nào trong cụm cluster. `Leaders` có trách nhiệm chấp nhận các thay đổi mới, và sao chép các thông tin này vào các node trong cụm cluster, sau đó cam kết các thay đổi này sau khi các node xác minh nhận. Trong mỗi cụm cluster `etcd` sẽ chỉ có duy nhất 1 `leader` tại bất kì thời điểm nào.

- Nếu node `leader` chết, hoặc trong 1 khoảng thời gian mà không còn phản hồi, các nút còn lại sẽ bắt đầu 1 cuộc bầu cử mới (`elections`) để chọn ra 1 `leader` mới. Mỗi node duy trì 1 bộ đếm thời gian bầu cử ngẫu nhiên thể hiện lượng thời gian mà node đó sẽ chờ trước khi kêu gọi 1 cuộc bầu cử mới và chọn chính nó làm ứng cử viên làm `leader`.

- Nếu 1 node trong cụm cluster không nhận được phản hồi từ node `leader` trước khi hết thời gian, node đó sẽ bắt đầu 1 nhiệm kỳ mới (`term`), đánh dấu chính nó là ứng cử viên làm `leader` và yêu cầu bỏ phiếu từ các node khác. Mỗi node bỏ phiếu cho ứng cử viên đầu tiên yêu cầu bỏ phiếu. Nếu 1 ứng cử viên nhận được 1 phiếu bầu từ phần lớn các node trong cụm, nó sẽ trở thành `leader` mới. Vì thời gian chờ bầu cử trên mỗi node là khác nhau, nên ứng cử viên đầu tiên thường trở thành `leader` mới. Tuy nhiên, nếu nhiều ứng cử viên tồn tại và nhận được cùng 1 số phiếu, thì cuộc bầu cử hiện tại sẽ kết thúc mà không có `leader` và 1 cuộc bầu cử mới sẽ bắt đầu với thời gian bầu cử ngẫu nhiên mới.

- *Như đã đề cập ở trên, mọi thay đổi phải được chuyển đến node `leader`. Thay vì chấp nhận và cam kết thay đổi ngay lập tức, `etcd` sử dụng thuật toán Raft để đảm bảo rằng phần lớn các node trong cụm cluster đều đồng ý về thay đổi. Node `leader` gửi giá trị mới được đề xuất cho mỗi node trong cụm. Các node sau đó gửi 1 thông báo xác nhận là đã nhận giá trị mới. Nếu phần lớn các node xác nhận đã nhận, `leader` cam kết giá trị mới và thông báo cho mỗi node rằng giá trị được cam kết với nhật ký. Điều này có nghĩa là mỗi thay đổi đòi hỏi một đại biểu từ các node trong cụm để được cam kết.

### 3. Etcd trong Kubernetes

- Công việc của `etcd` trong k8s là lưu trữ dữ liệu quan trọng cho các hệ thống phân tán. `Etcd` được biết đến như là kho dữ liệu chính của k8s, được sử dụng để lưu trữ dữ liệu cấu hình, trạng thái và metadata trong k8s. Vì k8s thường chạy trên 1 cụm nhiều máy khác nhau, nên nó là 1 hệ thống phân tán yêu cầu kho dữ liệu phân tán như `etcd`.

- `Etcd` giúp dễ dàng lưu trữ dữ liệu trên 1 cụm k8s và theo dõi các thay đổi, cho phép mọi node master từ cụm k8s đọc và ghi dữ liệu. Chức năng `watch` của `Etcd` được k8s sử dụng để theo dõi các thay đổi về trạng thái thực tế hoặc trạng thái mong muốn của hệ thống. Nếu chúng khác nhau, k8s thực hiện các thay đổi để dung hòa 2 trạng thái này.

- Mỗi lần thực hiện đọc bởi lệnh `kubectl`, dữ liệu sẽ được lấy trong `etcd`, mọi thay đổi được thực hiện ( `kubectl apply`) sẽ tạo hoặc cập nhập các mục vào trong `etcd`.

### 4. Một số khuyến nghị khi triển khai cluster etcd trong thực tế.

- Khi chạy các cụm etcd trong production, một số điểm cần lưu ý:

	* Vì `etcd` ghi dữ liệu vào đĩa, ssd được khuyến khích sử dụng
	
	* Nên sử dụng số lượng node trong cụm cluster etcd là số lẻ

	* Vì lí do hiệu suất, các cụm cluster etcd thường không nên có nhiều hơn bảy nút.
	
- Chi tiết hơn các bạn có thể tham khảo trực tiếp ở ***https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md***

### 5. Lab sử dụng kubeadm để triển khai cụm cluster etcd

- Mô hình:

- 3 server ubuntu 18.04
	
	+ 172.16.68.218 etcd1
	
	+ 172.16.68.219 etcd2
	
	+ 172.16.68.220 etcd3

### Thực hiện trên cả 3 node:

- Disable swap:
  
  ```
  swapoff -a
  ```

- Install Kubeadm Packages:
  
  ```
  apt update -y && apt upgrade -y
  
  apt-get install apt-transport-https curl -y
  
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
  
  vim /etc/apt/sources.list.d/kubernetes.list
  deb http://apt.kubernetes.io/ kubernetes-xenial main
  
  apt update
  apt install -y kubeadm kubelet kubectl
  ```
  
- Install Docker:

  ```
  apt-get install docker.io -y
  ```

- Configure the kubelet to be a service manager for etcd:

  ```
  cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
  [Service]
  ExecStart=
  ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true
  Restart=always
  EOF
  
  systemctl daemon-reload
  systemctl restart kubelet
  ```

### Thực hiện trên node etcd-1:

- Create configuration files for kubeadm:

```
export HOST0=172.16.68.218
export HOST1=172.16.68.219
export HOST2=172.16.68.220

# Create temp directories to store files that will end up on other hosts.
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

ETCDHOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=("infra0" "infra1" "infra2")

for i in "${!ETCDHOSTS[@]}"; do
HOST=${ETCDHOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
apiVersion: "kubeadm.k8s.io/v1beta1"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${ETCDHOSTS[0]}:2380,${NAMES[1]}=https://${ETCDHOSTS[1]}:2380,${NAMES[2]}=https://${ETCDHOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done  
```

- Tạo certificate authority:

```
etcd1# kubeadm init phase certs etcd-ca
```

- Tạo certificates cho tất cả các etcd node ( thực hiện trên node etcd1):

```
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/

# cleanup non-reusable certificates
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No need to move the certs because they are for HOST0

# clean up certs that should not be copied off this host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

- Thực hiện copy certificates and kubeadm configs tới 2 node etcd-2, etcd-3:

  ```
  etcd1# scp -r /tmp/${HOST1}/* ${HOST1}:
  etcd1# scp -r /tmp/${HOST2}/* ${HOST2}:
  ```

- Truy cập vào etcd-2 và thực hiện:

  ```
  etcd2# cd /root
  etcd2# mv pki /etc/kubernetes/
  ```

- Truy cập vào etcd-3 và thực hiện:

  ```
  etcd3# cd /root
  etcd3# mv pki /etc/kubernetes/
  ```

- Trên tất cả các node etcd, thực hiện tạo static manifest for etcd cluster với lệnh kubeadm:

  ```
  etcd1# kubeadm init phase etcd local --config=/tmp/172.16.68.218/kubeadmcfg.yaml

  etcd2# kubeadm init phase etcd local --config=/root/kubeadmcfg.yaml

  etcd3# kubeadm init phase etcd local --config=/root/kubeadmcfg.yaml
  ```

- Check cụm cluster etcd vừa khởi tạo (Lưu ý: k8s từ phiên bản 1.13 trở lên sử dụng ETCDCTL_API version 3):

  ```
  docker exec -it fc614b8ace41 /bin/sh
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.218:2379,https://172.16.68.219:2379,https://172.16.68.220:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key --write-out=table endpoint status

  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  |          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  | https://172.16.68.218:2379 | 42f932856eb887aa |  3.3.10 |  2.3 MB |     false |        80 |     118959 |
  | https://172.16.68.219:2379 | a41333865e3c22ac |  3.3.10 |  2.3 MB |      true |        80 |     118959 |
  | https://172.16.68.220:2379 | 7fc7860440987164 |  3.3.10 |  2.5 MB |     false |        80 |     118958 |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+	

  ```

- List các member trong cụm cluster etcd:

  ```
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.218:2379,https://172.16.68.219:2379,https://172.16.68.220:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key --write-out=table member list

  +------------------+---------+--------+----------------------------+----------------------------+
  |        ID        | STATUS  |  NAME  |         PEER ADDRS         |        CLIENT ADDRS        |
  +------------------+---------+--------+----------------------------+----------------------------+
  | 42f932856eb887aa | started | infra0 | https://172.16.68.218:2380 | https://172.16.68.218:2379 |
  | 7fc7860440987164 | started | infra2 | https://172.16.68.220:2380 | https://172.16.68.220:2379 |
  | a41333865e3c22ac | started | infra1 | https://172.16.68.219:2380 | https://172.16.68.219:2379 |
  +------------------+---------+--------+----------------------------+----------------------------+
  ```

- List key on k8s:

  ```
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.218:2379,https://172.16.68.219:2379,https://172.16.68.220:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only
  ```

- Add 1 node mới vào trong cụm etcd:

  - Trên node master trong cụm etcd ta thực hiện add member mới:

  ```
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.220:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member add infra3 --peer-urls=https://172.16.68.213:2380
  ```
  
  - Trên node mới ( 172.16.68.213):
  
  - Bước 1: copy 2 file ca.crt và ca.key từ etcd1 sang
  
  - Bước 2: Tạo file config `kubeadmcfg.yaml`:
    
	```
	apiVersion: "kubeadm.k8s.io/v1beta1"
    kind: ClusterConfiguration
    etcd:
        local:
            serverCertSANs:
            - "172.16.68.213"
            peerCertSANs:
            - "172.16.68.213"
            extraArgs:
                initial-cluster: infra0=https://172.16.68.218:2380,infra2=https://172.16.68.220:2380,infra3=https://172.16.68.213:2380,infra1=https://172.16.68.219:2380
                initial-cluster-state: existing
                name: infra3
                listen-peer-urls: https://172.16.68.213:2380
                listen-client-urls: https://172.16.68.213:2379
                advertise-client-urls: https://172.16.68.213:2379
                initial-advertise-peer-urls: https://172.16.68.213:2380
	```
	
  - Bước 3: Tạo certificates trên node mới:
    
	```
	kubeadm init phase certs etcd-server --config=kubeadmcfg.yaml
	
	kubeadm init phase certs etcd-peer --config=kubeadmcfg.yaml
	
	kubeadm init phase certs etcd-healthcheck-client --config=kubeadmcfg.yaml
	
	kubeadm init phase certs apiserver-etcd-client --config=kubeadmcfg.yaml

	```
   - Bước 4: Generate a static manifest với kubeadm:
   
   ```
   kubeadm init phase etcd local --config=/root/kubeadmcfg.yaml
   ```
   
- Backup etcd: 

  ```
  ETCDCTL_API=3 etcdctl --cert-file /etc/kubernetes/pki/etcd/peer.crt --key-file /etc/kubernetes/pki/etcd/peer.key --ca-file /etc/kubernetes/pki/etcd/ca.crt --endpoints https://172.16.68.219:2379 snapshot save /data/backup/etcd/snapshot.db
  ```

### 6. Tham khảo:

- https://kubernetes.io/docs/setup/independent/setup-ha-etcd-with-kubeadm/
- https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md



