# Dokumentasi Implementasi Apache CloudStack Private Cloud dan Deployment JobIQ

**Grup:** 8  
**Topik:** Implementasi Apache CloudStack Private Cloud pada VMware Workstation
**Tujuan Demo:** Membuat private cloud lokal yang dapat diakses melalui jaringan WiFi/hotspot, menjalankan instance Ubuntu, website nginx, dan aplikasi JobIQ.

---

## 1. Ringkasan Proyek

Pada praktikum ini saya membangun sebuah private cloud sederhana menggunakan **Apache CloudStack** di atas **VMware Workstation**. CloudStack digunakan sebagai platform manajemen cloud untuk membuat dan mengelola virtual machine, template, storage, host, system VM, dan instance.

Lab ini terdiri dari dua VM utama:

1. **cloudstack-lab** sebagai **Management Server**
2. **cloudstack-host** sebagai **Compute Host / KVM Host**

Setelah CloudStack berjalan, saya membuat sebuah **Ubuntu Cloud VM** dari template Ubuntu 24.04 cloud image. Di dalam instance Ubuntu tersebut, saya menjalankan:

* Web server sederhana menggunakan **nginx**
* Aplikasi web **JobIQ** berbasis Next.js
* Akses SSH untuk manajemen VM

Selain itu, saya juga mengatur agar dashboard CloudStack, JobIQ, dan website nginx dapat diakses dari perangkat lain selama masih berada dalam satu jaringan WiFi/hotspot.

---

## 2. Arsitektur Lab

### 2.1 Topologi Umum

```text
Laptop Windows
│
├── VMware Workstation
│   │
│   ├── cloudstack-lab
│   │   └── Apache CloudStack Management Server
│   │
│   └── cloudstack-host
│       ├── KVM / libvirt
│       ├── CloudStack Agent
│       ├── System VM
│       ├── Virtual Router
│       └── Ubuntu Instance
│
└── WiFi / Hotspot
    └── Device dosen / teman mengakses service melalui IP WiFi laptop
```

---

## 3. Tabel IP Address

| Komponen             | Fungsi                      |     IP Address | Keterangan                                 |
| -------------------- | --------------------------- | -------------: | ------------------------------------------ |
| Windows VMnet8       | Gateway internal VMware NAT |  `172.20.10.1` | Gateway untuk jaringan internal CloudStack |
| cloudstack-lab       | Management Server           |  `172.20.10.2` | Dashboard CloudStack                       |
| cloudstack-host      | Compute Host / KVM Host     |  `172.20.10.4` | Host yang menjalankan VM                   |
| Secondary Storage VM | System VM                   | `172.20.10.11` | Mengelola secondary storage/template       |
| Console Proxy VM     | System VM                   | `172.20.10.12` | Menangani View Console/noVNC               |
| Ubuntu Instance      | User VM                     | `172.20.10.13` | Menjalankan nginx dan JobIQ                |
| IP WiFi Laptop       | Akses dari device lain      |   berubah-ubah | Diambil dari `ipconfig`                    |

Subnet internal yang digunakan:

```text
Network: 172.20.10.0/28
Netmask: 255.255.255.240
Gateway: 172.20.10.1
```

---

## 4. Software dan Komponen yang Digunakan

| Komponen                   | Fungsi                                        |
| -------------------------- | --------------------------------------------- |
| VMware Workstation         | Menjalankan VM lab di laptop                  |
| Ubuntu Server 24.04        | OS untuk management server dan host           |
| Apache CloudStack 4.20.3.0 | Platform private cloud                        |
| KVM/libvirt                | Hypervisor di cloudstack-host                 |
| MySQL                      | Database CloudStack                           |
| nginx                      | Web server untuk template dan HTML demo       |
| NFS                        | Secondary storage CloudStack                  |
| Ubuntu Cloud Image 24.04   | Template untuk instance Ubuntu                |
| Next.js                    | Framework aplikasi JobIQ                      |
| Node.js/npm                | Runtime aplikasi JobIQ                        |
| Windows Portproxy          | Mengekspos service internal VM ke device lain |

---

## 5. Alasan Desain

### 5.1 Kenapa Menggunakan Ubuntu Server?

Ubuntu Server dipilih karena:

