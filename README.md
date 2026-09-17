# Laporan Praktikum Modul 1 — Wireshark & GNS3
**Komdat Jarkom 2026 | Kelompok K-49 Grup C | Prefix IP: 10.88.x.x**

dikerjakan oleh :

**Khalifa Suryadinarta** - **5027251104**

**Aura Syahzanani A** - **5027251123**

---

## Topologi Jaringan

<img width="922" height="653" alt="Whole GNS" src="https://github.com/user-attachments/assets/dbfca1f5-5697-4c4f-b635-ff00067fd5c2" />

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
- Chisa (`10.88.2.2`) → Alice (`10.88.1.2`) 
- Knights (`10.88.3.2`) → Chisa (`10.88.2.2`) 

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

Verifikasi: semua client dapat `ping 8.8.8.8` dan `ping google.com` 

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

<img width="917" height="560" alt="screenshot-20260917-002428" src="https://github.com/user-attachments/assets/20cdd846-ba47-42a7-b305-f8d38b391379" />


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

<img width="1920" height="1080" alt="screenshot-20260917-003210" src="https://github.com/user-attachments/assets/702bdd48-cc65-46d3-9daa-77f326ba72f3" />


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
	<img width="920" height="332" alt="screenshot-20260917-003939" src="https://github.com/user-attachments/assets/73819193-4cdb-4b94-afd8-5d92a217bbf6" />

- Mika hanya bisa read
	<img width="949" height="346" alt="screenshot-20260917-004257" src="https://github.com/user-attachments/assets/c54fb743-49f1-4afc-9642-a1a18c92cde0" />

- Eiri ditolak saat mencoba login 
	<img width="640" height="153" alt="screenshot-20260917-005024" src="https://github.com/user-attachments/assets/5aa880ba-73b0-42f1-bcd6-88a9aefd7708" />

---

## Nomor 8 — Knights Upload Dokumen Intelijen ke FTP Chisa

Knights menghubungkan FTP client ke Chisa menggunakan akun alice:

```bash
lftp -u alice,123 10.88.2.2
cd /var/wired/data
put /root/knights_report.txt
```

**Analisis Wireshark (filter: `ftp || ftp-data`):**

<img width="961" height="1012" alt="Pasted image 20260917011530" src="https://github.com/user-attachments/assets/83f0ef3b-f324-473a-a55d-77a053ddaed6" />


| Elemen          | Nilai                                          |
| --------------- | ---------------------------------------------- |
| Perintah upload | `STOR knights_report.txt`                      |
| Status sukses   | `226 File successfully transferred`            |
| Negosiasi PASV  | `227 Entering Passive Mode (10,88,2,2,212,58)` |
| Port data TCP   | 212×256 + 58 = **54330**                       |

---

## Nomor 9 — Mika Mengakses Protocol 7 dari FTP Chisa

Mika mengunduh `protocol7.txt` dari FTP server Chisa:
<img width="910" height="254" alt="Pasted image 20260917012226" src="https://github.com/user-attachments/assets/255bafe2-4260-4076-89ce-123ceda8f3c6" />

<img width="917" height="78" alt="Pasted image 20260917012422" src="https://github.com/user-attachments/assets/b6561323-1590-403d-a18d-f3b4dfcef908" />


```bash
lftp -u mika,mika1234 10.88.2.2
cd /var/wired/data
get protocol7.txt          # berhasil (read allowed)
put /etc/hostname          # gagal (553 Permission denied)
```

**Hasil:** Pembatasan read-only terbukti — Mika dapat download tapi tidak bisa upload.

<img width="954" height="1006" alt="Pasted image 20260917012144 (1)" src="https://github.com/user-attachments/assets/2f3c97fe-2e6f-4fa6-be17-3f0fcfe39d20" />


---

## Nomor 10 — Uji Ketahanan Koneksi Knights → Chisa (Ping)

```bash
ping -c 77 -s 128 -i 0.3 10.88.2.2
```

**Analisis Wireshark (filter: `icmp`):**
<img width="1600" height="415" alt="Pasted image 20260917012327" src="https://github.com/user-attachments/assets/33127a87-92a3-48c0-8e0c-5c4b19dd2450" />

