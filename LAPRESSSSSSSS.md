# Laporan Praktikum Modul 1 — Wireshark & GNS3
**Komdat Jarkom 2026 | Kelompok C | Prefix IP: 10.88.x.x**

---

## Topologi Jaringan

![[Whole GNS.png]]

---

## Skema Pengalamatan IP

| Perangkat   | Interface | Mode   | IP Address     | Netmask         | Gateway    |
|-------------|-----------|--------|----------------|-----------------|------------|
| RouterLain  | eth0      | DHCP   | Otomatis NAT   | Sesuai NAT      | Otomatis   |
| RouterLain  | eth1      | Static | 10.88.1.1      | 255.255.255.0   | -          |
| RouterLain  | eth2      | Static | 10.88.2.1      | 255.255.255.0   | -          |
| RouterLain  | eth3      | Static | 10.88.3.1      | 255.255.255.0   | -          |
| Alice       | eth0      | Static | 10.88.1.2      | 255.255.255.0   | 10.88.1.1  |
| Mika2       | eth0      | Static | 10.88.1.3      | 255.255.255.0   | 10.88.1.1  |
| Chisa       | eth0      | Static | 10.88.2.2      | 255.255.255.0   | 10.88.2.1  |
| Knights     | eth0      | Static | 10.88.3.2      | 255.255.255.0   | 10.88.3.1  |
| Eiri        | eth0      | Static | 10.88.3.3      | 255.255.255.0   | 10.88.3.1  |

---

## Nomor 1 - Pembangunan Topologi GNS3

Membangun topologi jaringan di GNS3 dengan:
- **RouterLain** sebagai router utama (Docker container, 4 interface)
- **NAT1** sebagai gateway internet
- **Switch1, Switch2, Switch3** sebagai Ethernet switch
- **Alice, Mika2, Chisa, Knights, Eiri** sebagai client (Docker container berbasis Alpine)

Semua node dikonfigurasi menggunakan prefix IP kelompok **10.88.x.x**.

Konfigurasi `/etc/network/interfaces` disimpan di config box GNS3 masing-masing node. Untuk memastikan konfigurasi diterapkan saat startup, dibuat `/root/init.sh` pada setiap node yang berisi:

```sh
#!/bin/sh
ip addr flush dev eth0
rm -f /run/ifstate
ifup eth0
```

File ini dipanggil otomatis oleh `/etc/alpinet-init.sh` saat container start.

---

## Nomor 2 — Koneksi Internet via NAT/DHCP pada eth0

Konfigurasi RouterLain `/etc/network/interfaces`:

```
auto eth0
iface eth0 inet dhcp
    up sysctl -w net.ipv4.ip_forward=1
    up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    ...

auto eth1
iface eth1 inet static
    address 10.88.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 10.88.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 10.88.3.1
    netmask 255.255.255.0
```

eth0 mendapat IP otomatis dari NAT1 via DHCP (`192.168.122.31`).

---

## Nomor 3 — Routing Antar Subnet

Seluruh client dikonfigurasi dengan static IP dan gateway mengarah ke RouterLain. IP forwarding diaktifkan di kernel RouterLain:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Verifikasi: client dari subnet berbeda dapat saling ping, contoh:
- Chisa (`10.88.2.2`) → Alice (`10.88.1.2`) ✅
- Knights (`10.88.3.2`) → Chisa (`10.88.2.2`) ✅

---

## Nomor 4 — NAT Masquerade & DNS Resolver

**NAT Masquerade** dikonfigurasi di RouterLain:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Menyebabkan paket dari client private IP (`10.88.x.x`) terlihat berasal dari IP router (`192.168.122.31`) saat keluar ke internet.

**DNS Resolver** dikonfigurasi di setiap client:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Ditambahkan sebagai `up` line di config box agar persisten:

```
up echo nameserver 8.8.8.8 > /etc/resolv.conf
```

Verifikasi: semua client dapat `ping 8.8.8.8` dan `ping google.com` ✅

---

## Nomor 5 — Script Verifikasi `/root/cek_status.sh`

Dibuat script verifikasi di RouterLain yang persisten di `/root/`:

```bash
#!/bin/bash
echo "=== Interface Summary ==="
ip -br a

echo ""
echo "=== NAT Table Status ==="
iptables -t nat -L -v -n
```

![[screenshot-20260917-002428.png]]

Script dipanggil dengan `/root/cek_status.sh` dan menampilkan:
- Status semua interface beserta IP address
- Status rule NAT Masquerade dan packet count

