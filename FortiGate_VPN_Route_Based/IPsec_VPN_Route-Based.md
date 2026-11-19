# Hướng dẫn cấu hình Site-to-Site IPsec VPN Route-Based trên FortiGate

## 1. Mô hình tham khảo

```text
FortiGate Site A
----------------------------
WAN Interface : wan2
Local LAN     : 10.10.100.0/24
Public IP     : <Public IP FortiGate>

        Internet
           |
           | IPsec IKEv2
           |
Remote Peer Public IP
103.205.98.78

        VCD / NSX Edge
----------------------------
Remote LAN    : 192.168.10.0/24
```

### VTI

```text
FortiGate VTI : 169.254.100.1
VCD Edge VTI  : 169.254.100.2
VTI Network   : 169.254.100.0/30
```

Luồng:

```text
10.10.100.0/24
      |
      v
FortiGate
Vpn_Site_Tpcoms
169.254.100.1
      |
      | IPsec
      |
169.254.100.2
VCD / NSX Edge
      |
      v
192.168.10.0/24
```

---

## 2. Tạo IPsec Tunnel

Vào:

```text
VPN
→ IPsec Tunnels
→ Create New
→ IPsec Tunnel / Custom
```

Đặt tên:

```text
Vpn_Site_Tpcoms
```

### Network

```text
Remote Gateway : Static IP Address
IP Address     : 103.205.98.78
Interface      : wan2

Local Gateway  : Disable
Mode Config    : Disable

NAT Traversal  : Enable
Dead Peer Detection : On Demand
```

> `Local Gateway` chỉ cần bật nếu interface WAN có nhiều IP và cần ép VPN sử dụng một IP cụ thể.

---

## 3. Authentication

```text
Authentication Method : Pre-shared Key
Pre-shared Key        : <PSK>

IKE Version           : 2
Peer ID               : Any peer ID
```

PSK phải giống nhau ở hai đầu.

---

## 4. Phase 1 Proposal

```text
Encryption       : AES256
Authentication   : SHA256

Diffie-Hellman   : Group 14
Key Lifetime     : 86400 seconds

Local ID         : để trống
```

Khuyến nghị chỉ để một proposal trong giai đoạn triển khai:

```text
AES256-SHA256
DH14
```

Điều này giúp dễ troubleshoot hơn và tránh hai peer negotiate sang một proposal ngoài dự kiến.

---

## 5. Phase 2 Selector

Vào:

```text
Phase 2 Selectors
→ Edit
```

Trong mô hình hiện tại:

```text
Local Address  : 10.10.100.0/24
Remote Address : 192.168.10.0/24
```

### Phase 2 Proposal

```text
Encryption       : AES256
Authentication   : SHA256

Replay Detection : Enable

PFS              : Enable
DH Group         : 14

Local Port       : All
Remote Port      : All
Protocol         : All

Key Lifetime     : 43200 seconds
```

Chỉ giữ một proposal:

```text
AES256-SHA256
```

Xóa các proposal không sử dụng, ví dụ:

```text
AES128-SHA1
AES256-SHA1
AES128-SHA256
AES-GCM
CHACHA20
...
```

### Lưu ý về Phase 2 Selector

FortiGate route-based vẫn có thể dùng selector cụ thể:

```text
10.10.100.0/24 <-> 192.168.10.0/24
```

Nếu phía VCD/NSX yêu cầu hoặc cấu hình selector rộng:

```text
0.0.0.0/0 <-> 0.0.0.0/0
```

thì cần điều chỉnh FortiGate để hai đầu tương thích.

---

## 6. Gán IP cho VTI

FortiGate tạo tunnel interface:

```text
Vpn_Site_Tpcoms
```

Vào:

```text
Network
→ Interfaces
→ Vpn_Site_Tpcoms
→ Edit
```

Cấu hình:

```text
Addressing Mode : Manual

IP:
169.254.100.1

Remote IP/Network Mask:
169.254.100.2 / 255.255.255.252
```

Có thể bật:

```text
PING = Enable
```

để troubleshoot VTI.

Không cần bật các dịch vụ quản trị khác nếu không có nhu cầu:

```text
HTTPS
SSH
SNMP
```

---

## 7. Static Route

Vào:

```text
Network
→ Static Routes
→ Create New
```

Route tới LAN phía VCD:

### Cách 1 - Route trực tiếp qua tunnel interface

```text
Destination : 192.168.10.0/24
Interface   : Vpn_Site_Tpcoms
Gateway     : để trống
Distance    : 10
```

Luồng:

```text
192.168.10.0/24
        ↓
Vpn_Site_Tpcoms
```

### Cách 2 - Dùng VTI peer làm next-hop

```text
Destination : 192.168.10.0/24
Gateway     : 169.254.100.2
Interface   : Vpn_Site_Tpcoms
Distance    : 10
```

Luồng:

```text
192.168.10.0/24
        ↓
169.254.100.2
        ↓
Vpn_Site_Tpcoms
```

Với mô hình peer VCD/NSX Edge có VTI, cách 2 giúp topology và routing rõ ràng hơn.

---

## 8. Tạo Firewall Address Objects

Vào:

```text
Policy & Objects
→ Addresses
```

### Local LAN

```text
Name   : NET_LOCAL_10.10.100.0_24
Subnet : 10.10.100.0/24
```

### Remote LAN

```text
Name   : NET_REMOTE_192.168.10.0_24
Subnet : 192.168.10.0/24
```

---

## 9. Firewall Policy LAN → VPN

Vào:

```text
Policy & Objects
→ IPv4 Policy
→ Create New
```