<img width="1600" height="420" alt="Pasted image 20260917012349" src="https://github.com/user-attachments/assets/b59a8779-c3e5-4e22-a79e-2fb4f01441cb" />



| Parameter      | Nilai                      |
|----------------|----------------------------|
| ICMP Type Echo Request | Type 8, Code 0    |
| ICMP Type Echo Reply   | Type 0, Code 0    |
| Panjang paket  | 170 bytes (128 payload + 42 header) |

**Statistik ping:**
<img width="917" height="78" alt="Pasted image 20260917012422 (1)" src="https://github.com/user-attachments/assets/8b98f976-9a08-4d72-a753-5d2fdbad2ab1" />


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

<img width="917" height="78" alt="Pasted image 20260917012422 (1)" src="https://github.com/user-attachments/assets/b39423b1-68bf-4d01-a4be-67b65c04bbb0" />

<img width="958" height="506" alt="Pasted image 20260917012456" src="https://github.com/user-attachments/assets/0335b34b-15f2-4418-8472-c86e37ff8ab3" />


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

<img width="1600" height="420" alt="Pasted image 20260917012605" src="https://github.com/user-attachments/assets/ce929165-6ab3-4978-a9fc-fb267a3eae6b" />

<img width="1600" height="424" alt="Pasted image 20260917012617" src="https://github.com/user-attachments/assets/b73f525a-4e56-4596-958e-9a8bfae03191" />

<img width="1600" height="420" alt="Pasted image 20260917012522" src="https://github.com/user-attachments/assets/e05e9f93-9ba0-4d7f-8e94-0dd617439e10" />



| Port | Response TCP Flag | Arti                    |
|------|------------------|-------------------------|
| 22   | SYN-ACK          | Port terbuka, koneksi diterima |
| 80   | SYN-ACK          | Port terbuka, koneksi diterima |
| 7777 | RST-ACK          | Port tertutup, koneksi ditolak |

---

## Nomor 13 - SSH Key-Based Authentication Mika → Knights

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

**Penjelasan Script**
Dikarenakan Sudah mendapatkan public key, disini dibikinkan shellscript untuk saat server di restart agar menghindari setup yang dilakukan di atas
```
#!/bin/sh
apk add openssh --quiet
if [ ! -f /root/ssh_host_rsa_key ]; then
    ssh-keygen -A
    cp /etc/ssh/ssh_host_*key* /root/
else
    cp /root/ssh_host_*key* /etc/ssh/
fi
adduser -D mika_admin
echo "mika_admin:123" | chpasswd
mkdir -p /home/mika_admin/.ssh
echo "ssh-rsa AAAA..." > /home/mika_admin/.ssh/authorized_keys
chmod 600 /home/mika_admin/.ssh/authorized_keys
chown -R mika_admin:mika_admin /home/mika_admin/.ssh
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config
/usr/sbin/sshd

```
Blok if [ ! -f /root/ssh_host_rsa_key ]

```
if [ ! -f /root/ssh_host_rsa_key ]; then
    ssh-keygen -A
    cp /etc/ssh/ssh_host_*key* /root/
else
    cp /root/ssh_host_*key* /etc/ssh/
fi
```

Blok ini menangani persistensi host key agar fingerprint Knights tidak berubah setiap restart.

[ ! -f /root/ssh_host_rsa_key ] — cek apakah file host key belum ada di /root/
! = NOT, -f = file exists
Jika belum ada: generate host key baru dengan ssh-keygen -A, lalu copy ke /root/ untuk disimpan
Jika sudah ada: copy host key yang tersimpan dari /root/ kembali ke /etc/ssh/ di mana sshd membutuhkannya
Mengapa penting: tanpa ini, setiap restart Knights menghasilkan host key baru → fingerprint berubah → Mika2 mendapat peringatan "REMOTE HOST IDENTIFICATION HAS CHANGED"

ssh-keygen -A — generate semua tipe host key sekaligus (RSA, ECDSA, ED25519). Host key adalah identitas server, bukan identitas user.


**Analisis Wireshark (filter: `ssh`):**

