# CCNP Multi-Site Network & Automated Network Assurance — Tài liệu ngữ cảnh dự án

> **Mục đích file này:** gói toàn bộ ngữ cảnh, quyết định thiết kế, kế hoạch triển khai và các lưu ý kỹ thuật của dự án để người dùng có thể đưa cho một AI agent khác (hoặc một phiên chat mới) và làm tiếp mà không phải giải thích lại từ đầu.
> **Ngày lập:** 2026-10-25 · **Ngôn ngữ làm việc của người dùng:** tiếng Việt (thuật ngữ kỹ thuật giữ nguyên tiếng Anh) · **Trạng thái dự án:** đang ở giai đoạn thiết kế/kế hoạch, **chưa dựng lab nào**.

---

## 0. Hướng dẫn dành cho AI agent đọc file này

1. **Trả lời bằng tiếng Việt**, giữ thuật ngữ tiếng Anh (OSPF, DMVPN, NHRP, ...). Người dùng là sinh viên, đã có CCNA, đang học CCNP; giải thích "tại sao" chứ không chỉ "làm gì".
2. **Không đưa Python vào việc push cấu hình.** Quy tắc cốt lõi của dự án: cấu hình thiết bị làm bằng tay trên CLI (để học CCNP); Python/pyATS chỉ dùng để **thu thập trạng thái, kiểm chứng, báo cáo** (Network Assurance).
3. **Đừng bịa output của lệnh `show` hay schema của Genie parser.** Những chỗ phụ thuộc phiên bản image/phiên bản pyATS được đánh dấu **[CẦN XÁC MINH]**. Hãy yêu cầu người dùng chạy lệnh thật và dán output trước khi kết luận.
4. **Không commit image Cisco (IOL/IOSv) hay mật khẩu thật** lên git. Mật khẩu đi qua biến môi trường.
5. Mỗi khi người dùng làm xong một phase, hãy dùng **tiêu chí nghiệm thu** của phase đó (mục 5 trở đi) để kiểm tra, rồi mới sang phase kế tiếp. Nếu một bước lỗi, quay lại debug theo thứ tự underlay → overlay → routing → dịch vụ.
6. Nếu bạn muốn đề xuất thay đổi thiết kế lớn (đổi giao thức, thêm MPLS, thêm IPv6...), hãy hỏi người dùng trước và ghi lý do, vì phạm vi dự án đã được cố ý giữ vừa phải.
7. Phần **Phụ lục A** là bản thảo gốc người dùng thảo luận với Gemini; **Phụ lục B** là ghi chú về repo tham khảo trên GitHub. Hai phần này là *tham khảo*, không phải thiết kế cuối cùng. Thiết kế cuối cùng nằm ở mục 3–6.

---

## 1. Bối cảnh người dùng và mục tiêu

| Hạng mục | Nội dung |
|---|---|
| Người dùng | Sinh viên chuyên ngành Điện tử - Viễn thông |
| Chứng chỉ / học tập | Đã có CCNA; đang ôn CCNP Enterprise (ENCOR / ENARSI) |
| Kỹ năng | Nền tảng Python; từng làm một dự án nhỏ tự động hóa cấu hình bằng Netmiko/Pexpect (Day 0/Day 1 provisioning, quản lý OOB) |
| Môi trường | GNS3 trên **Windows Desktop**, dùng image **Cisco IOU/IOL (L2/L3)** để tiết kiệm tài nguyên và tránh nested virtualization |
| Mục tiêu | Một dự án cá nhân cấp độ CCNP để **đưa vào CV xin thực tập Network Engineer** |
| Repo tham khảo | `github.com/cismembrane/gns3-labs` (của người khác; chỉ dùng làm tài liệu tham khảo để dựng topology riêng, **không phải dự án của người dùng**) |

**Tên dự án:** Thiết kế Mạng Đa Chi nhánh (Multi-Site) & Tự động hóa Kiểm định Trạng thái (Automated Network Assurance).

**Yêu cầu chức năng do người dùng đặt ra:**

- Topology: 1 HQ (Core/Distribution dự phòng bằng HSRP), 2 Branch, vùng mô phỏng Internet/ISP trung gian.
- Underlay: OSPF multi-area trong LAN, eBGP từ Edge Router ra ISP.
- Overlay & bảo mật: DMVPN Phase 3 + IPsec qua Internet; chạy EIGRP hoặc iBGP qua tunnel; có Route Redistribution.
- Assurance: dùng Cisco pyATS/Genie (kết hợp Netmiko nếu cần) để lấy snapshot trạng thái (BGP neighbors, OSPF, HSRP, IPsec tunnel), parse ra JSON, kiểm tra logic lỗi (tunnel down, mất BGP peer...), xuất báo cáo HTML/Markdown.

---

## 2. Nhật ký trao đổi (theo thứ tự thời gian)

> Các yêu cầu của người dùng được giữ gần nguyên văn; phần trả lời của trợ lý được tóm lại ở mức quyết định/kết luận, vì nội dung chi tiết đã được phát triển đầy đủ ở mục 3–7.

**Lượt 1 — Hỏi về repo GitHub.**
Người dùng gửi link repo `cismembrane/gns3-labs` (nhánh `main` và `ci/lint-baseline`) và hỏi trợ lý có đọc được không. *Kết quả:* GitHub chặn truy cập tự động nên không đọc trực tiếp được; trợ lý đề nghị dán nội dung hoặc upload file.

**Lượt 2 — Hai file README.**
Người dùng dán hai bản README của repo. *Kết quả:* trợ lý tóm tắt cấu trúc repo (lab BGP/OSPF/EIGRP/DMVPN/HSRP/GLBP + Ansible, CI lint, verify bằng `assert`) và nêu khác biệt giữa hai bản (bản mới có k8s-metallb-bgp, bảng nền tảng, mục Automation and Validation, CI).

**Lượt 3 — Nhờ nhận xét bản thảo với Gemini.**
Người dùng gửi bản thảo dự án (xem Phụ lục A) và nhờ nhận xét. *Kết quả chính:*
- Điểm mạnh: kịch bản doanh nghiệp thực tế, hướng Network Assurance khác với "chỉ cấu hình", chọn pyATS/Genie hợp lý.
- Vấn đề kỹ thuật: (1) chọn image chưa nhất quán — c7200 là router, không làm Core Switch HSRP/STP được, cần IOL L2/IOSvL2 hoặc IOL L3; (2) pyATS không hỗ trợ native trên Windows → dùng WSL2/Linux VM; (3) DMVPN trên hạ tầng 2 ISP chưa nói rõ NBMA address của hub; (4) redistribution OSPF–BGP dễ loop, nên làm có kiểm soát bằng route-map/tag/prefix-list; (5) phạm vi quá rộng → chia phase; (6) thiếu kịch bản lỗi/failover làm bằng chứng cho assurance; (7) không commit image Cisco lên GitHub.

**Lượt 4 — Làm rõ về repo.**
Người dùng: *"repo trên chỉ là phần tham khảo để tôi tạo lên topo thôi."* *Kết quả:* trợ lý rút lại nhận xét "trùng lặp với repo của bạn", và khuyên: học cấu trúc thư mục/README/verify/lint, nhưng tự thiết kế địa chỉ/AS/config, ghi nguồn nếu lấy ý tưởng, chốt một bộ image thống nhất.

**Lượt 5 — "ENARSI không nhiều nhỉ, chỉ thấy DMVPN liên quan".**
*Kết quả:* đối chiếu với blueprint 300-410. Dự án đã chạm nhóm Layer 3 (route-map, redistribution, chống loop, EIGRP/OSPF/BGP) và Infrastructure Services (NAT, DHCP, NTP, Syslog); còn thiếu PBR, VRF-Lite, BFD, IPv6 routing, BGP policy nâng cao, Infrastructure Security (ACL, uRPF, AAA, CoPP), IP SLA/tracking, SNMP, MPLS L3VPN. Đề xuất bổ sung gọn: IP SLA + tracking + BFD, BGP policy, PBR, VRF-Lite, hardening Edge; MPLS L3VPN để tùy chọn cuối cùng. (Xem ma trận ở mục 9.)

**Lượt 6 — Yêu cầu lộ trình triển khai (prompt chính).**
Người dùng yêu cầu trợ lý đóng vai *Senior Network Architect & NetDevOps Mentor* và thiết kế Implementation Plan chia Phase từ dễ đến khó. Với mỗi Phase cần: (1) mục tiêu và topology, (2) hạng mục cấu hình CLI, (3) tiêu chí nghiệm thu bằng lệnh `show`, (4) vị trí tích hợp pyATS/Genie; nhấn mạnh pitfalls của DMVPN Phase 3 và pyATS trên GNS3/Windows. *Kết quả:* lộ trình 9 phase (Phase 0–8), bốn quyết định thiết kế nền tảng (mục 3), và danh sách pitfalls (mục 7, 8).

**Lượt 7 — "Continue".** Trợ lý hoàn tất lộ trình ở trên.

**Lượt 8 — Yêu cầu file này.**
*"Hãy ghi lại chi tiết cuộc trò chuyện thành file .md để tôi có thể đưa vào context hoặc các AI agent khác để cùng làm project này... giữ lại nguyên chat + giải thích thêm chuyên sâu về từng phase."*

---

## 3. Các quyết định thiết kế đã chốt