Cấu hình:

```text
Name:
LAN_to_VPN_tpcoms

Incoming Interface:
User

Outgoing Interface:
Vpn_Site_Tpcoms

Source:
NET_LOCAL_10.10.100.0_24

Destination:
NET_REMOTE_192.168.10.0_24

Schedule:
always

Service:
ALL

Action:
ACCEPT

NAT:
Disable
```

> `User` là interface LAN trong lab hiện tại. Nếu môi trường khác thì chọn đúng interface chứa subnet local.

---

## 10. Firewall Policy VPN → LAN

```text
Name:
VPN_tpcoms_to_LAN

Incoming Interface:
Vpn_Site_Tpcoms

Outgoing Interface:
User

Source:
NET_REMOTE_192.168.10.0_24

Destination:
NET_LOCAL_10.10.100.0_24

Schedule:
always

Service:
ALL

Action:
ACCEPT

NAT:
Disable
```

Quan trọng:

```text
NAT = OFF
```

ở cả hai chiều.

Trong lab có thể để `Service = ALL`; production nên giới hạn theo đúng port/protocol cần sử dụng.

---

## 11. Tổng hợp thông số cấu hình

| Thành phần | Giá trị |
|---|---|
| VPN Type | Route-Based |
| Tunnel | `Vpn_Site_Tpcoms` |
| WAN Interface | `wan2` |
| Remote Gateway | `103.205.98.78` |
| IKE | IKEv2 |
| Authentication | PSK |
| Phase 1 Encryption | AES256 |
| Phase 1 Authentication | SHA256 |
| Phase 1 DH | 14 |
| Phase 1 Lifetime | 86400s |
| Phase 2 Encryption | AES256 |
| Phase 2 Authentication | SHA256 |
| PFS | Enable |
| Phase 2 DH | 14 |
| Phase 2 Lifetime | 43200s |
| Local LAN | `10.10.100.0/24` |
| Remote LAN | `192.168.10.0/24` |
| FortiGate VTI | `169.254.100.1` |
| VCD Edge VTI | `169.254.100.2` |
| VTI Network | `169.254.100.0/30` |
| NAT on VPN Policy | Disable |

---

## 12. Routing phía VCD/NSX Edge

Đầu VCD/NSX Edge sẽ cấu hình chiều ngược lại:

```text
Destination:
10.10.100.0/24

Next-hop:
169.254.100.1
```

Mô hình:

```text
                IKEv2 / IPsec
              AES256-SHA256
                   DH14
                     |
                     |
10.10.100.0/24       |                  192.168.10.0/24
      |              |                         |
      v              |                         v
+------------+       |                  +--------------+
| FortiGate  |=======|==================| VCD NSX Edge |
+------------+                          +--------------+
   VTI .1                                  VTI .2
169.254.100.1                           169.254.100.2
        \___________________________________/
                   169.254.100.0/30
```

---

## 13. Kiểm tra sau khi cấu hình

### GUI

Vào:

```text
VPN
→ IPsec Monitor
```

Tunnel cần hiển thị trạng thái UP.

### CLI - Tunnel summary

```bash
get vpn ipsec tunnel summary
```

### CLI - Chi tiết tunnel

```bash
diagnose vpn tunnel list name Vpn_Site_Tpcoms
```

### CLI - Kiểm tra IKE

```bash
diagnose vpn ike gateway list
```

### CLI - Kiểm tra route

```bash
get router info routing-table details 192.168.10.1
```

Kỳ vọng route đi qua:

```text
Vpn_Site_Tpcoms
```

### Ping VTI

```bash
execute ping-options source 169.254.100.1
execute ping 169.254.100.2
execute ping-options reset
```

Nếu VTI ping được thì test tiếp traffic LAN-to-LAN:

```text
10.10.100.x
→ 192.168.10.x
```

---

## 14. Troubleshooting nhanh

### Kiểm tra packet

```bash
diagnose sniffer packet any 'host 10.10.100.10 and host 192.168.10.10' 4 0 l
```

### Debug flow

```bash
diagnose debug reset
diagnose debug flow filter clear
diagnose debug flow filter saddr 10.10.100.10
diagnose debug flow show function-name enable
diagnose debug flow trace start 100
diagnose debug enable
```

Dừng debug:

```bash
diagnose debug disable
diagnose debug reset
```

### Thứ tự kiểm tra khi VPN không chạy

```text
1. Phase 1 / IKE
2. Phase 2
3. VTI IP
4. Static Route
5. Firewall Policy
6. NAT phải Disable
7. Routing phía VCD/NSX
8. Firewall phía VCD/NSX
```

---

## 15. Best Practice Production

- Dùng **IKEv2**.
- Ưu tiên **AES256 + SHA256** hoặc AES-GCM nếu cả hai đầu hỗ trợ và đã thống nhất.
- Tránh SHA1/DH Group 2/5 cho triển khai mới.
- Chỉ cho phép đúng local/remote subnet cần thiết.
- Tắt NAT trên site-to-site VPN.
- Bật logging cho firewall policy trong giai đoạn triển khai/troubleshoot.
- Dùng PSK đủ dài và ngẫu nhiên; không lưu PSK plaintext trong tài liệu dùng chung.
- Nếu hệ thống hỗ trợ certificate-based authentication thì nên cân nhắc cho production quy mô lớn.
- Có thể tạo blackhole route cho remote subnet với administrative distance cao hơn để tránh traffic đi nhầm default route khi tunnel down.
- Sau khi tunnel hoạt động ổn định, giới hạn `Service = ALL` về đúng port/protocol cần thiết.