<img width="954" height="1008" alt="Pasted image 20260917012652" src="https://github.com/user-attachments/assets/b9bde793-e200-49b4-92ef-31de2c9ba286" />


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

## Soal 14 — Brute-Force Login Web
**File**: `wired_bruteforce.pcapng`

### Apa yang terjadi
Eiri melancarkan serangan *brute-force* otomatis ke form login web milik Alice, mencoba berbagai kombinasi password terhadap satu username tetap secara berulang.

### Cara pembuktian

<img width="1918" height="1009" alt="image" src="https://github.com/user-attachments/assets/cf6d1e35-af53-478f-aa92-3a2ab25270aa" />

Filter `http.request.method == "POST"` diterapkan untuk menampilkan seluruh percobaan login ke endpoint `/login.php`. Terlihat puluhan permintaan POST beruntun dari IP yang sama — pola khas serangan otomatis, bukan login manual manusia. 

Untuk menemukan percobaan yang **berhasil**, filter diganti menjadi `http.response`, yang menampilkan pola: puluhan respons `401 Unauthorized` diikuti **satu** respons `200 OK` di paket paling akhir. 

<img width="1897" height="951" alt="image" src="https://github.com/user-attachments/assets/09882ee2-eb19-48c9-bdf4-2f61f53c6b2d" />


Paket sukses itu ditelusuri lebih dalam dengan klik kanan → **Follow → HTTP Stream**, yang menampilkan payload request lengkap (`username=lain_admin&password=...`) beserta response HTML `Success! Login successful.` Dari header response juga terbaca `Server: Apache/2.4.62` dan `X-Powered-By: PHP/8.3.14`. Header `User-Agent` pada setiap request POST juga mengungkap tool yang dipakai: `Fuzz Faster U Fool v2.1.0-dev` (ffuf).

<img width="1354" height="1012" alt="image" src="https://github.com/user-attachments/assets/738028c5-e0fd-4541-8c71-a803e89dabe3" />

### Temuan
| Item | Hasil |
|---|---|
| IP Penyerang | `172.26.7.50` |
| IP Target : Port | `172.26.7.100 : 8080` |
| Password berhasil | `wired_pr0tocol_7` |
| Web server | `Apache/2.4.62` |

### Validasi Socket Server
```
nc [IP_Group] 3401   
```

<img width="1192" height="604" alt="image" src="https://github.com/user-attachments/assets/8e864b90-cd9f-4375-bed2-62020f9a1782" />

---
## Soal 15 — Keylogger USB HID
**File**: `wired_usb_hid.pcap`

### Apa yang terjadi
Eiri memasang keyboard USB berbahaya (fisik) pada node Alice yang merekam setiap ketukan tombol (keystroke) dan mengirimkannya diam-diam.

### Cara pembuktian
Filter `usb.idVendor || usb.idProduct` menampilkan paket **Device Descriptor** — respons awal saat perangkat USB pertama kali dikenali sistem.

<img width="1918" height="1008" alt="image" src="https://github.com/user-attachments/assets/c811d6a8-ddd7-4634-bdd3-d97a4fd59c2a" />

Di dalam paket ini terbaca `idVendor: Logitech, Inc. (0x046d)` dan `idProduct: Keyboard K120 (0xc31c)`. Nomor **Device Address** ditemukan dengan filter `usb.device_address != 0`, menunjukkan angka `7`.

<img width="946" height="424" alt="image" src="https://github.com/user-attachments/assets/51ab616e-80b8-4879-967c-e27977262cc8" />


<img width="1039" height="546" alt="image" src="https://github.com/user-attachments/assets/cae6187d-ecf9-4999-af60-688c6dad7eea" />

Untuk membaca pesan rahasia, filter diganti ke `usb.capdata` yang menampilkan seluruh paket data mentah HID (`Leftover Capture Data`, 8 byte per paket: byte pertama = modifier/Shift, byte ketiga = kode tombol). 

Karena membaca puluhan paket satu-satu tidak efisien, seluruh paket di-*export* via **File → Export Packet Dissections → As Plain Text**, lalu diolah otomatis: setiap kode HID (misal `0x1a`) dicocokkan ke tabel HID Usage ID standar (`0x1a` = huruf **w**), dan modifier `0x02` (Shift) menandai huruf tersebut ditulis kapital. Hasil rangkaian huruf membentuk kalimat utuh.

