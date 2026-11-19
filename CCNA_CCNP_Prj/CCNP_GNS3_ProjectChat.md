Title: Check out this chat

URL Source: https://chatgpt.com/share/6afc2933-6118-83ec-a08f-7d286f95ecef

Markdown Content:
**Phase 0 hoàn thành khi bạn có một nền tảng lab ổn định: các thiết bị giữ được cấu hình sau khi khởi động lại, máy AUTO SSH được vào thiết bị qua OOB, và pyATS đọc/parse được trạng thái thực tế.**

Tôi hướng dẫn theo source mới, với phương án cụ thể:

*   Windows chạy GNS3 GUI và VMware Workstation.
*   GNS3 VM chạy các thiết bị IOL.
*   Một Ubuntu VM riêng tên **AUTO** chạy pyATS.
*   AUTO nối mạng OOB qua một mạng VMware Host-only riêng.

Đây là lựa chọn triển khai trong phạm vi thiết kế đã có. Nếu bạn đã cài một phần, hãy kiểm tra nghiệm thu rồi chuyển tới bước tiếp theo.

**Quy ước:** lệnh Cisco chạy trên console thiết bị; lệnh Bash chạy trong Ubuntu AUTO. Các giá trị `<...>` phải thay bằng thông tin của bạn.

* * *

**1. Hình dung topology hoàn chỉnh trước khi dựng**

Sơ đồ dưới dùng các đường nối đơn giản; đây là **topology đích của mục 4**, chưa yêu cầu cấu hình toàn bộ ở Phase 0.

mermaid

```
flowchart TB
    subgraph WAN["Internet mô phỏng"]
        IA["ISP-A · AS 65100"]
        IB["ISP-B · AS 65200"]
        IA ---|"203.0.113.16/30"| IB
    end

    subgraph HQ["HQ · AS 65010"]
        R1["HQ-R1 · Edge / Hub"]
        R2["HQ-R2 · Edge"]
        D1["HQ-DS1"]
        D2["HQ-DS2"]
        AC["HQ-ACC"]
        PC["HQ-PC · VLAN 10"]
        SRV["HQ-SRV · VLAN 20"]

        R1 ---|"10.255.1.0/30 · Area 0"| D1
        R1 ---|"10.255.1.4/30 · Area 0"| D2
        R2 ---|"10.255.1.8/30 · Area 0"| D1
        R2 ---|"10.255.1.12/30 · Area 0"| D2
        D1 ---|"10.255.1.16/30 · Area 10"| D2
        D1 ---|"Trunk VLAN 10,20"| AC
        D2 ---|"Trunk VLAN 10,20"| AC
        AC --- PC
        AC --- SRV
    end

    subgraph B1["Branch 1 · AS 65021"]
        BR1["BR1-R · Spoke"]
        SW1["BR1-SW"]
        P1["BR1-PC"]
        BR1 --- SW1 --- P1
    end

    subgraph B2["Branch 2 · AS 65022"]
        BR2["BR2-R · Spoke"]
        SW2["BR2-SW"]
        P2["BR2-PC"]
        BR2 --- SW2 --- P2
    end

    IA ---|"203.0.113.0/30"| R1
    IB ---|"203.0.113.4/30"| R2
    IA ---|"203.0.113.8/30"| BR1
    IB ---|"203.0.113.12/30"| BR2
```

Các kết nối logic triển khai ở phase sau:

*   iBGP: HQ-R1 ↔ HQ-R2, dùng Loopback0 và đi qua mạng HQ.
*   DMVPN: HQ-R1 là hub; BR1-R và BR2-R là spoke.
*   Tunnel100: `172.16.100.0/24`.
*   LAN: HQ Users `10.10.10.0/24`, HQ Servers `10.10.20.0/24`, BR1 `10.20.10.0/24`, BR2 `10.30.10.0/24`.

**Lưu ý khi nối dây:** đường DS1–DS2 trong thiết kế hiện tại là **link routed thuộc Area 10**. Hai đường DS–ACC mới là trunk L2.

Mạng OOB được bổ sung độc lập với các đường trên:

