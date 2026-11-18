# HSRP — Dự phòng gateway

## Tại sao cần HSRP?

Các host cần 1 địa chỉ gateway cố định để thoát mạng. Nếu chỉ cấu hình IP thật của MSW1 làm gateway thì khi MSW1 hỏng, toàn bộ host mất kết nối dù MSW2 vẫn đang chạy. HSRP giải quyết vấn đề này bằng cách tạo ra một **IP ảo** (Virtual IP) dùng chung giữa 2 switch — host chỉ trỏ về IP ảo đó, không quan tâm switch nào đang thực sự xử lý.

---

## Nguyên lý hoạt động

```
Host → default-router 192.168.10.1  (IP ảo HSRP)
                │
        ┌───────┴────────┐
        │                │
      MSW1             MSW2
  (ACTIVE, .2)     (STANDBY, .3)
  priority 110     priority 100
```

- **Active**: Đang giữ IP ảo, nhận và chuyển tiếp traffic.
- **Standby**: Lắng nghe gói Hello từ Active (mỗi 3 giây). Nếu sau 10 giây không thấy Hello → chuyển sang Active.
- **Preempt**: Switch priority cao hơn sẽ tự giành lại vai trò Active khi khởi động lại — không cần can thiệp thủ công.

---

## Cấu hình Building 1

| VLAN | Group | IP ảo | MSW1 (Active) | Priority | MSW2 (Standby) | Priority |
|------|-------|-------|---------------|----------|----------------|----------|
| 10 – HR | 10 | 192.168.10.1 | 192.168.10.2 | **110** | 192.168.10.3 | 100 |
| 20 – Sales | 20 | 192.168.20.1 | 192.168.20.2 | **110** | 192.168.20.3 | 100 |
| 30 – Finance | 30 | 192.168.30.1 | 192.168.30.2 | **110** | 192.168.30.3 | 100 |

MSW1 là Active cho tất cả VLAN Building 1 (priority 110 > 100).

**Đoạn cấu hình MSW1 (mỗi VLAN):**
```
interface Vlan10
 ip address 192.168.10.2 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 110
 standby 10 preempt
```

**Đoạn cấu hình MSW2 (mỗi VLAN):**
```
interface Vlan10
 ip address 192.168.10.3 255.255.255.0
 standby 10 ip 192.168.10.1
 standby 10 priority 100
 standby 10 preempt
```

---

## Cấu hình Building 2

| VLAN | Group | IP ảo | MSW3 (Active) | Priority | MSW4 (Standby) | Priority |
|------|-------|-------|---------------|----------|----------------|----------|
| 40 – HR | 40 | 192.168.40.1 | 192.168.40.2 | **210** | 192.168.40.3 | 200 |
| 50 – Sales | 50 | 192.168.50.1 | 192.168.50.2 | **210** | 192.168.50.3 | 200 |
| 60 – Finance | 60 | 192.168.60.1 | 192.168.60.2 | **210** | 192.168.60.3 | 200 |

MSW3 là Active cho tất cả VLAN Building 2 (priority 210 > 200).

> Dùng 210/200 cho Building 2 thay vì 110/100 để dễ phân biệt khi xem `show standby` — khoảng cách 10 giữa 2 switch vẫn như nhau, chỉ là quy ước.

---

## Kịch bản failover

**MSW1 bị hỏng:**
1. MSW1 dừng gửi Hello trên trunk nối MSW2.
2. MSW2 chờ hết hold-time (10s) rồi chuyển trạng thái: Standby → Active.
3. MSW2 gửi Gratuitous ARP thông báo `192.168.10.1 = MAC của MSW2`.
4. Tất cả host cập nhật ARP cache — traffic tiếp tục qua MSW2.
5. DHCP: host đang có lease tiếp tục dùng bình thường; host mới sẽ nhận IP từ pool dự phòng của MSW2 (dải .128–.254).

**MSW1 khởi động lại:**
1. MSW1 quảng bá HSRP với priority 110.
2. Vì có `preempt`, MSW1 tự giành lại vai trò Active.
3. Traffic quay về MSW1, MSW2 trở về Standby.

---

## Lệnh kiểm tra

```
show standby brief
show standby vlan 10
show standby vlan 10 | inc State
```

Kết quả mẫu trên MSW1:
```
                     P indicates configured to preempt.
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   110 P Active   local           192.168.10.3    192.168.10.1
Vl20        20   110 P Active   local           192.168.20.3    192.168.20.1
Vl30        30   110 P Active   local           192.168.30.3    192.168.30.1
```