<img width="624" height="409" alt="image" src="https://github.com/user-attachments/assets/56c76d08-2441-4e5c-b4f0-9f3b3ea4580a" />

<img width="615" height="479" alt="image" src="https://github.com/user-attachments/assets/efa9625a-67a4-49c6-b43d-7e8eeb333918" />

### Temuan
| Item | Hasil |
|---|---|
| Vendor ID / Product ID | `0x046d` / `0xc31c` (Logitech K120) |
| Device Address | `7` |
| Pesan rahasia | `Wired_Protocol_7_is_alive_2026` |

### Validasi Socket Server
```
nc [IP_Group] 3402
```

<img width="1234" height="576" alt="image" src="https://github.com/user-attachments/assets/80d022ab-0f9f-4edf-a43b-86f5d38d006d" />

---

## Soal 16 — Pencurian File via FTP
**File**: `wired_ftp_theft.pcapng`

### Apa yang terjadi
Eiri menaruh file malware di server FTP miliknya sendiri, lalu file tersebut diunduh dari sistem korban (Knights).

### Cara pembuktian
Karena capture berisi beberapa sesi FTP berbeda (percobaan login legit dan sesi berbahaya bercampur), pencarian dipersempit dengan filter `ftp contains "knights_payload"`. 

<img width="1918" height="982" alt="image" src="https://github.com/user-attachments/assets/80a5791c-80cd-4c19-bca7-7baf69a740af" />


Filter ini langsung menunjukkan 3 paket kunci: request `SIZE knights_payload.exe`, request `RETR knights_payload.exe` (perintah download), dan respons server `150 Opening BINARY mode data connection for knights_payload.exe (524288 bytes)` — dari sinilah ukuran file diketahui secara eksplisit tanpa perlu menghitung manual.

Untuk mengetahui banner server dan kredensial login, sesi yang sama ditelusuri balik ke awal menggunakan filter `ip.addr == 198.51.100.7 && ftp`.

<img width="1911" height="721" alt="image" src="https://github.com/user-attachments/assets/4a92004d-1089-4b66-8141-77bcdd8c6e5c" />


Di awal sesi terlihat baris `Response: 220 Welcome to Wired FTP Server (vsftpd 3.0.5)` sebagai banner, diikuti `Request: USER knights_agent` dan `Request: PASS N4v1_s3cur3_2026` sebagai kredensial yang dipakai.

### Temuan
| Item | Hasil |
|---|---|
| IP Server FTP penyerang | `198.51.100.7` |
| Banner software | `vsftpd 3.0.5` |
| Kredensial | `knights_agent` / `N4v1_s3cur3_2026` |
| Ukuran file malware | `524288 bytes` |

### Validasi Socker Server
```
nc [IP_Group] 3403  
```

<img width="1228" height="573" alt="image" src="https://github.com/user-attachments/assets/276b10e3-f964-4bed-96ef-988c464283f2" />

---

## Soal 17 — HTTP C2 Download Malware
**File**: `wired_http_c2.pcap`

### Apa yang terjadi
Alice membuat halaman web, dan Eiri memanfaatkan celah di halaman itu agar sistem Alice otomatis mengunduh file executable berbahaya dari server C2 (Command & Control) miliknya.

### Cara pembuktian
Filter `http.request` menampilkan 3 request GET dalam capture; dua di antaranya request normal (`/style.css`, `/`), dan satu lagi mencurigakan: `GET /navi_agent.exe HTTP/1.1`.

<img width="1918" height="885" alt="image" src="https://github.com/user-attachments/assets/f69c3722-c3a4-49c3-b10f-f899cea5c01c" />

Paket ini diklik untuk melihat detail header, dan langsung terbaca `Host: wired-update.net` serta baris otomatis Wireshark `[Full request URI: http://wired-update.net/navi_agent.exe]`.

<img width="954" height="427" alt="image" src="https://github.com/user-attachments/assets/68e74b91-9c02-4d02-b5cd-9f8ae4960b86" />

