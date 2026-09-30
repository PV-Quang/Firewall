# RUNBOOK – Cấu hình HA 2 OPNsense trên VMware Cloud Director

## 1. Mục tiêu

Tài liệu này hướng dẫn triển khai mô hình **High Availability Active/Passive** cho hai firewall OPNsense chạy trên VMware Cloud Director.

Giải pháp sử dụng:

- **CARP** để cung cấp Virtual IP trên LAN và WAN.
- **pfsync** để đồng bộ firewall states.
- **XMLRPC Sync** để đồng bộ cấu hình từ Primary sang Backup.
- **Unicast CARP** để phù hợp với môi trường VMware Cloud Director / NSX.
- **MAC Learning / MAC Discovery** ở lớp VMware networking để CARP virtual MAC hoạt động đúng.
- Một network riêng cho pfsync.

Mục tiêu sau triển khai:

- Client LAN sử dụng một gateway duy nhất.
- Không cần thay đổi default gateway khi firewall failover.
- LAN và WAN VIP được chuyển từ Primary sang Backup khi Primary mất hoàn toàn.
- Firewall state được đồng bộ giữa hai node.
- Có thể sử dụng WAN CARP VIP làm Source NAT IP chung sau khi WAN VIP đã được provider network hỗ trợ đầy đủ.

---

# 2. Thiết kế IP

## 2.1. OPNsense-01 – Primary

| Interface | IP Address | Chức năng |
|---|---|---|
| WAN | `61.14.236.216/26` | WAN node Primary |
| LAN | `30.0.0.1/24` | LAN node Primary |
| PFSYNC | `40.0.0.11/24` | HA state/config synchronization |

## 2.2. OPNsense-02 – Backup

| Interface | IP Address | Chức năng |
|---|---|---|
| WAN | `61.14.236.215/26` | WAN node Backup |
| LAN | `30.0.0.10/24` | LAN node Backup |
| PFSYNC | `40.0.0.12/24` | HA state/config synchronization |

## 2.3. CARP Virtual IP

| Network | CARP VIP | VHID | CARP virtual MAC |
|---|---|---:|---|
| WAN | `61.14.236.231/26` | `1` | `00:00:5e:00:01:01` |
| LAN | `30.0.0.10/24` | `2` | `00:00:5e:00:01:02` |

## 2.4. Default gateway

WAN upstream gateway:

```text
61.14.236.193
```

LAN client gateway:

```text
30.0.0.1
```

---

# 3. Kiến trúc

```text
                              Internet
                                 |
                       Upstream Gateway
                        61.14.236.193
                                 |
             +-------------------+-------------------+
             |                                       |
        OPNsense-01                            OPNsense-02
        PRIMARY                               BACKUP
             |                                       |
      WAN 61.14.236.216                  WAN 61.14.236.215
             \                                       /
              +------ WAN CARP VIP ----------------+
                     61.14.236.231/26
                     VHID 1

      LAN 30.0.0.1                       LAN 30.0.0.10
             \                                       /
              +------ LAN CARP VIP ----------------+
                     30.0.0.10/24
                     VHID 2
                              |
                           Clients
                     GW = 30.0.0.10

      PFSYNC 40.0.0.11  <------->  PFSYNC 40.0.0.12
```

Mỗi OPNsense sử dụng **3 network adapter**:

```text
NIC 1 = WAN
NIC 2 = LAN
NIC 3 = PFSYNC
```

Không cần tạo NIC thứ tư cho WAN CARP VIP.

---

# 4. Chuẩn bị trên vCenter

> Phần này áp dụng cho các network/port group được quản lý trực tiếp bởi vSphere.
>
> Nếu network của VCD là **NSX-backed Segment**, không cấu hình MAC Learning trực tiếp trên vCenter port group để thay thế cho NSX Segment Profile. Với NSX-backed network, thực hiện thêm Phần B.

## Port-group security nếu network được quản lý trực tiếp bởi vCenter

Nếu WAN/LAN là Distributed Port Group hoặc Standard Port Group **không do NSX quản lý**, CARP có thể cần cho phép guest sử dụng virtual MAC khác với MAC được gán cho vNIC.

Trong vSphere Client:

```text
Networking
→ chọn Distributed Port Group
→ Configure / Edit Settings
→ Security
```

Áp dụng trên port group chuyên dụng cho OPNsense HA:

```text
MAC Address Changes = Accept
Forged Transmits    = Accept
```