| # | Quyết định | Lý do |
|---|---|---|
| D1 | **Redistribution thật nằm ở hub HQ giữa OSPF và EIGRP overlay**, có route-map + tag + prefix-list. BGP *không* được redistribute cả bảng vào OSPF; Edge chỉ `default-information originate` xuống OSPF. | Đúng thực tế (không bơm bảng Internet vào IGP) và vẫn thể hiện đủ kiến thức redistribution/chống loop của ENARSI. |
| D2 | **pyATS chạy trên Linux** (WSL2 hoặc VM Linux), thiết bị được quản lý qua mạng **OOB bằng SSH**; không dùng telnet vào console GNS3. | pyATS không hỗ trợ chính thức Windows; console telnet dễ lỗi prompt/cổng/syslog xen vào. |
| D3 | **Test-driven networking:** tiêu chí nghiệm thu mỗi phase được viết thành test pyATS ngay trong phase đó. | Có bằng chứng kiểm chứng từng bước; cuối dự án có sẵn bộ test hồi quy. |
| D4 | Dùng **Unicon (của pyATS)** làm lớp kết nối chính; Netmiko là tùy chọn (lệnh thô ngoài Genie). | pyATS đã có kết nối riêng; tránh trùng lặp công cụ. |
| D5 | **IPsec bật ở phase riêng** sau khi GRE/NHRP chạy sạch. | Dễ tách lỗi: lỗi tunnel vs lỗi mã hóa. |
| D6 | **Mạng OOB 192.168.99.0/24** không quảng bá vào bất kỳ giao thức định tuyến nào. | Giữ đường quản lý/test độc lập với lỗi của mạng đang test. (VRF-Lite cho OOB là mở rộng tùy chọn.) |
| D7 | **Chốt một bộ image thống nhất** (IOL L3 + IOL L2), không trộn Dynamips c7200/c3660 và IOSv. | Dễ dựng lại, tiết kiệm tài nguyên, tránh khác biệt hành vi. |
| D8 | **Dual-hub (HQ-R2 làm hub thứ hai) là tùy chọn ở Phase 7.** Ban đầu hub duy nhất là HQ-R1. | Giữ phạm vi vừa phải; nhưng ghi rõ điểm yếu: ISP-A lỗi thì DMVPN mất hub. |
| D9 | **MPLS L3VPN, IPv6 đầy đủ** để ngoài phạm vi mặc định; có thể thêm ở phần mở rộng. | Nặng tài nguyên/dễ vỡ phạm vi. |

---

## 4. Thiết kế topology và địa chỉ (đặc tả để dựng lab)

> Đây là đề xuất của trợ lý, chưa được người dùng triển khai. Người dùng có thể đổi quy ước, nhưng nếu đổi hãy cập nhật nhất quán toàn bộ bảng dưới.

### 4.1 Danh sách thiết bị

| Hostname | Vai trò | Loại node | Loopback0 | OOB (192.168.99.x) |
|---|---|---|---|---|
| HQ-R1 | Edge + DMVPN hub, nối ISP-A | IOL L3 | 10.255.0.1/32 | .11 |
| HQ-R2 | Edge dự phòng, nối ISP-B | IOL L3 | 10.255.0.2/32 | .12 |
| HQ-DS1 | Distribution/Core (L3), HSRP | IOL L2/L3 | 10.255.0.11/32 | .13 |
| HQ-DS2 | Distribution/Core (L3), HSRP | IOL L2/L3 | 10.255.0.12/32 | .14 |
| HQ-ACC | Access switch (L2) | IOL L2 | — | .15 |
| HQ-SRV | Linux server (DNS/Syslog/NTP/DHCP tùy chọn) | Docker/Linux node | — | .16 |
| ISP-A | Mô phỏng ISP A | IOL L3 | Lo0 203.0.113.100/32 ("Internet" mục tiêu) | .31 |
| ISP-B | Mô phỏng ISP B | IOL L3 | Lo0 203.0.113.200/32 | .32 |
| BR1-R | Router chi nhánh 1 (DMVPN spoke), nối ISP-A | IOL L3 | 10.255.0.21/32 | .21 |
| BR2-R | Router chi nhánh 2 (DMVPN spoke), nối ISP-B | IOL L3 | 10.255.0.22/32 | .22 |
| BR1-SW / BR2-SW | Switch chi nhánh | IOL L2 | — | tùy chọn |
| BR1-PC / BR2-PC / HQ-PC | Host kiểm tra (VPCS hoặc Linux nhẹ) | VPCS/Alpine | — | — |
| OOB-SW | Switch OOB nối mọi thiết bị | IOL L2 | — | — |
| AUTO | Máy Linux chạy pyATS (hoặc WSL2 nếu mạng thông) | Linux VM | — | .10 |

Khoảng **14–16 node**, nên cấp cho GNS3 VM khoảng **8–12 GB RAM**. IOL: cấu hình đủ số Ethernet adapter (mỗi adapter = 4 cổng; ví dụ `e0/0–e0/3` và `e1/0–e1/3`).

### 4.2 AS number

| Thực thể | AS |
|---|---|
| HQ | 65010 |
| BR1 | 65021 |
| BR2 | 65022 |
| ISP-A | 65100 |
| ISP-B | 65200 |

### 4.3 Địa chỉ LAN, tunnel, quản lý

| Mạng | Prefix | Ghi chú |
|---|---|---|
| HQ VLAN 10 (Users) | 10.10.10.0/24 | HSRP VIP .1; DS1 .2; DS2 .3 |
| HQ VLAN 20 (Servers) | 10.10.20.0/24 | HSRP VIP .1; DS1 .2; DS2 .3; HQ-SRV .10 |
| BR1 LAN | 10.20.10.0/24 | gateway BR1-R .1 |
| BR2 LAN | 10.30.10.0/24 | gateway BR2-R .1 |
| Loopback | 10.255.0.0/24 | /32 theo bảng 4.1 |
| Link nội bộ HQ (hạ tầng) | 10.255.1.0/24 chia /30 | tách khỏi 10.10.0.0/16 để summary ở ABR gọn |
| DMVPN tunnel (Tunnel100) | 172.16.100.0/24 | hub .1; BR1-R .11; BR2-R .12; (hub 2 tùy chọn: HQ-R2 .2) |
| OOB | 192.168.99.0/24 | xem bảng 4.1 |
| Underlay "public" | 203.0.113.0/24 | dải tài liệu RFC 5737, dùng cho mô phỏng |

### 4.4 Link nội bộ HQ (OSPF)

| Link | Subnet | Địa chỉ | Area |
|---|---|---|---|
| HQ-R1 ↔ HQ-DS1 | 10.255.1.0/30 | R1 .1, DS1 .2 | 0 |
| HQ-R1 ↔ HQ-DS2 | 10.255.1.4/30 | R1 .5, DS2 .6 | 0 |
| HQ-R2 ↔ HQ-DS1 | 10.255.1.8/30 | R2 .9, DS1 .10 | 0 |
| HQ-R2 ↔ HQ-DS2 | 10.255.1.12/30 | R2 .13, DS2 .14 | 0 |
| HQ-DS1 ↔ HQ-DS2 | 10.255.1.16/30 | DS1 .17, DS2 .18 | 10 |
| DS1/DS2 ↔ HQ-ACC | trunk L2 (VLAN 10, 20) | — | — |

Loopback của mọi router/DS thuộc **Area 0** (cần để iBGP giữa HQ-R1/HQ-R2 hoạt động qua loopback). Các SVI VLAN 10/20 thuộc **Area 10** (passive).

### 4.5 Underlay "Internet" và BGP

| Link | Subnet | Địa chỉ |
|---|---|---|
| ISP-A ↔ HQ-R1 | 203.0.113.0/30 | ISP-A .1, HQ-R1 .2 (**NBMA của hub**) |
| ISP-B ↔ HQ-R2 | 203.0.113.4/30 | ISP-B .5, HQ-R2 .6 |
| ISP-A ↔ BR1-R | 203.0.113.8/30 | ISP-A .9, BR1-R .10 (NBMA spoke 1) |
| ISP-B ↔ BR2-R | 203.0.113.12/30 | ISP-B .13, BR2-R .14 (NBMA spoke 2) |
| ISP-A ↔ ISP-B | 203.0.113.16/30 | ISP-A .17, ISP-B .18 |

Phiên BGP: HQ-R1–ISP-A, HQ-R2–ISP-B, BR1-R–ISP-A, BR2-R–ISP-B (eBGP, kết nối trực tiếp, không cần `ebgp-multihop`); ISP-A–ISP-B (eBGP); **HQ-R1–HQ-R2 là iBGP** peer qua loopback.

### 4.6 Gán cổng gợi ý (IOL)

- HQ-R1: `e0/0` ISP-A, `e0/1` DS1, `e0/2` DS2, `e1/0` OOB. HQ-R2: `e0/0` ISP-B, `e0/1` DS1, `e0/2` DS2, `e1/0` OOB.
- HQ-DS1/DS2: `e0/0` R1, `e0/1` R2, `e0/2` DS còn lại, `e0/3` ACC, `e1/0` OOB.
- HQ-ACC: `e0/0` DS1, `e0/1` DS2, `e0/2` HQ-SRV (VLAN 20), `e0/3` HQ-PC (VLAN 10), `e1/0` OOB.
- BR*-R: `e0/0` ISP, `e0/1` LAN, `e1/0` OOB. ISP-A/B: một cổng cho mỗi link + OOB.
- **[CẦN XÁC MINH]** cách đưa cổng OOB lên L2 IOL / DS (dùng `no switchport` hoặc SVI VLAN riêng không trunk) tùy image của người dùng.

### 4.7 Sơ đồ khối (ASCII)

```
                 ┌──────────── ISP-A (AS65100) ════ ISP-B (AS65200) ────────────┐
                 │  .0/30           │ .8/30                │ .12/30      .4/30 │
              HQ-R1 (hub,AS65010)   BR1-R (AS65021)       BR2-R (AS65022)    HQ-R2
                 │ ╲                                                         ╱ │
                 │   ╲___________ iBGP (Lo0) ______________________________╱   │
              HQ-DS1 ───── (area 10) ───── HQ-DS2                              │
                 └──────────── HQ-ACC (VLAN10/20) ─── HQ-PC, HQ-SRV            │
                       (HQ-DS1/DS2 cũng nối HQ-R2; Tunnel100 mGRE: HQ-R1 ↔ BR1-R ↔ BR2-R)
   OOB-SW ═══ mọi thiết bị (192.168.99.0/24) ═══ AUTO (pyATS)
```

---

## 5. Lộ trình triển khai chi tiết (Phase 0 → 8)

**Quy ước mỗi phase:** (A) Mục tiêu & lý do · (B) Kiến thức nền cần hiểu · (C) Cấu hình mẫu (khung, không phải bản cuối) · (D) Tiêu chí nghiệm thu · (E) Kịch bản lỗi để thử · (F) Tích hợp pyATS · (G) Pitfalls · (H) Sản phẩm commit lên git.
Cấu hình mẫu chỉ mang tính tham khảo; **người dùng phải tự gõ và hiểu từng dòng** vì mục tiêu là học CCNP. Nhớ `write memory` và chụp snapshot GNS3 sau mỗi phase.

### Phase 0 — Nền tảng lab và mặt phẳng quản lý

**(A) Mục tiêu.** Có một môi trường ổn định để mọi phase sau dựa vào: GNS3 + GNS3 VM chạy IOL, mạng OOB, và máy Linux chạy pyATS kết nối SSH được vào mọi thiết bị.