1. Kompatibel dengan CloudStack dan KVM.
2. Ringan dibandingkan OS desktop.
3. Mudah dikonfigurasi melalui terminal.
4. Banyak dokumentasi dan package tersedia melalui `apt`.
5. Cocok untuk kebutuhan server seperti CloudStack Management, KVM host, NFS, nginx, dan Node.js.

### 5.2 Kenapa Menggunakan Ubuntu Cloud Image?

Ubuntu Cloud Image digunakan karena image ini memang dirancang untuk cloud environment. Keuntungannya:

1. Boot lebih cepat.
2. Sudah mendukung cloud-init.
3. Cocok dijadikan template CloudStack.
4. Lebih ringan daripada ISO installer biasa.
5. Bisa langsung dipakai untuk membuat VM tanpa instalasi manual dari ISO.

File image yang digunakan:

```text
noble-server-cloudimg-amd64.img
```

Kemudian didaftarkan ke CloudStack sebagai template dalam format QCOW2.

### 5.3 Kenapa Menggunakan nginx?

nginx digunakan untuk dua kebutuhan:

1. Menyediakan file template Ubuntu Cloud Image melalui HTTP.
2. Menjalankan website HTML sederhana di dalam Ubuntu VM.

CloudStack dapat mengambil template dari URL HTTP, sehingga nginx cocok untuk menyajikan file `.qcow2`.

### 5.4 Kenapa Menggunakan VMnet8 NAT?

Awalnya VM menggunakan Bridged Network. Masalahnya, saat laptop pindah WiFi/hotspot, IP VM bisa ikut berubah. Hal ini membuat CloudStack tidak stabil karena CloudStack membutuhkan IP yang konsisten.

Akhirnya digunakan **VMnet8 NAT custom** dengan subnet tetap:

```text
172.20.10.0/28
```

Keuntungannya:

1. IP CloudStack tetap walaupun laptop pindah WiFi.
2. cloudstack-lab tetap `172.20.10.2`.
3. cloudstack-host tetap `172.20.10.4`.
4. Ubuntu VM tetap `172.20.10.13`.
5. Akses dari device luar cukup lewat Windows portproxy.

---

## 6. Konfigurasi VMware Network

### 6.1 Virtual Network Editor

Pada VMware Workstation:

```text
Edit → Virtual Network Editor → Change Settings
```

VMnet8 diubah menjadi:

```text
Type: NAT
Subnet IP: 172.20.10.0
Subnet Mask: 255.255.255.240
Gateway IP: 172.20.10.1
```

DHCP sebaiknya tidak digunakan untuk IP penting CloudStack. IP utama dibuat statis.

### 6.2 Network Adapter VM

Pada kedua VM:

```text
cloudstack-lab
cloudstack-host
```

Network Adapter diubah menjadi:

```text
Custom: Specific virtual network → VMnet8
```

Tidak lagi menggunakan Bridged.

---

## 7. Konfigurasi IP Statis

### 7.1 cloudstack-lab

Login ke management server:

```powershell
ssh cloud@172.20.10.2
```

Konfigurasi netplan:

```bash
sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg > /dev/null <<'EOF'
network: {config: disabled}
EOF

sudo tee /etc/netplan/50-cloud-init.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 172.20.10.2/28
      routes:
        - to: default
          via: 172.20.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
EOF

sudo netplan apply
```

Cek:

```bash
ip -4 addr
ip route
ping -c 4 172.20.10.4
```

### 7.2 cloudstack-host

Login ke compute host:

```powershell
ssh host@172.20.10.4
```

Pastikan host berada di IP:

```text
172.20.10.4/28
```

Cek:

```bash
ip -4 addr
ip route
ping -c 4 172.20.10.2
```

---

## 8. Konfigurasi CloudStack Management Server

Pada `cloudstack-lab`, service utama yang digunakan:

```bash
sudo systemctl status mysql --no-pager
sudo systemctl status nginx --no-pager
sudo systemctl status nfs-server --no-pager
sudo systemctl status cloudstack-management --no-pager
```

Jika dashboard tidak dapat dibuka, restart service:

```bash
sudo systemctl restart mysql
sudo systemctl restart nginx
sudo systemctl restart nfs-server
sudo systemctl restart cloudstack-management
```