---

## Nomor 6 — Analisis Traffic Anomali (Wireshark + Traffic Generator)

Traffic generator dijalankan pada node **Mika2**. Wireshark capture dilakukan pada link Mika2 → Switch1.

**Display filter yang diterapkan:**
```
dns || icmp
```

**Hasil:**

![[screenshot-20260917-003210.png]]

- Total paket: 50
- Paket DNS dan ICMP: 45 (90%)
- ICMP: ping traffic ke `1.1.1.1` dan `8.8.8.8`
- DNS: domain lookup ke `8.8.8.8` dan `1.1.1.1`

---

## Nomor 7 — FTP Server pada Node Chisa

**Software:** pure-ftpd (dipilih karena vsftpd tidak kompatibel dengan Alpine tanpa PAM)

**Setup:**
```bash
apk add pure-ftpd
adduser -D alice && echo "alice:123" | chpasswd
adduser -D mika  && echo "mika:mika1234" | chpasswd
adduser -D eiri  && echo "eiri:eiri1234" | chpasswd
passwd -l eiri                          # blacklist eiri
mkdir -p /var/wired/data
chown alice:alice /var/wired/data
chmod 775 /var/wired/data               # alice=rwx, others=r-x
pure-ftpd -l unix -B
```

**Kebijakan akses:**

| User  | Akses           | Mekanisme                        |
|-------|-----------------|----------------------------------|
| alice | Read + Write    | Owner direktori (chmod 775)      |
| mika  | Read only       | Others permission (r-x)          |
| eiri  | Blacklist       | `passwd -l eiri` (account lock)  |

Script setup disimpan di `/root/setup_ftp.sh` untuk persistensi.

**Bukti konfigurasi:**
- Alice berhasil membuat `signal_alice.txt` di `/var/wired/data`
	![[screenshot-20260917-003939.png]] 
- Mika hanya bisa read
	![[screenshot-20260917-004257.png]]
  
- Eiri ditolak saat mencoba login 
	![[screenshot-20260917-005024.png]]
---

## Nomor 8 — Knights Upload Dokumen Intelijen ke FTP Chisa

Knights menghubungkan FTP client ke Chisa menggunakan akun alice:

```bash
lftp -u alice,123 10.88.2.2
cd /var/wired/data
put /root/knights_report.txt
```

**Analisis Wireshark (filter: `ftp || ftp-data`):**

![[Pasted image 20260917011530.png]]

| Elemen          | Nilai                                          |
| --------------- | ---------------------------------------------- |
| Perintah upload | `STOR knights_report.txt`                      |
| Status sukses   | `226 File successfully transferred`            |
| Negosiasi PASV  | `227 Entering Passive Mode (10,88,2,2,212,58)` |
| Port data TCP   | 212×256 + 58 = **54330**                       |

---

## Nomor 9 — Mika Mengakses Protocol 7 dari FTP Chisa

Mika mengunduh `protocol7.txt` dari FTP server Chisa:
![[Pasted image 20260917012226.png]]

![[Pasted image 20260917012242.png]]

```bash
lftp -u mika,mika1234 10.88.2.2
cd /var/wired/data
get protocol7.txt          # berhasil (read allowed)
put /etc/hostname          # gagal (553 Permission denied)
```

**Hasil:** Pembatasan read-only terbukti — Mika dapat download tapi tidak bisa upload.
![[Pasted image 20260917012144.png]]

---

## Nomor 10 — Uji Ketahanan Koneksi Knights → Chisa (Ping)

```bash
ping -c 77 -s 128 -i 0.3 10.88.2.2
```

**Analisis Wireshark (filter: `icmp`):**
![[Pasted image 20260917012327.png]]

![[Pasted image 20260917012349.png]]



| Parameter      | Nilai                      |
|----------------|----------------------------|
| ICMP Type Echo Request | Type 8, Code 0    |
| ICMP Type Echo Reply   | Type 0, Code 0    |
| Panjang paket  | 170 bytes (128 payload + 42 header) |

**Statistik ping:**
![[Pasted image 20260917012422.png]]

---

## Nomor 11 — Kelemahan Protokol Telnet

**Setup di Chisa:**
```bash
adduser -D phantom_user
echo "phantom_user:wired_ghost" | chpasswd
telnetd
```

**Koneksi dari Eiri:**
```bash
telnet 10.88.2.2
# login: phantom_user
# Password: wired_ghost
```

**Analisis Wireshark (Follow TCP Stream):**

