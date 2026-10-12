# Lab: Enterprise Network Segmentation — PT Mitra Solusi Teknologi

**VLSM + VLAN + Trunk + Router-on-a-Stick (ROAS) + Static Routing**
**Tools:** Cisco Packet Tracer

---

## 1. Latar Belakang

PT Mitra Solusi Teknologi punya kantor pusat (**HQ**) di **Makassar** dan kantor cabang (**Branch**) di **Palopo**. HQ dipecah jadi beberapa VLAN per departemen, di-routing lewat 1 router (Router-on-a-Stick), lalu disambungkan ke Branch lewat **router transit** yang merepresentasikan jalur ISP/Internet antara dua site — bukan link langsung HQ↔Branch.

---

## 2. Topologi

```mermaid
graph LR
    subgraph HQ["HQ - Makassar (192.168.30.0/24)"]
        PCm[PC Management<br/>VLAN 10] --- SW0[Switch0]
        PCf[PC Finance<br/>VLAN 20] --- SW0
        SW0 ==Trunk== SW4[Switch4]
        PCit[PC IT/Networking<br/>VLAN 30] --- SW4
        PCmk[PC Marketing<br/>VLAN 40] --- SW4
        SW4 ==Trunk dot1Q<br/>10,20,30,40==> R1[R1 - HQ<br/>Router-on-a-Stick]
    end

    R1 ---|"20.0.0.0/30"| R3[INTERNET<br/>Router Transit]
    R3 ---|"10.0.0.0/30"| R2[R2 - Branch]

    subgraph Branch["Branch - Palopo"]
        R2 --- SW2[Switch2]
        SW2 --- PCs["PC9-PC18<br/>192.168.40.0/27"]
    end
```

- **R1** = router HQ, ROAS (subinterface per VLAN)
- **INTERNET** = router transit, murni routing, tidak ada VLAN/host di belakangnya
- **R2** = router Branch, 1 interface ke LAN, 1 interface ke transit

---

## 3. Perhitungan VLSM (HQ — 192.168.30.0/24)

| VLAN | Nama | Network | CIDR | Gateway (Subinterface R1) | Interface |
|---|---|---|---|---|---|
| 10 | MANAGEMENT | 192.168.30.192 | /28 | 192.168.30.206 | Gi0/0.10 |
| 20 | FINANCE | 192.168.30.160 | /27 | 192.168.30.190 | Gi0/0.20 |
| 30 | IT_NETWORKING | 192.168.30.64 | /26 | 192.168.30.126 | Gi0/0.30 |
| 40 | MARKETING | 192.168.30.128 | /27 | 192.168.30.158 | Gi0/0.40 |

Trunk di kedua switch HQ (Switch4/SW1 dan Switch0/SW2) sudah di-pruning hanya allow VLAN yang dipakai (`10,20,30,40` di sisi router, `10,20` di sisi Switch0), dengan **native VLAN 99** (dedicated, tidak dipakai host manapun) — sudah tidak ada sisa VLAN 50 di mana pun.

---

## 4. WAN Link (point-to-point, gaya ISP)

| Link | Subnet | R1-HQ | INTERNET | R2-Branch |
|---|---|---|---|---|
| HQ ↔ Internet | 20.0.0.0/30 | 20.0.0.1 (Gi0/0/0) | 20.0.0.2 (Gi0/0/0) | — |
| Internet ↔ Branch | 10.0.0.0/30 | — | 10.0.0.1 (Gi0/1/0) | 10.0.0.1 (Gi0/0/0) *(lihat catatan)* |

> Catatan penomoran IP di link kedua: `INTERNET` pakai `10.0.0.2` dan `R2` pakai `10.0.0.1` — sudah benar dan saling berpasangan, baris tabel di atas cuma menunjukkan subnet-nya sama.

---

## 5. Branch — Palopo

| Interface | IP Address | Keterangan |
|---|---|---|
| R2 Gi0/0 | 192.168.40.30/27 | Gateway LAN Branch (192.168.40.0/27, usable .1–.30) |
| PC9–PC18 | 192.168.40.x | 10 PC, statis, dalam range .1–.29 |

