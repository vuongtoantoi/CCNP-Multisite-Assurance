Chúc mừng bạn đã hoàn thành CCNA và bước sang lộ trình CCNP. Với nền tảng đã có về Python và dự án NetDevOps (tập trung vào Day 0/Day 1 provisioning và OOB management) trước đó, CV của bạn cần một dự án tiếp theo nhấn mạnh vào **Kiến trúc mạng (Architecture), Định tuyến nâng cao (Advanced Routing) và Tự động hóa kiểm định (Network Assurance)**.  
Dưới đây là một đề án cấp độ CCNP Enterprise, thiết kế tối ưu cho GNS3 trên Windows Desktop và hoàn toàn khác biệt với dự án cấu hình tự động trước đó.

## **Tên dự án: Thiết kế Kiến trúc Mạng Đa Chi nhánh (Multi-Site) & Tự động hóa Kiểm định Trạng thái (Automated Network Assurance)**

Dự án này mô phỏng một doanh nghiệp có Trụ sở chính (HQ) và 2 chi nhánh (Branch), yêu cầu tính sẵn sàng cao, bảo mật đường truyền qua Internet và khả năng tự động kiểm tra xem mạng có đang hoạt động đúng thiết kế hay không.

### **1\. Kiến trúc Topology trong GNS3**

Để GNS3 trên Windows chạy mượt mà, không xung đột ảo hóa (nested virtualization), bạn nên dùng các image nhẹ như **Cisco IOU/IOL (L2/L3)** hoặc **Dynamips c7200**.

* **Vùng HQ (Trụ sở chính):** 2 Core Switch (chạy HSRP/VRRP, STP tinh chỉnh), 2 Router biên (Edge) kết nối ra hai ISP khác nhau để dự phòng. Phía sau là cụm Server (sử dụng các máy ảo Linux nhẹ như Alpine/Debian) đóng vai trò Web/DNS/Syslog.  
* **Vùng WAN/ISP:** Mô phỏng môi trường Internet bằng một vài Router chạy BGP để bơm default route xuống các site.  
* **Vùng Branch 1 & 2:** Mỗi chi nhánh có 1 Router biên và các Access Switch cấp mạng xuống cho end-user.

### **2\. Các công nghệ cốt lõi triển khai (Bám sát CCNP ENCOR & ENARSI)**

* **Định tuyến nâng cao (Advanced Routing):**  
  * Chạy **OSPF multi-area** nội bộ tại HQ.  
  * Chạy **eBGP** giữa các Edge Router và ISP để học default route và quảng bá dải IP public.  
  * Sử dụng **Route Redistribution** (phân phối lại tuyến) giữa OSPF và BGP, áp dụng Route-map và Prefix-list để chống loop (routing loop) và tối ưu đường đi.  
* **Bảo mật & VPN (Security & Overlay):**  
  * Triển khai **DMVPN (Dynamic Multipoint VPN) Phase 3** kết hợp **IPsec** để tạo đường hầm bảo mật trực tiếp giữa HQ và các Branch, cũng như giữa Branch với Branch (Spoke-to-Spoke) qua môi trường Internet.  
  * Chạy **EIGRP hoặc iBGP** qua đường hầm DMVPN để trao đổi thông tin định tuyến nội bộ của doanh nghiệp.  
* **Kiến thức bên lề (Hệ thống & Dịch vụ):**  
  * Thiết lập NAT (PAT) tại các Edge Router.  
  * Cấu hình DHCP, NTP và đẩy log hệ thống về một Syslog Server tập trung.

### **3\. Tích hợp Python: Network Assurance & State Validation**

Thay vì dùng Python để tự động *đẩy cấu hình* (như dự án NetDevOps cũ), bạn sẽ dùng Python để **kiểm tra sức khỏe và trạng thái mạng (Network Assurance)**. Trong môi trường thực tế, sau khi cấu hình hoặc có sự cố, Network Engineer cần script để thẩm định nhanh hệ thống.

* **Công cụ:** Sử dụng thư viện **Cisco pyATS / Genie** (công cụ chuyên dụng để test mạng của Cisco) kết hợp với Netmiko.  
* **Kịch bản Script:**  
  * Script tự động đăng nhập vào toàn bộ Router/Switch lấy snapshot trạng thái mạng (Routing table, BGP neighbors, OSPF neighbors, HSRP state, IPsec Tunnel status).  
  * Phân tích cú pháp (parse) output từ CLI thành định dạng JSON/Dictionary.  
  * Viết logic so sánh: Tự động cảnh báo đỏ nếu BGP neighbor bị "Idle/Active" thay vì "Established", hoặc nếu tunnel DMVPN bị "Down", hoặc HSRP bị mất Active router.  
  * Xuất kết quả kiểm tra ra một file báo cáo Markdown hoặc HTML tự động.

### **Tài liệu tham khảo & Hướng dẫn**

> 1. **Về GNS3 & IOU/IOL:**  
   * Tham khảo tài liệu chính thức từ trang chủ GNS3 về việc cài đặt *GNS3 VM* trên VMware Workstation để chạy các image IOU/IOL mượt nhất trên Windows: docs.gns3.com.  
> 2. **Về CCNP Routing & DMVPN:**  
   * *CCNP and CCIE Enterprise Core ENCOR 350-401 Official Cert Guide* (Chương BGP và OSPF).  
   * *CCNP Enterprise Advanced Routing ENARSI 300-410 Official Cert Guide* (Chương DMVPN và Route Redistribution).  
   * Kênh Youtube *NetworkLessons* hoặc *Jeremy's IT Lab* (rất chi tiết về DMVPN và IPsec).  
> 3. **Về Python pyATS / Genie:**  
   * Trang chủ tài liệu pyATS của Cisco: \[pubhub.devnetcloud.com/media/pyats/docs/\](https\://pubhub.devnetcloud.com/media/pyats/docs/).  
   * Khóa học miễn phí trên Cisco DevNet: *Introduction to pyATS*.

Dự án này đưa vào CV sẽ thể hiện bạn không chỉ biết cấu hình thiết bị (CCNA), hiểu sâu về kiến trúc định tuyến và mạng diện rộng (CCNP), mà còn biết áp dụng lập trình vào việc vận hành và giảm thiểu rủi ro cho hệ thống thực tế.