![[Pasted image 20260917012442.png]]

![[Pasted image 20260917012456.png]]

Kredensial `phantom_user` dan `wired_ghost` terlihat jelas dalam plaintext di TCP stream.

**Mengapa setiap karakter dikirim dalam paket TCP terpisah:**

mengapa terpisah? karena telnet memasuki character session, dimana dari situ telnet hanya menerima input per karakter, ini juga karena telnet dibuat untuk interactive remote sessions

---

## Nomor 12 — Port Scan Alice → Knights (Netcat)

**Setup di Knights:**
```bash
nc -l -p 22 &    # simulasi SSH
nc -l -p 80 &    # simulasi HTTP
# port 7777 dibiarkan tertutup
```

**Scan dari Alice:**
```bash
nc -zv 10.88.3.2 22    # Connection succeeded
nc -zv 10.88.3.2 80    # Connection succeeded
nc -zv 10.88.3.2 7777  # Connection refused
```

**Analisis Wireshark (filter: `tcp && ip.addr == 10.88.3.2`):**

![[Pasted image 20260917012605.png]]

![[Pasted image 20260917012617.png]]

![[Pasted image 20260917012522.png]]


| Port | Response TCP Flag | Arti                    |
|------|------------------|-------------------------|
| 22   | SYN-ACK          | Port terbuka, koneksi diterima |
| 80   | SYN-ACK          | Port terbuka, koneksi diterima |
| 7777 | RST-ACK          | Port tertutup, koneksi ditolak |

---

## Nomor 13 — SSH Key-Based Authentication Mika → Knights

**Setup di Knights:**
```bash
apk add openssh
ssh-keygen -A
adduser -D mika_admin
passwd mika_admin        # set password untuk unlock account
/usr/sbin/sshd
```

**Generate key di Mika2:**
```bash
ssh-keygen -t rsa -b 2048 -f /root/.ssh/id_rsa -N ""
```

**Copy public key ke Knights:**
```bash
# di Knights:
echo "<public_key>" > /home/mika_admin/.ssh/authorized_keys
chmod 600 /home/mika_admin/.ssh/authorized_keys
```

**sshd_config di Knights:**
```
PasswordAuthentication no
PubkeyAuthentication yes
```

**Koneksi dari Mika2:**
```bash
ssh mika_admin@10.88.3.2
```

**Analisis Wireshark (filter: `ssh`):**

![[Pasted image 20260917012652.png]]

| Paket                     | Keterangan                                    |
|---------------------------|-----------------------------------------------|
| Protocol Version Exchange | `SSH-2.0-OpenSSH_10.2` (client dan server)    |
| Key Exchange Init         | Negosiasi algoritma enkripsi                  |
| PQ/T Hybrid Key Exchange  | Pertukaran kunci menggunakan mlkem768x25519   |
| Encrypted packets         | Semua data setelah handshake terenkripsi      |

**Mengapa kredensial tidak terlihat seperti Telnet:**

SSH mengenkripsi seluruh sesi menggunakan **chacha20-poly1305** setelah key exchange. Pertukaran kunci menggunakan kriptografi asimetris sehingga shared secret tidak pernah dikirim melalui jaringan. Berbeda dengan Telnet yang mengirim semua data dalam plaintext, SSH membungkus setiap byte dalam enkripsi.

---

## Catatan Teknis

### Persistensi Konfigurasi

Karena container Docker bersifat ephemeral, persistensi dicapai dengan:

1. **`/root/init.sh`** — dipanggil otomatis oleh `/etc/alpinet-init.sh` saat startup:
```sh
#!/bin/sh
ip addr flush dev eth0
rm -f /run/ifstate
ifup eth0
```

2. **Config box GNS3** — menyimpan `/etc/network/interfaces` per node

3. **`/root/cek_status.sh`** — script verifikasi RouterLain

4. **`/root/setup_ftp.sh`** — script setup FTP server Chisa

### Tools yang Digunakan

| Tool        | Fungsi                                  |
| ----------- | --------------------------------------- |
| GNS3        | Network simulator                       |
| Wireshark   | Packet capture dan analisis             |
| pure-ftpd   | FTP server di Chisa                     |
| lftp        | FTP client di node client               |
| netcat (nc) | Port scanning dan TCP listener          |
| OpenSSH     | SSH server di Knights                   |
| telnetd     | Telnet server di Chisa (demo kelemahan) |
| tshark      | CLI Wireshark untuk analisis pcapng     |