| Thành phần | Cách nối OOB |
| --- | --- |
| Router | Một cổng Ethernet riêng nối OOB-SW |
| Switch | Một access port VLAN quản lý riêng nối OOB-SW |
| AUTO | Card mạng OOB nối qua Cloud tới OOB-SW |
| HQ-SRV | Card quản lý riêng, nếu triển khai ngay |
| PC thử nghiệm | Không cần nối OOB |

**Nghiệm thu bước 1**

Bạn phân biệt được:

*   **Mạng nghiệp vụ:** các kết nối trong sơ đồ.
*   **Overlay:** tunnel logic triển khai sau.
*   **OOB:** đường quản lý và kiểm thử độc lập.

Phase 0 tập trung vào **OOB, SSH và assurance**, chưa cấu hình OSPF/BGP/DMVPN.

* * *

**2. Chuẩn bị phần mềm và kiểm tra tài nguyên**

Cần có:

| Thành phần | Mục đích |
| --- | --- |
| GNS3 GUI | Vẽ và điều khiển topology |
| GNS3 VM tương thích với phiên bản GUI | Chạy thiết bị IOL |
| VMware Workstation | Chạy GNS3 VM và AUTO |
| Image IOL L3 hợp lệ | Router HQ, Branch, ISP |
| Image IOL L2 có tính năng phù hợp | DS và access switch |
| Ubuntu Server 24.04 LTS x86-64 | Máy AUTO |

GNS3 hướng dẫn sử dụng GNS3 VM cho IOU/IOL trên Windows; bạn cần import image vào đúng compute server GNS3 VM.

**Phân bổ tham khảo**, cần điều chỉnh theo RAM thực tế:

| RAM máy Windows | Cách bắt đầu |
| --- | --- |
| 16 GB | GNS3 VM khoảng 6 GB, AUTO 2 GB; bật thiết bị theo nhóm |
| 32 GB trở lên | GNS3 VM 8–12 GB, AUTO 2–4 GB |
| CPU | Bắt đầu GNS3 VM 4 vCPU, AUTO 2 vCPU nếu máy đủ tài nguyên |

Không cần bật toàn bộ topology ngay từ đầu.

**Nghiệm thu bước 2**

*   GNS3 mở được.
*   GNS3 VM khởi động bình thường.
*   GNS3 nhận compute server của VM.
*   AUTO khởi động được.
*   Windows không bị thiếu RAM nghiêm trọng khi chạy hai VM.

Nếu GNS3 VM chưa kết nối được với GUI, xử lý xong trước khi import hàng loạt thiết bị.

* * *

**3. Import image và kiểm tra tính năng trước**

Trong GNS3, vào phần tương ứng:

text

```
Edit → Preferences → IOS on UNIX / IOU Devices → New
```

Tên menu có thể khác theo phiên bản. Chọn chạy trên **GNS3 VM**, rồi tạo template:

text

```
IOL-L3
IOL-L2
```

Với router và DS, bố trí đủ cổng để có:

text

```
Ethernet0/0 → Ethernet0/3
Ethernet1/0 → Ethernet1/3
```

Trước tiên chỉ tạo:

*   Một IOL L3.
*   Một IOL L2 dự kiến làm DS.

Khởi động và kiểm tra:

cisco

```
show version
show ip interface brief
```

Trên node dự kiến làm DS, kiểm tra bằng trợ giúp cú pháp:

cisco

```
configure terminal
interface Vlan99
 standby ?
exit
ip routing ?
end
```

Mục tiêu là xác định image có khả năng làm **multilayer switch với SVI, routing, HSRP và STP**. Chưa cần cấu hình HSRP.

**Không mặc định image mang nhãn “L2” hay “L3” sẽ có đủ tất cả tính năng.** Nếu image DS thiếu tính năng, cần chọn lại trước khi dựng phần campus. Yêu cầu license cũng phụ thuộc image và GNS3; không mặc định IOL luôn chạy không cần license.

**Nghiệm thu bước 3**

*   IOL L3 và IOL L2 khởi động tới CLI.
*   Thấy đủ interface dự kiến.
*   Có tên image và phiên bản IOS thực tế.
*   Image DS chấp nhận các nhóm lệnh cần cho SVI/HSRP/routing.
*   Không có lỗi image/license chưa xử lý.

