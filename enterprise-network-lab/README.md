# README

# Enterprise Network Lab — Cisco Packet Tracer

Mục tiêu: Xây dựng hệ thống mạng dành cho doanh nghiệp với 2 tòa nhà giả định
Công nghệ sử dụng: DHCP,OSPF,Routing Inter-VLAN,HSRP, NTP.
Tóm tắt: Thiết kế hệ thống có dự phòng đầy đủ, định tuyến động liên tòa nhà, cấp IP tự động không mất phiên khi failover.

---

## Topology tổng quan

```mermaid
graph LR
  subgraph B1["  BUILDING 1  "]
    direction TB
    SW1["SW1\nVLAN 10 - HR"]:::acc
    SW2["SW2\nVLAN 20 - Sales"]:::acc
    SW3["SW3\nVLAN 30 - Finance"]:::acc
    MSW1["MSW1\nHSRP Active\npriority 110\n10.10.1.2/30"]:::active
    MSW2["MSW2\nHSRP Standby\npriority 100\n10.10.1.6/30"]:::standby
    SW1 --> MSW1
    SW2 --> MSW1
    SW3 --> MSW1
    SW1 --> MSW2
    SW2 --> MSW2
    SW3 --> MSW2
    MSW1 <--> MSW2
  end

  MSW1 --- |"Gi0/1\n10.10.1.0/30"| R1
  MSW2 --- |"Gi0/1\n10.10.1.4/30"| R1

  R1["R1\nOSPF / NTP Master\nSe0/0/0 DCE\n10.10.10.1/30"]:::router
  R1 <--> |"Serial\n10.10.10.0/30"| R2
  R2["R2\nOSPF\nSe0/0/0 DTE\n10.10.10.2/30"]:::router

  R2 --- |"Gi0/1\n10.10.2.0/30"| MSW3
  R2 --- |"Gi0/0\n10.10.2.4/30"| MSW4

  subgraph B2["  BUILDING 2  "]
    direction TB
    MSW3["MSW3\nHSRP Active\npriority 210\n10.10.2.2/30"]:::active
    MSW4["MSW4\nHSRP Standby\npriority 200\n10.10.2.6/30"]:::standby
    SW4["SW4\nVLAN 40 - HR"]:::acc
    SW5["SW5\nVLAN 50 - Sales"]:::acc
    SW6["SW6\nVLAN 60 - Finance"]:::acc
    MSW3 <--> MSW4
    MSW3 --> SW4
    MSW3 --> SW5
    MSW3 --> SW6
    MSW4 --> SW4
    MSW4 --> SW5
    MSW4 --> SW6
  end

  classDef active fill:#1e3a5f,stroke:#38bdf8,color:#e2e8f0
  classDef standby fill:#2a2d3e,stroke:#64748b,color:#e2e8f0
  classDef router fill:#1e3a2a,stroke:#4ade80,color:#e2e8f0
  classDef acc fill:#2a1f3a,stroke:#a78bfa,color:#e2e8f0
```

---

## Thiết bị

| Thiết bị | Vai trò | Ghi chú |
| --- | --- | --- |
| R1 | Router trung tâm B1, NTP master | DCE, clock rate 64000 |
| R2 | Router trung tâm B2 | DTE |
| MSW1 | Distribution B1, HSRP Active | priority 110, DHCP .11–.127 |
| MSW2 | Distribution B1, HSRP Standby | priority 100, DHCP .128–.254 |
| MSW3 | Distribution B2, HSRP Active | priority 210, DHCP .11–.127 |
| MSW4 | Distribution B2, HSRP Standby | priority 200, DHCP .128–.254 |
| SW1–SW3 | Access layer B1 | VLAN 10/20/30 |
| SW4–SW6 | Access layer B2 | VLAN 40/50/60 |

---

## Tính năng đã cấu hình

| Tính năng | Chi tiết |
| --- | --- |
| VLAN | 10/20/30 (B1 — HR/Sales/Finance), 40/50/60 (B2 — HR/Sales/Finance) |
| 802.1Q Trunk | Tất cả link switch–switch và switch–MSW |
| Inter-VLAN routing | SVI trên MSW + `ip routing` |
| HSRP | Dự phòng gateway, preempt bật trên cả 2 switch |
| OSPF Area 0 | Single area, passive interface trên SVI |
| WAN Serial | R1 (DCE) ↔︎ R2 (DTE), 64 Kbps |
| DHCP | Split-pool: primary .11–.127, standby .128–.254 |
| NTP | R1 làm master stratum 1, tất cả thiết bị sync về R1 |

---

## Cơ chế dự phòng

**HSRP** — Dự phòng gateway:

- MSW1/MSW3 là Active (priority cao hơn).
- MSW2/MSW4 là Standby, giám sát qua Hello packet mỗi 3 giây.
- Khi Active hỏng: Standby tự lên Active trong vòng 10 giây.
- Khi Active phục hồi: `preempt` giúp giành lại vai trò tự động.

**DHCP split-pool** — Dự phòng cấp IP:

- Cả hai switch đều chạy DHCP cùng lúc với dải không trùng nhau.
- Khi switch chính hỏng: switch dự phòng vẫn cấp IP bình thường từ dải của nó.
- Không có IP conflict, không mất DHCP khi failover.

**OSPF** — Định tuyến động:

- Tự động cập nhật bảng định tuyến khi topo thay đổi.
- Passive interface trên SVI: quảng bá subnet nhưng không gửi Hello xuống host.

---

## Cấu trúc thư mục

```
enterprise-network-lab/
├── README.md                        ← file này
├── packet-tracer/
│   └── INDIVIDUAL_LAB.pkt
├── configs/
│   ├── routers/
│   │   ├── R1.txt
│   │   └── R2.txt
│   ├── building1/
│   │   ├── access-switches/
│   │   │   ├── SW1.txt
│   │   │   ├── SW2.txt
│   │   │   └── SW3.txt
│   │   └── multilayer-switches/
│   │       ├── MSW1.txt
│   │       └── MSW2.txt
│   └── building2/
│       ├── access-switches/
│       │   ├── SW4.txt
│       │   ├── SW5.txt
│       │   └── SW6.txt
│       └── multilayer-switches/
│           ├── MSW3.txt
│           └── MSW4.txt
└── docs/
    ├── dia-chi-ip.md                ← topology chi tiết + bảng địa chỉ IP đầy đủ
    ├── hsrp.md                      ← cơ chế HSRP, cấu hình, kịch bản failover
    ├── ospf.md                      ← cơ chế OSPF, network statements, passive interface
    └── dhcp.md                      ← split-pool DHCP, lý do thiết kế, cách test
```

---

## Lệnh xác minh nhanh

```bash
# OSPF
show ip ospf neighbor          ! phải thấy 3 láng giềng FULL trên R1
show ip route ospf             ! kiểm tra route cross-building

# HSRP
show standby brief             ! trạng thái Active/Standby từng VLAN

# DHCP
show ip dhcp binding           ! danh sách IP đã cấp
show ip dhcp pool              ! dải địa chỉ, số lượng đã dùng

# Kết nối
ping 192.168.40.1 source vlan10   ! B1 VLAN10 → B2 VLAN40 gateway
```