Cek dashboard:

```bash
curl -I http://172.20.10.2:8080/client/
```

Akses dari browser laptop:

```text
http://172.20.10.2:8080/client
```

---

## 9. Konfigurasi CloudStack Host

Pada `cloudstack-host`, service utama:

```bash
sudo systemctl status libvirtd --no-pager
sudo systemctl status cloudstack-agent --no-pager
```

Restart jika diperlukan:

```bash
sudo systemctl restart libvirtd
sudo systemctl restart cloudstack-agent
```

Cek VM yang berjalan di KVM:

```bash
sudo virsh list --all
```

Contoh VM yang muncul:

```text
r-xx-VM       Virtual Router
s-xx-VM       Secondary Storage VM
v-xx-VM       Console Proxy VM
i-2-58-VM     Ubuntu user instance
```

---

## 10. Konfigurasi CloudStack UI

Melalui dashboard CloudStack:

```text
http://172.20.10.2:8080/client
```

Dilakukan konfigurasi:

1. Zone
2. Pod
3. Cluster
4. Host
5. Primary Storage
6. Secondary Storage
7. System VM Template
8. Ubuntu VM Template
9. Instance Ubuntu

### 10.1 Zone

Zone dibuat dengan nama:

```text
zone1
```

Hypervisor:

```text
KVM
```

### 10.2 Host

Host yang ditambahkan:

```text
cloudstack-host
IP: 172.20.10.4
Hypervisor: KVM
```

### 10.3 Primary Storage

Primary storage menggunakan local storage/libvirt di host.

### 10.4 Secondary Storage

Secondary storage menggunakan NFS dari management server:

```text
172.20.10.2:/export/secondary
```

---

## 11. Registrasi Template Ubuntu Cloud Image

### 11.1 Download Ubuntu Cloud Image

Image Ubuntu Cloud 24.04 didownload:

```bash
sudo mkdir -p /var/www/html/iso
cd /var/www/html/iso

sudo wget -O ubuntu-24-cloud.qcow2 https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
sudo chmod 644 ubuntu-24-cloud.qcow2
```

Cek URL:

```bash
curl -I http://172.20.10.2/iso/ubuntu-24-cloud.qcow2
```

Target output:

```text
HTTP/1.1 200 OK
```

### 11.2 Register Template dari CloudStack UI

Pada CloudStack UI:

```text
Images → Templates → Register Template from URL
```

Isi:

```text
URL: http://172.20.10.2/iso/ubuntu-24-cloud.qcow2
Name: ubuntu-24-cloud
Description: Ubuntu 24.04 Cloud Image
Zone: zone1
Hypervisor: KVM
Format: QCOW2
OS Type: Other Ubuntu (64-bit)
Arch: x86_64
HVM: checked
```

Setelah itu tunggu sampai template status:

```text
Ready
```

---

## 12. Membuat Ubuntu Instance

Dari CloudStack UI:

```text
Compute → Instances → Add Instance
```

Template yang digunakan:

```text
ubuntu-24-cloud
```

Instance yang dibuat:

```text
Name: ubuntu
IP: 172.20.10.13
```

User-data/cloud-init digunakan untuk membuat user dan password Ubuntu. Password tidak ditulis di dokumentasi ini. Gunakan password yang disimpan di catatan pribadi.

Contoh user-data dengan placeholder:

```yaml
#cloud-config
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false

chpasswd:
  list: |
    ubuntu:<PASSWORD_VM_UBUNTU>
  expire: false

ssh_pwauth: true
```

---

## 13. Konfigurasi Disk Tambahan pada Ubuntu VM

Setelah instance Ubuntu dibuat, dilakukan pengecekan disk:

```bash
lsblk
df -h
```

Ditemukan:

```text
vda = root disk
vdb = data disk 15G
```

Karena root disk kecil, aplikasi diletakkan di data disk `vdb`.

Mount disk:

```bash
sudo mkdir -p /srv/web
sudo mount /dev/vdb /srv/web
```

Agar mount otomatis setelah reboot:

```bash
echo '/dev/vdb /srv/web ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

Cek:

```bash
df -h
```

Target:

```text
/dev/vdb mounted on /srv/web
```

---

## 14. Setup nginx HTML Website

Install nginx:

```bash
sudo apt update
sudo apt install -y nginx
```

Buat folder web:

```bash
sudo mkdir -p /srv/web/html
echo "<h1>Hello from CloudStack Ubuntu Cloud Image</h1>" | sudo tee /srv/web/html/index.html
```

Edit konfigurasi nginx:

```bash
sudo nano /etc/nginx/sites-available/default
```

Ubah root menjadi:

```nginx
root /srv/web/html;
```

Test konfigurasi:

```bash
sudo nginx -t
```

Restart nginx:

```bash
sudo systemctl enable --now nginx
sudo systemctl restart nginx
```

Cek dari Ubuntu VM:

```bash
curl -I http://localhost
```

Cek dari laptop:

```text
http://172.20.10.13
```

---

## 15. Deploy Aplikasi JobIQ

### 15.1 Clone Repository

Masuk ke data disk:

```bash
cd /srv/web
sudo chown -R ubuntu:ubuntu /srv/web
```

Clone repository JobIQ:

```bash
git clone -b TRUE_AI https://github.com/AlexanderChristhian/JobIQ.git
cd JobIQ/src
```

### 15.2 Setup npm Cache

Karena root disk kecil, cache npm dipindahkan ke `/srv/web`:

```bash
mkdir -p /srv/web/.npm-cache
npm config set cache /srv/web/.npm-cache
npm config get cache
```

Install dependency:

```bash
npm install --cache /srv/web/.npm-cache
```

### 15.3 Build JobIQ

Build aplikasi:

```bash
npm run build
```

Jika berhasil:

```bash
npm start
```

Akses dari laptop:

```text
http://172.20.10.13:3000
```

---

## 16. Perbaikan Error JobIQ

### 16.1 Error Login karena Cookie Secure

Masalah:

```text
API login berhasil via curl, tetapi browser tidak bisa login.
```

Penyebab:

```text
Cookie session diset Secure saat aplikasi berjalan di HTTP biasa, bukan HTTPS.
```

File yang dicek:

```bash
grep -R "secure:" -n app lib | head -50
grep -R "jobiq_session" -n app lib | head -50
```

File yang diubah:

```text
app/api/auth/login/route.ts
app/api/auth/signup/route.ts
```

Bagian cookie diubah agar tidak menggunakan secure cookie pada HTTP lab:

```ts
secure: false,
```

Setelah itu:

```bash
npm run build
npm start
```

### 16.2 Error Next.js Google Font

Masalah:

```text
Failed to fetch `Manrope` from Google Fonts.
```

Penyebab:

```text
VM mengalami timeout ketika Next.js mencoba mengambil font dari Google Fonts saat build.
```

File yang diubah:

```text
app/layout.tsx
```

Sebelumnya:

```ts
import { Manrope } from "next/font/google";
const manrope = Manrope({ subsets: ["latin"] });
<body className={`${manrope.className} antialiased`}>
```

Diubah menjadi:

```tsx
<body className="font-sans antialiased">
```

Import `Manrope` dan konstanta `manrope` dihapus.

Build ulang:

```bash
npm run build
npm start
```

---

## 17. Membuat Service Bisa Diakses Device Lain

Masalah utama: IP internal CloudStack berada di VMnet8, sehingga device lain di WiFi tidak bisa langsung mengakses:

```text
172.20.10.2
172.20.10.13
```

Solusi: menggunakan Windows `netsh interface portproxy`.

### 17.1 Cek IP WiFi Laptop

Di Windows:

```cmd
ipconfig
```

Cari:

```text
Wireless LAN adapter Wi-Fi
IPv4 Address
```

Contoh:

```text
192.168.18.13
```

### 17.2 Portproxy

Jalankan CMD/PowerShell sebagai Administrator:

```cmd
netsh interface portproxy reset

netsh interface portproxy add v4tov4 listenaddress=IP_WIFI_LAPTOP listenport=8080 connectaddress=172.20.10.2 connectport=8080
netsh interface portproxy add v4tov4 listenaddress=IP_WIFI_LAPTOP listenport=3000 connectaddress=172.20.10.13 connectport=3000
netsh interface portproxy add v4tov4 listenaddress=IP_WIFI_LAPTOP listenport=8081 connectaddress=172.20.10.13 connectport=80