Ghi kết quả vào ghi chú môi trường; chưa cần thay đổi kiến trúc nếu chưa thử image thực tế.

* * *

**4. Tạo project và bố trí thiết bị**

Tạo project:

text

```
CCNP-Multisite-Assurance
```

Đặt tên node và hostname thống nhất:

text

```
HQ-R1
HQ-R2
HQ-DS1
HQ-DS2
HQ-ACC
ISP-A
ISP-B
BR1-R
BR2-R
BR1-SW
BR2-SW
OOB-SW
```

Có thể thêm HQ-PC, HQ-SRV, BR1-PC và BR2-PC ngay để bố trí sơ đồ, rồi để chúng tắt.

Riêng **OOB-SW**, dùng IOL L2 có ít nhất 16 cổng Ethernet để đủ nối các node. Có thể bắt đầu bằng OOB-SW + HQ-R1, nghiệm thu rồi mở rộng.

Địa chỉ quản lý:

| Thiết bị | IP OOB | Cách gán |
| --- | --- | --- |
| AUTO | `192.168.99.10/24` | Card OOB của Ubuntu |
| HQ-R1 | `192.168.99.11/24` | Ethernet1/0 |
| HQ-R2 | `192.168.99.12/24` | Ethernet1/0 |
| HQ-DS1 | `192.168.99.13/24` | SVI VLAN 99 |
| HQ-DS2 | `192.168.99.14/24` | SVI VLAN 99 |
| HQ-ACC | `192.168.99.15/24` | SVI VLAN 99 |
| HQ-SRV | `192.168.99.16/24` | Card quản lý, nếu dựng ngay |
| BR1-R | `192.168.99.21/24` | Ethernet1/0 |
| BR2-R | `192.168.99.22/24` | Ethernet1/0 |
| BR1-SW | `192.168.99.23/24` | SVI VLAN 99 |
| BR2-SW | `192.168.99.24/24` | SVI VLAN 99 |
| ISP-A | `192.168.99.31/24` | Ethernet1/0 |
| ISP-B | `192.168.99.32/24` | Ethernet1/0 |
| OOB-SW | `192.168.99.2/24` | SVI VLAN 99 |

Các IP `.2`, `.23`, `.24` là bổ sung cho những địa chỉ quản lý còn để tùy chọn trong source.

**Nghiệm thu bước 4**

*   Tên node không trùng nhau.
*   Đã ghi rõ loại image cho từng vai trò.
*   Có bảng OOB thống nhất.
*   Các cổng dành cho OOB không bị dùng cho WAN/LAN.

* * *

**5. Nối AUTO với OOB qua VMware Host-only**

AUTO dùng hai card:

| Card | Kết nối | Vai trò |
| --- | --- | --- |
| NIC 1 | VMware NAT | Internet để cài phần mềm |
| NIC 2 | Host-only riêng | OOB `192.168.99.0/24` |

**5.1. Tạo mạng Host-only**

Trong VMware Virtual Network Editor:

1.   Chọn một VMnet chưa sử dụng, ví dụ `VMnet2`.
2.   Đặt loại **Host-only**.
3.   Đặt subnet `192.168.99.0/24`.
4.   Tắt VMware DHCP trên VMnet này.
5.   Kiểm tra IP card VMware phía Windows, tránh trùng bảng OOB; có thể dùng `.254`.

Nếu mạng thật/VPN của máy đang dùng `192.168.99.0/24`, phải giải quyết xung đột trước và cập nhật nhất quán bảng địa chỉ.

**5.2. Gắn card mạng**

Tắt VM trước khi sửa phần cứng:

*   AUTO: NIC 2 → Custom `VMnet2`.
*   GNS3 VM: thêm một NIC riêng → Custom `VMnet2`.
*   Giữ nguyên các NIC hiện có của GNS3 VM.

**5.3. Tạo Cloud trên đúng server**

Trong GNS3:

1.   Thêm node `Cloud-OOB`.
2.   Chọn chạy Cloud trên **GNS3 VM**.
3.   Trong Cloud, chọn interface Linux tương ứng NIC vừa thêm.
4.   Nối Cloud với OOB-SW.

