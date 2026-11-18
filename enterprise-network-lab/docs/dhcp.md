# DHCP — Cấp phát IP tự động có dự phòng

## Đặt DHCP ở đâu?

DHCP được cấu hình trên **multilayer switch** (MSW1–MSW4), không phải trên router. Lý do: MSW đã có SVI trực tiếp trong từng VLAN và đóng vai trò default gateway. Khi DHCP server nằm cùng thiết bị với gateway, không cần cấu hình `ip helper-address` để relay DHCP request qua các subnet khác — đơn giản hơn và ít điểm lỗi hơn.

---

## Vấn đề: HSRP + DHCP chỉ trên 1 switch

Nếu chỉ cấu hình DHCP trên MSW1 (switch Active):

```
MSW1 (Active)   → chạy DHCP
MSW2 (Standby)  → không có DHCP
```

Khi MSW1 hỏng, HSRP failover thành công — MSW2 tiếp quản IP ảo, host vẫn có gateway. Nhưng **không có DHCP server nào đang chạy**. Host mới vào mạng hoặc host cần gia hạn lease sẽ không nhận được IP.

---

## Giải pháp: Split-pool (chia dải theo 2 switch)

Cả 2 switch đều chạy DHCP cùng lúc, nhưng mỗi switch phụ trách **dải địa chỉ không trùng nhau**:

```
MSW1 (Primary)  → cấp 192.168.10.11 – 192.168.10.127
MSW2 (Standby)  → cấp 192.168.10.128 – 192.168.10.254
```

- MSW1 exclude .128–.254 → chỉ cấp từ .11 đến .127.
- MSW2 exclude .1–.127 → chỉ cấp từ .128 đến .254.
- Địa chỉ .1–.10 được exclude trên cả hai → dành cho gateway và hạ tầng.

**Kết quả:**
- Bình thường: MSW1 cấp cho host mới, MSW2 cấp khi MSW1 đầy hoặc khi có 2 DHCP discover đến cùng lúc.
- Khi MSW1 hỏng: MSW2 tiếp tục cấp IP trong dải .128–.254. Host đang giữ lease .11–.127 vẫn dùng được đến khi hết hạn (lease 1 ngày).
- Khi MSW1 phục hồi: MSW1 chạy lại bình thường, tự cấp tiếp dải của mình.

Không xung đột địa chỉ. Không mất DHCP khi failover.

---

## Cấu hình Building 1

### MSW1 — Pool chính (cấp .11–.127)
```
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.10.128 192.168.10.254

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 1
```
Tương tự cho VLAN 20 và VLAN 30.

### MSW2 — Pool dự phòng (cấp .128–.254)
```
ip dhcp excluded-address 192.168.10.1 192.168.10.127

ip dhcp pool VLAN10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 1
```
Tương tự cho VLAN 20 và VLAN 30.

---

## Cấu hình Building 2

### MSW3 — Pool chính (cấp .11–.127)
```
ip dhcp excluded-address 192.168.40.1 192.168.40.10
ip dhcp excluded-address 192.168.40.128 192.168.40.254
```

### MSW4 — Pool dự phòng (cấp .128–.254)
```
ip dhcp excluded-address 192.168.40.1 192.168.40.127
```
Tương tự cho VLAN 50 và VLAN 60.

---

## DNS

Tất cả pool DHCP đẩy `dns-server 8.8.8.8` về cho host:
```
dns-server 8.8.8.8
```
Trong môi trường thực tế sẽ dùng IP của DNS server nội bộ. Trong lab này dùng 8.8.8.8 để đơn giản.

---

## Lệnh kiểm tra

```
show ip dhcp pool           ! tên pool, dải địa chỉ, số lượng đã cấp
show ip dhcp binding        ! danh sách địa chỉ đang được thuê
show ip dhcp conflict       ! phát hiện xung đột IP
show ip dhcp statistics     ! đếm Discover/Offer/Request/Ack
```

**Test cơ bản:** Đặt PC trong VLAN 10 chế độ DHCP. Host phải nhận được:
- IP: 192.168.10.11–127 (từ MSW1) hoặc .128–254 (từ MSW2)
- Gateway: 192.168.10.1
- DNS: 8.8.8.8
- Lease: 1 ngày

**Test failover:** Tắt MSW1, thử xin DHCP. Host vẫn nhận được IP trong dải .128–.254 từ MSW2.
