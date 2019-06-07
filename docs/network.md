### 1. Giới thiệu:

- Một trong những phần phức tạp và quan trọng nhất khi tìm hiểu về k8s là phần Network. Trong phần này, tôi sẽ giới thiệu về network của 1 pod, cách thức kết nối giữa các pods trong 1 cụm cluster k8s.

### 2. K8s Networking Model

- Về cốt lõi, Network trong k8s có 1 triết lí thiết kế cơ bản quan trọng là: **Mỗi Pod có 1 IP duy nhất**.

- IP của 1 pod được chia sẻ bởi tất cả các container trong Pod này, và mặc định nó có thể kết nối từ tất cả các Pod khác trong 1 cụm k8s. 

- Bạn có bao giờ để ý thấy 1 số container `pause` đang chạy trên tất cả các node trong k8s? 
  
  ![alt](../images/containerpause.png)

- Chúng được gọi là các `sandbox containers`, có 1 nhiệm vụ duy nhất là lưu giữ `network namespace` (netns) được chia sẻ bởi tất cả các container trong 1 pod. Với cơ chế này, trong 1 Pod khi 1 container chết hoặc 1 container mới được tạo thì IP Pod sẽ không bị thay đổi.

### 3. Việc kết nối giữa các Pod trên 1 node

 ![alt](../images/rootnetworkns.png)
 
- Trên mỗi node trong cụm k8s (chạy hệ điều hành linux) sẽ có 1 `root network namespace` ( root netns). Interface chính `eth0` nằm trong `root network namespace` này.

- Tương tự, mỗi pod sẽ có netns riêng, với 1 virtual ethernet (veth) kết nối nó tới `root netns`. Về cơ bản, `veth` là 1 đường ống với 2 đầu, 1 đầu được gắn với root netns và đầu còn lại gắn với pod netns.

  ![alt](../images/podnetns1.png)  

- Bạn có thể liệt kê tất cả các network interface trên node với lệnh `ifconfig` hoặc `ip a`.

- Trong k8s, sẽ sử dụng 1 bridge `cbr0` để các pod trên 1 node nói chuyện với nhau.

  ![alt](../images/podnetns2.png)
  
- Bạn có thể list các bridge bằng lệnh `brctl show`.

- Traffic flows:
  
  ![alt](../images/podnetns.gif)

  1. Gói tin từ pod1 đi vào root netns tại `vethxx`.
  
  2. Gói tin được chuyển tới bridge `cbr0`, để phát hiện ra đích đến của gói tin bằng cách sử dụng `ARP request`, nói rằng pod nào có IP này? 
  
  3. `vethyyy` nói rằng nó có IP đó, vì vậy bridge `cbr0` sẽ biết nơi chuyển tiếp gói tin.
  
  4. Gói tin được gửi đến `vethyyy`, và gửi đến pod2.
  
### 4. Việc kết nối giữa các pod trên các node khác nhau

- Như đề cập ở trên, các pod cũng cần kết nối được trên tât cả các node trong 1 cụm k8s. Để làm được điều này, có thể sử dụng L2 (ARP across nodes), L3 (IP routing giữa các node) hoặc overlay networks. Một số plugins network được sử dụng trong k8s: flannel, calico, weave, cannal...

- Mỗi node trong cụm k8s sẽ được gán 1 `CIDR block` duy nhất (1 dải địa chỉ IP) để cấp cho các pods, vì vậy mỗi pod có 1 IP duy nhất và không bị conflict với các pods trên 1 node khác trong cụm k8s.

- Phần tiếp theo, mình sẽ nói về một số kiểu network được sử dụng trong k8s: `flannel`, `calico`, `weave`.

### 5. Flannel - overlay networks

- Mặc định trong k8s, Overlay networks không được yêu cầu, tuy nhiên, nó sẽ hữu ích trong 1 số tình huống cụ thể: như khi không đủ lượng IP để cấp cho các pods, hoặc mạng không thể xử lý được các routes bổ sung. Một trường hợp thường thấy là khi có giới hạn số hops mà bảng định tuyến của các nhà cung cấp cloud có thể xử lý. Ví dụ: Để đảm bảo hiệu suất mạng, bảng định tuyến AWS chỉ hỗ trợ tối đa 50 hops. Vì vậy nếu chúng ta có hơn 50 nodes trong cụm k8s, bảng định tuyến AWS sẽ không đủ. Trong trường hợp như vậy, sử dụng overlay networks sẽ có hiệu quả.