Trong console GNS3 VM có thể kiểm tra:

bash

```
ip -br link
```

Đối chiếu MAC address với VMware để nhận diện NIC. **Không đoán rằng NIC mới luôn là****`eth2`****.**

Cloud ở đây đưa frame Ethernet giữa lab và VMnet2; không cần gán IP `192.168.99.x` cho NIC đó trên GNS3 VM.

**Nghiệm thu bước 5**

*   AUTO và NIC Cloud của GNS3 VM cùng nối `VMnet2`.
*   Cloud chạy trên đúng GNS3 VM.
*   Cloud nối vào OOB-SW.
*   Chưa cần gateway trong mạng OOB.

* * *

**6. Cấu hình OOB-SW và cổng quản lý Cisco**

**6.1. OOB-SW**

Ví dụ dùng 16 cổng `e0/0` tới `e3/3`:

cisco

```
enable
configure terminal
hostname OOB-SW

vlan 99
 name OOB-MGMT
exit

interface range Ethernet0/0 - 3
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

interface range Ethernet1/0 - 3
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

interface range Ethernet2/0 - 3
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

interface range Ethernet3/0 - 3
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

interface Vlan99
 ip address 192.168.99.2 255.255.255.0
 no shutdown
exit

end
write memory
```

Nếu image dùng cách đặt tên cổng khác, dùng tên thực tế từ `show interfaces status`.

Không cần trunk trên OOB-SW. Các link quản lý này đều truyền frame untagged.

**6.2. Router**

Ví dụ HQ-R1:

cisco

```
enable
configure terminal
hostname HQ-R1

interface Ethernet1/0
 description OOB_TO_OOB-SW
 ip address 192.168.99.11 255.255.255.0
 no shutdown
exit

end
write memory
```

Áp dụng cùng mẫu cho HQ-R2, BR1-R, BR2-R, ISP-A và ISP-B, thay IP theo bảng.

**6.3. Switch DS/access**

Ví dụ HQ-DS1:

cisco

```
enable
configure terminal
hostname HQ-DS1

vlan 99
 name OOB-MGMT
exit

interface Ethernet1/0
 description OOB_TO_OOB-SW
 switchport mode access
 switchport access vlan 99
 no shutdown
exit

interface Vlan99
 ip address 192.168.99.13 255.255.255.0
 no shutdown
exit

end
write memory
```

Làm tương tự cho HQ-DS2, HQ-ACC và hai switch Branch.

Ở các phase sau, **không cho VLAN 99 vào trunk nghiệp vụ** và không đưa subnet OOB vào OSPF/EIGRP/BGP.

> Cách này tách đường dây quản lý khỏi mạng nghiệp vụ. Khi bật routing trên thiết bị ở phase sau, vẫn cần ACL hoặc VRF nếu muốn kiểm soát chặt khả năng truy cập giữa các mạng.

**Nghiệm thu bước 6**

Chạy trên router:

cisco

```
show ip interface brief
ping 192.168.99.2
```

Chạy trên switch:

cisco

```
show vlan brief
show interfaces Ethernet1/0 switchport
show ip interface brief
ping 192.168.99.2
```

Kết quả cần đạt:

*   Router có cổng OOB `up/up`, đúng IP.
*   Switch có VLAN 99 và SVI `Vlan99 up/up`.
*   Cổng quản lý thuộc access VLAN 99.
*   Thiết bị ping được OOB-SW.

Nếu SVI `down`, kiểm tra VLAN đã tạo và có cổng VLAN 99 thực sự hoạt động, chờ STP ổn định.

* * *

**7. Cấu hình địa chỉ OOB trên AUTO**

Trong Ubuntu:

bash

```
ip -br address
ip route
```

Xác định card NAT và card OOB bằng MAC address. Ví dụ dưới giả sử:

text

```
ens33 = NAT
ens37 = OOB
```

**Phải thay theo tên thực tế.**

Sửa file Netplan đang quản lý hai card này trong `/etc/netplan/`. Không tạo thêm file có cấu hình trùng interface.

Cấu hình tương đương:

yaml

```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      dhcp4: false
      addresses:
        - 192.168.99.10/24
```

Áp dụng:

