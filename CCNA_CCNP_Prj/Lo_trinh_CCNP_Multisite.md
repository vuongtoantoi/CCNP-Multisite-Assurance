# Lộ trình triển khai CCNP Multi-Site & Network Assurance

Lộ trình gồm 9 phase (0 đến 8), đi từ dễ đến khó. Trước khi vào từng phase, cần chốt bốn điều chỉnh thiết kế.

## Bốn quyết định thiết kế nên chốt trước

1. **Redistribution đặt ở đâu.** Không redistribute BGP vào OSPF (cả bảng định tuyến Internet) hoặc ngược lại. BGP chỉ lo underlay: Edge học default route từ ISP và đẩy xuống OSPF bằng `default-information originate`. Redistribution thật nằm ở hub HQ giữa OSPF và EIGRP overlay, có route-map và tag chống loop. Cách này vừa đúng thực tế vừa đúng nội dung ENARSI.
2. **pyATS chạy trên Linux, không chạy native trên Windows.** Dùng WSL2, một máy Linux ảo, hoặc container có sẵn Python. Thiết bị quản lý qua một mạng OOB riêng bằng SSH, không dùng telnet vào console GNS3.
3. **Viết test cùng lúc với từng phase.** Tiêu chí nghiệm thu của mỗi phase trở thành một test pyATS. Đừng để đến cuối mới viết code, vì khi đó không có cách kiểm chứng từng bước.
4. **Netmiko là phần tùy chọn.** pyATS tự có lớp kết nối (Unicon), nên Netmiko chỉ cần khi muốn chạy lệnh thô ngoài Genie. Vẫn có thể ghi "pyATS + Netmiko" lên CV nếu thực sự dùng.

## Topology và địa chỉ tổng thể

| Thành phần | Quy ước |
|---|---|
| HQ | HQ-R1 (nối ISP-A), HQ-R2 (nối ISP-B), DS1/DS2 (HSRP), 1 access switch, 1–2 server Linux (DNS/Syslog/NTP) |
| Branch | B1 nối ISP-A, B2 nối ISP-B, mỗi nơi 1 router + 1 switch + 1 host |
| ISP | ISP-A và ISP-B, peering eBGP với nhau, mỗi ISP có một loopback đóng vai "Internet" |
| AS number | HQ 65010, B1 65021, B2 65022, ISP-A 65100, ISP-B 65200 |
| LAN | HQ 10.10.0.0/16, B1 10.20.0.0/16, B2 10.30.0.0/16 |
| Underlay "public" | 203.0.113.0/24 chia /30: ISP-A↔HQ-R1 .0/30, ISP-B↔HQ-R2 .4/30, ISP-A↔B1 .8/30, ISP-B↔B2 .12/30, ISP-A↔ISP-B .16/30 |
| Tunnel DMVPN | 172.16.100.0/24 (hub .1, B1 .11, B2 .12) |
| Loopback | 10.255.0.0/24 |
| OOB quản lý | 192.168.99.0/24 (không quảng bá vào OSPF/BGP/EIGRP) |

Tổng khoảng 14 node IOL, nên cấp cho GNS3 VM khoảng 8–12 GB RAM.

---

## Phase 0 – Nền tảng lab và mặt phẳng quản lý

- **Mục tiêu / topology:** GNS3 + GNS3 VM, import image IOL, dựng một switch OOB nối mọi thiết bị và một node Linux (hoặc WSL2) để chạy pyATS.
- **CLI:** `hostname`, `ip domain-name`, `crypto key generate rsa modulus 2048`, `ip ssh version 2`, user local `privilege 15`, `line vty 0 4` với `login local` và `transport input ssh`, `logging synchronous`, IP quản lý trên interface OOB, `write memory`.
- **Nghiệm thu:** `show ip interface brief`, `show ip ssh`, và SSH từ máy Linux vào từng thiết bị thành công.
- **pyATS:** viết `testbed.yaml` (`os: ios`), kết nối và chạy `pyats parse "show version" --testbed-file testbed.yaml`. Mật khẩu lấy từ biến môi trường, không ghi cứng.

## Phase 1 – Campus HQ: VLAN, STP, HSRP

