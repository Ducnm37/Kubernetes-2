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
	
	+ 172.16.68.210 etcd0
	
	+ 172.16.68.211 etcd1
	
	+ 172.16.68.212 etcd2

### Install cfssl (Cloudflare ssl) thực hiện trên node etcd0:

- Install and config:

  ```
  1. wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  
  2. wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  
  3. chmod +x cfssl*
  
  4. mv cfssl_linux-amd64 /usr/local/bin/cfssl
  
  5. mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
  ```

- Verify the installation:
  
  ```
  cfssl version
  ```
  
### Generating the TLS certificates:

- Tạo file cấu hình CA (certificate authority):

  ```
  vim ca-config.json
  {
    "signing": {
      "default": {
        "expiry": "8760h"
      },
      "profiles": {
        "kubernetes": {
          "usages": ["signing", "key encipherment", "server auth", "client auth"],
          "expiry": "8760h"
        }
      }
    }
  }
  ```
  
- Create the certificate authority signing request configuration file.
  
  ```
  vim ca-csr.json
  {
    "CN": "Kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
    {
      "C": "IE",
      "L": "VNPT",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "VNPT Co."
    }
   ]
  }
  ```

- Generate the certificate authority certificate and private key.

  ```
  cfssl gencert -initca ca-csr.json | cfssljson -bare ca
  ```

- Verify that the ca-key.pem and the ca.pem were generated.
  
  ```
  ll
  ```

### Creating the certificate for the Etcd cluster

- Create the certificate signing request configuration file.

  ```
  vim kubernetes-csr.json
  {
    "CN": "kubernetes",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
    {
      "C": "IE",
      "L": "VNPT",
      "O": "Kubernetes",
      "OU": "Kubernetes",
      "ST": "VNPT Co."
    }
   ]
  }
  ```
- Generate the certificate and private key.

  ```
  cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=172.16.68.210,172.16.68.211,172.16.68.212,172.16.68.215,127.0.0.1,kubernetes.default \
  -profile=kubernetes kubernetes-csr.json | \
  cfssljson -bare kubernetes
  ```
  
- Verify that the kubernetes-key.pem and the kubernetes.pem file were generated.

  ```
  ll
  ```

- Copy the certificate to each nodes on etcd cluster:

  ```
  scp ca.pem kubernetes.pem kubernetes-key.pem root@172.16.68.211:~
  
  scp ca.pem kubernetes.pem kubernetes-key.pem root@172.16.68.212:~
  ```

### Install and config Etcd cluster:

##### Trên node etcd0:

- 1.Create a configuration directory for Etcd.
  
  ```
  mkdir /etc/etcd /var/lib/etcd
  ```

- 2.Move the certificates to the configuration directory.

  ```
  mv ~/ca.pem ~/kubernetes.pem ~/kubernetes-key.pem /etc/etcd
  ```
  
- 3.Download the etcd binaries.

  ```
  wget https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz
  ```
  
- 4.Extract the etcd archive.

  ```
  tar xvzf etcd-v3.3.9-linux-amd64.tar.gz
  ```
  
- 5.Move the etcd binaries to /usr/local/bin.

  ```
  mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
  ```

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd0 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.210:2380 \
    --listen-peer-urls https://172.16.68.210:2380 \
    --listen-client-urls https://172.16.68.210:2379 \
    --advertise-client-urls https://172.16.68.210:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```
  
##### Trên node etcd1:

- Thực hiện các bước 1 -> 5 tương tự như trên node etcd0

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd1 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.211:2380 \
    --listen-peer-urls https://172.16.68.211:2380 \
    --listen-client-urls https://172.16.68.211:2379 \
    --advertise-client-urls https://172.16.68.211:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```

##### Trên node etcd2:

- Thực hiện các bước 1 -> 5 tương tự như trên node etcd0

- 6.Create an etcd systemd unit file.

  ```
  vim /etc/systemd/system/etcd.service
  
  [Unit]
  Description=etcd
  Documentation=https://github.com/coreos

  [Service]
  ExecStart=/usr/local/bin/etcd \
    --name etcd2 \
    --cert-file=/etc/etcd/kubernetes.pem \
    --key-file=/etc/etcd/kubernetes-key.pem \
    --peer-cert-file=/etc/etcd/kubernetes.pem \
    --peer-key-file=/etc/etcd/kubernetes-key.pem \
    --trusted-ca-file=/etc/etcd/ca.pem \
    --peer-trusted-ca-file=/etc/etcd/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --initial-advertise-peer-urls https://172.16.68.212:2380 \
    --listen-peer-urls https://172.16.68.212:2380 \
    --listen-client-urls https://172.16.68.212:2379 \
    --advertise-client-urls https://172.16.68.212:2379 \
    --initial-cluster-token etcd-cluster-0 \
    --initial-cluster etcd0=https://172.16.68.210:2380,etcd1=https://172.16.68.211:2380,etcd2=https://172.16.68.212:2380 \
    --initial-cluster-state new \
    --data-dir=/var/lib/etcd
  Restart=on-failure
  RestartSec=5

  [Install]
  WantedBy=multi-user.target
  ```

##### Trên cả 3 node etcd:

- Reload the daemon configuration.
  
  ```
  systemctl daemon-reload
  ```
  
- Enable etcd to start at boot time.
  
  ```
  systemctl enable etcd
  ```
  
- Start etcd.

  ```
  systemctl start etcd
  ```

- Verify etcd-cluster:
  
  ```
  ETCDCTL_API=3 etcdctl --write-out=table member list
  +------------------+---------+-------+----------------------------+----------------------------+
  |        ID        | STATUS  | NAME  |         PEER ADDRS         |        CLIENT ADDRS        |
  +------------------+---------+-------+----------------------------+----------------------------+
  | ab84f8691a2c742c | started | etcd0 | https://172.16.68.210:2380 | https://172.16.68.210:2379 |
  | e1a8c0925289a69d | started | etcd1 | https://172.16.68.211:2380 | https://172.16.68.211:2379 |
  | f6a2daf825ab64d5 | started | etcd2 | https://172.16.68.212:2380 | https://172.16.68.212:2379 |
  +------------------+---------+-------+----------------------------+----------------------------+

  
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.210:2379,https://172.16.68.211:2379,https://172.16.68.212:2379 --cacert=/etc/etcd/ca.pem --cert=/etc/etcd/kubernetes.pem --key=/etc/etcd/kubernetes-key.pem --write-out=table endpoint status
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  |          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  | https://172.16.68.210:2379 | ab84f8691a2c742c |   3.3.9 |  1.9 MB |      true |         6 |       8213 |
  | https://172.16.68.211:2379 | e1a8c0925289a69d |   3.3.9 |  1.9 MB |     false |         6 |       8213 |
  | https://172.16.68.212:2379 | f6a2daf825ab64d5 |   3.3.9 |  1.9 MB |     false |         6 |       8213 |
  +----------------------------+------------------+---------+---------+-----------+-----------+------------+
  ```
- List key on etcd ( sau khi kết nối tới node-master trong k8s )

  ```
  ETCDCTL_API=3 etcdctl --endpoints=https://172.16.68.210:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key get / --prefix --keys-only
  ```

### 6. Tham khảo:

- https://github.com/etcd-io/etcd/tree/master/Documentation
- https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md
- https://blog.inkubate.io/install-and-configure-a-multi-master-kubernetes-cluster-with-kubeadm/