bash

```
sudo netplan generate
sudo netplan try
```

Nếu hoạt động đúng, xác nhận giữ cấu hình.

**Không đặt default gateway trên ens37.** Default route đi Internet nằm ở card NAT.

Kiểm tra:

bash

```
ip -br address
ip route
ping -c 4 192.168.99.2
ping -c 4 192.168.99.11
```

**Nghiệm thu bước 7**

*   Card OOB có `192.168.99.10/24`.
*   Có connected route tới `192.168.99.0/24` qua card OOB.
*   Default route đi qua card NAT.
*   Ping tới `.2` và `.11` thành công.
*   AUTO vẫn truy cập được kho phần mềm Internet.

**Chưa ping được thì chưa chuyển sang SSH/pyATS.**

* * *

**8. Bật SSH trên toàn bộ thiết bị Cisco**

Làm trước trên HQ-R1, sau đó lặp lại cho các thiết bị còn lại.

Thay hai mật khẩu bằng giá trị riêng của bạn:

cisco

```
enable
configure terminal

ip domain-name lab.local
username admin privilege 15 secret <LAB_PASS>
enable secret <LAB_ENABLE>
ip ssh version 2

line console 0
 logging synchronous
 exec-timeout 15 0
exit

line vty 0 4
 login local
 transport input ssh
 logging synchronous
 exec-timeout 15 0
exit

end
```

Tạo RSA key:

cisco

```
configure terminal
crypto key generate rsa modulus 2048
end
write memory
```

Nếu cú pháp một dòng không được hỗ trợ:

cisco

```
crypto key generate rsa
```

Rồi nhập `2048` khi được hỏi. Không tạo lại key nếu thiết bị đã có key phù hợp.

Kiểm tra:

cisco

```
show ip ssh
show crypto key mypubkey rsa
show running-config | section line vty
```

Nếu thiết bị có thêm VTY `5 15`, áp dụng cùng cấu hình SSH cho các line đó.

Từ AUTO:

bash

```
ssh admin@192.168.99.11
```

Sau khi đăng nhập:

cisco

```
show privilege
show version
exit
```

Lặp lại cho toàn bộ IP Cisco trong bảng OOB.

Nếu OpenSSH báo không tương thích thuật toán, chỉ bổ sung đúng thuật toán mà thiết bị yêu cầu. Ví dụ:

bash

```
ssh \
  -o KexAlgorithms=+diffie-hellman-group14-sha1 \
  -o HostKeyAlgorithms=+ssh-rsa \
  admin@192.168.99.11
```

Không áp dụng tùy chọn này hàng loạt nếu SSH mặc định đã hoạt động.

**Nghiệm thu bước 8**

*   `show ip ssh` xác nhận SSH v2.
*   AUTO đăng nhập được bằng tài khoản `admin`.
*   Prompt khớp hostname, ví dụ `HQ-R1#`.
*   `show privilege` cho biết mức 15.
*   Đọc được `show version`.
*   Đăng nhập sai mật khẩu bị từ chối.

Đến đây cần hoàn thành SSH thủ công trước khi thử kết nối bằng Python.

* * *

**9. Cài pyATS trong virtual environment**

pyATS được phát hành cho Linux/macOS; trang gói hiện nêu Python từ 3.10 trở lên. Hướng dẫn này dùng Python mặc định của Ubuntu 24.04, rồi kiểm tra kết quả cài đặt thực tế.

Trong AUTO:

bash

```
sudo apt update
sudo apt install -y \
  python3 python3-venv python3-pip \
  openssh-client git
```

Tạo thư mục làm việc:

bash

```
mkdir -p ~/ccnp-multisite-assurance
cd ~/ccnp-multisite-assurance

mkdir -p docs configs/phase0
mkdir -p assurance/tests assurance/lib
mkdir -p assurance/snapshots/phase0 assurance/reports
```

Tạo và kích hoạt môi trường:

bash

```
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install "pyats[full]"
```

Cách cài `pyats[full]` được hướng dẫn trong tài liệu pyATS.

Kiểm tra:

bash

```
python --version
python -m pip check
pyats version check
python -c "import pyats, genie, unicon; print('IMPORT_OK')"
```

