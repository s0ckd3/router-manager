# Router Manager — Nền tảng gateway Ubuntu cho multi-WAN, policy routing và proxy egress

**Đối tượng:** kỹ sư mạng, SRE, vận hành ISP/lab, chủ hạ tầng farm MMO hoặc proxy seller — đã quen Linux, PPPoE, policy routing, nftables và mô hình “một máy làm nhiều việc”.

 ![https://github.com/s0ckd3/router-manager/blob/main/so-do-tham-khao.png?raw=true](https://github.com/s0ckd3/router-manager/blob/main/so-do-tham-khao.png?raw=true)

---

## 1. Định vị sản phẩm

Router Manager là **soft router** chạy trên **Ubuntu Server LTS**, quản trị qua **SPA web** (REST `/api/v1`). Trạng thái cấu hình được **khai báo hóa** (declarative): lưu trong DB, render ra artifact runtime (netplan, nftables, dnsmasq, unit gost, ip rule/policy), apply có validate — không coi file rời trên disk là nguồn sự thật.

Hai trục năng lực chính:

1. **Data plane / control plane mạng:** multi-uplink PPPoE, NAT, nftables, DHCP/DNS, policy routing và **egress binding** theo client LAN.
2. **Proxy egress:** stack proxy SOCKS5/HTTP (fork gost in-process), gắn **per-pppX**, backconnect public WAN, shared upstream, seller lease — tối ưu workload **stream proxy / MMO** (UDP relay, đổi IP theo session PPP, health check).

Khác biệt so với router OS (OpenWrt, pfSense, RouterOS): về **khả năng định tuyến**, hệ sinh thái router OS đã có policy routing, multi-WAN, failover — nhưng vận hành ở layer rule/script, UI phân mảnh, khó **gom proxy + reconcile PPP + egress client** trên cùng lifecycle. Router Manager **chuẩn hóa workflow** cho mô hình “nhiều pppX + nhiều proxy + gán client” trên **một host Ubuntu**, đồng thời tận dụng **hạ tầng server** (RAM/CPU, NIC multi-port, Docker, monitoring, AI inference sidecar) mà appliance router khó cấp.

---

## 2. Ubuntu làm converged edge — không chỉ “cài thêm router”
 ![https://github.com/s0ckd3/router-manager/blob/main/so-do-tham-khao.png?raw=true](https://github.com/s0ckd3/router-manager/blob/main/router-manager_why-ubuntu.PNG?raw=true)

### 2.1 Tại sao không khóa vào appliance

| Khía cạnh | Router OS trên appliance | Ubuntu + Router Manager |
|-----------|-------------------------|-------------------------|
| Multi-WAN PPPoE | Có (tùy platform) | pppd + profile DB, hook IPUp/Down, reconcile nền |
| Policy routing / egress | Rule thủ công, fwmark, tables | Egress service + UI; autopilot sau PPP event |
| Proxy SOCKS5/HTTP scale | Thường cần host khác | gost per-stack trên cùng máy, reload từng instance |
| Collocation workload | Hạn chế flash/RAM | Docker/Podman, VM nhẹ, Prometheus node, LLM local |
| Backup / DR | Export config vendor | Snapshot DB + file backup module, systemd unit |
| Mở rộng NIC | Model cố định | PCIe multi-port Intel, SR-IOV (tùy lab) |

Một **server tổng** Ubuntu đóng vai trò:

- **Edge gateway:** default route, NAT, firewall, DNS forwarder nội bộ.
- **PPP concentrator:** mỗi hợp đồng ISP → một session `pppX` (hoặc macvlan parent → ppp child), map 1:1 với **uplink logic** trong UI.
- **Egress broker:** client LAN ↔ `wan_ppp` (pure policy route) hoặc `proxy_stream` (SOCKS5/HTTP upstream nội bộ).
- **Compute host phụ:** container registry nội bộ, panel, job AI, bastion — **cùng NUMA**, giảm hop và chi phí rack.

### 2.2 Topology tham chiếu

```
                    [ Ubuntu Server — Router Manager ]
    NIC WAN0 ──► ppp0 ──► public IP₀ ──► proxy stack₀ / NAT₀
    NIC WAN1 ──► ppp1 ──► public IP₁ ──► proxy stack₁ / NAT₁
    NIC WAN2 ──► ppp2 (backup uplink) ──► policy failover (ops-defined)
    NIC LAN  ──► br-lan / routed LAN ──► clients + DHCP reservations
                      │
                      ├── egress: MAC → table/rule → pppX | local proxy
                      └── Docker / monitoring / optional AI stack
```

**Line uplink dự phòng:** vận hành có thể khai báo profile PPP primary/secondary; khi `pppX` down, hook + reconcile worker đồng bộ trạng thái DB với kernel (ip link, default route, egress rule) — admin vẫn override qua UI/API. Chi tiết failover phụ thuộc policy bạn đặt (không thay thế hoàn toàn SD-WAN appliance nếu yêu cầu sub-second SLA).

### 2.3 Collocation có kiểm soát

Trên cùng host, đội vận hành thường chạy song song:

- **Router Manager** (`systemd`): control plane mạng + proxy.
- **Container runtime:** checker, exporter, panel nội bộ — traffic qua `br-lan` hoặc management VLAN.
- **Observability:** node metrics, log shipper — tích hợp dashboard có sẵn.
- **AI / batch:** inference local hoặc crawler — **tách cgroup/CPU set** để không starve pppd/gost khi farm đông.

Router OS trên box 512MB–2GB khó cân bằng; Ubuntu cho phép **right-size** RAM (32–128GB) và **nic partitioning** rõ ràng (management vs WAN vs LAN).

---

## 3. Data plane: multi-WAN, PPPoE, policy routing

![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_seller_multi-wan.PNG?raw=true)

### 3.1 PPPoE và mapping pppX ↔ uplink

- CRUD profile PPPoE (user/pass, interface parent, metric).
- **Start/stop/reconnect** từng session; **PPP hooks** (`IPUp`/`IPDown`) báo control plane không cần polling liên tục.
- **Reconcile background:** drift giữa DB và runtime (interface up nhưng thiếu IPv4, proxy enabled nhưng ppp down) được xử lý theo chu kỳ + sau event.
- **MACVLAN multi-WAN:** nhiều session trên cùng physical NIC khi ISP cho phép — giảm chi phí port.

Mỗi `pppX` mang **public IPv4** (và IPv6 nếu triển khai), là anchor cho:

- NAT masquerade riêng tầng egress.
- **Backconnect proxy** listen trên IP public của `pppX`.
- Policy route / fwmark (roadmap mở rộng mark packet đầy đủ).

### 3.2 Policy routing và egress binding
![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_seller_binding.PNG?raw=true)
| Chế độ (`route_mode`) | Hành vi | Use case |
|----------------------|---------|----------|
| `wan_ppp` | Traffic client đi thẳng qua `pppoe_profile_id` — policy routing, không qua local gost | Game client native, cần egress IP = IP của line |
| `proxy_stream` | Client dùng upstream SOCKS5/HTTP do router publish | Tool/Launcher hỗ trợ proxy, central quota |

Gán qua **Client Policy** (egress): theo MAC, pool, hoặc rule — phù hợp farm MMO tách **identity mạng per host**. Kết hợp **manual policy routes** trên Routing page cho traffic không gắn DHCP client.
![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_seller_client.PNG?raw=true)
**Autopilot chain** (không thay thế admin CRUD): `lanautopilot` → `dhcp/lan_watch` → `egress` → `networkreconcile` — giảm Apply thủ công sau khi DB đã persist (PPP flapping, client mới trên LAN).

### 3.3 L2/L3 nội bộ

- LAN **bridge hoặc routed**, netplan render + apply.
- **dnsmasq:** DHCP pool, reservation, DNS forwarder; **DNS domain block** cho policy nội bộ.
- **nftables:** zone firewall, access-open WAN có kiểm soát; reload **slice** thay vì flush toàn bộ table khi có thể.
- **NAT + port forward / DNAT:** auto cho proxy WAN (`wan_use_dnat`, `auto_wan_firewall`) — giảm rule tay khi scale port.

### 3.4 Device mode

- **`router`:** đầy đủ NAT, firewall reconcile, proxy DNAT.
- **`switch`:** tắt một phần reconcile FW/DNAT — dùng khi máy chỉ là L2/L3 bridge trong topology lớn hơn.

---

## 4. Proxy plane: stream egress cho MMO và seller
![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_proxy.PNG?raw=true)
### 4.1 Stack proxy per WAN

- **Quick create stack** trên `ProxyLinesPage`: gắn profile PPP, slot port, credential, `listen_mode` (LAN / **public WAN backconnect**).
- **gost fork in-process:** mỗi proxy instance cô lập parse config — reload **một** service khi đổi credential/port, không restart toàn bộ daemon.
- **SOCKS5 + UDP relay:** bật `socks5_udp_enabled` + `open_firewall_udp` — bắt buộc cho MMO/voice; kiểm tra qua **Proxy Check** (TCP/UDP probe).
- **PPP watchdog:** tự phục hồi `pppX` khi mất IPv4 trên interface — giảm proxy “sống nhưng egress chết”.

### 4.2 Client Policy và enforcement roadmap

- Metadata assignment: `explicit` / `access_binding` / routing — roadmap **enforce** đầy đủ (hiện một phần metadata-first).
- **External proxy pool:** upstream chain cho client LAN hoặc shared tier.
- **Shared Proxy (10000–20000):** local listener + upstream chain, FW/DNAT reconcile khi Apply.
- **Seller:** lease, expiry, public token page — mô hình proxy-as-a-service.

### 4.3 TCP/IP fingerprint

- Preset **OS-like TCP profile** per `pppX` (sysctl + dialer) — giảm fingerprint mismatch khi platform correlate stack với TTL/window; UI chỉ expose preset, không bắt operator chỉnh từng sysctl.

### 4.4 Kiểm tra chất lượng

- Check local proxy từ router (TCP/UDP).
- So sánh với danh sách proxy ngoài — phù hợp QA trước khi đưa line vào pool bán.

---

## 5. Vận hành: apply pipeline, safe mode, observability

### 5.1 Apply pattern (chuẩn cho admin có trách nhiệm production)

```
Validate → Backup artifact → Render → Syntax check (nft -c, dnsmasq --test)
→ Executor apply → Verify → [SafeMode timer] → Confirm / Rollback → Action log
```

- **Apply All:** orchestration forwarding → LAN → DHCP → NAT → firewall → PPP → routing → egress → WAN balance → proxy network reconcile — trả `steps[]` cho UI audit.
- **Reload granular:** một slice nft, một instance gost, dnsmasq — giảm blast radius khi farm đang online.

**Safe mode** (NAT/Firewall và một số module): cửa sổ xác nhận trước khi commit vĩnh viễn — quan trọng khi quản trị qua SSH session trên cùng đường management.

### 5.2 Khởi động và recovery
![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_auto_restore.PNG?raw=true)
Boot: load config → SQLite → scan NIC → **startup recovery** (PPP restore, proxy restore, network access reconcile) → background workers (PPP reconcile, monitoring, LAN autopilot, optional bootstrap `auto_apply_on_start`).

Thiết kế hướng tới **stateful edge**: sau reboot, máy tự đưa data plane về gần trạng thái DB đã lưu — admin không phải click Apply lặp cho từng module đã persist.

### 5.3 Giám sát

- WAN health, PPP status, traffic metrics.
- Network status cache cho UI — phân biệt “config DB” vs “runtime thực tế”.
- Action log cho audit thay đổi (không log secret).

---

## 6. So sánh với router OS — góc nhìn kỹ sư triển khai

| Tiêu chí | Router OS (OpenWrt/pfSense/…) | Router Manager @ Ubuntu |
|----------|------------------------------|-------------------------|
| Độ sâu routing | Rất cao nếu bạn master CLI/GUI | Cao cho mô hình multi-PPP + egress + proxy; một số tính năng đang roadmap (fwmark packet, assignment enforce) |
| Time-to-value proxy farm | Cần tích hợp rời (gost, 3proxy, script) | Tích hợp sẵn UI + lifecycle |
| Học curve cho team junior | Dốc | Thấp hơn cho workflow chuẩn; senior vẫn cần hiểu Linux networking |
| Mở rộng phần cứng | Theo model | PCIe NIC, RAM, disk tùy ý |
| Collocation | Khó | Docker, VM, AI, hosting — **cùng host** |
| API automation | Thường không thống nhất | REST `/api/v1`, envelope ổn định, field additive |

**Kết luận thực dụng:** 

Nếu bạn chỉ cần VPN site-to-site hoặc firewall đơn giản, router OS có thể đủ. Nếu bạn vận hành **hàng chục pppX**, **hàng trăm proxy**, **egress per-MAC** cho MMO/seller — và muốn **một Ubuntu** vừa gateway vừa colocate dịch vụ — Router Manager là lớp **orchestration** được viết cho bài toán đó, không phải bản sao GUI của pfSense.

![ảnh](https://github.com/s0ckd3/router-manager/blob/main/router-manager_all.PNG?raw=true)
---

## 7. Use case điển hình (kỹ thuật)

### 7.1 Farm MMO / studio acc

-  N NIC WAN → N profile PPPoE → N `pppX` public IP.
-  Máy client DHCP reservation → egress `wan_ppp` map cố định `ppp2`, `ppp5`, …
-  Hoặc `proxy_stream` → local SOCKS5 trên router, credential rotate theo slot.
-  UDP enabled + fingerprint preset theo nhóm acc.
-  Reconnect PPP theo lịch → đổi IP có kiểm soát; health check trước khi đưa line vào rotation.

### 7.2 Proxy seller / shared pool

-  Backconnect listen trên public IP của `pppX`.
-  Seller lease + public page; shared tier upstream chain.
-  Port check diagnostic trước khi publish catalog.

### 7.3 Lab ISP / training

-  MACVLAN + multi-session trên một port.
-  Routing page + WAN balance (lưu ý xung đột ip rule — vận hành cần review policy table).
-  Device mode `switch` khi lab chỉ cần L3 segment.

### 7.4 Converged small DC / rack edge

-  Management interface **protected** (không đổi IP/FW blind qua autopilot).
-  Production WAN/LAN trên NIC còn lại.
-  Cùng máy: Router Manager + Prometheus + Docker workloads nội bộ.

---

## 8. Yêu cầu triển khai (tóm tắt)

- **OS:** Ubuntu Server 22.04+ LTS (khuyến nghị LTS đang được support).
- **Privileges:** quyền quản trị network namespace, systemd, pppd, nftables — cài qua script/packaging dự án.
- **Phần cứng:** multi-port NIC (Intel i350/i225 phổ biến lab); RAM theo số lượng gost instance + container colocate.
- **Remote ops:** luôn giữ đường management khi apply NAT/FW; dùng safe mode và backup trước thay đổi rủi ro.

Chi tiết cài đặt: `docs/INSTALL.md`, `setup_router/README.md` — ngoài phạm vi giới thiệu này.

---

## 9. Tóm tắt

Router Manager biến **Ubuntu Server** thành **converged edge gateway**: multi-uplink PPPoE (`pppX`), policy routing và egress binding, nftables/NAT/DHCP/DNS, kết hợp **proxy stack gost** (SOCKS5/HTTP, UDP, backconnect WAN, fingerprint) cho workload **stream proxy và MMO**. So với router OS thuần, lợi thế nằm ở **workflow tích hợp**, **API/ UI thống nhất**, **reload granular**, **autopilot sau persist DB**, và khả năng **colocate** Docker, monitoring, AI trên cùng một server tổng — mô hình mà appliance khó đáp ứng cùng mức linh hoạt phần cứng.

## 10. Tài liệu liên kết:
- [https://2movn.com/bai-viet/router-manager-cong-cu-mmo-hieu-qua](https://2movn.com/bai-viet/router-manager-cong-cu-mmo-hieu-qua)

---