Không bật `Promiscuous Mode` nếu không có yêu cầu cụ thể.

Nếu đang sử dụng MAC Learning ở Distributed Port Group:

```text
MAC Learning = Enabled
Forged Transmits = Accept
```

Không nên bật đồng thời:

```text
MAC Learning = Enabled
Promiscuous Mode = Accept
```

trên các phiên bản vSphere hiện đại vì đây không phải tổ hợp cấu hình được khuyến nghị.

### Khuyến nghị

Tạo dedicated Port Group chỉ dành cho HA firewall thay vì thay security policy của một port group dùng chung.

---

# 5. Chuẩn bị trên VMware Cloud Director / NSX

## 5.1. Tạo ba network

Cần ba network logic.

### WAN Network

WAN phải cung cấp L2 connectivity cho cả hai OPNsense.

Các IP sử dụng:

```text
OPNsense-01 = 61.14.236.216/26
OPNsense-02 = 61.14.236.215/26
CARP VIP    = 61.14.236.231/26

Gateway     = 61.14.236.193
```

WAN có thể là:

```text
Direct Org VDC Network
```

hoặc network tương đương được provider expose từ External Network.

WAN VIP `.231` phải:

- Thuộc subnet `/26`.
- Không được cấp cho VM khác.
- Không nằm trong conflict với IP pool đang sử dụng.
- Được upstream/provider network cho phép sử dụng.

---

## 5.2. LAN Network

Tạo một Isolated Org VDC Network:

```text
Name: OPN-LAN-HA
Subnet: 10.0.0.0/24
```

Gắn vào:

```text
OPNsense-01 LAN
OPNsense-02 LAN
Client / workload cần sử dụng firewall
```

Không cấu hình gateway VCD trên network này nếu OPNsense đóng vai trò gateway.

Gateway của workload sẽ là:

```text
30.0.0.1
```

---

## 5.3. PFSYNC Network

Tạo network riêng:

```text
Name: OPN-PFSYNC
Subnet: 40.0.0.0/24
Type: Isolated
```

Network này chỉ nên gắn vào:

```text
OPNsense-01 PFSYNC NIC
OPNsense-02 PFSYNC NIC
```

Không dùng network này cho workload khác.

---

## 5.4. Gắn network cho hai VM

### OPNsense-01

```text
NIC 1 → WAN Network
NIC 2 → OPN-LAN-HA
NIC 3 → OPN-PFSYNC
```

### OPNsense-02

```text
NIC 1 → WAN Network
NIC 2 → OPN-LAN-HA
NIC 3 → OPN-PFSYNC
```

NIC order phải giống nhau.

---

# 6. NSX Segment Profile cho CARP

> Áp dụng khi Org VDC Network của VCD được backed bởi NSX Segment.


## 6.1. Tạo MAC Discovery Profile

Trong NSX Manager:

```text
Networking
→ Segments
→ Segment Profiles
→ MAC Discovery
→ Add MAC Discovery Profile
```

Ví dụ:

```text
Name: OPNsense-CARP-MAC-Discovery
```

Thiết lập:

```text
MAC Change:               Yes
MAC Learning:             Yes
MAC Learning Aging Time:  600
Unknown Unicast Flooding: Yes
MAC Limit:                4096
MAC Limit Policy:         Allow
```

Trong môi trường đã triển khai, `Unknown Unicast Flooding = Yes` là cần thiết để traffic tới CARP vMAC trên LAN được forward đúng.

---

## 6.2. Gán profile vào LAN Segment

Trong NSX Manager:

```text
Networking
→ Segments
→ chọn Segment backing cho OPN-LAN-HA
→ Edit
→ Segment Profiles
```

Chọn:

```text
MAC Discovery Profile:
OPNsense-CARP-MAC-Discovery
```

Save.

---


# 7. Cấu hình LAN CARP VIP

## 7.1. Primary

Trên OPNsense-01:

```text
Interfaces
→ Virtual IPs
→ Settings
→ Add
```

Cấu hình:

```text
Mode / Type: CARP
Interface: LAN
Address: 30.0.0.10/24
VHID Group: 2
Password: <CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 0
Peer IPv4: 30.0.0.2
```

Save và Apply.

---

## 7.2. Backup

Trên OPNsense-02:

```text
Mode / Type: CARP
Interface: LAN
Address: 30.0.0.10/24
VHID Group: 2
Password: <same CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 100
Peer IPv4: 30.0.0.1
```

Save và Apply.

---

# 8. Cấu hình WAN CARP VIP