Ghi lại phiên bản đã cài:

bash

```
python -m pip freeze > requirements-lock.txt
```

Mỗi lần mở terminal mới, kích hoạt lại:

bash

```
cd ~/ccnp-multisite-assurance
source .venv/bin/activate
```

**Nghiệm thu bước 9**

*   `pip check` không báo xung đột dependency.
*   Import được `pyats`, `genie`, `unicon`.
*   In ra `IMPORT_OK`.
*   Có `requirements-lock.txt` ghi phiên bản thực tế.

Nếu báo không có bản phân phối phù hợp, kiểm tra Python, kiến trúc CPU và phiên bản package; không chuyển sang cài trực tiếp trên Windows.

* * *

**10. Tạo testbed và kiểm tra cấu trúc**

Trong thư mục dự án, tạo:

text

```
assurance/testbed.yaml
```

Nội dung:

yaml

```
testbed:
  name: ccnp-multisite
  credentials:
    default:
      username: "%ENV{LAB_USER}"
      password: "%ENV{LAB_PASS}"
    enable:
      password: "%ENV{LAB_ENABLE}"

devices:
  HQ-R1:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.11

  HQ-R2:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.12

  HQ-DS1:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.13

  HQ-DS2:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.14

  HQ-ACC:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.15

  BR1-R:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.21

  BR2-R:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.22

  BR1-SW:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.23

  BR2-SW:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.24

  ISP-A:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.31

  ISP-B:
    os: ios
    type: router
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.32

  OOB-SW:
    os: ios
    type: switch
    connections:
      cli:
        protocol: ssh
        ip: 192.168.99.2
```

Nếu một thiết bị cần thuật toán SSH cũ, bổ sung **riêng tại****`connections.cli`****của thiết bị đó**:

yaml

```
ssh_options: >-
          -o KexAlgorithms=+diffie-hellman-group14-sha1
          -o HostKeyAlgorithms=+ssh-rsa
```

Nhập thông tin đăng nhập mà không ghi mật khẩu vào file:

bash

```
export LAB_USER=admin

read -rsp "LAB_PASS: " LAB_PASS
echo
export LAB_PASS

read -rsp "LAB_ENABLE: " LAB_ENABLE
echo
export LAB_ENABLE
```

Mẫu trên giả sử các thiết bị dùng chung tài khoản lab. Nếu khác nhau, cần credentials riêng theo thiết bị.

Kiểm tra:

bash

```
pyats validate testbed assurance/testbed.yaml
```

**Nghiệm thu bước 10**

*   Testbed được validate thành công.
*   Tên thiết bị trong YAML khớp hostname trên CLI.
*   Địa chỉ IP khớp bảng OOB.
*   File không chứa mật khẩu thật.
*   Không đưa HQ-SRV/VPCS vào testbed `os: ios`.

Validate thành công chỉ chứng minh cấu trúc file hợp lệ, **chưa chứng minh SSH hoạt động**.

* * *

**11. Viết bài kiểm tra đầu tiên: SSH → thu thập → parse → báo cáo**

Có một chi tiết cần xử lý để đúng nguyên tắc dự án: **Unicon có thể chạy lệnh khởi tạo khi kết nối**. Ta tắt nhóm lệnh khởi tạo cấu hình bằng `init_config_commands=[]`; chỉ giữ các lệnh EXEC điều chỉnh hiển thị của phiên SSH. Cisco cũng hướng dẫn dùng tham số này để tránh tự đưa thiết bị vào chế độ cấu hình.

Tạo file:

text

```
assurance/tests/phase0_smoke.py
```

Nội dung:

python