**(B) Kiến thức nền.**
- IOL (IOS on Linux) chạy như tiến trình Linux trong GNS3 VM nên **không cần KVM/nested virtualization** (khác với IOSv/QEMU). GNS3 VM vẫn cần chạy trên VMware/VirtualBox. IOL thường không cần file `iourc` license (bản IOU cũ thì cần) **[CẦN XÁC MINH theo image của bạn]**.
- Vì sao cần OOB: để test/quản lý không phụ thuộc vào mạng đang được test. Nếu OSPF/BGP hỏng mà bạn vẫn SSH được thì mới debug được.
- Vì sao SSH thay telnet console: console GNS3 đổi cổng sau khi restart, banner/prompt/syslog làm hỏng pattern parse.

**(C) Cấu hình mẫu (mọi thiết bị).**
```
hostname HQ-R1
ip domain-name lab.local
username admin privilege 15 secret <LAB_PASS>
crypto key generate rsa modulus 2048      ! hostname + domain-name phải có trước
ip ssh version 2
line con 0
 logging synchronous
line vty 0 4
 login local
 transport input ssh
 logging synchronous
 exec-timeout 0 0
interface Ethernet1/0
 description OOB
 ip address 192.168.99.11 255.255.255.0
 no shutdown
```

**(D) Nghiệm thu.**
- `show ip interface brief` — cổng OOB `up/up` với IP đúng.
- `show ip ssh` — SSH enabled, version 2.
- Từ máy AUTO: `ssh admin@192.168.99.11` (và các thiết bị khác) thành công; `ping` thủ công thành công trước khi chạy bất kỳ script nào.

**(E) Kịch bản lỗi.** Thử đăng nhập sai mật khẩu, thử SSH khi chưa có RSA key để hiểu thông báo lỗi.

**(F) pyATS — điểm bắt đầu.** Tạo `testbed.yaml` (xem mục 6.1), kết nối và chạy:
`pyats parse "show version" --testbed-file testbed.yaml --devices HQ-R1`.
Mục tiêu: chứng minh chuỗi *WSL2/Linux → OOB → SSH → Genie parse* hoạt động trước khi có bất kỳ giao thức định tuyến nào.

**(G) Pitfalls.** Xem mục 8 (WSL2 không thấy GNS3 VM, SSH thuật toán cũ, IOL chưa boot xong...). Quên `write memory` → mất cấu hình khi restart.

**(H) Commit.** `docs/addressing.md`, `assurance/testbed.yaml` (không chứa mật khẩu), `configs/phase0/*.cfg`.

---

### Phase 1 — Campus HQ: VLAN, RSTP, HSRP

**(A) Mục tiêu.** HQ có gateway dự phòng: DS1/DS2 chạy HSRP, STP root khớp với HSRP Active để đường đi tối ưu.

**(B) Kiến thức nền.**
- **HSRP v2** có Active/Standby; `preempt` cho phép router ưu tiên cao giành lại Active. HSRP Active nên **trùng STP root** của VLAN đó, nếu không lưu lượng từ access lên gateway phải đi vòng qua link giữa hai DS.
- **Load sharing theo VLAN:** DS1 là root/Active cho VLAN 10; DS2 là root/Active cho VLAN 20. Mỗi DS đóng vai trò backup cho VLAN còn lại.
- Rapid-PVST hội tụ nhanh hơn PVST cổ điển; `allowed vlan` hạn chế VLAN trên trunk để tránh mở rộng miền STP không cần thiết.

**(C) Cấu hình mẫu (DS1; DS2 đảo ngược priority/root).**
```
spanning-tree mode rapid-pvst
spanning-tree vlan 10 root primary
spanning-tree vlan 20 root secondary
vlan 10
 name USERS
vlan 20
 name SERVERS
interface Ethernet0/3
 switchport trunk encapsulation dot1q      ! tùy image
 switchport mode trunk
 switchport trunk allowed vlan 10,20
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt
interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 90       ! DS2 đặt 110 để làm Active VLAN 20
 standby 20 preempt
```
**[CẦN XÁC MINH]** Image IOL L2 của bạn có hỗ trợ SVI, `ip routing`, HSRP hay không. Nếu không, dùng IOL L3 làm DS (cổng `e0/x` kiểu router-on-a-stick hoặc switchport module nếu image có) và để IOL L2 chỉ làm access.

**(D) Nghiệm thu.**
- `show vlan brief`, `show interfaces trunk` — đúng VLAN allowed.
- `show spanning-tree vlan 10` / `vlan 20` — root đúng DS, cổng ở trạng thái FWD/BLK hợp lý.
- `show standby brief` — VLAN 10: DS1 Active/DS2 Standby; VLAN 20: ngược lại.
- Host HQ-PC ping được VIP 10.10.10.1 và HQ-SRV.

**(E) Kịch bản lỗi.** Tắt uplink DS1→ACC (hoặc shutdown DS1): xác nhận DS2 lên Active cho VLAN 10, ping từ HQ-PC không đứt quá vài giây; bật lại DS1: nhờ `preempt` DS1 giành lại Active.

**(F) pyATS.** Test đầu tiên: mỗi group HSRP có đúng một Active và một Standby; Active khớp intent (`intent.yaml`); STP root đúng thiết bị. Dùng parser cho `show standby brief` hoặc `learn hsrp` **[CẦN XÁC MINH schema thực tế bằng `pyats parse ... --output`]**.

**(G) Pitfalls.** HSRP Active ≠ STP root (suboptimal path); trunk allowed VLAN không khớp hai đầu; không bật preempt nên failback không xảy ra; HSRP version 1 và 2 ở hai đầu không nói chuyện được với nhau.

**(H) Commit.** `configs/phase1/`, `assurance/tests/test_hsrp.py`, ghi chú kết quả failover vào `docs/test-results.md`.

---

### Phase 2 — OSPF multi-area tại HQ

**(A) Mục tiêu.** Có IGP nội bộ HQ với multi-area có chủ đích, để Edge/DS biết nhau và có nền cho iBGP, NTP, Syslog sau này.

**(B) Kiến thức nền.**
- **Thiết kế area:** Area 0 gồm các link Edge↔DS và loopback; Area 10 gồm các SVI người dùng/server (passive) và link DS1–DS2. DS1/DS2 là **ABR**.
- **Summarization tại ABR:** `area 10 range 10.10.0.0 255.255.0.0` gom các prefix của Area 10 thành một LSA Type 3 và tạo route Null0 (discard) trên ABR để chống loop. Đây là điểm ăn tiền cho "multi-area có lý do".
- **Network type point-to-point** trên link router–router bỏ bầu DR/BDR, hội tụ nhanh và gọn adjacency. `passive-interface default` rồi chỉ mở các cổng thật sự cần.
- Giai đoạn này **chưa** có hub/redistribution; Edge sẽ nhận default từ BGP ở Phase 3.
- Mở rộng tùy chọn: thêm một router "HQ-SRV-R" để Area 10 có router nội bộ và đặt Area 10 là totally stubby (`area 10 stub no-summary` ở ABR).

**(C) Cấu hình mẫu (DS1).**
```
router ospf 1
 router-id 10.255.0.11
 passive-interface default
 no passive-interface Ethernet0/0
 no passive-interface Ethernet0/1
 no passive-interface Ethernet0/2
 area 10 range 10.10.0.0 255.255.0.0
interface Loopback0
 ip address 10.255.0.11 255.255.255.255
 ip ospf 1 area 0
interface Ethernet0/0
 ip address 10.255.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
interface Ethernet0/2
 ip address 10.255.1.17 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 10
interface Vlan10
 ip ospf 1 area 10
interface Vlan20
 ip ospf 1 area 10
! xác thực (ví dụ MD5; dùng SHA nếu image hỗ trợ)
interface Ethernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 <KEY>
```

**(D) Nghiệm thu.**
- `show ip ospf neighbor` — HQ-R1 và HQ-R2 mỗi router 2 neighbor FULL; HQ-DS1/DS2 mỗi router 3 neighbor FULL.
- `show ip ospf interface brief`, `show ip protocols`.
- `show ip ospf database` — thấy LSA Type 3 `10.10.0.0/16` ở Area 0 (đã summarize), không còn các /24 riêng lẻ.
- `show ip route ospf` trên Edge — thấy 10.10.0.0/16 (O IA) và loopback; ping/traceroute từ HQ-PC đến loopback Edge.

**(E) Kịch bản lỗi.** Gỡ `no passive-interface` hoặc đổi network type một đầu để xem lỗi mismatch; thử lệch key MD5 để thấy neighbor không lên.

**(F) pyATS.** **Baseline snapshot đầu tiên.** Test: mọi OSPF neighbor `FULL` và đúng số lượng theo `intent.yaml`; kiểm tra có LSA summary mong đợi. Lưu snapshot JSON làm "golden state".

**(G) Pitfalls.** Quên `ip ospf network point-to-point` ở một đầu (hello/dead lệch network type); MTU mismatch kẹt ở ExStart; router-id trùng; loopback không nằm trong OSPF làm iBGP Phase 3 không lên.

**(H) Commit.** `configs/phase2/`, `assurance/tests/test_ospf.py`, `snapshots/golden/phase2/*.json` (có thể gitignore thư mục snapshots, giữ vài mẫu).

---

### Phase 3 — ISP, eBGP, iBGP giữa hai Edge, PAT và chi nhánh ra Internet

**(A) Mục tiêu.** HQ và hai chi nhánh ra "Internet" (loopback ISP) qua underlay BGP, HQ có hai Edge/hai ISP với chính sách ưu tiên rõ ràng. Mục tiêu thêm: dựng sẵn underlay mà DMVPN sẽ chạy trên đó.

**(B) Kiến thức nền.**
- **Hai Edge cùng AS cần iBGP với nhau** (qua loopback, `update-source`, `next-hop-self`). Nếu không có, từng Edge chỉ biết đường của ISP mình, chính sách không nhất quán, và dễ blackhole.
- **Chính sách chọn đường ra:** `local-preference` 200 cho đường học từ ISP-A, 100 từ ISP-B → toàn AS 65010 ưu tiên đi ra ISP-A, ISP-B là dự phòng. (Chiều vào nếu cần dùng AS-path prepend; xem phần mở rộng ở mục 9.)
- **ISP chỉ gửi default** cho khách (`default-originate` + prefix-list chặn các route khác), và **khách không được trở thành transit** (không quảng bá lại route của ISP kia).
- **Default xuống OSPF:** `default-information originate` (không có `always`) trên cả hai Edge, với metric R1 thấp hơn R2 (ví dụ 10 vs 100) để R1 được ưu tiên. Không redistribute BGP vào OSPF.
- **PAT** trên Edge: traffic từ LAN ra ngoài được dịch sang địa chỉ interface outside; return traffic quay về đúng Edge vì PAT dùng địa chỉ của chính Edge đó. Không có NAT stateful failover, nên nếu R1 sập, các phiên đang mở sẽ đứt (chấp nhận).
- **NBMA address cho DMVPN** (203.0.113.2/.10/.14) chính là địa chỉ outside của các router → phải đi qua underlay (mục 7).