## 8.1. Primary

Trên OPNsense-01:

```text
Interfaces
→ Virtual IPs
→ Settings
→ Add
```

Cấu hình:

```text
Mode / Type: CARP
Interface: WAN
Address: 61.14.236.231/26
VHID Group: 1
Password: <CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 0
Peer IPv4: 61.14.236.215
```

---

## 8.2. Backup

Trên OPNsense-02:

```text
Mode / Type: CARP
Interface: WAN
Address: 61.14.236.231/26
VHID Group: 1
Password: <same CARP_SHARED_SECRET>
Advertising Base: 1
Advertising Skew: 100
Peer IPv4: 61.14.236.216
```

---

# 9. Cấu hình pfsync

## 9.1. Primary

Trên OPNsense-01:

```text
System
→ High Availability
→ Settings
```

Phần State Synchronization:

```text
Synchronize all states via: PFSYNC
Sync compatibility: OPNsense 24.7 or above
Synchronize Peer IP: 40.0.0.12
Defer pfsync: Disable
```

`Disable preempt`:

```text
Unchecked
```

---

## 9.2. Backup

Trên OPNsense-02:

```text
System
→ High Availability
→ Settings
```

Cấu hình:

```text
Synchronize all states via: PFSYNC
Sync compatibility: OPNsense 24.7 or above
Synchronize Peer IP: 40.0.0.11
```

Không cấu hình XMLRPC synchronization theo chiều Backup → Primary.

---

# 10. Cấu hình XMLRPC Sync

XMLRPC chỉ cấu hình trên Primary.

Trên OPNsense-01:

```text
System
→ High Availability
→ Settings
```

Phần:

```text
Configuration Synchronization Settings (XMLRPC Sync)
```

Cấu hình:

```text
Perform synchronization: Enabled
Synchronize Config: 40.0.0.12
Remote System Username: <HA_SYNC_USER>
Remote System Password: <PASSWORD>
```

Có thể dùng `root`, nhưng trong production nên sử dụng account riêng có quyền phù hợp nếu policy tổ chức cho phép.

Chọn các mục cần synchronize.

Khuyến nghị:

```text
Firewall Rules
NAT
Virtual IPs
Aliases
DHCP
IPsec
OpenVPN
Services cần thiết khác
```

Không cấu hình XMLRPC từ OPNsense-02 về OPNsense-01.

---

# 11. Cấu hình Preempt

Trên cả hai node:

```text
System
→ High Availability
→ Settings
```

Để:

```text
Disable preempt = Unchecked
```

Mục tiêu là node có priority cao hơn được phép trở lại MASTER khi hệ thống ổn định.

Primary sử dụng:

```text
AdvSkew = 0
```

Backup sử dụng:

```text
AdvSkew = 100
```

---

# 12. Cấu hình client LAN

Client phía LAN không sử dụng IP vật lý của từng firewall làm gateway.

Không sử dụng:

```text
30.0.0.1
30.0.0.2
```

Sử dụng:

```text
Default Gateway = 30.0.0.10
```

Ví dụ:

```text
IP Address:      30.0.0.100
Subnet Mask:     255.255.255.0
Default Gateway: 30.0.0.10
DNS:             theo thiết kế
```

---

# 13. Outbound NAT

Có hai giai đoạn triển khai.

## 13.1. Giai đoạn ban đầu

Có thể giữ Automatic Outbound NAT để xác nhận HA LAN/WAN và failover node.

Trong trường hợp này:

```text
Traffic qua Primary → public source 61.14.236.216
Traffic qua Backup  → public source 61.14.236.215
```

---

## 13.2. Giai đoạn production với shared WAN public IP (cấu hình trên cả 2 FW)

Sau khi WAN CARP VIP `61.14.236.231` đã được upstream/VCD/NSX hỗ trợ đầy đủ, chuyển sang:

```text
Firewall
→ NAT
→ Outbound
```

Chọn:

```text
Hybrid Outbound NAT
```

Tạo rule:

```text
Interface: WAN
TCP/IP Version: IPv4
Protocol: any
Source: 30.0.0.0/24
Destination: any
Translation / target: 61.14.236.231
```

Mục tiêu:

```text
Client
  ↓
30.0.0.10
  ↓
Active OPNsense
  ↓
SNAT
  ↓
61.14.236.231
  ↓
Internet
```

Khi failover:

```text
Primary → Backup
```

public source IP vẫn giữ:

```text
61.14.236.231
```