```
import argparse
import json
import sys
from datetime import datetime, timezone
from pathlib import Path

from genie.testbed import load

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--testbed",
        default="assurance/testbed.yaml",
    )
    parser.add_argument(
        "--devices",
        nargs="+",
        help="Chỉ kiểm tra các thiết bị được chỉ định",
    )
    args = parser.parse_args()

    tb = load(args.testbed)
    names = args.devices or list(tb.devices)

    unknown = [name for name in names if name not in tb.devices]
    if unknown:
        parser.error(f"Không có trong testbed: {unknown}")

    stamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
    root = Path("assurance/snapshots/phase0") / stamp
    root.mkdir(parents=True, exist_ok=True)

    results = []

    for name in names:
        dev = tb.devices[name]
        device_dir = root / name
        device_dir.mkdir()

        result = {
            "device": name,
            "status": "FAIL",
            "stage": "connect",
        }

        try:
            dev.connect(
                via="cli",
                init_config_commands=[],
                init_exec_commands=[
                    "terminal length 0",
                    "terminal width 512",
                ],
                log_stdout=False,
                connection_timeout=60,
            )

            result["stage"] = "collect"
            raw = dev.execute("show version", timeout=60)
            (device_dir / "show_version.txt").write_text(
                raw, encoding="utf-8"
            )

            interfaces = dev.execute(
                "show ip interface brief", timeout=60
            )
            (device_dir / "show_ip_interface_brief.txt").write_text(
                interfaces, encoding="utf-8"
            )

            result["stage"] = "parse"
            parsed = dev.parse("show version", output=raw)

            if not isinstance(parsed, dict) or not parsed:
                raise ValueError("Parser không trả về dictionary có dữ liệu")

            (device_dir / "show_version.json").write_text(
                json.dumps(
                    parsed,
                    ensure_ascii=False,
                    indent=2,
                    default=str,
                ),
                encoding="utf-8",
            )

            result["status"] = "PASS"
            result["stage"] = "complete"

        except Exception as exc:
            result["error"] = f"{type(exc).__name__}: {exc}"

        finally:
            try:
                dev.disconnect()
            except Exception as exc:
                result["disconnect_error"] = str(exc)
                result["status"] = "FAIL"

        results.append(result)
        print(f"{name}: {result['status']} [{result['stage']}]")

    (root / "summary.json").write_text(
        json.dumps(results, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )

    passed = sum(item["status"] == "PASS" for item in results)
    print(f"\nPASS: {passed}/{len(results)}")
    print(f"Evidence: {root}")

    return 0 if passed == len(results) else 1

if __name__ == "__main__":
    sys.exit(main())
```

Chạy trước trên HQ-R1:

bash

```
python assurance/tests/phase0_smoke.py --devices HQ-R1
```

Sau khi thành công, chạy toàn bộ:

bash

```
python assurance/tests/phase0_smoke.py
```

Script này:

1.   Kết nối lần lượt từng thiết bị, tránh gây tải đồng thời.
2.   Đọc `show version` và `show ip interface brief`.
3.   Parse chính output `show version` vừa thu thập.
4.   Lưu raw output, JSON và kết quả.
5.   Trả mã lỗi nếu có thiết bị thất bại.

**Không có lệnh cấu hình hoặc lưu cấu hình trong script.**

**Nghiệm thu bước 11**

Với một thiết bị:

*   Kết nối SSH thành công.
*   Có file raw `show_version.txt`.
*   Có file `show_version.json` với dữ liệu thực tế.
*   Có file `show_ip_interface_brief.txt`.
*   Kết quả HQ-R1 là `PASS`.

Với toàn bộ testbed trên:

*   Đạt **12/12 PASS** nếu đã dựng đủ 12 thiết bị.
*   Kiểm tra raw interface output để xác nhận IP OOB đúng.
*   `summary.json` không có lỗi chưa xử lý.

Đây là **smoke test kết nối và parser**, chưa kiểm định OSPF/BGP/HSRP.

Nếu parse lỗi nhưng raw output đã thu được, SSH có thể vẫn tốt; hãy đọc lỗi parser thay vì sửa mạng một cách phỏng đoán. Chưa truy cập các key cụ thể của Genie trước khi xem JSON thực tế.

* * *

**12. Kiểm tra lưu cấu hình và tính độc lập của OOB**

**12.1. Kiểm tra sau khởi động lại**

Trên mỗi thiết bị, lưu cấu hình thủ công:

cisco

```
write memory
```

Sau đó:

1.   Stop/start một thiết bị trong GNS3.
2.   Chờ boot hoàn tất.
3.   Kiểm tra lại hostname, IP OOB, SSH.
4.   Chạy lại smoke test cho thiết bị đó.