Untuk kode status, digunakan petunjuk `[Response in frame: 31]` yang ada di detail paket — klik untuk lompat langsung ke paket respons, yang menunjukkan `HTTP/1.1 200 OK` dengan header `Content-Disposition: attachment; filename="navi_agent.exe"`, mengonfirmasi file benar-benar terkirim sebagai unduhan.

<img width="954" height="427" alt="image" src="https://github.com/user-attachments/assets/01ab1da3-e56d-41cc-be49-3fb424e981cd" />


### Temuan
| Item | Hasil |
|---|---|
| Domain (Host) | `wired-update.net` |
| IP server penyerang | `203.0.113.42` |
| File executable | `navi_agent.exe` |
| Kode status HTTP | `200 OK` |

### Validasi Socket Server
```
nc [IP_Group] 3404
```

<img width="1273" height="571" alt="image" src="https://github.com/user-attachments/assets/a3b1e102-eb5c-47cb-8204-f7481b711fb3" />

---

## Soal 18 — Penanaman Malware via SMB
**File**: `wired_smb_transfer.pcapng`

### Apa yang terjadi
Eiri mengganti taktik dengan menanamkan file malware langsung ke sistem korban menggunakan protokol *file sharing* SMB (umum dipakai untuk berbagi file di jaringan Windows).

### Cara pembuktian
Filter `smb2 || smb` langsung menampilkan seluruh alur transaksi secara berurutan dan sudah cukup jelas dari kolom **Info** tanpa perlu Follow Stream

<img width="1918" height="1044" alt="image" src="https://github.com/user-attachments/assets/4e8dc742-8c4f-4b04-b545-559efe4dbf2d" />

mulai dari `Negotiate Protocol`, `Session Setup`, kemudian `Tree Connect Request, Tree: '\\10.7.1.50\ADMIN$'` — ini bagian penting, karena `ADMIN$` adalah *share* administratif tersembunyi Windows yang hanya bisa diakses dengan kredensial admin, indikasi penyerang sudah memiliki akses istimewa. Diikuti `Create Request, File: System32\wired_trojan_payload.exe` dan `Write Request Len:1028 ... File: System32\wired_trojan_payload.exe` yang menunjukkan proses penulisan file malware langsung ke folder sistem korban.

### Temuan
| Item | Hasil |
|---|---|
| Protokol dieksploitasi | SMB2 |
| IP Pengirim → Penerima | `10.7.3.100` → `10.7.1.50` |
| Folder tujuan | `System32` (via share `ADMIN$`) |
| File malware | `wired_trojan_payload.exe` |

### Validasi Socket Server
```
nc [IP_Group] 3405 
```

<img width="1213" height="691" alt="image" src="https://github.com/user-attachments/assets/c7d10851-e516-4ea0-a2cf-42191cce3df9" />

---

## Soal 19 — Email Pemerasan via SMTP
**File**: `wired_smtp_threat.pcap`

### Apa yang terjadi
Eiri mengirim email pemerasan (ransomware note) melalui SMTP tanpa enkripsi, mengancam akan membocorkan data korban jika tidak membayar tebusan.

### Cara pembuktian
Filter `smtp` menampilkan beberapa sesi email berbeda dalam satu capture (ada 3 pasangan IP yang saling berkomunikasi).

<img width="1918" height="991" alt="image" src="https://github.com/user-attachments/assets/01f5490e-1d89-434f-b389-80524f441233" />

Karena isi email tidak langsung terlihat dari list paket, setiap sesi ditelusuri satu per satu dengan klik kanan paket → **Follow → TCP Stream**, dicoba stream demi stream sampai ditemukan stream dari `185.234.72.19` (`attacker@darkwired.net`) menuju `203.0.113.100` (server `mail.protocol7.co.jp`, Postfix) 

<img width="1153" height="849" alt="image" src="https://github.com/user-attachments/assets/25331def-29c6-44ba-ad86-36c666945306" />


isinya jelas berupa ancaman: klaim password bocor (`pr0tocol_7_user`), klaim infeksi *ransomware*, tuntutan pembayaran 2 BTC, batas waktu **72 jam (3 hari)**, dan baris `MailClientID: 7719980706` di bagian akhir pesan.


