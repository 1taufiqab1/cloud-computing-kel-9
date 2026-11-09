# Alur Eksperimen Cloud Computing Kelompok 9 Versi 2

## Ringkasan konfigurasi

| Komponen | Pengaturan |
|---|---|
| Laptop/host | Windows 11, RAM 16 GB, Intel Core i7-10750H |
| VirtualBox – VM Proxmox | 4 vCPU, RAM 5 GB, disk sistem 32 GB |
| Ubuntu VM di Proxmox | 3 vCPU, RAM 3 GB, disk sistem 8 GB |
| Storage data A | VDI 8 GB, tanpa LUKS di host Proxmox |
| Storage data B | VDI 8 GB, LUKS di host Proxmox |
| Virtual disk data di Ubuntu | RAW 4 GB per disk |
| Ukuran file FIO | 512 MiB per disk |
| Durasi setiap pengujian | 40 detik |
| Pengulangan | 10 kali untuk setiap kondisi dan workload |
| Workload | Sequential read, sequential write, random read, random write |
| Jeda antar-run | 3 detik |
| Jeda antar-kelompok workload | 30 detik |
| Jumlah pengujian utama | 80 run (4 workload × 2 kondisi × 10 pengulangan) |

**Kondisi A** = storage tanpa enkripsi. **Kondisi B** = storage dengan LUKS di host Proxmox.

---

## 1. Siapkan aplikasi dan ISO