**(C) Cấu hình mẫu.**
```
! HQ-R1
router bgp 65010
 bgp router-id 10.255.0.1
 bgp log-neighbor-changes
 neighbor 203.0.113.1 remote-as 65100
 neighbor 203.0.113.1 description ISP-A
 neighbor 10.255.0.2 remote-as 65010
 neighbor 10.255.0.2 update-source Loopback0
 address-family ipv4
  neighbor 203.0.113.1 activate
  neighbor 203.0.113.1 route-map FROM-ISPA in
  neighbor 10.255.0.2 activate
  neighbor 10.255.0.2 next-hop-self
route-map FROM-ISPA permit 10
 set local-preference 200
router ospf 1
 default-information originate metric 10 metric-type 1
!
ip access-list standard NAT-INSIDE
 permit 10.10.0.0 0.0.255.255
interface Ethernet0/0
 ip nat outside
interface Ethernet0/1
 ip nat inside
interface Ethernet0/2
 ip nat inside
ip nat inside source list NAT-INSIDE interface Ethernet0/0 overload

! ISP-A
router bgp 65100
 neighbor 203.0.113.2  remote-as 65010
 neighbor 203.0.113.10 remote-as 65021
 neighbor 203.0.113.18 remote-as 65200
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.252
  network 203.0.113.8 mask 255.255.255.252
  network 203.0.113.100 mask 255.255.255.255
  neighbor 203.0.113.2  default-originate
  neighbor 203.0.113.10 default-originate
  neighbor 203.0.113.2  prefix-list DEFAULT-ONLY out
  neighbor 203.0.113.10 prefix-list DEFAULT-ONLY out
```
Chi nhánh: eBGP với ISP tương ứng, nhận default, PAT LAN chi nhánh ra Internet, **chưa** có tunnel. Với HQ-R2: local-pref 100 từ ISP-B, `default-information originate metric 100`. Tùy chọn: cấp một dải public nhỏ (ví dụ 198.51.100.0/29) làm NAT pool và `network` + route Null0 để "quảng bá dải IP public" như bản thảo gốc.

**(D) Nghiệm thu.**
- `show ip bgp summary` — mọi phiên `Established` (cột State/PfxRcd là số, không phải Idle/Active).
- `show ip bgp` — thấy default từ ISP-A với local-pref 200 là best trên HQ-R1/HQ-R2.
- `show ip route 0.0.0.0` — trên HQ-R1 là `B*`; trên DS là `O*E1`.
- `show ip nat translations`, `show ip nat statistics`.
- Từ HQ-PC, ping `203.0.113.100` và `203.0.113.200`; từ BR1-PC/BR2-PC ping cả hai.
- Từ BR2-R ping 203.0.113.2 (NBMA của hub): chứng minh underlay thông liên ISP trước khi dựng tunnel.

**(E) Kịch bản lỗi.** Shutdown link HQ-R1–ISP-A: default ở HQ nên chuyển sang đường qua HQ-R2/ISP-B (sau thời gian phát hiện của BGP); bật lại và quan sát chuyển ngược lại.

**(F) pyATS.** Test: mọi BGP neighbor `Established` theo `intent.yaml`, có default route, bảng NAT có entry sau khi sinh traffic. Đây là test BGP đầu tiên.

**(G) Pitfalls.** iBGP không đặt `update-source Loopback0`/`next-hop-self`; quên `activate` neighbor trong address-family; eBGP ra thiết bị không kết nối trực tiếp mà không có `ebgp-multihop`; quên `network` hoặc route Null0 nên prefix không được quảng bá; local-pref đặt ở chiều `out` thay vì `in`; NAT ACL chọn nhầm traffic nội bộ cần đi qua tunnel (xử lý khi sang Phase 4/5).

**(H) Commit.** `configs/phase3/`, `assurance/tests/test_bgp.py`, bảng chính sách BGP trong `docs/design-decisions.md`.

---

### Phase 4 — DMVPN Phase 3 (chưa mã hóa)

**(A) Mục tiêu.** Tunnel mGRE hub-and-spoke chạy với NHRP; xác nhận cơ chế redirect/shortcut của Phase 3 hoạt động **trước khi** thêm IPsec và routing phức tạp.

**(B) Kiến thức nền.**
- **mGRE + NHRP:** mỗi spoke đăng ký (NHRP registration) địa chỉ NBMA của mình với hub (NHS). Hub lưu ánh xạ tunnel-IP → NBMA. Spoke muốn nói với spoke khác thì hỏi NHRP.
- **Khác biệt giữa các phase DMVPN:** Phase 1 chỉ hub-spoke; Phase 2 spoke-to-spoke bằng NHRP resolution nhưng yêu cầu hub giữ next-hop nguyên bản; **Phase 3** dùng `ip nhrp redirect` (hub) và `ip nhrp shortcut` (spoke) — hub có thể gửi **summary hoặc default**, gói đầu tiên đi qua hub, hub gửi NHRP redirect, spoke gửi resolution request và cài **route NHRP cụ thể (shortcut)** để các gói sau đi thẳng.
- **MTU/MSS:** GRE thêm 24 byte (và IPsec thêm nữa), nên đặt `ip mtu 1400` và `ip tcp adjust-mss 1360` trên tunnel.
- **Source của tunnel:** hub dùng `Ethernet0/0` (203.0.113.2). Lưu ý: với hai ISP, đây là điểm yếu — xử lý ở Phase 7 (dual-hub/hai cloud).

**(C) Cấu hình mẫu.**
```
! HUB (HQ-R1)
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

! SPOKE (BR1-R)  — cú pháp NHS có thể khác theo phiên bản IOS
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
Để test trước khi có EIGRP, có thể tạm dùng route tĩnh trỏ về hub; sang Phase 5 sẽ thay bằng EIGRP.

**(D) Nghiệm thu.**
- `show dmvpn` — mỗi spoke thấy hub ở trạng thái `UP`; hub thấy cả hai spoke `UP` (loại `D` dynamic).
- `show ip nhrp` — ánh xạ tunnel-IP ↔ NBMA đúng.
- Ping 172.16.100.1/.11/.12 giữa các thành viên.
- Spoke-to-spoke (sau khi có routing ở Phase 5): `traceroute` từ LAN BR1 sang LAN BR2 lần đầu thấy hop là hub, lần sau đi thẳng; `show dmvpn detail` có thêm entry dynamic giữa hai spoke; `show ip route nhrp` có route shortcut.

**(E) Kịch bản lỗi.** Xóa `ip nhrp shortcut` trên một spoke → thấy traffic spoke-to-spoke vẫn đi qua hub (hành vi giống Phase 1/2). Đổi `network-id` hoặc `tunnel key` một đầu → spoke không đăng ký được.

**(F) pyATS.** Test: hub thấy đúng số spoke ở trạng thái `UP` theo intent. Nếu Genie không có parser phù hợp cho `show dmvpn`/`show ip nhrp` ở phiên bản của bạn **[CẦN XÁC MINH]**, tự viết parser bằng regex hoặc TextFSM — là điểm cộng cho dự án.

**(G) Pitfalls.** Xem mục 7 (đầy đủ). Tóm tắt: quên `redirect`/`shortcut`; lệch NHRP network-id/auth/tunnel key; MTU/MSS; recursive routing; gói đầu tiên đi qua hub là bình thường.

**(H) Commit.** `configs/phase4/`, `assurance/tests/test_dmvpn.py`.

---

### Phase 5 — IPsec, EIGRP overlay và Route Redistribution

**(A) Mục tiêu.** Mã hóa tunnel bằng IPsec, trao đổi route LAN giữa HQ và chi nhánh qua EIGRP trên Tunnel100, và redistribute **hai chiều tại hub (HQ-R1) giữa EIGRP và OSPF** một cách có kiểm soát. Đây là phase khó nhất và là "trái tim" CCNP/ENARSI của dự án.

**(B) Kiến thức nền.**
- **IPsec bảo vệ GRE:** dùng `mode transport` vì GRE đã đóng gói; ISAKMP/IKEv1 với pre-shared key dạng **wildcard** (`address 0.0.0.0 0.0.0.0`) vì spoke-to-spoke cần thiết lập SA với địa chỉ NBMA động. `tunnel protection ipsec profile` gắn vào tunnel (không dùng crypto map).
- **EIGRP trên mGRE:** hub học route từ spoke nhưng mặc định **split-horizon** không cho quảng bá lại ra đúng interface. Với DMVPN Phase 3 có hai cách: (1) hub gửi **summary** (`ip summary-address eigrp 100 10.0.0.0 255.0.0.0` trên Tunnel100) — spoke-to-spoke dựa vào NHRP redirect/shortcut; (2) tắt split-horizon (`no ip split-horizon eigrp 100`) và giữ next-hop (`no ip next-hop-self eigrp 100`) để spoke học route cụ thể của nhau. Cách (1) gọn và đúng tinh thần Phase 3. Lưu ý: summary tạo route **Null0 (AD 5)** trên hub — mọi traffic 10.x không có route cụ thể sẽ bị discard ở hub (chủ ý).
- **Redistribution chống loop:** ở một điểm duy nhất (hub) loop khó xảy ra, nhưng nếu sau này thêm HQ-R2 làm hub (Phase 7) thì mutual redistribution tại **hai điểm** sẽ tạo loop/route feedback. Luôn **gắn tag khi vào** và **deny theo tag khi ra**, kết hợp prefix-list để chỉ cho phép đúng các prefix cần thiết.
- **Route Null0 do summary bị redistribute nhầm:** vì route summary 10.0.0.0/8 là route EIGRP trong RIB, `redistribute eigrp` có thể kéo nó vào OSPF. Prefix-list `BRANCH-LANS` chặn việc này.
- **Recursive routing:** tuyệt đối không để địa chỉ NBMA (203.0.113.x) hoặc default route học được qua tunnel. Không redistribute default/underlay vào EIGRP; spoke giữ default từ BGP/ISP; kiểm tra `show ip route 203.0.113.2` trên spoke không đi qua Tunnel100.

**(C) Cấu hình mẫu.**
```
! --- IPsec (cả hub và spoke) — thuật toán phụ thuộc image [CẦN XÁC MINH] ---
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key <PSK> address 0.0.0.0 0.0.0.0
crypto isakmp keepalive 30 5
crypto ipsec transform-set TS-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
crypto ipsec profile DMVPN-PROF
 set transform-set TS-DMVPN