netsh interface portproxy show all
```

Contoh jika IP WiFi laptop adalah `192.168.18.13`:

```cmd
netsh interface portproxy reset

netsh interface portproxy add v4tov4 listenaddress=192.168.18.13 listenport=8080 connectaddress=172.20.10.2 connectport=8080
netsh interface portproxy add v4tov4 listenaddress=192.168.18.13 listenport=3000 connectaddress=172.20.10.13 connectport=3000
netsh interface portproxy add v4tov4 listenaddress=192.168.18.13 listenport=8081 connectaddress=172.20.10.13 connectport=80

netsh interface portproxy show all
```

### 17.3 Firewall Rule

Jalankan sekali:

```cmd
netsh advfirewall firewall add rule name="CloudStack Dashboard 8080" dir=in action=allow protocol=TCP localport=8080
netsh advfirewall firewall add rule name="JobIQ 3000" dir=in action=allow protocol=TCP localport=3000
netsh advfirewall firewall add rule name="Ubuntu HTML 8081" dir=in action=allow protocol=TCP localport=8081
```

### 17.4 Link Demo untuk Device Lain

Jika IP WiFi laptop adalah:

```text
192.168.18.13
```

Maka link demo:

```text
CloudStack Dashboard:
http://192.168.18.13:8080/client

JobIQ:
http://192.168.18.13:3000

HTML nginx:
http://192.168.18.13:8081
```

Syarat:

```text
Device dosen/teman harus berada dalam jaringan WiFi/hotspot yang sama.
```

---

## 18. Konfigurasi View Console

View Console CloudStack menggunakan Console Proxy VM:

```text
Console Proxy IP: 172.20.10.12
```

Masalah yang sempat terjadi:

```text
Dashboard bisa dibuka, tetapi View Console timeout.
```

Setelah dicek:

```powershell
Test-NetConnection 172.20.10.12 -Port 80
```

Awalnya gagal dari Windows, tetapi berhasil dari cloudstack-host. Artinya Console Proxy hidup, tetapi Windows tidak dapat langsung mengaksesnya.

### 18.1 Cek Interface Index VMnet8

Di Windows:

```cmd
netsh interface ipv4 show interfaces
```

VMnet8 memiliki index:

```text
16
```

### 18.2 Tambah Persistent Route

Jalankan CMD/PowerShell sebagai Administrator:

```cmd
route delete 172.20.10.12
arp -d 172.20.10.12
route -p add 172.20.10.12 mask 255.255.255.255 172.20.10.4 metric 1 IF 16
```

Cek:

```powershell
Test-NetConnection 172.20.10.12 -Port 80
```

Target:

```text
TcpTestSucceeded : True
```

### 18.3 Fix Tambahan di cloudstack-host

Jika View Console gagal setelah host restart, jalankan di `cloudstack-host`:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.cloudbr0.rp_filter=0

sudo iptables -I FORWARD 1 -s 172.20.10.1/32 -d 172.20.10.12/32 -j ACCEPT
sudo iptables -I FORWARD 1 -s 172.20.10.12/32 -d 172.20.10.1/32 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 172.20.10.1/32 -d 172.20.10.12/32 -j SNAT --to-source 172.20.10.4
```

Setelah itu View Console dapat dibuka dari laptop utama.

Catatan:

```text
View Console dari device dosen/teman belum tentu otomatis bisa karena URL console menggunakan IP internal 172.20.10.12.
Untuk demo, View Console dibuka dari laptop utama.
```

---

## 19. Kendala yang Dihadapi dan Solusi

### 19.1 Template URL 404

Masalah:

```text
curl -I http://172.20.10.2/iso/ubuntu-24-cloud.qcow2
HTTP/1.1 404 Not Found
```

Penyebab:

```text
File qcow2 awalnya berada di VM yang salah, sehingga nginx di management server tidak menemukan file tersebut.
```

Solusi:

```bash
sudo mkdir -p /var/www/html/iso
sudo mv /tmp/ubuntu-24-cloud.qcow2 /var/www/html/iso/
sudo chmod 644 /var/www/html/iso/ubuntu-24-cloud.qcow2
```