---

## 6. Static Routing

**R1 (HQ):**
```
ip route 192.168.40.0 255.255.255.224 20.0.0.2
```

**INTERNET (transit):**
```
ip route 192.168.30.64 255.255.255.192 20.0.0.1
ip route 192.168.30.128 255.255.255.224 20.0.0.1
ip route 192.168.30.160 255.255.255.224 20.0.0.1
ip route 192.168.30.192 255.255.255.240 20.0.0.1
ip route 192.168.40.0 255.255.255.224 10.0.0.1
```

**R2 (Branch):**
```
ip route 192.168.30.64 255.255.255.192 10.0.0.2
ip route 192.168.30.128 255.255.255.224 10.0.0.2
ip route 192.168.30.160 255.255.255.224 10.0.0.2
ip route 192.168.30.192 255.255.255.240 10.0.0.2
```

> 💡 **Potensi perbaikan:** rute di `INTERNET` dan `R2` saat ini satu baris per subnet VLAN HQ (4 baris). Karena semua subnet itu masih ada di dalam `192.168.30.0/24`, ini bisa disederhanakan jadi **1 baris summary**: `ip route 192.168.30.0 255.255.255.0 <next-hop>` — kecuali kamu sengaja mau tetap pisah per subnet untuk alasan kontrol routing yang lebih granular. Bukan salah, cuma opsional buat didiskusikan di README sebagai trade-off.

---

## 7. Verifikasi (sudah dites, hasil nyata)

Ping dari PC di HQ ke gateway VLAN lain dan ke Branch:

| Tujuan | Keterangan | Hasil |
|---|---|---|
| 192.168.30.161 (GW Finance) | Inter-VLAN via ROAS | ✅ Berhasil |
| 192.168.30.65 (GW IT) | Inter-VLAN via ROAS | ✅ Berhasil |
| 192.168.30.193 (GW Management) | Inter-VLAN via ROAS | ✅ Berhasil |
| 192.168.40.1 (Branch) | Static routing lintas 2 hop (via INTERNET) | ✅ Berhasil, 0% loss |

*(Request timeout di percobaan pertama tiap tujuan itu normal — ARP resolution, bukan masalah jaringan.)*

Output mentah (apa adanya dari Command Prompt PC, bukan ringkasan) ada di [`verifikasi/ping-test.txt`](verifikasi/ping-test.txt).

`show vlan brief` & `show interface trunk` final (SW1/Switch4 dan SW2/Switch0):

| Switch | Native VLAN | Allowed VLAN (trunk) | VLAN 50 sisa? |
|---|---|---|---|
| SW1 (Switch4) | 99 ✅ | 10,20 (ke SW2), 10,20,30,40 (ke R1) | Tidak ada ✅ |
| SW2 (Switch0) | 99 ✅ | 10,20 | Tidak ada ✅ |

`Gi0/0.50` di R1 juga sudah dihapus — tidak muncul lagi di `show ip interface brief`.

**Masih perlu dites (opsional, lanjutan):**
- [ ] PC-ke-PC langsung antar VLAN (bukan cuma PC-ke-gateway)
- [ ] Cabut kabel trunk sebentar → pastikan inter-VLAN mati total

---

## 8. Struktur Repo

```
enterprise-vlan-vlsm-roas-static/
├── README.md
├── topologi.pkt
└── configs/
    ├── r1-hq.cfg
    ├── internet-transit.cfg
    ├── r2-branch.cfg
    ├── sw1-hq.cfg
    ├── sw2-hq.cfg
    └── sw1-branch-palopo.cfg
└── verifikasi/
    └── ping-test.txt
```

## 9. Push ke GitHub

```bash
cd nama-folder-lab
git init
git add .
git commit -m "Lab: VLSM + VLAN + Trunk + ROAS + Static Routing (3-router topology) - PT Mitra Solusi Teknologi"
git branch -M main
git remote add origin https://github.com/sahrulashar/enterprise-vlan-vlsm-roas-static.git
git push -u origin main
```
