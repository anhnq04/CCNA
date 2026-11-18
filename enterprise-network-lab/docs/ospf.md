# OSPF — Định tuyến động

## Tổng quan

Lab dùng **OSPF single area (Area 0)** để cung cấp định tuyến động giữa hai tòa nhà. Tất cả thiết bị Layer 3 (R1, R2, MSW1–MSW4) đều tham gia OSPF. Switch access (SW1–SW6) chỉ hoạt động ở Layer 2, không chạy OSPF.

---

## Sơ đồ quan hệ láng giềng

```
MSW1 ──10.10.1.0/30──  R1 ──10.10.10.0/30──  R2 ──10.10.2.0/30──  MSW3
MSW2 ──10.10.1.4/30──┘                           └──10.10.2.4/30──  MSW4
```

Mỗi link /30 là một adjacency riêng. OSPF hội tụ xong thì R1 biết đường đến cả VLAN 40/50/60 bên Building 2, và ngược lại.

---

## Network statements

### R1
```
router ospf 1
 network 10.10.1.0 0.0.0.3 area 0     ! nối MSW1
 network 10.10.1.4 0.0.0.3 area 0     ! nối MSW2
 network 10.10.10.0 0.0.0.3 area 0    ! WAN serial đến R2
```

### R2
```
router ospf 1
 network 10.10.2.0 0.0.0.3 area 0     ! nối MSW3
 network 10.10.2.4 0.0.0.3 area 0     ! nối MSW4
 network 10.10.10.0 0.0.0.3 area 0    ! WAN serial đến R1
```

### MSW1 (Building 1 Active)
```
router ospf 1
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Vlan30
 network 10.10.1.0 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

### MSW2 (Building 1 Standby) — tương tự, dùng subnet /30 riêng
```
router ospf 1
 passive-interface Vlan10
 passive-interface Vlan20
 passive-interface Vlan30
 network 10.10.1.4 0.0.0.3 area 0
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

MSW3 và MSW4 cấu hình tương tự với subnet 10.10.2.x.

---

## Passive interface là gì và tại sao cần?

```
passive-interface Vlan10
```

Nếu không có lệnh này, OSPF sẽ gửi gói Hello ra tất cả interface đang được khai báo trong `network` — kể cả SVI VLAN 10, 20, 30. Điều đó có nghĩa là switch sẽ liên tục gửi Hello xuống phía host, trong khi host không chạy OSPF và không bao giờ trả lời. Vừa tốn băng thông, vừa sinh log lỗi không cần thiết.

Lệnh `passive-interface` trên SVI:
- **Ngăn** gửi/nhận Hello trên cổng đó (không có adjacency với host).
- **Vẫn quảng bá** subnet của SVI đó vào OSPF để các thiết bị khác biết đường đến VLAN đó.

---

## Kết quả xác minh (đã test trên lab)

`show ip ospf neighbor` trên R1:
```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.1.2         1   FULL/DR         00:00:39    10.10.1.2       GigabitEthernet0/1
10.10.1.6         1   FULL/DR         00:00:35    10.10.1.6       GigabitEthernet0/0
10.10.10.2        0   FULL/  -        00:00:39    10.10.10.2      Serial0/0/0
```

3 láng giềng FULL:
- R1 ↔ MSW1 (Gi0/1)
- R1 ↔ MSW2 (Gi0/0)
- R1 ↔ R2 (Se0/0/0)

---

## Route quảng bá

Sau khi OSPF hội tụ, bảng định tuyến của R1 sẽ có:
```
O    192.168.40.0/24 [110/x] via 10.10.10.2   ! VLAN 40 B2 qua R2
O    192.168.50.0/24 [110/x] via 10.10.10.2   ! VLAN 50 B2 qua R2
O    192.168.60.0/24 [110/x] via 10.10.10.2   ! VLAN 60 B2 qua R2
```

Và bảng định tuyến của R2 sẽ có các route đến VLAN 10/20/30 Building 1 theo chiều ngược lại.

---

## Lệnh kiểm tra

```
show ip ospf neighbor              ! kiểm tra trạng thái láng giềng (cần FULL)
show ip route ospf                 ! các route học qua OSPF
show ip ospf interface brief       ! interface nào đang tham gia OSPF
show ip ospf database              ! LSDB
ping 192.168.40.1 source vlan10    ! test thông 2 building
```