Cek ulang:

```bash
curl -I http://172.20.10.2/iso/ubuntu-24-cloud.qcow2
```

Target:

```text
HTTP/1.1 200 OK
```

---

### 19.2 Template “Connection Refused” / Not Ready

Masalah:

```text
Template sempat Not Ready dan status zone menunjukkan Connection refused.
```

Solusi:

1. Pastikan URL template dapat diakses.
2. Pastikan nginx berjalan.
3. Tunggu proses download template selesai.
4. Refresh CloudStack UI.

Akhirnya template berubah menjadi:

```text
Ready
```

---

### 19.3 IP Management Server Berubah

Masalah:

```text
cloudstack-lab berubah dari 172.20.10.2 menjadi 172.20.10.5
```

Penyebab:

```text
DHCP VMnet8 memberikan IP dinamis.
```

Solusi:

1. Disable cloud-init network config.
2. Set IP statis via netplan ke `172.20.10.2`.
3. Apply netplan.
4. Restart CloudStack service.

---

### 19.4 CloudStack Dashboard 503

Masalah:

```text
curl -I http://localhost:8080/client/
HTTP/1.1 503 Service Unavailable
```

Penyebab:

```text
CloudStack Management belum siap atau sedang error setelah perubahan IP/network.
```

Solusi:

```bash
sudo systemctl restart mysql
sudo systemctl restart nginx
sudo systemctl restart nfs-server
sudo systemctl restart cloudstack-management
```

Tunggu 2–3 menit lalu cek ulang.

---

### 19.5 Root Disk Ubuntu VM Kecil

Masalah:

```text
/dev/vda1 hanya sekitar 2.4G dan cepat penuh.
```

Solusi:

Data aplikasi dipindahkan ke disk tambahan:

```text
/dev/vdb → /srv/web
```

Mount:

```bash
sudo mkdir -p /srv/web
sudo mount /dev/vdb /srv/web
```

Persist:

```bash
echo '/dev/vdb /srv/web ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

---

### 19.6 npm EACCES

Masalah:

```text
npm config set cache /srv/web/.npm-cache --global
EACCES: permission denied, mkdir '/usr/etc'
```

Penyebab:

```text
Command global mencoba menulis ke direktori sistem.
```

Solusi:

Gunakan konfigurasi lokal user:

```bash
mkdir -p /srv/web/.npm-cache
npm config set cache /srv/web/.npm-cache
npm config get cache
```

---

### 19.7 npm ETIMEDOUT

Masalah:

```text
npm install mengalami ETIMEDOUT
```

Penyebab:

```text
Koneksi internet VM tidak stabil.
```

Solusi:

1. Ulangi `npm install`.
2. Gunakan cache di `/srv/web`.
3. Pastikan DNS dan route aktif.
4. Jika perlu gunakan jaringan yang lebih stabil.

---

### 19.8 Next.js Build Gagal karena Google Font

Masalah:

```text
Failed to fetch `Manrope` from Google Fonts.
```

Solusi:

Hapus penggunaan:

```ts
import { Manrope } from "next/font/google";
```

Ganti body menjadi:

```tsx
<body className="font-sans antialiased">
```

Build ulang:

```bash
npm run build
```

---

### 19.9 Login JobIQ Tidak Berhasil

Masalah:

```text
API login berhasil via curl, tetapi browser tidak berpindah halaman setelah login.
```

Penyebab:

```text
Cookie session menggunakan Secure flag padahal aplikasi berjalan di HTTP.
```

Solusi:

Pada file login dan signup route, cookie diubah:

```ts
secure: false,
```

Build ulang:

```bash
npm run build
npm start
```

---

### 19.10 System VM Disconnected

Masalah:

```text
Console Proxy VM dan Secondary Storage VM sempat Running tetapi Agent state Disconnected.
```

Solusi:

1. Reboot system VM dari UI.
2. Jika tidak berhasil, destroy system VM agar CloudStack membuat ulang.
3. Restart service CloudStack Management dan CloudStack Agent.

Command dari host:

```bash
sudo virsh list --all
sudo virsh destroy v-xx-VM
sudo virsh start v-xx-VM
```

Atau destroy dari CloudStack UI.

---

### 19.11 View Console Timeout

Masalah:

```text
View Console membuka URL 172.20.10.12 tetapi timeout.
```

Cek dari Windows:

```powershell
Test-NetConnection 172.20.10.12 -Port 80
```

Cek dari host:

```bash
curl -I http://172.20.10.12/resource/noVNC/vnc.html
```

Hasilnya:

```text
Dari Windows gagal.
Dari cloudstack-host berhasil.
```

Solusi:

1. Tambahkan route persistent Windows ke consoleproxy melalui cloudstack-host.
2. Aktifkan forwarding dan SNAT di cloudstack-host.

---

## 20. Checklist Demo

Sebelum demo, cek:

### 20.1 CloudStack Internal

```text
http://172.20.10.2:8080/client
```

### 20.2 Ubuntu VM

```powershell
ssh ubuntu@172.20.10.13
```

### 20.3 HTML nginx

```text
http://172.20.10.13
```

### 20.4 JobIQ

```text
http://172.20.10.13:3000
```

### 20.5 Device Lain

Setelah portproxy:

```text
http://IP_WIFI_LAPTOP:8080/client
http://IP_WIFI_LAPTOP:3000
http://IP_WIFI_LAPTOP:8081
```

### 20.6 View Console

```powershell
Test-NetConnection 172.20.10.12 -Port 80
```

Target:

```text
TcpTestSucceeded : True
```

---

## 21. Prosedur Jika Pindah WiFi

Setiap pindah WiFi:

1. Jalankan:

```cmd
ipconfig
```

2. Ambil IP dari:

```text
Wireless LAN adapter Wi-Fi → IPv4 Address
```

3. Jalankan CMD/PowerShell sebagai Administrator:

```cmd
netsh interface portproxy reset