interface Tunnel100
 tunnel protection ipsec profile DMVPN-PROF
! Nếu image không hỗ trợ: thử aes 128 / sha / group 2 / esp-sha-hmac trước khi nghi ngờ thiết kế

! --- EIGRP (hub HQ-R1) ---
router eigrp 100
 network 172.16.100.0 0.0.0.255
 redistribute ospf 1 metric 10000 100 255 1 1400 route-map OSPF-TO-EIGRP
interface Tunnel100
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
 ! (cách 2) no ip split-horizon eigrp 100 / no ip next-hop-self eigrp 100

! --- EIGRP (spoke BR1-R) ---
router eigrp 100
 network 172.16.100.0 0.0.0.255
 network 10.20.10.0 0.0.0.255
 passive-interface Ethernet0/1
 eigrp stub connected

! --- Redistribution tại hub ---
ip prefix-list BRANCH-LANS seq 10 permit 10.20.0.0/16 le 24
ip prefix-list BRANCH-LANS seq 20 permit 10.30.0.0/16 le 24
ip prefix-list HQ-PREFIXES seq 10 permit 10.10.0.0/16 le 24
ip prefix-list HQ-PREFIXES seq 20 permit 10.255.0.0/24 le 32
route-map EIGRP-TO-OSPF deny 10
 match tag 200
route-map EIGRP-TO-OSPF permit 20
 match ip address prefix-list BRANCH-LANS
 set tag 100
 set metric-type type-1
route-map OSPF-TO-EIGRP deny 10
 match tag 100
route-map OSPF-TO-EIGRP permit 20
 match ip address prefix-list HQ-PREFIXES
 set tag 200
router ospf 1
 redistribute eigrp 100 subnets metric 20 route-map EIGRP-TO-OSPF
```

**(D) Nghiệm thu.**
- IPsec: `show crypto isakmp sa` (trạng thái `QM_IDLE`), `show crypto ipsec sa` (counter `#pkts encaps/decaps` **tăng** khi ping), `show crypto session`.
- EIGRP: `show ip eigrp neighbors` (hub có 2 spoke), `show ip route eigrp` (spoke có `D 10.0.0.0/8` qua hub; hub có `D` cho 10.20.10.0/24 và 10.30.10.0/24).
- Redistribution: trên HQ-DS1, `show ip route 10.20.10.0 255.255.255.0` → `O E1`, **Route tag 100**; trên hub, `show route-map`, `show ip protocols`; `show ip ospf database external` chỉ có các LAN chi nhánh, **không** có 10.0.0.0/8.
- Không flapping: `show logging` không có `%TUN-5-RECURDOWN`; `show ip route 203.0.113.2` trên spoke không qua Tunnel100.
- End-to-end: BR1-PC ping HQ-PC và BR2-PC; traceroute lần 1 qua hub, lần 2 đi thẳng (shortcut); `show dmvpn detail` thấy tunnel động BR1–BR2 với IPsec.

**(E) Kịch bản lỗi.** (1) Sai PSK một đầu → `show crypto isakmp sa` ở trạng thái `MM_NO_STATE`/không có SA. (2) Cố tình redistribute default vào EIGRP để quan sát recursive routing và tunnel flap (chỉ thử trong lab, rồi gỡ). (3) Bỏ `deny match tag` trong route-map để quan sát route feedback (cần hai điểm redistribution mới thấy rõ; làm ở Phase 7).

**(F) pyATS.** Test: IPsec SA ở trạng thái active cho mọi cặp mong đợi; số EIGRP neighbor đúng; route LAN chi nhánh trên DS mang tag 100; không có route NBMA học qua tunnel. Thêm **test "sanity" cho redistribution:** duyệt `show ip route` của hub/DS và khẳng định mọi route redistributed có tag hợp lệ.

**(G) Pitfalls.** Xem mục 7 (đặc biệt recursive routing, summary Null0 bị redistribute, split-horizon).

**(H) Commit.** `configs/phase5/`, `assurance/tests/test_ipsec.py`, `test_eigrp.py`, `test_redistribution.py`, `docs/redistribution-design.md` (sơ đồ luồng route + bảng tag/prefix-list).

---

### Phase 6 — Dịch vụ hạ tầng và hardening

**(A) Mục tiêu.** Hoàn thiện các dịch vụ mà mạng doanh nghiệp cần, đồng thời tạo dữ liệu log/thời gian chuẩn cho assurance.

**(B) Kiến thức nền.**
- **DHCP:** chi nhánh cấp DHCP cục bộ trên router; HQ dùng `ip helper-address 10.10.20.10` (relay tới server) hoặc pool trên DS. DNS trỏ về HQ-SRV.
- **NTP có xác thực:** HQ-R1 `ntp master`; mọi thiết bị khác `ntp server 10.255.0.1 key 1`; branch đến NTP qua tunnel (loopback 10.255.0.0/24 phải có route — đã nằm trong prefix-list `HQ-PREFIXES`).
- **Syslog:** timestamp có mili giây và `logging source-interface Loopback0`, log về HQ-SRV (10.10.20.10). **Đồng bộ thời gian là điều kiện** để đối chiếu log với snapshot.
- **Hardening:** ACL giới hạn VTY chỉ từ OOB, `service password-encryption`, tắt `ip http server`, banner, `enable secret`. Nếu bật `aaa new-model` phải thêm phương thức local fallback để **không tự khóa mình**.

**(C) Cấu hình mẫu.**
```
! BR1-R
ip dhcp excluded-address 10.20.10.1 10.20.10.99
ip dhcp pool BR1-LAN
 network 10.20.10.0 255.255.255.0
 default-router 10.20.10.1
 dns-server 10.10.20.10
! HQ DS (SVI VLAN 10)
interface Vlan10
 ip helper-address 10.10.20.10
! NTP
ntp authenticate
ntp authentication-key 1 md5 <NTPKEY>
ntp trusted-key 1
ntp source Loopback0
ntp server 10.255.0.1 key 1        ! (client); HQ-R1: ntp master 3
! Syslog
service timestamps log datetime msec localtime show-timezone
service timestamps debug datetime msec
logging source-interface Loopback0
logging host 10.10.20.10
logging trap informational
! Hardening
ip access-list standard MGMT-ONLY
 permit 192.168.99.0 0.0.0.255
line vty 0 4
 access-class MGMT-ONLY in
no ip http server
no ip http secure-server
service password-encryption
```

**(D) Nghiệm thu.**
- `show ip dhcp binding` (chi nhánh cấp lease cho PC), PC chi nhánh nhận IP/gateway/DNS.
- `show ntp status` ("Clock is synchronized"), `show ntp associations`.
- `show logging` (có dòng "Logging to 10.10.20.10"), kiểm tra log thực sự đến server (`tail /var/log/syslog` trên HQ-SRV).
- `show access-lists`, SSH từ ngoài OOB bị từ chối.

**(E) Kịch bản lỗi.** Làm lệch key NTP để quan sát mất đồng bộ; shutdown một interface rồi kiểm tra log xuất hiện đúng timestamp ở server.

**(F) pyATS.** Test NTP đồng bộ trên mọi thiết bị; test số lease DHCP; có thể thêm bước đọc log syslog trên server để đối chiếu với thời điểm gây lỗi ở Phase 7.

**(G) Pitfalls.** NTP không đồng bộ vì không có route đến 10.255.0.1 qua tunnel; syslog không qua được vì ACL/NAT; áp dụng `access-class` sai khiến chính AUTO bị chặn; `aaa new-model` không có local fallback.

**(H) Commit.** `configs/phase6/`, `assurance/tests/test_services.py`.

---

### Phase 7 — Chịu lỗi, failover và kịch bản sự cố

**(A) Mục tiêu.** Biến dự án từ "chạy được" thành "chạy được khi có sự cố và chứng minh được điều đó" — đây là phần tạo khác biệt cho CV.

**(B) Kiến thức nền.**
- **Điểm yếu thiết kế:** hub duy nhất (HQ-R1) nằm trên ISP-A; ISP-A lỗi → DMVPN mất hub. Hai hướng khắc phục:
  1. **Dual-cloud (khuyến nghị):** HQ-R2 là hub của Tunnel101 (172.16.101.0/24, `network-id 101`, key 101, NBMA 203.0.113.6); mỗi spoke có hai tunnel, hai NHS; dùng `delay`/`bandwidth` hoặc offset-list để ưu tiên Tunnel100.
  2. **Dual-hub trên một cloud:** thêm HQ-R2 (172.16.100.2) làm NHS thứ hai; hai hub cần mối liên hệ NHRP/EIGRP với nhau.
- Khi có hub thứ hai, **redistribution xảy ra ở hai điểm** → bắt buộc dùng tag và deny theo tag (đã thiết kế ở Phase 5). Cũng cần so sánh AD/metric để HQ ưu tiên một đường.
- **IP SLA + tracking:** BGP chỉ phát hiện sự cố khi mất peer (hold timer mặc định 180 s). IP SLA `icmp-echo` tới "Internet" (203.0.113.100) phát hiện **ISP up nhưng upstream đứt**. Kết quả track có thể điều khiển `default-information originate route-map` có điều kiện (kỹ thuật: route tĩnh Null0 gắn `track` làm điều kiện tồn tại cho route-map) **[CẦN XÁC MINH trên image]**.
- **BFD** (`bfd interval 300 min_rx 300 multiplier 3`, `neighbor X fall-over bfd`) rút ngắn thời gian phát hiện lỗi link; **[CẦN XÁC MINH]** IOL có hỗ trợ BFD đầy đủ không.
- **Mở rộng tùy chọn có nhiều giá trị ENARSI:** BGP policy chiều vào (AS-path prepend), PBR (ép syslog/subnet nhất định đi ISP-B), VRF-Lite cho OOB, hardening Edge (ACL anti-spoof, uRPF).

**(C) Cấu hình mẫu (ý tưởng).**
```
ip sla 10
 icmp-echo 203.0.113.100 source-interface Ethernet0/0
 frequency 5
ip sla schedule 10 life forever start-time now
track 10 ip sla 10 reachability
!
router bgp 65010
 neighbor 203.0.113.1 fall-over bfd        ! nếu BFD khả dụng
```