- **Mục tiêu / topology:** DS1/DS2 dự phòng cho nhau, access switch nối cả hai, VLAN 10 (Users), VLAN 20 (Servers).
- **CLI:** VLAN, trunk (chỉ cho phép các VLAN cần), `spanning-tree mode rapid-pvst`, root primary ở DS1 và secondary ở DS2, SVI, HSRP v2 với `preempt`, priority cao hơn ở DS1 cho VLAN 10. Có thể đảo ngược cho VLAN 20 để chia tải.
- **Nghiệm thu:** `show vlan brief`, `show interfaces trunk`, `show spanning-tree vlan 10`, `show standby brief`. Làm thử: tắt uplink DS1 rồi kiểm tra DS2 lên Active, host vẫn ping được gateway.
- **pyATS:** test đầu tiên là mỗi group HSRP có đúng một Active và một Standby, STP root đúng thiết bị.

> **Lưu ý:** kiểm tra image IOL L2 có hỗ trợ SVI, `ip routing` và HSRP không. Nếu không, dùng IOL L3 làm DS và để L2 cho access.

## Phase 2 – OSPF multi-area tại HQ

- **Mục tiêu / topology:** Area 0 gồm các link DS↔Edge, Area 10 gồm các SVI người dùng (passive) và link DS1–DS2. DS1/DS2 là ABR.
- **CLI:** `router ospf` với router-id theo loopback, `passive-interface default`, `ip ospf network point-to-point` trên link router, xác thực OSPF (key chain nếu image hỗ trợ, nếu không thì MD5), `area 10 range 10.10.0.0 255.255.0.0` để summarization tại ABR.
- **Nghiệm thu:** `show ip ospf neighbor` (FULL), `show ip ospf interface brief`, `show ip ospf database` (có LSA type 3 đã summarize), `show ip route ospf`, ping/traceroute giữa các subnet.
- **pyATS:** đây là baseline snapshot đầu tiên. Tạo `intent.yaml` ghi số neighbor mong đợi của từng router, test so sánh trạng thái thật với intent.

## Phase 3 – ISP, eBGP, iBGP giữa hai Edge và PAT

- **Mục tiêu / topology:** Dựng ISP-A và ISP-B (eBGP với nhau), HQ-R1/HQ-R2 peer eBGP với ISP tương ứng, B1/B2 kết nối ISP và có default route.
- **CLI:**
  - ISP dùng `default-originate` xuống khách và `network` để quảng bá các /30 của mình.
  - Giữa HQ-R1 và HQ-R2 phải có **iBGP** (peer bằng loopback, `update-source`, `next-hop-self`).
  - Chính sách: local-preference 200 cho đường qua ISP-A, 100 cho ISP-B.
  - PAT trên Edge (`ip nat inside/outside`, ACL chọn lưu lượng).
  - `default-information originate` vào OSPF.
- **Nghiệm thu:** `show ip bgp summary` (Established), `show ip bgp`, `show ip route 0.0.0.0`, `show ip nat translations`, `show ip nat statistics`. Từ host HQ ping loopback "Internet" của cả hai ISP.
- **pyATS:** test mọi BGP peer Established theo intent, có default route, bảng NAT có entry. Đây là test BGP đầu tiên.

## Phase 4 – DMVPN Phase 3 (chưa mã hóa)

- **Mục tiêu / topology:** Hub là HQ-R1, spoke là B1 và B2. Làm sạch GRE trước, bật IPsec ở phase sau để dễ tách lỗi.
- **CLI (khung cấu hình, cú pháp NHS tùy phiên bản IOS):**