1. [Install VirtualBox **platform packages 7.2.22**.](https://www.virtualbox.org/wiki/Downloads)
2. [Download ISO **Proxmox VE 9.2**.](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)
3. [Download ISO **Ubuntu Server 26.04.1 LTS**.](https://ubuntu.com/download/server#how-to-install-tab-lts)
4. Opsional: cek SHA256 untuk memastikan file download tidak corrupt.

```powershell
Get-FileHash "path:lokasi\file" -Algorithm SHA256
```

| File | SHA256 |
|---|---|
| `VirtualBox-7.2.22-175245-Win.exe` | [`afd0ff1656cbe04ed88ce619594ae021419d97cce89fa9bdc443e9a963f1713f`](https://download.virtualbox.org/virtualbox/7.2.22/SHA256SUMS) |
| `proxmox-ve_9.2-1.iso` | [`4e88fe416df9b527624a175f24c9aa07c714d3332afb1ee3dbf3879573ef2c6c`](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso) |
| `ubuntu-26.04.1-live-server-amd64.iso` | [`cc8a95cde20f6ced61a322420de00f10cc3c90ced545daa46cb9c1a117f1d927`](https://releases.ubuntu.com/resolute/SHA256SUMS) |

## 2. Buat VM Proxmox di VirtualBox

1. Buat VM baru dan pilih ISO Proxmox VE sebagai media instalasi.
2. Isi konfigurasi:
   - Name: `PVE-Lab`
   - OS: Linux
   - Distribution: Debian
   - Version: Debian 13 Trixie (64-bit)
   - Base Memory: **5120 MB**
   - Number of CPUs: **4**
   - Disk: **32 GB**

![gambar1](https://lh3.googleusercontent.com/d/1efS_YexYjVgA6P-UlXnREDxr-gAE0vLe)

### 2.1. Cek virtualisasi Windows

1. Buka Task Manager → Performance.
2. Pastikan `Virtualization` berstatus **Enabled**.
3. Kalau belum aktif, aktifkan Intel Virtualization Technology (VT-x) melalui BIOS/UEFI laptop.

![gambar2](https://lh3.googleusercontent.com/d/1wXrQY2xNd5v-G6B8I1CAhaNlPYJg6Y1r)

### 2.2. Jika Proxmox tidak bisa mengakses VT-x

Pada konfigurasi laptop ini, nested VT-x baru berjalan setelah fitur virtualisasi Windows yang bentrok dimatikan. Konsekuensinya, WSL/fitur terkait tidak bisa dipakai sementara. Aktifkan kembali setelah eksperimen selesai.

**Matikan Memory Integrity:**

1. Start → Windows Security.
2. Device security → Core isolation details.
3. Matikan `Memory integrity`.
4. Jangan restart dulu.

**Matikan fitur virtualisasi Windows:**

1. Tekan `Windows + R`.
2. Jalankan `optionalfeatures.exe`.
3. Hilangkan centang `Virtual Machine Platform` dan `Windows Subsystem for Linux`.
4. Jangan restart dulu.

**Cegah hypervisor Windows berjalan saat boot:**

1. Klik kanan Start → Terminal (Admin).
2. Jalankan:

```powershell
bcdedit /set hypervisorlaunchtype off
```

3. Restart laptop.

### 2.3. Aktifkan Nested VT-x

Buka PowerShell Windows, lalu jalankan:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "PVE-Lab" --nested-hw-virt=on
```

### 2.4. Port forwarding

Langkah ini adalah konfigurasi awal yang pernah dicoba. Akses Proxmox yang dipakai pada konfigurasi akhir adalah melalui Adapter 2 Host-only di `192.168.56.10` (lihat bagian 6).

1. Buka Settings VM → ubah tampilan dari Basic ke Expert.
2. Network → Port Forwarding → klik tanda `+` pojok kanan atas.
3. Isi:
   - Protocol: TCP
   - Host IP: `127.0.0.1`
   - Host Port: `8006`
   - Guest IP: `10.0.2.15`
   - Guest Port: `8006`

![gambar3](https://lh3.googleusercontent.com/d/14IziFoD7k5EclyCaPa3xdNDqyMQDkUke)
![gambar4](https://lh3.googleusercontent.com/d/1qJTOsHQlCpvnMU-jPx7q1VIWTpnXyf6t)

## 3. Install Proxmox VE

1. Nyalakan VM `PVE-Lab`.
2. Pilih **Install Proxmox VE (Graphical)**.
3. Ikuti pengaturan awal:
   - Langsung Next
   - Next
   - Country: Indonesia
   - Time zone: Asia/Jakarta
   - Keyboard: US English
   - Email: 1taufiqab1@gmail.com `isi alamat email sendiri`
   - Password: Proxmox1!
   - Management interface: biarkan default apa adanya
   - Hostname (FQDN): `pve.lab.test`
   - IP Address (CIDR): `10.0.2.15/24`
   - Gateway: `10.0.2.2`
   - DNS Server: `10.0.2.3`
4. Klik Next dan tunggu sampai instalasi selesai.

### 3.1. Jika VM kembali ke layar installer

1. Power off VM.
2. VirtualBox → Settings → Storage.
3. Pilih ISO Proxmox yang masih terpasang pada Optical Drive.
4. Klik ikon CD di sebelah kanan → Remove.

![gambar5](https://lh3.googleusercontent.com/d/1FS3v1iRM79dlzVBZ93-dx6s5vZBUtKIu)

5. Buka System → Motherboard.
6. Hilangkan centang Optical dan pastikan Hard Disk berada di urutan paling atas. Bisa atur urutan dengan tombol atas bawah di sampingnya.
7. Save.

| Sebelum | Sesudah |
|---|---|
| ![gambar6](https://lh3.googleusercontent.com/d/1PYaSPkyjH4srRIJBNnzYTEyDh35LTEr3) | ![gambar7](https://lh3.googleusercontent.com/d/16pNq7SFjMzJ6Xtsziv3K70vXb8ay6WQC)  |

### 3.2. Masuk ke dashboard Proxmox

1. Nyalakan VM dan tunggu sampai Proxmox selesai boot sampai muncul seperti gambar di bawah.

![gambar8](https://lh3.googleusercontent.com/d/1qza2Bt5DRQYBrWBWQkn1dRxDrtsh9J4j)

2. Akses melalui browser `https://127.0.0.1:8006/`.
3. Login:
   - Username: `root`
   - Password: Proxmox1!
   - Realm: `Linux PAM standard authentication`
4. Buka node `pve` → Repositories.
5. Disable dua repository enterprise.
6. Add → pilih repository **No-Subscription** → Langsung next next saja.

![gambar9](https://lh3.googleusercontent.com/d/1MtzteLOfJWra5tTt81-bnrTPYAcmv7jD)

7. Buka node `pve` → Shell, jalankan:

Note: pastikan pakai wifi karena ukuran update lumayan besar.

```bash
apt update
apt full-upgrade -y
reboot
```



## 4. Buat dua virtual disk data di VirtualBox

1. Matikan Proxmox dengan aman dari shell:

```bash
poweroff
```

2. VirtualBox → pilih VM `PVE-Lab` → Settings → Storage.
3. Pilih controller SATA → Klik icon Add Hard Disk → Create.

![gambar10](https://lh3.googleusercontent.com/d/1Myqc8jE3oG-NbUbMb9O9sy08u-iYF-yb)

4. Buat dua disk VDI, masing-masing **8 GB**:
   - `PVE-Lab_1` = storage biasa/tanpa LUKS.
   - `PVE-Lab_2` = storage yang akan memakai LUKS.
5. Klik Choose untuk memasang kedua disk.
6. Pastikan urutan disk dari atas ke bawah:
   - `PVE-Lab.vdi`
   - `PVE-Lab_1.vdi`
   - `PVE-Lab_2.vdi`

![gambar11](https://lh3.googleusercontent.com/d/14SmY1yCI4LW_Cocsp-vntVpFBi31B-wq)

## 5. Siapkan storage A dan B di Proxmox

1. Nyalakan VM Proxmox dan buka dashboard melalui `https://127.0.0.1:8006/`.
2. Buka node `pve` → Shell.
3. Jalankan command-command di bawah ini.

### 5.1. Storage A — tanpa enkripsi

```bash
mkfs.ext4 -L PVE-Lab_1 /dev/sdb
```

### 5.2. Storage B — dengan LUKS

Install cryptsetup:

```bash
apt install cryptsetup systemd-cryptsetup -y
```

Format partisi/disk dengan LUKS:

```bash
cryptsetup luksFormat /dev/sdc
```

Ketik `YES` dengan huruf kapital, lalu masukkan passphrase `123` agar mudah diingat.

Buka container LUKS dan buat filesystem ext4:

```bash
cryptsetup luksOpen /dev/sdc pve-lab2-crypt
mkfs.ext4 -L PVE-Lab_2 /dev/mapper/pve-lab2-crypt
```

### 5.3. Cek, auto-unlock LUKS, dan mount storage

```bash
lsblk -f
```

Pemetaan disk pada konfigurasi ini:
- `/dev/sdb` = `PVE-Lab_1` = tanpa LUKS.
- `/dev/sdc` = `PVE-Lab_2` = memakai LUKS.

Buat direktori mount:

```bash
mkdir -p /mnt/pve/PVE-Lab_1
mkdir -p /mnt/pve/PVE-Lab_2
```

Ambil UUID disk:

```bash
blkid /dev/sdb /dev/sdc
```

Buat keyfile agar LUKS terbuka otomatis saat boot:

```bash
dd if=/dev/urandom of=/root/luks-pve-lab2.key bs=512 count=8
chmod 400 /root/luks-pve-lab2.key
cryptsetup luksAddKey /dev/sdc /root/luks-pve-lab2.key
```

Edit `/etc/crypttab`:

```bash
nano /etc/crypttab
```

Tambahkan satu baris berikut. Ganti `UUID_LUKS_DEV_SDC` dengan UUID `/dev/sdc` dari hasil `blkid`:

```text
pve-lab2-crypt UUID=UUID_LUKS_DEV_SDC /root/luks-pve-lab2.key luks
```

Edit `/etc/fstab`:

```bash
nano /etc/fstab
```

Tambahkan dua baris di bagian paling bawah. Ganti `UUID_DEV_SDB` dengan UUID `/dev/sdb` dari hasil `blkid`:

```text
UUID=UUID_DEV_SDB /mnt/pve/PVE-Lab_1 ext4 defaults,nofail 0 2
/dev/mapper/pve-lab2-crypt /mnt/pve/PVE-Lab_2 ext4 defaults,nofail 0 2
```

Muat ulang konfigurasi dan mount:

```bash
systemctl daemon-reload
mount -a
```

Cek mount point:

```bash
findmnt --mountpoint /mnt/pve/PVE-Lab_1
findmnt --mountpoint /mnt/pve/PVE-Lab_2
```

### 5.4. Daftarkan storage di dashboard

1. Datacenter → Storage → Add → Directory.
2. Tambahkan storage pertama:
   - ID: `PVE-Lab_1`
   - Directory: `/mnt/pve/PVE-Lab_1`
3. Tambahkan storage kedua:
   - ID: `PVE-Lab_2`
   - Directory: `/mnt/pve/PVE-Lab_2`
4. Aktifkan pengaman agar Proxmox menolak membuat disk di folder kosong jika LUKS belum terbuka:

```bash
pvesm set PVE-Lab_2 --is_mountpoint yes
```

5. Cek status dari shell:

```bash
pvesm status
```

## 6. Atur akses Proxmox melalui Host-only Adapter

Pada praktik sebelumnya, akses dashboard/web shell melalui port forwarding NAT sempat mengalami putus-sambung. Untuk akses dashboard yang stabil, gunakan Adapter 2 Host-only. Adapter 1 NAT tetap dipertahankan untuk koneksi internet Proxmox.

1. Dari shell Proxmox, matikan VM Proxmox dengan aman:

```bash
poweroff
```

2. VirtualBox → pilih VM `PVE-Lab` → Settings → Network.
3. Adapter 1 tetap NAT.
4. Adapter 2: centang Enable Network Adapter, lalu pilih **Host-only Adapter**.
5. Di PowerShell Windows, cek alamat adapter:

```powershell
ipconfig
```

6. Pada konfigurasi laptop ini, `Ethernet adapter Ethernet 3` memiliki IP `192.168.56.1`.
7. Nyalakan VM Proxmox.
8. Login melalui console. `Bukan lewat browser ya, tapi langsung di VM Proxmoxnya yang di buka lewat VirtualBox`. Sama pakai `root` passwordnya `Proxmox1!`.
9. Edit konfigurasi jaringan:

```bash
nano /etc/network/interfaces
```

10. Tambahkan bagian berikut sebelum baris `source /etc/network/interfaces.d/*`:

```text
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10/24
```

11. Simpan, lalu terapkan:

```bash
ifreload -a
```

12. Mulai sekarang, akses dashboard Proxmox melalui `https://192.168.56.10:8006`.

## 7. Buat VM Ubuntu di dalam Proxmox

### 7.1. Upload ISO

1. Nyalakan VM Proxmox di VirtualBox.
2. Akses dashboard Proxmox melalui browser `https://192.168.56.10:8006`.
3. Klik ikon `▽` di sebelah kiri node `pve` → local (pve) → ISO Images → Upload.
4. Pilih `ubuntu-26.04.1-live-server-amd64.iso` yang sudah diunduh. Masukkan SHA256nya `cc8a95cde20f6ced61a322420de00f10cc3c90ced545daa46cb9c1a117f1d927`

![gambar12](https://lh3.googleusercontent.com/d/1F8Etbbw8Q1bTyaviZJHG-FilzIy_Ph11)

### 7.2. Create VM

> [!IMPORTANT]
> Pastikan LUKS sudah ter-mount sebelum membuat VM Ubuntu.

Jalankan command ini di shell Proxmox.

```bash
findmnt --mountpoint /mnt/pve/PVE-Lab_2
pvesm status
```

Jika keluar output di bawah ini, maka sudah aman untuk membuat VM Ubuntu.

```text
root@pve:~# findmnt --mountpoint /mnt/pve/PVE-Lab_2
TARGET             SOURCE                     FSTYPE OPTIONS
/mnt/pve/PVE-Lab_2 /dev/mapper/pve-lab2-crypt ext4   rw,relatime
root@pve:~# pvesm status
Name             Type     Status     Total (KiB)      Used (KiB) Available (KiB)        %
PVE-Lab_1         dir     active         8154588            2076         7716700    0.03%
PVE-Lab_2         dir     active         8138532            2072         7701468    0.03%
local             dir     active        14145416         8515136         4889928   60.20%
local-lvm     lvmthin     active        12378112               0        12378112    0.00%
```
Jika mountpoint PVE-Lab_2 sudah muncul, dan Used PVE-Lab_2 kurang lebih sama seperti PVE-Lab_1, maka aman untuk membuat VM Ubuntu. Itu artinya LUKS auto-unlock sudah berjalan dengan benar, karena sebelumnya ada error dimana VM ternyata dibuat diluar LUKS dikarenakan LUKS masih terkunci.

Klik **Create VM** di pojok kanan atas dan isi:

**General**
- Node: `pve`
- VM ID: `100`
- Name: `Ubuntu-IO-Test`

**OS**
- ISO image: `ubuntu-26.04.1-live-server-amd64.iso`
- Version: `7.x - 2.6 Kernel`

**System**
- Tidak perlu centang Qemu Agent.
- Pengaturan lain biarkan default.

**Disks**
1. `scsi0`: 
   - Storage `local-lvm`
   - size **8 GB**
2. Add `scsi1`
   - Storage `PVE-Lab_1`
   - size **4 GB**
   - Format **Raw**.
3. Add `scsi2`
   - Storage `PVE-Lab_2`
   - size **4 GB**
   - Format **Raw**.

**CPU**
- Cores: **3**
- Type: `host`

**Memory**
- Memory: **3072 MB**

### 7.3. Nyalakan VM Ubuntu

1. Klik ikon `▽` di sebelah kiri node `pve` → VM `100 (Ubuntu-IO-Test)` → Start.
2. Buka Console untuk melihat proses instalasi. Console ada di bawah tab summary.

![gambar13](https://lh3.googleusercontent.com/d/1BZEGzDz76ruogGswmE7c_STlxdJ-JqC2)

## 8. Install Ubuntu

Pada installer, pilih bahasa `English` dan `Ubuntu Server`.

### 8.1. Pengaturan jaringan Ubuntu

1. Pilih interface `ens18`, lalu tekan Enter.
2. Buka Edit IPv4.
3. Ubah metode ke Manual dan isi:
   - Subnet: `10.0.2.0/24`
   - Address: `10.0.2.16`
   - Gateway: `10.0.2.2`
   - Name servers: `10.0.2.3`
   - Search domains: kosong
   - Save
   - Tunggu "applying changes" selesai baru tekan Done
4. Proxy dikosongkan, lalu Done.
5. Biarkan mirror address default dan tunggu sampai tulisan ini muncul `mirror location passed test`.
6. Pengaturan lainnya: Next → Next → Continue.

### 8.2. Profil Ubuntu

- Your name: `Ubuntu Lab`
- Your server's name: `ubuntu-io-test`
- Pick a username: `labuser`
- Password: `Proxmox1!`
- Skip Ubuntu Pro.
- Centang `Install OpenSSH server` tanpa SSH key.
- Featured server snaps jangan centang. Langsung saja next.
- Tunggu instalasi selesai.

Setelah instalasi, jika kelihatan stuck ? `tekan enter sekali aja`, lalu login dengan user `labuser` dan password `Proxmox1!`.

## 9. Siapkan filesystem data di Ubuntu

Jalankan perintah berikut di terminal VM Ubuntu. Tahap `mkfs` hanya dilakukan saat menyiapkan disk pertama kali; jangan diulang pada disk yang sudah berisi data.

```bash
sudo mkfs.ext4 -F -L BIASA /dev/sdb
sudo mkfs.ext4 -F -L LUKS /dev/sdc
sudo mkdir -p /mnt/BIASA /mnt/LUKS
sudo blkid /dev/sdb /dev/sdc
```

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Tambahkan dua baris berikut di bagian paling bawah. Ganti UUID sesuai hasil `blkid`:

```text
UUID=UUID_DEV_SDB /mnt/BIASA ext4 defaults,nofail 0 2
UUID=UUID_DEV_SDC /mnt/LUKS ext4 defaults,nofail 0 2
```

Muat ulang, mount, dan berikan akses ke `labuser`:

```bash
sudo systemctl daemon-reload
sudo mount -a
sudo chown labuser:labuser /mnt/BIASA /mnt/LUKS
```

## 10. Install FIO dan sysstat di Ubuntu

Jalankan di VM Ubuntu:

```bash
sudo ln -sfn /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo mv /etc/apt/sources.list.d/cdrom.sources /etc/apt/sources.list.d/cdrom.sources.disabled
sudo apt update
sudo apt install -y fio sysstat
```

## 11. Siapkan Proxmox untuk benchmark

Jalankan di shell Proxmox:

```bash
apt update
apt install -y tmux sysstat htop
```

### 11.1. Tes SSH dari Proxmox ke Ubuntu

Di shell Proxmox:

```bash
ssh labuser@10.0.2.16 'hostname'
```

Saat pertama kali terhubung, jawab `yes` untuk menerima host key.

Buat SSH key di Proxmox:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_pve_lab -C "pve-to-ubuntu-io-test"
```

Kosongkan passphrase saat diminta. Lalu jalankan command ini untuk menyalin public key ke Ubuntu:

```bash
ssh-copy-id -i /root/.ssh/id_ed25519_pve_lab.pub labuser@10.0.2.16
```

Tes koneksi tanpa password:

```bash
ssh -i /root/.ssh/id_ed25519_pve_lab -o BatchMode=yes labuser@10.0.2.16 'hostname && whoami && fio --version'
```

## 12. Jalankan benchmark

Benchmark dijalankan dari shell Proxmox. FIO berjalan di Ubuntu melalui SSH, sedangkan `mpstat` merekam CPU Proxmox selama FIO berlangsung.

### 12.1. Parameter FIO

| Workload | `rw` | Block size | I/O depth |
|---|---|---|---|
| Sequential read | `read` | 1 MiB | 16 |
| Sequential write | `write` | 1 MiB | 16 |
| Random read | `randread` | 4 KiB | 32 |
| Random write | `randwrite` | 4 KiB | 32 |

Pengaturan umum: file 512 MiB, `direct=1`, `ioengine=libaio`, runtime 40 detik, ramp time 0, 10 pengulangan. Urutan A/B bergantian. Seed random berubah untuk setiap pasangan pengulangan, tetapi seed A dan B pada pasangan yang sama dibuat identik. Total waktu FIO aktif adalah 53 menit 20 detik; jeda, persiapan, dan proses analisis membuat waktu keseluruhan lebih panjang.

### 12.2. Buat script benchmark

Di shell Proxmox:

```bash
nano /root/run_fio_experiment.sh
```

Masukkan script berikut:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# =========================
# KONFIGURASI
# =========================
REMOTE="labuser@10.0.2.16"
SSH_KEY="/root/.ssh/id_ed25519_pve_lab"

OUT="/root/experiment-results"

SIZE="512M"
RUNTIME=40
RAMP_TIME=0
REPEATS=10
CASE_PAUSE=3
GROUP_PAUSE=30
MIN_FREE_KIB=1300000

FILE_A="/mnt/BIASA/fio-benchmark.bin"
FILE_B="/mnt/LUKS/fio-benchmark.bin"

CPU_PID=""
EXPECTED_RUNS=80

# =========================
# INISIALISASI
# =========================
if [[ "$EUID" -ne 0 ]]; then
    echo "Jalankan script sebagai root."
    exit 1
fi

if [[ -z "${TMUX:-}" ]]; then
    echo "Jalankan script di dalam tmux."
    exit 1
fi

for cmd in mpstat stdbuf python3 ssh flock tee; do
    command -v "$cmd" >/dev/null || {
        echo "Command belum tersedia: $cmd"
        exit 1
    }
done

if [[ ! -r "$SSH_KEY" ]]; then
    echo "SSH key tidak ditemukan: $SSH_KEY"
    exit 1
fi

if [[ -e "$OUT" ]]; then
    echo "Folder hasil sudah ada: $OUT"
    echo "Pindahkan atau hapus folder tersebut sebelum eksperimen baru."
    exit 1
fi

mkdir -p "$OUT/cpu" "$OUT/fio"
exec > >(tee -a "$OUT/run.log") 2>&1

cleanup() {
    if [[ -n "$CPU_PID" ]] && kill -0 "$CPU_PID" 2>/dev/null; then
        kill -TERM "$CPU_PID" 2>/dev/null || true
        wait "$CPU_PID" 2>/dev/null || true
    fi
}

trap cleanup EXIT

# Cegah dua benchmark berjalan bersamaan.
exec 9>/run/lock/fio-luks-experiment.lock
if ! flock -n 9; then
    echo "Benchmark lain sedang berjalan."
    exit 1
fi

ssh_remote() {
    ssh \
        -i "$SSH_KEY" \
        -o BatchMode=yes \
        -o ConnectTimeout=10 \
        -o ServerAliveInterval=15 \
        -o ServerAliveCountMax=3 \
        "$REMOTE" "$@"
}

echo "=========================================="
echo "LUKS STORAGE PERFORMANCE EXPERIMENT"
echo "Folder hasil: $OUT"
echo "Total pengujian: $EXPECTED_RUNS runs"
echo "=========================================="

# =========================
# PREFLIGHT
# =========================
echo
echo "[1] Memeriksa Ubuntu, FIO, storage, dan ruang kosong..."

ssh_remote "MIN_FREE_KIB=$MIN_FREE_KIB bash -s" <<'REMOTE'
set -e

hostname
fio --version

for d in /mnt/BIASA /mnt/LUKS; do
    mountpoint -q "$d"
    test -w "$d"

    available_kib=$(df -Pk "$d" | awk 'NR == 2 {print $4}')

    if [ "$available_kib" -lt "$MIN_FREE_KIB" ]; then
        echo "Ruang kosong tidak mencukupi: $d"
        exit 1
    fi
done

df -h /mnt/BIASA /mnt/LUKS
REMOTE

# =========================
# PERSIAPAN FILE FIO
# =========================
prepare_file() {
    local name="$1"
    local path="$2"

    echo
    echo "Menyiapkan $name..."

    ssh_remote "fio \
        --name=prepare \
        --filename='$path' \
        --size='$SIZE' \
        --rw=write \
        --bs=1M \
        --ioengine=libaio \
        --iodepth=1 \
        --numjobs=1 \
        --direct=1 \
        --overwrite=1 \
        --end_fsync=1 \
        --group_reporting \
        --eta=never" > /dev/null

    echo "$name selesai."
}

echo
echo "[2] Menyiapkan file kerja..."

prepare_file "Storage A" "$FILE_A"
prepare_file "Storage B" "$FILE_B"

# =========================
# FUNGSI SATU PENGUJIAN
# =========================
run_case() {
    local condition="$1"
    local workload="$2"
    local rw="$3"
    local bs="$4"
    local depth="$5"
    local rep="$6"
    local seed="$7"
    local file="$8"

    local name="${condition}_${workload}_r${rep}"
    local cpu_log="$OUT/cpu/${name}_CPU.txt"
    local fio_log="$OUT/fio/${name}.json"
    local fio_command
    local status=0

    echo
    echo "=========================================="
    echo "RUN: $name | Seed: $seed"
    echo "RW: $rw | BS: $bs | QD: $depth"
    echo "=========================================="

    # Rekam CPU Proxmox.
    stdbuf -oL mpstat -P ALL 1 > "$cpu_log" 2>&1 &
    CPU_PID=$!

    sleep 2

    if ! kill -0 "$CPU_PID" 2>/dev/null; then
        echo "mpstat gagal berjalan."
        CPU_PID=""
        return 1
    fi

    # Konfigurasi FIO.
    fio_command="fio \
        --name='$workload' \
        --filename='$file' \
        --size='$SIZE' \
        --rw='$rw' \
        --bs='$bs' \
        --ioengine=libaio \
        --iodepth='$depth' \
        --numjobs=1 \
        --direct=1 \
        --overwrite=1 \
        --time_based \
        --runtime='$RUNTIME' \
        --ramp_time='$RAMP_TIME' \
        --group_reporting \
        --eta=never \
        --randseed='$seed' \
        --randrepeat=1 \
        --output-format=json"

    echo "Menjalankan FIO: $name"

    if ssh_remote "$fio_command" > "$fio_log"; then
        echo "FIO selesai: $name"
    else
        status=$?
        echo "FIO gagal: $name (exit status $status)"
    fi

    # Hentikan pencatatan CPU.
    kill -TERM "$CPU_PID" 2>/dev/null || true
    wait "$CPU_PID" 2>/dev/null || true
    CPU_PID=""

    if [[ "$status" -ne 0 ]]; then
        return "$status"
    fi

    # Validasi JSON.
    python3 -m json.tool "$fio_log" > /dev/null

    echo "Selesai: $name"
    sleep "$CASE_PAUSE"
}

# =========================
# MATRIKS PENGUJIAN
# =========================
declare -A CONFIG

CONFIG[seqread]="read 1M 16"
CONFIG[seqwrite]="write 1M 16"
CONFIG[randread]="randread 4k 32"
CONFIG[randwrite]="randwrite 4k 32"

WORKLOADS=(seqread seqwrite randread randwrite)

run_count=0
group_number=0

echo
echo "[3] Memulai benchmark..."
echo "Total run: $EXPECTED_RUNS"

for workload in "${WORKLOADS[@]}"; do

    read -r rw bs depth <<< "${CONFIG[$workload]}"

    echo
    echo "######## WORKLOAD: $workload ########"

    for ((rep=1; rep<=REPEATS; rep++)); do

        rep_label="$(printf '%02d' "$rep")"
        seed="$rep"

        # Urutan A/B bergantian.
        if (( rep % 2 == 1 )); then
            CONDITIONS=(A B)
        else
            CONDITIONS=(B A)
        fi

        for condition in "${CONDITIONS[@]}"; do

            if [[ "$condition" == "A" ]]; then
                file="$FILE_A"
            else
                file="$FILE_B"
            fi

            run_case \
                "$condition" \
                "$workload" \
                "$rw" \
                "$bs" \
                "$depth" \
                "$rep_label" \
                "$seed" \
                "$file"

            run_count=$((run_count + 1))

        done
    done

    group_number=$((group_number + 1))

    if (( group_number < ${#WORKLOADS[@]} )); then
        echo "Jeda ${GROUP_PAUSE} detik..."
        sleep "$GROUP_PAUSE"
    fi

done

if (( run_count != EXPECTED_RUNS )); then
    echo "Jumlah run tidak sesuai: $run_count / $EXPECTED_RUNS"
    exit 1
fi

echo
echo "=========================================="
echo "BENCHMARK SELESAI"
echo "=========================================="
echo "Total run: $run_count"
echo "Folder hasil: $OUT"
```

Simpan file, lalu beri izin eksekusi:

```bash
chmod +x /root/run_fio_experiment.sh
```

### 12.3. Jalankan benchmark

Di shell Proxmox:

```bash
tmux new -s fio-main
```

Di dalam session tmux:

```bash
/root/run_fio_experiment.sh
```

Untuk keluar dari tampilan tmux tanpa menghentikan benchmark: tekan `Ctrl+B`, lalu `D`. Untuk masuk kembali:

```bash
tmux attach -t fio-main
```

### 12.4. Memantau penggunaan CPU, RAM, proses yang sedang berjalan, dan beban sistem secara real time

Di shell Proxmox saat script benchmark masih berjalan di dalam session tmux:

```bash
htop
```

Untuk keluar dari `htop` tekan `F10`.

## 13. Analisis keselarasan log CPU dengan FIO

Script ini memakai `job_start` dan `job_runtime` dari JSON FIO untuk memilih interval `mpstat` yang berada di dalam waktu pengujian FIO.

### 13.1. Buat script

Di shell Proxmox:

```bash
nano /root/analyze_aligned.py
```

Masukkan:

```python
import csv
import json
import re
import statistics
from pathlib import Path
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

ROOT = Path("/root/experiment-results")
if not ROOT.is_dir():
    raise SystemExit(
        f"Folder hasil tidak ditemukan: {ROOT}. Jalankan benchmark terlebih dahulu."
    )

print("Folder hasil:", ROOT)

FIO_DIR = ROOT / "fio"
CPU_DIR = ROOT / "cpu"

OUT_COMBINED = ROOT / "combined_results.csv"

EXPECTED_RUNS = 80
TZ = ZoneInfo("Asia/Jakarta")

PATTERN = re.compile(
    r"([AB])_(seqread|seqwrite|randread|randwrite)_r(\d+)"
)


def parse_mpstat_time(parts, reference):
    """Ambil waktu mpstat dan pilih tanggal terdekat dari waktu FIO."""
    clock_text = parts[0]

    if len(parts) > 1 and parts[1].upper() in ("AM", "PM"):
        clock = datetime.strptime(
            clock_text + " " + parts[1], "%I:%M:%S %p"
        ).time()
    else:
        clock = datetime.strptime(clock_text, "%H:%M:%S").time()

    candidates = []
    for offset in (-1, 0, 1):
        day = reference.date() + timedelta(days=offset)
        candidates.append(datetime.combine(day, clock, tzinfo=TZ))

    return min(
        candidates,
        key=lambda x: abs((x - reference).total_seconds())
    )


def parse_cpu_log(path, fio_start, fio_end):
    """Ambil sampel mpstat yang berada di dalam rentang waktu FIO."""
    samples = []

    margin = timedelta(milliseconds=200)
    safe_start = fio_start + margin
    safe_end = fio_end - margin

    for line in path.read_text(errors="replace").splitlines():
        parts = line.split()

        if "all" not in parts or parts[0].startswith("Average"):
            continue

        idx = parts.index("all")
        if idx < 1:
            continue

        try:
            sample_end = parse_mpstat_time(parts, fio_start)
            sample_start = sample_end - timedelta(seconds=1)

            values = [
                float(x) for x in parts[idx + 1:idx + 11]
            ]

        except (ValueError, IndexError):
            continue

        if len(values) != 10:
            continue

        # Urutan nilai mpstat:
        # usr, nice, sys, iowait, irq, soft, steal, guest, gnice, idle
        iowait = values[3]
        idle = values[9]

        if sample_start >= safe_start and sample_end <= safe_end:
            samples.append((iowait, idle))

    return samples


# --------------------------------------------------
# BACA FIO DAN SELARASKAN DENGAN LOG CPU
# --------------------------------------------------

per_run = []
fio_files = sorted(FIO_DIR.glob("[AB]_*.json"))

for fio_path in fio_files:
    match = PATTERN.fullmatch(fio_path.stem)
    if not match:
        continue

    condition, workload, repeat = match.groups()

    cpu_path = CPU_DIR / f"{fio_path.stem}_CPU.txt"

    if not cpu_path.exists():
        print("Log CPU hilang:", cpu_path.name)
        continue

    data = json.loads(fio_path.read_text())

    if not data.get("jobs"):
        print("Data job FIO kosong:", fio_path.name)
        continue

    job = data["jobs"][0]

    io_type = "read" if workload in ("seqread", "randread") else "write"
    metrics = job[io_type]
    clat = metrics["clat_ns"]

    if "mean" not in clat:
        print("Mean latency tidak tersedia:", fio_path.name)
        continue

    fio_start = datetime.fromtimestamp(
        int(job["job_start"]) / 1000,
        tz=TZ
    )

    runtime_ms = int(job["job_runtime"])
    fio_end = fio_start + timedelta(milliseconds=runtime_ms)

    samples = parse_cpu_log(cpu_path, fio_start, fio_end)

    if not samples:
        print("Tidak ada sampel CPU yang selaras:", fio_path.name)
        continue

    # Rata-rata komponen idle dan iowait untuk menghitung CPU usage.
    avg_iowait = statistics.mean(sample[0] for sample in samples)
    avg_idle = statistics.mean(sample[1] for sample in samples)

    cpu_usage = round(
        100 - avg_idle - avg_iowait,
        4
    )

    result = {
        "condition": condition,
        "workload": workload,
        "repeat": repeat,
        "seed": int(repeat),
        "throughput_MiB_s": round(metrics["bw_bytes"] / (1024 ** 2), 3),
        "IOPS": round(metrics["iops"], 3),
        "mean_latency_ms": round(clat["mean"] / 1_000_000, 6),
        "cpu_usage_pct": cpu_usage,
        "cpu_samples": len(samples),
        "fio_runtime_ms": runtime_ms,
        "fio_error": job["error"],
    }

    per_run.append(result)


if not per_run:
    raise SystemExit(
        "Tidak ada data CPU yang berhasil disejajarkan."
    )


# --------------------------------------------------
# SIMPAN HASIL GABUNGAN DAN VALIDASI
# --------------------------------------------------

with OUT_COMBINED.open("w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=list(per_run[0].keys()))
    writer.writeheader()
    writer.writerows(per_run)

failed = [r for r in per_run if int(r["fio_error"]) != 0]

print("\nANALISIS SELESAI")
print("Records:", len(per_run), "/", EXPECTED_RUNS)
print(OUT_COMBINED)

if len(per_run) == EXPECTED_RUNS and not failed:
    print("\nVALIDATION: SUCCESS — 80 run lengkap, tanpa error FIO.")
else:
    print(
        f"\nVALIDATION: WARNING — records={len(per_run)}, "
        f"error FIO={len(failed)}"
    )
```

### 13.2. Jalankan analisis CPU

Jalankan perintah berikut. Script langsung membaca folder hasil tetap:

```bash
python3 /root/analyze_aligned.py
```

File yang dihasilkan:
- `combined_results.csv`

## 14. Analisis statistik hasil gabungan

### 14.1. Buat script statistik

Di shell Proxmox:

```bash
nano /root/analyze_final.py
```

Masukkan:

```python
import csv
import statistics
from pathlib import Path

ROOT = Path("/root/experiment-results")

if not ROOT.is_dir():
    raise SystemExit(
        f"Folder hasil tidak ditemukan: {ROOT}. Jalankan benchmark terlebih dahulu."
    )

print("Folder hasil:", ROOT)

INPUT = ROOT / "combined_results.csv"

EXPECTED_RUNS = 80
REPEATS = 10

METRICS = [
    "throughput_MiB_s",
    "IOPS",
    "mean_latency_ms",
    "cpu_usage_pct",
]

WORKLOADS = [
    "seqread",
    "seqwrite",
    "randread",
    "randwrite",
]

CONDITIONS = ["A", "B"]

# Format repeat dari script benchmark: 01 sampai 10.
REPEAT_LABELS = [f"{i:02d}" for i in range(1, REPEATS + 1)]


# --------------------------------------------------
# LOAD DAN VALIDASI DATA
# --------------------------------------------------

if not INPUT.is_file():
    raise SystemExit(f"File input tidak ditemukan: {INPUT}")

with INPUT.open(newline="") as f:
    reader = csv.DictReader(f)

    required_columns = {
        "condition",
        "workload",
        "repeat",
        "fio_error",
        *METRICS,
    }

    if not reader.fieldnames or not required_columns.issubset(reader.fieldnames):
        missing = required_columns - set(reader.fieldnames or [])
        raise SystemExit(f"Kolom CSV tidak lengkap: {sorted(missing)}")

    rows = list(reader)

if len(rows) != EXPECTED_RUNS:
    raise SystemExit(
        f"Diharapkan {EXPECTED_RUNS} records, ditemukan {len(rows)}"
    )

data = {}

for row in rows:
    key = (
        row["condition"],
        row["workload"],
        row["repeat"],
    )

    if key in data:
        raise SystemExit(f"Record duplikat: {key}")

    try:
        fio_error = int(row["fio_error"])
    except (ValueError, TypeError):
        raise SystemExit(f"Nilai fio_error tidak valid pada: {key}")

    if fio_error != 0:
        raise SystemExit(f"FIO error pada: {key}")

    # Pastikan seluruh metrik berisi angka yang valid.
    for metric in METRICS:
        try:
            float(row[metric])
        except (ValueError, TypeError):
            raise SystemExit(
                f"Nilai {metric} tidak valid pada: {key}"
            )

    data[key] = row


# Pastikan seluruh kombinasi workload, kondisi, dan repeat tersedia.
for workload in WORKLOADS:
    for condition in CONDITIONS:
        for repeat in REPEAT_LABELS:
            key = (condition, workload, repeat)

            if key not in data:
                raise SystemExit(f"Record hilang: {key}")

print("Validasi dataset: PASSED")
print("Total records:", len(data))


# --------------------------------------------------
# RINGKASAN A/B
# --------------------------------------------------

summary = []

for workload in WORKLOADS:
    for metric in METRICS:
        a_values = [
            float(data[("A", workload, r)][metric])
            for r in REPEAT_LABELS
        ]

        b_values = [
            float(data[("B", workload, r)][metric])
            for r in REPEAT_LABELS
        ]

        mean_a = statistics.mean(a_values)
        mean_b = statistics.mean(b_values)
        diff = mean_b - mean_a
        change_pct = diff / mean_a * 100 if mean_a != 0 else None

        summary.append({
            "workload": workload,
            "metric": metric,
            "n": len(a_values),
            "A_mean": round(mean_a, 4),
            "A_sd": round(statistics.stdev(a_values), 4),
            "B_mean": round(mean_b, 4),
            "B_sd": round(statistics.stdev(b_values), 4),
            "B_minus_A": round(diff, 4),
            "change_pct": round(change_pct, 2) if change_pct is not None else None,
        })


# --------------------------------------------------
# SIMPAN DAN TAMPILKAN HASIL
# --------------------------------------------------

SUMMARY_PATH = ROOT / "summary.csv"

with SUMMARY_PATH.open("w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=list(summary[0].keys()))
    writer.writeheader()
    writer.writerows(summary)

print("\nRINGKASAN A/B")
print("=" * 85)

for row in summary:
    change = (
        f'{row["change_pct"]:+.2f}%'
        if row["change_pct"] is not None
        else "N/A"
    )

    print(
        f'{row["workload"]:12} {row["metric"]:18} '
        f'A={row["A_mean"]:10.4f}±{row["A_sd"]:.4f} '
        f'B={row["B_mean"]:10.4f}±{row["B_sd"]:.4f} '
        f'Delta={row["B_minus_A"]:+.4f} Change={change}'
    )

print("\nFile hasil:")
print(SUMMARY_PATH)
print("\nANALISIS SELESAI")
```

### 14.2. Jalankan script statistik

Jalankan perintah berikut. Script langsung membaca folder hasil yang sama:

```bash
python3 /root/analyze_final.py
```

File hasil:
- summary.csv — berisi mean, standar deviasi, selisih rata-rata B terhadap A, dan persentase perubahan.

## 15. File hasil yang perlu disimpan

Semua hasil berada di folder berikut:

```bash
/root/experiment-results/
├── run.log
├── fio/
│   └── 80 file JSON
├── cpu/
│   └── 80 file mpstat
├── combined_results.csv
└── summary.csv
```

## 16. Aktifkan kembali Memory Integrity dan virtualisasi Windows yang dimatikan pada langkah 2.2.

**Mengatur hypervisor Windows otomatis menyala saat boot**

1. Klik kanan Start → Terminal (Admin).
2. Jalankan:

```powershell
bcdedit /set hypervisorlaunchtype auto
```

3. Jangan restart dulu.

**Aktifkan kembali fitur virtualisasi Windows**

1. Tekan `Windows + R`.
2. Jalankan `optionalfeatures.exe`.
3. Centang `Virtual Machine Platform` dan `Windows Subsystem for Linux`.
4. Jangan restart dulu.

**Aktifkan Memory Integrity**

1. Start → Windows Security.
2. Device security → Core isolation details.
3. Nyalakan `Memory integrity`.
4. Restart laptop.

**Cek apakah VBS aktif**

1. Jalankan di powershell:

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
```

2. Harapannya setelah semua langkah pada tahap 16 ini dilakukan:

| Baris | Sebelum dilakukan tahap 16 | Setelah dilakukan tahap 16 |
|---|---|---|
| VirtualizationBasedSecurityStatus | 0 | 2 |
| SecurityServicesConfigured | {0} | {2} |
| SecurityServicesRunning | {0} | {2} |

3. Juga cek WSL Ubuntumu apakah bisa menjalankan perintah