**(D) Nghiệm thu (bảng kịch bản lỗi — ghi lại thời gian hội tụ thực đo).**

| Kịch bản | Hành động | Kỳ vọng | Lệnh xác nhận |
|---|---|---|---|
| ISP-A lỗi | shutdown link HQ-R1–ISP-A | Default ở HQ chuyển sang ISP-B; (với dual-cloud) DMVPN chuyển sang Tunnel101 | `show ip bgp`, `show ip route 0.0.0.0`, `show dmvpn`, `show ip eigrp neighbors` |
| DS1 lỗi | shutdown HQ-DS1 | HSRP VLAN 10 sang DS2; OSPF hội tụ; ping HQ-PC không đứt lâu | `show standby brief`, `show ip ospf neighbor` |
| Spoke ngắt | shutdown BR1-R e0/0 | Hub thấy BR1 `DOWN`; BR2 không ảnh hưởng | `show dmvpn`, `show ip eigrp neighbors` |
| Hub lỗi | shutdown HQ-R1 | Chi nhánh ra Internet bình thường (BGP độc lập); overlay phụ thuộc hub dự phòng | `show dmvpn`, ping qua Tunnel101 |
| Khôi phục | bật lại tất cả | Mọi thứ về trạng thái golden | chạy lại toàn bộ test suite |

**(F) pyATS.** **Snapshot trước/sau lỗi + diff.** Quy trình: snapshot → gây lỗi → đợi → snapshot → diff → đối chiếu với kỳ vọng. Loại các trường dao động (uptime, counter, timestamp) khỏi diff. Ghi lại thời gian từ lúc gây lỗi đến khi test pass.

**(G) Pitfalls.** Dual-hub nhưng quên tag → loop route; spoke chọn hub không ưu tiên do metric/delay; HSRP preempt gây flap khi test lặp; IP SLA test dùng địa chỉ không đi qua đúng ISP.

**(H) Commit.** `configs/phase7/`, `docs/test-results.md` (bảng thời gian hội tụ), `assurance/tests/test_failover.py`, `assurance/lib/compare.py`.

---

### Phase 8 — Hoàn thiện nền tảng assurance và trình bày

**(A) Mục tiêu.** Gộp mọi test thành một bộ chạy được bằng một lệnh, sinh báo cáo, và đóng gói để đưa vào CV.

**(B) Việc cần làm.**
- Chuẩn hóa `intent.yaml` làm **nguồn chuẩn (source of truth)** cho mọi test (mục 6.2).
- Dùng **AEtest** để có pass/fail, ghi log có cấu trúc; chạy qua **job file** (`pyats run job`).
- Thu snapshot JSON có timestamp, so sánh với golden; xuất **HTML** (Jinja2 hoặc log của pyATS/Easypy) và **Markdown**; thu bản sao `show running-config` để kiểm soát thay đổi (đây là *đọc* cấu hình, không push).
- README có sơ đồ topology, bảng địa chỉ, bảng kết quả hội tụ, video demo ngắn (3–5 phút), hướng dẫn dựng lại lab (không kèm image).
- (Tùy chọn) lint YAML/Markdown/Python bằng pre-commit + GitHub Actions; không chạy lab trên CI, nhưng có thể chạy kiểm tra cú pháp/parse trên snapshot mẫu.

**(D) Nghiệm thu.** Chạy suite ở trạng thái bình thường → toàn pass. Gây một lỗi có chủ đích (shutdown tunnel, ngắt BGP peer) → báo cáo đánh dấu đúng thiết bị/đúng kiểm tra, kèm bằng chứng (output đã parse).

**(F) pyATS.** Đây là điểm hoàn thiện, không phải điểm bắt đầu — mọi test đã được viết dần từ Phase 0–7.

**(H) Commit.** Toàn bộ cấu trúc repo ở mục 10.

---

## 6. Đặc tả lớp Assurance (pyATS/Genie)

> Mọi đoạn code dưới đây là **khung (skeleton)**. Schema của Genie parser phụ thuộc phiên bản — luôn kiểm tra bằng `pyats parse "<lệnh>" --testbed-file testbed.yaml --devices <TÊN> --output out/` trước khi viết logic truy cập key. Đánh dấu **[CẦN XÁC MINH]**.

### 6.1 `testbed.yaml` (mẫu)

```yaml
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
        ssh_options: >-
          -o KexAlgorithms=+diffie-hellman-group14-sha1
          -o HostKeyAlgorithms=+ssh-rsa
          -o StrictHostKeyChecking=no
  # lặp lại cho HQ-R2 (.12), HQ-DS1 (.13), HQ-DS2 (.14), HQ-ACC (.15),
  # BR1-R (.21), BR2-R (.22), ISP-A (.31), ISP-B (.32)
```
Mật khẩu lấy từ biến môi trường (`export LAB_USER=... LAB_PASS=... LAB_ENABLE=...`), **không ghi cứng** vào file. `ssh_options` bật thuật toán cũ mà OpenSSH hiện đại tắt mặc định.

### 6.2 `intent.yaml` (mẫu — nguồn chuẩn cho trạng thái mong đợi)

```yaml
ospf:
  HQ-R1:  {neighbors: 2}
  HQ-R2:  {neighbors: 2}
  HQ-DS1: {neighbors: 3}
  HQ-DS2: {neighbors: 3}
hsrp:
  HQ-DS1: {10: Active, 20: Standby}
  HQ-DS2: {10: Standby, 20: Active}
stp_root:
  vlan10: HQ-DS1
  vlan20: HQ-DS2
bgp:
  HQ-R1:
    neighbors: {203.0.113.1: 65100, 10.255.0.2: 65010}
  HQ-R2:
    neighbors: {203.0.113.5: 65200, 10.255.0.1: 65010}
  BR1-R: {neighbors: {203.0.113.9: 65100}}
  BR2-R: {neighbors: {203.0.113.13: 65200}}
dmvpn:
  hub: HQ-R1
  spokes: {BR1-R: 172.16.100.11, BR2-R: 172.16.100.12}
eigrp:
  HQ-R1: {neighbors: 2}
redistribution:
  tags: {eigrp_to_ospf: 100, ospf_to_eigrp: 200}
ntp:
  server: 10.255.0.1
```

### 6.3 Ma trận kiểm tra

| Nhóm | Lệnh thu thập | Điều kiện PASS | Mức độ khi FAIL |
|---|---|---|---|
| HSRP | `show standby brief` | Mỗi group đúng 1 Active + 1 Standby; Active khớp intent | Critical |
| STP | `show spanning-tree vlan N` | Root bridge đúng thiết bị | Major |
| OSPF | `show ip ospf neighbor` | Mọi neighbor `FULL`; số lượng khớp intent | Critical |
| OSPF LSDB | `show ip ospf database` | Có LSA summary 10.10.0.0/16 | Major |
| BGP | `show ip bgp summary` | Mọi neighbor `Established` (có prefix count); AS đúng intent | Critical |
| Default route | `show ip route 0.0.0.0` | Tồn tại; next-hop đúng ISP ưu tiên | Critical |
| NAT | `show ip nat translations` | Có entry sau khi sinh traffic | Minor |
| DMVPN | `show dmvpn` | Hub thấy đủ spoke `UP`; spoke thấy hub `UP` | Critical |
| IPsec | `show crypto isakmp sa`, `show crypto ipsec sa` | `QM_IDLE`; counter encaps/decaps > 0 | Critical |
| EIGRP | `show ip eigrp neighbors` | Đủ neighbor theo intent | Critical |
| Redistribution | `show ip route` (hub/DS) | Route LAN chi nhánh mang tag 100; không có route NBMA qua tunnel | Major |
| NTP | `show ntp status` | `synchronized` | Minor |
| Reachability | ping ma trận (dùng `device.ping()` hoặc `ping` qua CLI) | Mọi cặp trong intent thông | Critical |

### 6.4 Khung AEtest (ví dụ BGP)

```python
# assurance/tests/test_bgp.py   — SKELETON [CẦN XÁC MINH schema]
import yaml
from pyats import aetest
from genie.metaparser.util.exceptions import SchemaEmptyParserError

class CommonSetup(aetest.CommonSetup):
    @aetest.subsection
    def load_intent_and_connect(self, testbed):
        self.parent.parameters["intent"] = yaml.safe_load(open("intent.yaml"))
        testbed.connect(log_stdout=False, learn_hostname=True)

class BgpEstablished(aetest.Testcase):
    @aetest.test
    def check(self, testbed, intent, steps):
        for name, exp in intent["bgp"].items():
            with steps.start(f"{name}", continue_=True) as step:
                dev = testbed.devices[name]
                try:
                    out = dev.parse("show ip bgp summary")
                except SchemaEmptyParserError:
                    step.failed(f"{name}: không có BGP neighbor nào")
                nbrs = out["vrf"]["default"]["neighbor"]
                for ip, asn in exp["neighbors"].items():
                    n = nbrs.get(ip)
                    if n is None:
                        step.failed(f"{name}: thiếu neighbor {ip}")
                    af = next(iter(n["address_family"].values()))
                    # state_pfxrcd là số prefix nếu Established, là tên trạng thái nếu chưa
                    if not str(af["state_pfxrcd"]).isdigit():
                        step.failed(f"{name}: {ip} ở trạng thái {af['state_pfxrcd']}")

class CommonCleanup(aetest.CommonCleanup):
    @aetest.subsection
    def disconnect(self, testbed):
        testbed.disconnect()

if __name__ == "__main__":
    from genie.testbed import load
    aetest.main(testbed=load("testbed.yaml"))
```

### 6.5 Snapshot và diff (ý tưởng)

```python
# assurance/lib/snapshot.py  — SKELETON
import json, time
from genie.metaparser.util.exceptions import SchemaEmptyParserError

COMMANDS = ["show ip ospf neighbor", "show ip bgp summary", "show standby brief",
            "show ip eigrp neighbors", "show dmvpn", "show crypto isakmp sa"]

def snapshot(testbed, outdir):
    ts = time.strftime("%Y%m%d-%H%M%S")
    for name, dev in testbed.devices.items():
        data = {}
        for cmd in COMMANDS:
            try:
                data[cmd] = dev.parse(cmd)
            except SchemaEmptyParserError:
                data[cmd] = {}
            except Exception as e:      # parser không có / lệnh lỗi
                data[cmd] = {"__error__": str(e)}
        with open(f"{outdir}/{name}-{ts}.json", "w") as f:
            json.dump(data, f, indent=2, default=str)
```
So sánh: dùng `genie.utils.diff.Diff(before, after, exclude=[...])` rồi `findDiff()` **[CẦN XÁC MINH API ở phiên bản bạn cài]**; luôn đưa uptime, counter, timer, timestamp vào danh sách loại trừ.