```
! HUB
interface Tunnel100
 ip address 172.16.100.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp authentication DMVPN
 ip nhrp map multicast dynamic
 ip nhrp redirect
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100

! SPOKE
interface Tunnel100
 ip address 172.16.100.11 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp network-id 100
 ip nhrp authentication DMVPN
 ip nhrp nhs 172.16.100.1 nbma 203.0.113.2 multicast
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

- **Nghiệm thu:** `show dmvpn` (spoke ở trạng thái UP), `show ip nhrp`, ping các địa chỉ tunnel. Sau khi có routing (Phase 5): traceroute từ LAN B1 sang LAN B2 lần đầu đi qua hub, lần sau đi thẳng, kèm `show ip route nhrp` và `show dmvpn detail`.
- **pyATS:** test các spoke đã đăng ký với hub (đúng số entry) và, sau Phase 5, test spoke-to-spoke có entry động.

## Phase 5 – IPsec, EIGRP overlay và redistribution

- **Mục tiêu / topology:** Bảo vệ tunnel bằng IPsec, chạy EIGRP trên Tunnel100, redistribute hai chiều tại hub giữa EIGRP và OSPF.
- **CLI:**
  - **IPsec:** `crypto isakmp policy`, key dùng wildcard (`address 0.0.0.0`) vì spoke có thể đổi địa chỉ, transform-set với `mode transport`, `crypto ipsec profile`, `tunnel protection ipsec profile ...` trên cả hub và spoke.
  - **EIGRP** trên tunnel; hub dùng `ip summary-address eigrp` (hoặc `no ip split-horizon eigrp` nếu không summarize).
  - **Redistribution:** từ EIGRP vào OSPF gắn tag 100, từ OSPF vào EIGRP gắn tag 200, route-map deny các route đã mang tag của chiều kia. Không đẩy default hay các /30 underlay vào overlay.
- **Nghiệm thu:** `show crypto isakmp sa` (QM_IDLE), `show crypto ipsec sa` (counter encaps/decaps tăng), `show ip eigrp neighbors`, `show ip route eigrp`, `show ip ospf database external`, `show route-map`. Kiểm tra bảng routing không dao động (không flapping) sau khi bật redistribution.
- **pyATS:** test IPsec SA lên, EIGRP neighbor đủ, và một test "sanity" kiểm tra route redistributed mang đúng tag.

## Phase 6 – Dịch vụ và hardening

- **Mục tiêu / topology:** Hoàn thiện DHCP, NTP, Syslog và bảo mật cơ bản.
- **CLI:**
  - DHCP trên router chi nhánh, `ip helper-address` hoặc DHCP server cho HQ.
  - NTP master tại HQ với xác thực, chi nhánh làm client qua tunnel.
  - `service timestamps log datetime msec`, `logging host` tới server Linux, `logging source-interface Loopback0`.
  - ACL cho VTY, tắt dịch vụ thừa.
- **Nghiệm thu:** `show ip dhcp binding`, `show ntp associations`, `show ntp status`, `show logging`, kiểm tra log đến server, `show access-lists`.
- **pyATS:** thêm test NTP đồng bộ, vì timestamp chính xác là điều kiện để đối chiếu log với snapshot.

## Phase 7 – Chịu lỗi và failover

- **Mục tiêu / topology:** Bổ sung IP SLA, object tracking và BFD ở Edge, kèm kịch bản lỗi. Nếu còn thời gian, thêm HQ-R2 làm hub thứ hai (dual-hub), vì hiện tại hub nằm trên ISP-A nên DMVPN sẽ rớt khi ISP-A lỗi.
- **CLI:** `ip sla`, `track`, default route có điều kiện hoặc BGP theo tracking, `bfd`. Với hub thứ hai, chú ý redistribution tại hai điểm dễ gây loop, nên tag phải chặt.
- **Nghiệm thu:** `show ip sla statistics`, `show track`, `show bfd neighbors`, và với mỗi kịch bản lỗi ghi lại thời gian hội tụ: tắt ISP-A, tắt DS1, ngắt một spoke.
- **pyATS:** chụp snapshot trước và sau lỗi rồi chạy diff. Loại các trường dao động (uptime, counter) ra khỏi so sánh.

## Phase 8 – Hoàn thiện nền tảng assurance

- **Mục tiêu:** Gộp mọi test thành một bộ chạy được bằng một lệnh.
- **Việc cần làm:**
  - Dùng AEtest để có pass/fail rõ ràng.
  - `intent.yaml` làm nguồn chuẩn.
  - Lưu snapshot JSON có timestamp.
  - Xuất báo cáo HTML (Easypy hoặc Jinja2 tự viết) và Markdown.
  - Lưu bản sao `show running-config` để kiểm soát thay đổi.
- **Nghiệm thu:** chạy toàn bộ suite ở trạng thái bình thường (toàn pass), rồi gây lỗi có chủ đích và xem báo cáo đánh dấu đúng chỗ.
- **Trình bày cho CV:** README với sơ đồ, bảng kết quả hội tụ, video demo ngắn, và lint/CI nếu muốn.

---

## Lỗi hay gặp khi làm DMVPN Phase 3 và IPsec

- **Quên `ip nhrp redirect` trên hub hoặc `ip nhrp shortcut` trên spoke.** Khi đó traffic spoke-to-spoke vẫn đi qua hub như Phase 1/2, và dễ tưởng nhầm là đã hoạt động.
- **Recursive routing.** Nếu EIGRP/OSPF quảng bá địa chỉ NBMA (203.0.113.x) qua chính tunnel, tunnel sẽ flap kèm log `%TUN-5-RECURDOWN`. Redistribution ở Phase 5 rất dễ gây ra lỗi này, nên NBMA chỉ nên đi qua underlay.
- **Split horizon và next-hop trên hub.** Nếu hub không summarize mà bật EIGRP, spoke sẽ không học được LAN của nhau, trừ khi tắt split-horizon (và xem lại `next-hop-self`). Cách gọn hơn trong Phase 3 là hub gửi summary.
- **MTU/MSS.** Thiếu `ip mtu 1400` và `ip tcp adjust-mss` thì ping nhỏ chạy nhưng TCP lớn bị treo. Kiểm tra bằng ping có DF bit với kích thước lớn.
- **Gói đầu tiên đi qua hub.** Mất 1–2 gói ping lần đầu là bình thường, đừng nhầm với lỗi.
- **Sai lệch tham số.** `NHRP network-id`, `authentication`, `tunnel key`, hoặc ISAKMP policy/transform-set không khớp hai đầu. Với IPsec, trạng thái `MM_NO_STATE` trong `show crypto isakmp sa` nghĩa là đàm phán IKE chưa xong.
- **CEF.** Đảm bảo `ip cef` đang bật.
- **Image IOL.** Nếu DMVPN + IPsec có hành vi lạ dù cấu hình đúng, thử đổi phiên bản image IOL (nhánh 15.x) trước khi nghi ngờ thiết kế.
- **Mất cấu hình.** Luôn `write memory`, chụp snapshot GNS3 sau mỗi phase và commit file config vào git.

## Lỗi hay gặp khi chạy pyATS với GNS3 trên Windows

- **WSL2 không thấy mạng của GNS3 VM.** WSL2 mặc định dùng NAT nên không tới được dải host-only của GNS3 VM. Cách xử lý: bật mirrored networking (nếu phiên bản WSL hỗ trợ), thêm route, hoặc chạy pyATS trong một VM Linux riêng cùng dải mạng. Nên thử `ping` và `ssh` thủ công trước khi chạy script.
- **Cloud node nối vào card mạng Windows.** Bridge ra Wi-Fi thường không ổn định, nên ưu tiên host-only của GNS3 VM.
- **SSH thuật toán cũ.** IOS cũ chỉ hỗ trợ các thuật toán mà OpenSSH hiện đại tắt mặc định, nên kết nối có thể bị từ chối dù thông tin đăng nhập đúng. Dùng RSA ≥ 2048, SSH v2 và, nếu cần, bật thêm thuật toán legacy ở phía client.
- **Dùng console telnet.** Cổng đổi sau khi khởi động lại project, banner và prompt phá pattern, syslog chen vào output làm Genie parse lỗi. Vì vậy cần `logging synchronous` và nên dùng SSH.
- **`SchemaEmptyParserError`.** Xảy ra khi lệnh trả về rỗng (ví dụ `show ip bgp summary` khi chưa có neighbor). Bắt exception này và coi là kết quả hợp lệ ("không có neighbor").
- **Parser không phủ hết mọi lệnh.** Một số lệnh DMVPN/NHRP có thể thiếu parser ở phiên bản của bạn. Khi đó tự viết parser bằng regex hoặc TextFSM, và đó cũng là điểm cộng cho dự án.
- **`learn` rất nặng.** Nó chạy nhiều lệnh một lúc nên chậm trên IOL. Ưu tiên parse từng lệnh cần thiết.
- **Kết nối song song quá nhiều.** GNS3 VM sẽ bị nghẽn. Giới hạn số thiết bị chạy đồng thời và đặt timeout hợp lý.
- **Diff có nhiễu.** Uptime và counter luôn đổi, nên cần danh sách trường bị loại trừ khi so sánh snapshot.
- **Môi trường Python.** Dùng virtualenv, và kiểm tra phiên bản Python mà bản pyATS bạn cài hỗ trợ trước khi cài.