Thử trên HQ-R1 trước, sau đó trên một switch.

**12.2. Kiểm tra OOB độc lập**

Nếu bạn đã nối dây mạng nghiệp vụ:

1.   Giữ nguyên Ethernet1/0 OOB.
2.   Shutdown các cổng nghiệp vụ của HQ-R1.
3.   Từ AUTO thử SSH và chạy smoke test HQ-R1.
4.   Khôi phục các cổng theo trạng thái dự kiến.

**12.3. Thử phát hiện lỗi có chủ đích**

Chọn HQ-R1, dùng console GNS3 để:

cisco

```
configure terminal
interface Ethernet1/0
 shutdown
end
```

Chạy:

bash

```
python assurance/tests/phase0_smoke.py --devices HQ-R1
```

Kết quả phải là FAIL ở giai đoạn kết nối. Khôi phục qua console:

cisco

```
configure terminal
interface Ethernet1/0
 no shutdown
end
```

Chạy lại, kết quả phải trở về PASS.

**Nghiệm thu bước 12**

*   Restart không làm mất cấu hình quản lý.
*   Ngắt mạng nghiệp vụ không làm mất SSH OOB.
*   Ngắt cổng OOB khiến test phát hiện lỗi.
*   Khôi phục cổng OOB khiến test PASS trở lại.
*   Có bằng chứng trước lỗi, trong lỗi và sau khôi phục.

* * *

**13. Lưu bằng chứng và chốt Phase 0**

Trong thư mục dự án nên có:

text

```
docs/environment.md
docs/addressing.md
docs/phase0-results.md
configs/phase0/
assurance/testbed.yaml
assurance/tests/phase0_smoke.py
assurance/snapshots/phase0/
requirements-lock.txt
```

Trong `environment.md`, ghi:

*   RAM/CPU máy Windows.
*   Phiên bản GNS3 GUI và GNS3 VM.
*   VMware và Ubuntu.
*   Tên image, phiên bản IOS.
*   Cách nối VMnet2/Cloud.
*   Phiên bản Python/pyATS thực tế.

Khi lưu cấu hình vào `configs/phase0/`, **thay cả mật khẩu dạng hash/secret và các khóa bằng placeholder trước khi đưa lên Git**. Running-config có mật khẩu băm vẫn là dữ liệu cần bảo vệ.

Ví dụ `.gitignore`:

gitignore

```
.venv/
.env
__pycache__/
*.pyc

assurance/snapshots/
assurance/reports/
*.log

*.bin
*.image
*.qcow2
*.vmdk
*.ova
iourc
.iourc
```

Không dùng quy tắc bỏ qua toàn bộ `*.yaml`, vì testbed không chứa mật khẩu là một phần cần quản lý phiên bản.

**Nghiệm thu bước 13 — điều kiện hoàn thành Phase 0**

| Hạng mục | Điều kiện đạt |
| --- | --- |
| GNS3 | Project mở lại được, thiết bị boot bình thường |
| Image | Có phiên bản thực tế; DS có tính năng phù hợp |
| OOB | IP đúng, không trùng, interface `up/up` |
| AUTO | OOB `.10/24`, default route qua NAT |
| SSH | Đăng nhập thủ công được vào mọi thiết bị thuộc phạm vi |
| pyATS | Virtual environment hoạt động, testbed hợp lệ |
| Thu thập | Có raw output và JSON `show version` thực tế |
| Smoke test | Tất cả thiết bị đã dựng đều PASS |
| Phát hiện lỗi | Ngắt OOB → FAIL; khôi phục → PASS |
| Lưu cấu hình | Restart router và switch vẫn SSH được |
| Bằng chứng | Có ghi chú môi trường và kết quả |
| Git | Không chứa image, mật khẩu hoặc khóa thật |

**Để tôi đối chiếu lần đầu, bạn gửi 5 nhóm thông tin:** cấu hình RAM/CPU máy; tên hai image IOL; ảnh phần GNS3 VM/server; `show version` của một router và một DS; `ip -br address` cùng `ip route` trên AUTO. Từ đó có thể điều chỉnh chính xác tên cổng, tài nguyên và cách nối OOB trước khi cấu hình hàng loạt.