---

## 7. Pitfalls khi cấu hình DMVPN Phase 3 và IPsec

| # | Triệu chứng | Nguyên nhân thường gặp | Cách xử lý / kiểm tra |
|---|---|---|---|
| 1 | Spoke-to-spoke vẫn luôn đi qua hub | Thiếu `ip nhrp redirect` trên hub hoặc `ip nhrp shortcut` trên spoke (lúc đó chỉ là Phase 1/2) | Kiểm tra `show ip nhrp`, `show ip route nhrp`, `show dmvpn detail` có entry `D` giữa hai spoke |
| 2 | Tunnel flap, log `%TUN-5-RECURDOWN` | **Recursive routing:** địa chỉ NBMA (hoặc default) học qua chính tunnel (thường do redistribution/summary) | Không redistribute underlay/default vào overlay; `show ip route <NBMA>` phải đi qua interface physical |
| 3 | Spoke không đăng ký được với hub | Lệch `ip nhrp network-id`, `authentication`, `tunnel key`; sai NBMA của NHS | `show dmvpn`, `debug nhrp` (cẩn thận), so sánh config hai đầu |
| 4 | Ping nhỏ được, TCP/HTTP bị treo | MTU/MSS: GRE + IPsec thêm overhead; PMTUD bị chặn | `ip mtu 1400`, `ip tcp adjust-mss 1360`; test `ping size 1400 df-bit` |
| 5 | Spoke không học route LAN của spoke khác | EIGRP **split-horizon** trên hub; hoặc `next-hop-self` | Dùng summary từ hub (Phase 3) hoặc tắt split-horizon + `no ip next-hop-self eigrp` |
| 6 | Hub redistribute route `10.0.0.0/8` (Null0) vào OSPF | Route summary là route EIGRP trong RIB, bị `redistribute eigrp` kéo theo | Prefix-list `BRANCH-LANS` trong route-map `EIGRP-TO-OSPF` |
| 7 | Mất 1–2 gói ping đầu tiên | Hành vi bình thường: gói đầu qua hub, sau đó NHRP resolution | Không phải lỗi; thử ping lần hai |
| 8 | `show crypto isakmp sa` trống hoặc `MM_NO_STATE` | PSK sai; policy ISAKMP lệch; ACL/NAT chặn UDP 500/ESP/GRE; NBMA không thông | Ping NBMA trước; so khớp policy; dùng wildcard key; kiểm tra `show crypto isakmp policy` |
| 9 | Có SA nhưng counter `encaps/decaps` không tăng hoặc tăng một chiều | Routing/NHRP chưa đúng; asymmetric; transform-set/mode lệch (`transport` vs `tunnel`) | `show crypto ipsec sa`, kiểm tra hai đầu cùng `mode transport` |
| 10 | NAT phá tunnel | NAT áp lên traffic tunnel hoặc spoke đứng sau NAT | Không đánh dấu Tunnel100 là `ip nat outside`; nếu spoke sau NAT thì cần NAT-T (mặc định bật ở IKEv1 khi phát hiện NAT) và NHRP NAT-aware |
| 11 | Chạy đúng nhưng hội tụ chậm | Hello/hold EIGRP mặc định trên mGRE; không có BFD; NHRP registration timer | Tinh chỉnh timer có kiểm soát; ghi lại số liệu đo thực |
| 12 | Hành vi lạ dù config đúng | Image IOL lỗi/khác phiên bản; thiếu `ip cef` | Bật `ip cef`; thử đổi image IOL (nhánh 15.x) trước khi nghi ngờ thiết kế |
| 13 | Hub duy nhất trên một ISP | Thiết kế: ISP-A lỗi → mất overlay | Xử lý ở Phase 7 (dual-cloud/dual-hub) |
| 14 | Route loop sau khi thêm hub thứ hai | Mutual redistribution ở hai điểm không có tag | Gắn tag khi vào, deny theo tag khi ra, prefix-list; kiểm tra `show ip route <prefix>` tag |

**Thứ tự debug khuyến nghị:** underlay (ping NBMA) → GRE/NHRP (`show dmvpn`) → IPsec (`show crypto ...`) → IGP overlay (`show ip eigrp neighbors`) → redistribution (`show ip route`, tag) → ứng dụng (ping/traceroute từ host).

---

## 8. Pitfalls khi chạy pyATS với GNS3 trên Windows

| # | Vấn đề | Giải thích | Cách xử lý |
|---|---|---|---|
| 1 | pyATS không chạy native trên Windows | Hỗ trợ chính thức: Linux/macOS | Dùng WSL2, VM Linux, hoặc container; dùng virtualenv; kiểm tra phiên bản Python mà bản pyATS hỗ trợ |
| 2 | WSL2 không ping/SSH được GNS3 VM | WSL2 mặc định NAT, khác dải host-only/VMnet của GNS3 VM | Bật mirrored networking (nếu phiên bản WSL hỗ trợ), thêm route, hoặc dùng VM Linux riêng cùng dải mạng. **Luôn ping/ssh thủ công trước** |
| 3 | Cloud node nối card mạng Windows không ổn định | Bridge ra Wi-Fi thường không hoạt động | Ưu tiên host-only của GNS3 VM hoặc node Linux trong GNS3 làm bastion |
| 4 | SSH bị từ chối dù đúng mật khẩu | IOS cũ chỉ có thuật toán KEX/host key cũ mà OpenSSH hiện đại tắt | RSA ≥ 2048, `ip ssh version 2`; thêm `ssh_options` (`+diffie-hellman-group14-sha1`, `+ssh-rsa`) |
| 5 | Dùng telnet console: prompt/cổng lung tung | Cổng console đổi khi restart; banner và syslog chen vào output | Dùng SSH qua OOB; `logging synchronous`; `exec-timeout 0 0` |
| 6 | `SchemaEmptyParserError` | Lệnh trả rỗng (ví dụ chưa có BGP neighbor) | Bắt exception, coi là kết quả hợp lệ ("không có") |
| 7 | Không có parser cho một lệnh | Độ phủ parser khác nhau theo phiên bản/OS | Viết parser riêng (regex/TextFSM) cho `show dmvpn`/`show ip nhrp` nếu cần |
| 8 | `learn` rất chậm | Chạy nhiều lệnh một lúc, IOL xử lý chậm | Ưu tiên `parse` từng lệnh cần thiết |
| 9 | GNS3 VM nghẽn khi kết nối song song | Nhiều SSH đồng thời | Giới hạn song song (pool size), tăng timeout |
| 10 | Diff snapshot nhiễu | uptime, counter, timer luôn thay đổi | Danh sách `exclude` cố định, lưu trong repo |
| 11 | Lệnh chạy khi router chưa boot xong | IOL khởi động chậm | Chờ ổn định; thêm bước kiểm tra sẵn sàng trong `CommonSetup` |
| 12 | Mất cấu hình khi restart | Quên `write memory`; không lưu project GNS3 | `write memory` sau mỗi bước; snapshot GNS3 mỗi phase; commit `configs/` |
| 13 | Rò rỉ mật khẩu | Ghi cứng trong testbed/script | Biến môi trường (`%ENV{...}`), `.env.example` không chứa giá trị thật, `.gitignore` đúng |
| 14 | Sai giờ giữa thiết bị | Không có NTP | Hoàn thiện Phase 6 trước khi đối chiếu log/snapshot |

---

## 9. Đối chiếu ENARSI (300-410) và hướng mở rộng

**Đã bao phủ trong kế hoạch:** route-map, prefix-list, tag, redistribution và chống loop, administrative distance (khi so sánh route), EIGRP trên overlay, OSPF multi-area + summarization, eBGP/iBGP + policy cơ bản, DMVPN, NAT/PAT, DHCP, NTP, Syslog; hardening cơ bản (ACL VTY, tắt dịch vụ thừa).

**Chưa bao phủ (có thể thêm có chọn lọc):** PBR; VRF-Lite (ví dụ cho OOB); BFD; IPv6 routing (OSPFv3, EIGRP IPv6, BGP IPv6); BGP nâng cao (AS-path prepend chiều vào, community, MED); Infrastructure Security (uRPF, AAA đầy đủ, CoPP); IP SLA/tracking (đã đưa vào Phase 7); SNMP; MPLS L3VPN.

**Thứ tự ưu tiên mở rộng gợi ý:** (1) IP SLA + tracking + BFD (Phase 7) → (2) BGP policy chiều vào → (3) PBR → (4) VRF-Lite cho OOB → (5) hardening Edge (uRPF/ACL) → (6) IPv6 nhẹ → (7) MPLS L3VPN (cuối cùng, nặng tài nguyên).

Nguồn đối chiếu blueprint: các trang đào tạo tổng hợp mục thi 300-410; người dùng nên xác nhận lại trên trang chính thức của Cisco vì blueprint có thể thay đổi theo phiên bản.

---

## 10. Cấu trúc repo đề xuất

```
ccnp-multisite-assurance/
├── README.md                    # tổng quan, sơ đồ, cách dựng lại, bảng kết quả hội tụ
├── docs/
│   ├── topology.png
│   ├── addressing.md            # bảng ở mục 4
│   ├── design-decisions.md      # D1–D9 và lý do
│   ├── redistribution-design.md # luồng route, tag, prefix-list
│   └── test-results.md          # thời gian hội tụ từng kịch bản
├── gns3/                        # ghi chú import project; KHÔNG chứa image Cisco
├── configs/
│   ├── phase0/ … phase7/        # <hostname>.cfg đã xác nhận chạy
├── assurance/
│   ├── testbed.yaml             # không có mật khẩu
│   ├── intent.yaml
│   ├── jobs/assurance_job.py
│   ├── tests/                   # test_hsrp.py, test_ospf.py, test_bgp.py, test_dmvpn.py,
│   │                            # test_ipsec.py, test_eigrp.py, test_redistribution.py,
│   │                            # test_services.py, test_failover.py
│   ├── lib/                     # snapshot.py, compare.py, report.py
│   ├── templates/report.html.j2
│   ├── snapshots/               # (gitignore, giữ vài mẫu golden)
│   └── reports/                 # HTML/Markdown sinh ra
├── .env.example                 # liệt kê biến cần có, không có giá trị thật
├── .gitignore
├── .pre-commit-config.yaml      # yamllint, markdownlint, ruff/flake8
└── .github/workflows/lint.yml
```