- Về cơ bản, cơ chế overlay networks sẽ đóng gói `packet-in-packet` đi qua underlay networks (hạ tầng mạng vật lí cơ bản).

- Traffic flows trong overlay networks, cụ thể ở đây là `flannel` sử dụng backend `VXLAN`:

  ![alt](../images/flannel.gif)

- Luồng gói tin gửi từ pod1 đến pod4 (trên node khác):
 
	* 1. Gói tin từ `pod1` netns tại `eth0` trên pod 1 được gửi vào root netns tại `vethxxx`.
  
  	* 2. Gói tin chuyển tới `cbr0`, và gửi yêu cầu ARP để tìm destination của gói tin.
  
  	* 3a. Vì không pod nào trên node này có địa chỉ IP cho `pod4`, nên bridge `cbr0` gửi nó tới `flannel0`. Vì bảng định tuyến của node được cấu hình với `flannel0` làm mục tiêu cho range IP pod. Các thông tin cấu hình này được lưu ở `etcd`.
  
  	* 3b. Khi tiến trình `flanneld` daemon nói chuyện với `kubernetes apiserver` hoặc `etcd`, nó sẽ biết về tất cả các IP của pod và các dải IP của pod đang nằm trên node nào. Vì vậy, `flannel` sẽ ánh xạ các IP pod với IP của các node. `flannel0` lấy gói tin này và đóng gói vào trong gói UDP với các header bổ sung thay đổi IP nguồn và IP đích đến các node tương ứng và gửi nó đến 1 port đặc biệt `vxlan` (thường là 8472).
  
    ![alt](../images/packetvxlan.png)
	
	* 3c. Gói tin này được đóng gói được gửi qua eth0.
	
	* 4. Gói tin rời khỏi node với sourceIP là IP node1 và destIP là IP node2.
	
	* 5. Bảng định tuyến của mạng vật lí đã biết cách định tuyến giữa các node, do đó nó gửi gói tin đến node2.
	
	* 6a. Gói tin này đến `eth0` của node2. Do port là port đặc biệt `vxlan`, kernel sẽ gửi gói tin đến `flannel0`.
	
	* 6b. `flannel0` sẽ thực hiện de-capsulates và gửi nó vào trong root netns.
	
	* 6c. `Vì IP forwarding được enabled`, kernel sẽ chuyển tiếp gói tin tới `cbr0`.
	
	* 7. Bridge `cbr0` sẽ lấy gói tin và thực hiện 1 request ARP và phát hiện ra răng IP này thuộc về `vethyyy`.
	
	* 8. Gói tin sẽ đi qua `vethyyy` và được gửi tới pod4.

- Với Flannel sử dụng công nghệ `vxlan`: nhanh nhưng không có mã hóa giữa các node, nó phù hợp với mô hình private.
	
- Flannel không cung cấp triển khai NetworkPolicy resource. Vậy NetworkPolicy là gì ?

### 6. NetworkPolicy trong k8s

- Mặc định trong 1 cụm cluster không sử dụng NetworkPolicy, tất cả các Pod sẽ nói chuyện được với nhau. Ví dụ:

  ![alt](../images/networkpolicy1.png)
  
- Trước khi cấu hình Networkpolicy, các pod web sẽ nói chuyện được với các db pod. Để tăng tính bảo mật, mình sẽ cấu hình 1 NetworkPolicy chỉ cho phép các pod backend mới có thể kết nối được đến pod database, các pod web sẽ không được phép kết nối tới các pod db.

  ```
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: backend-access-ingress
  spec:
    podSelector:
      matchLabels:
        app: myapp
        role: backend
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: myapp
            role: web
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: db-access-ingress
  spec:
    podSelector:
      matchLabels:
        app: myapp
        role: db
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: myapp
            role: backend
  ```

- Sau khi config NetworkPolicy: 
  
  ![alt](../images/networkpolicy2.png)
  

  
  
  


  
  