netsh interface portproxy add v4tov4 listenaddress=IP_BARU listenport=8080 connectaddress=172.20.10.2 connectport=8080
netsh interface portproxy add v4tov4 listenaddress=IP_BARU listenport=3000 connectaddress=172.20.10.13 connectport=3000
netsh interface portproxy add v4tov4 listenaddress=IP_BARU listenport=8081 connectaddress=172.20.10.13 connectport=80

netsh interface portproxy show all
```

4. Link baru menjadi:

```text
CloudStack:
http://IP_BARU:8080/client

JobIQ:
http://IP_BARU:3000

HTML:
http://IP_BARU:8081
```

View Console tidak perlu diubah hanya karena pindah WiFi, karena View Console menggunakan jaringan internal VMnet8.

---

## 22. Catatan Keamanan

1. Password Linux VM tidak ditulis di dokumentasi.
2. Password disimpan di catatan pribadi.
3. Untuk dosen/teman, cukup berikan akses dashboard CloudStack jika diperlukan.
4. Jangan memberikan password user Linux seperti `cloud`, `host`, atau `ubuntu` ke pihak lain.
5. Portproxy hanya dibuka saat demo.
6. Setelah demo, portproxy dapat dihapus:

```cmd
netsh interface portproxy reset
```

---

## 23. Kesimpulan

Lab ini berhasil membangun private cloud sederhana menggunakan Apache CloudStack. Management Server dan Compute Host berjalan di atas VMware Workstation dengan jaringan internal VMnet8 NAT agar IP tetap stabil meskipun laptop berpindah WiFi.

CloudStack berhasil digunakan untuk:

1. Mengelola host KVM.
2. Mengelola primary dan secondary storage.
3. Menjalankan system VM.
4. Mendaftarkan template Ubuntu Cloud Image.
5. Membuat instance Ubuntu.
6. Menjalankan nginx dan aplikasi JobIQ pada instance.
7. Mengekspos dashboard, JobIQ, dan HTML nginx ke device lain melalui portproxy Windows.

Kendala terbesar selama proses adalah perubahan IP akibat network mode, error template URL, root disk kecil, error npm/Next.js akibat internet timeout, cookie login HTTP, dan masalah View Console karena Console Proxy VM. Semua kendala tersebut berhasil ditangani melalui konfigurasi IP statis, VMnet8 NAT, portproxy, perbaikan konfigurasi aplikasi, serta routing khusus untuk Console Proxy.