### Temuan
| Item | Hasil |
|---|---|
| Email korban | `victim@protocol7.co.jp` |
| Password diklaim bocor | `pr0tocol_7_user` |
| Jenis malware | Ransomware |
| Batas waktu | 72 jam (3 hari) |
| MailClientID | `7719980706` |

- **IP Penyerang**: `185.234.72.19` (dari `attacker@darkwired.net`)
- **IP Server SMTP**: `203.0.113.100` (`mail.protocol7.co.jp`, Postfix)
- **Tuntutan**: 2 BTC ke alamat `bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh`

### Validasi Socket Server
```
nc [IP_Group] 3406   # Soal 19 ✓
```

<img width="1231" height="682" alt="image" src="https://github.com/user-attachments/assets/afb7bb9b-d625-41dd-98b8-bfa58ccfad0d" />

---

## Soal 20 — Komunikasi Malware via TLS Terenkripsi
**File**: `wired_tls_decrypt.pcapng` + `keyslogfile.txt`

### Apa yang terjadi
Sebagai taktik terakhir, Eiri menyembunyikan komunikasi malware (C2 beacon) di balik kanal HTTPS/TLS agar tidak mudah dicurigai sebagai traffic berbahaya.

### Cara pembuktian
Karena traffic terenkripsi, langkah pertama adalah mendaftarkan file `keyslogfile.txt` ke Wireshark melalui **Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename**. Setelah keylog terpasang, filter `tls.handshake.type == 1` menampilkan paket **Client Hello**, yang detailnya langsung menunjukkan `TLSv1.2` sebagai versi protokol dan `SNI=example.com` sebagai domain tujuan.

<img width="1918" height="969" alt="image" src="https://github.com/user-attachments/assets/4c43270e-715f-45d0-a61c-eb252db84693" />

Untuk membuktikan dekripsi berhasil, filter diganti ke `http` — jika keylog berhasil diterapkan, Wireshark otomatis menampilkan isi HTTP asli di balik enkripsi TLS (ditandai tab tambahan **"Decrypted TLS"** di panel bawah).

<img width="1918" height="996" alt="image" src="https://github.com/user-attachments/assets/db1d09db-c5d4-4496-afeb-70df2982eab2" />


Paket HTTP yang muncul diklik kanan → **Follow → TLS Stream**, menampilkan request lengkap: `HEAD / HTTP/1.1` dengan header `User-Agent: curl/7.62.0` — User-Agent ini mencurigakan karena menandakan traffic dibuat oleh script/tool otomatis, bukan browser manusia biasa, ciri khas *beacon* C2 yang menyamar sebagai traffic HTTPS normal.

### Temuan
| Item | Hasil |
|---|---|
| Versi TLS | TLS 1.2 |
| Domain (SNI) | `example.com` |
| IP Server HTTPS | `93.184.216.34` |
| User-Agent | `curl/7.62.0` |
| HTTP Method & Path | `HEAD /` |

### Validasi Socket Server
```
nc [IP_Group] 3407   
```


<img width="1252" height="712" alt="image" src="https://github.com/user-attachments/assets/f2c070f1-fa42-4a17-8a0d-bd49926ee2ca" />

--

## Kesimpulan Umum

Analisis kasus menunjukkan eskalasi taktik serangan Eiri: dari *brute-force* web sederhana, pencurian data via *keylogger* fisik (USB HID), pencurian file via FTP, distribusi malware lewat HTTP dan SMB (*lateral movement*), ancaman pemerasan via email tanpa enkripsi, hingga penyamaran komunikasi C2 di balik kanal TLS. Setiap kasus dibuktikan langsung dari bukti paket mentah di Wireshark — baik lewat *display filter* spesifik, fitur **Follow Stream** untuk merekonstruksi percakapan penuh, maupun dekripsi TLS menggunakan *keylog file* — sehingga seluruh *Indicator of Compromise* (IP, kredensial, nama file, pesan tersembunyi) dapat diidentifikasi secara faktual dan dapat diverifikasi ulang.