**Vệ sinh git:** không commit image IOL/IOSv, mật khẩu, PSK, token; ghi nguồn nếu tham khảo ý tưởng từ repo khác (và tôn trọng giấy phép của repo đó; nếu repo không có LICENSE thì mặc định tác giả giữ toàn bộ bản quyền → không chép nguyên văn).

---

## 11. Câu hỏi mở và việc tiếp theo

**Thông tin còn thiếu (agent nên hỏi người dùng):**
1. Phiên bản/loại image IOL L2 và L3 người dùng có (tên file), image L2 có hỗ trợ SVI/HSRP không?
2. Cấu hình máy (RAM/CPU), dùng VMware hay VirtualBox cho GNS3 VM; WSL2 hay VM Linux riêng cho pyATS?
3. Có muốn làm dual-cloud ở Phase 7 hay chỉ dừng ở hub đơn?
4. Phiên bản Python/pyATS đã cài (nếu có).
5. Có muốn thêm các mục mở rộng ở mục 9 hay giữ nguyên phạm vi?

**Bước tiếp theo đề xuất:** làm Phase 0 (dựng GNS3 + OOB + pyATS hello world), rồi Phase 1 (VLAN/RSTP/HSRP). Với mỗi phase, người dùng dán output `show ...` để agent đối chiếu tiêu chí nghiệm thu, đồng thời viết test pyATS tương ứng.

---

## Phụ lục A — Bản thảo gốc (thảo luận của người dùng với Gemini), giữ nguyên nội dung

> *Chú ý: đây là tham khảo ban đầu, một số điểm đã được điều chỉnh ở mục 3 (xem nhận xét ở mục 2, lượt 3 và lượt 5).*

Chúc mừng bạn đã hoàn thành CCNA và bước sang lộ trình CCNP. Với nền tảng đã có về Python và dự án NetDevOps (tập trung vào Day 0/Day 1 provisioning và OOB management) trước đó, CV của bạn cần một dự án tiếp theo nhấn mạnh vào **Kiến trúc mạng (Architecture), Định tuyến nâng cao (Advanced Routing) và Tự động hóa kiểm định (Network Assurance)**.
Dưới đây là một đề án cấp độ CCNP Enterprise, thiết kế tối ưu cho GNS3 trên Windows Desktop và hoàn toàn khác biệt với dự án cấu hình tự động trước đó.

**Tên dự án: Thiết kế Kiến trúc Mạng Đa Chi nhánh (Multi-Site) & Tự động hóa Kiểm định Trạng thái (Automated Network Assurance)**

Dự án này mô phỏng một doanh nghiệp có Trụ sở chính (HQ) và 2 chi nhánh (Branch), yêu cầu tính sẵn sàng cao, bảo mật đường truyền qua Internet và khả năng tự động kiểm tra xem mạng có đang hoạt động đúng thiết kế hay không.

**1. Kiến trúc Topology trong GNS3**

Để GNS3 trên Windows chạy mượt mà, không xung đột ảo hóa (nested virtualization), bạn nên dùng các image nhẹ như **Cisco IOU/IOL (L2/L3)** hoặc **Dynamips c7200**.

- **Vùng HQ (Trụ sở chính):** 2 Core Switch (chạy HSRP/VRRP, STP tinh chỉnh), 2 Router biên (Edge) kết nối ra hai ISP khác nhau để dự phòng. Phía sau là cụm Server (sử dụng các máy ảo Linux nhẹ như Alpine/Debian) đóng vai trò Web/DNS/Syslog.
- **Vùng WAN/ISP:** Mô phỏng môi trường Internet bằng một vài Router chạy BGP để bơm default route xuống các site.
- **Vùng Branch 1 & 2:** Mỗi chi nhánh có 1 Router biên và các Access Switch cấp mạng xuống cho end-user.

**2. Các công nghệ cốt lõi triển khai (Bám sát CCNP ENCOR & ENARSI)**

- **Định tuyến nâng cao (Advanced Routing):**
  - Chạy **OSPF multi-area** nội bộ tại HQ.
  - Chạy **eBGP** giữa các Edge Router và ISP để học default route và quảng bá dải IP public.
  - Sử dụng **Route Redistribution** (phân phối lại tuyến) giữa OSPF và BGP, áp dụng Route-map và Prefix-list để chống loop (routing loop) và tối ưu đường đi.
- **Bảo mật & VPN (Security & Overlay):**
  - Triển khai **DMVPN (Dynamic Multipoint VPN) Phase 3** kết hợp **IPsec** để tạo đường hầm bảo mật trực tiếp giữa HQ và các Branch, cũng như giữa Branch với Branch (Spoke-to-Spoke) qua môi trường Internet.
  - Chạy **EIGRP hoặc iBGP** qua đường hầm DMVPN để trao đổi thông tin định tuyến nội bộ của doanh nghiệp.
- **Kiến thức bên lề (Hệ thống & Dịch vụ):**
  - Thiết lập NAT (PAT) tại các Edge Router.
  - Cấu hình DHCP, NTP và đẩy log hệ thống về một Syslog Server tập trung.

**3. Tích hợp Python: Network Assurance & State Validation**

Thay vì dùng Python để tự động *đẩy cấu hình* (như dự án NetDevOps cũ), bạn sẽ dùng Python để **kiểm tra sức khỏe và trạng thái mạng (Network Assurance)**. Trong môi trường thực tế, sau khi cấu hình hoặc có sự cố, Network Engineer cần script để thẩm định nhanh hệ thống.

- **Công cụ:** Sử dụng thư viện **Cisco pyATS / Genie** (công cụ chuyên dụng để test mạng của Cisco) kết hợp với Netmiko.
- **Kịch bản Script:**
  - Script tự động đăng nhập vào toàn bộ Router/Switch lấy snapshot trạng thái mạng (Routing table, BGP neighbors, OSPF neighbors, HSRP state, IPsec Tunnel status).
  - Phân tích cú pháp (parse) output từ CLI thành định dạng JSON/Dictionary.
  - Viết logic so sánh: Tự động cảnh báo đỏ nếu BGP neighbor bị "Idle/Active" thay vì "Established", hoặc nếu tunnel DMVPN bị "Down", hoặc HSRP bị mất Active router.
  - Xuất kết quả kiểm tra ra một file báo cáo Markdown hoặc HTML tự động.

**Tài liệu tham khảo & Hướng dẫn**

1. **Về GNS3 & IOU/IOL:** Tham khảo tài liệu chính thức từ trang chủ GNS3 về việc cài đặt *GNS3 VM* trên VMware Workstation để chạy các image IOU/IOL mượt nhất trên Windows: docs.gns3.com.
2. **Về CCNP Routing & DMVPN:**
   - *CCNP and CCIE Enterprise Core ENCOR 350-401 Official Cert Guide* (Chương BGP và OSPF).
   - *CCNP Enterprise Advanced Routing ENARSI 300-410 Official Cert Guide* (Chương DMVPN và Route Redistribution).
   - Kênh Youtube *NetworkLessons* hoặc *Jeremy's IT Lab* (rất chi tiết về DMVPN và IPsec).
3. **Về Python pyATS / Genie:**
   - Trang chủ tài liệu pyATS của Cisco: pubhub.devnetcloud.com/media/pyats/docs/
   - Khóa học miễn phí trên Cisco DevNet: *Introduction to pyATS*.

Dự án này đưa vào CV sẽ thể hiện bạn không chỉ biết cấu hình thiết bị (CCNA), hiểu sâu về kiến trúc định tuyến và mạng diện rộng (CCNP), mà còn biết áp dụng lập trình vào việc vận hành và giảm thiểu rủi ro cho hệ thống thực tế.

**Những điểm trong bản thảo đã được điều chỉnh ở thiết kế cuối:** (1) c7200 không làm được Core Switch → dùng IOL L2/L3; (2) redistribution OSPF–BGP toàn phần → thay bằng default-originate + redistribution EIGRP↔OSPF có kiểm soát ở hub; (3) bổ sung iBGP giữa hai Edge, chính sách local-pref, kịch bản lỗi và failover; (4) pyATS chạy trên Linux/WSL2, không native trên Windows.

---

## Phụ lục B — Ghi chú về repo tham khảo (`cismembrane/gns3-labs`)

> Repo của người khác, chỉ để tham khảo ý tưởng tổ chức. Nội dung dưới đây là **tóm tắt bằng lời của trợ lý** từ hai file README người dùng cung cấp, không sao chép nguyên văn. GitHub chặn truy cập tự động, nên chưa đọc được mã nguồn/config bên trong repo.

- **Nội dung:** tập hợp lab GNS3 cho CCNP ENCOR (routing, switching, VPN, first-hop redundancy) kết hợp Ansible. Các lab: `bgp-grafana-monitoring`, `ansible-bgp`, `dmvpn-phase3-ipsec`, `ansible-ospf`, `ansible-lab`, `redundant-ospf-hsrp-eem-dhcp`, `eigrp-delay-manipulation`, `glbp-basic`, `proxy-arp`; bản README mới hơn thêm `k8s-metallb-bgp`. Thư mục `restconf-netconf-scripts` chứa script Python RESTCONF/NETCONF cho IOS XE (DevNet sandbox).
- **Điểm học được về cách tổ chức:** mỗi lab một thư mục có README (topology, địa chỉ, cấu hình, cách kiểm tra) và `configs/`; tách playbook deploy và verify, verify dùng `assert` để fail khi BGP neighbor Idle/Active hoặc OSPF không có neighbor FULL; mật khẩu mã hóa bằng ansible-vault với file `.env.example`/`.vault_pass.example`; CI lint (yamllint, ansible-lint, markdownlint) và kiểm tra cú pháp playbook, kèm pre-commit; bảng ghi rõ image/nền tảng theo từng lab (IOSv/IOSvL2 qua QEMU và c7200/c3660 qua Dynamips).
- **Cách dùng đúng:** học cấu trúc và thói quen kỹ thuật; **tự thiết kế** địa chỉ, AS, config và giải thích bằng lời của mình; ghi nguồn khi lấy ý tưởng; không chép nguyên văn nếu repo không có giấy phép cho phép.

---

*Hết tài liệu. Nếu cập nhật thiết kế (đổi địa chỉ, thêm/bớt phase), hãy sửa các mục 3–5 và bảng ở mục 4 trước, vì các phần còn lại tham chiếu tới chúng.*
