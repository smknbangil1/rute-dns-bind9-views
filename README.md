# forwading dns menggunakan bind9 + views
# **Panduan Lengkap: Konfigurasi BIND9 sebagai Forwarder dengan Kebijakan Perutean di Ubuntu 24.04**  

**Oleh: PURWANTO**  
**Tanggal: 2025-06-23**  

BIND9 adalah salah satu server DNS paling populer di dunia. Dalam panduan ini, kita akan mengonfigurasi BIND9 di **Ubuntu Server 24.04** untuk berfungsi sebagai **DNS Forwarder** dengan kebijakan perutean khusus:  
- **Default Forwarding**: Meneruskan semua permintaan ke DNS utama (`172.16.200.2`).  
- **Pengecualian Forwarding**: Meneruskan permintaan dari IP tertentu ke DNS alternatif (`172.16.200.18`).  

Konfigurasi ini cocok untuk:  
✔️ Jaringan perusahaan dengan kebijakan DNS berbeda untuk beberapa klien.  
✔️ Membagi lalu lintas DNS berdasarkan kebutuhan keamanan.  
✔️ Mengoptimalkan resolusi DNS dengan beberapa upstream server.  

---

## **Langkah 1: Instal BIND9**  
Pastikan server Ubuntu 24.04 terupdate, lalu instal BIND9:  

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install bind9 bind9utils bind9-doc -y
```

---

## **Langkah 2: Konfigurasi Dasar (`named.conf.options`)**  
Edit file konfigurasi utama:  

```bash
sudo nano /etc/bind/named.conf.options
```

**Salin konfigurasi berikut:**  
```bind9
options {
    directory "/var/cache/bind";

    // Nonaktifkan zona default (karena hanya sebagai forwarder)
    empty-zones-enable no;
    recursion yes;

    // Izinkan rekursi hanya untuk klien lokal/RFC1918
    allow-recursion { 
        10.0.0.0/8;
        172.16.0.0/12; 
        192.168.0.0/16; 
    };

    // Blok transfer zona (tidak diperlukan untuk forwarder)
    allow-transfer { none; };

    // Aktifkan logging untuk debugging
    logging {
        channel query_log {
            file "/var/log/bind/query.log";
            severity debug 3;
            print-time yes;
        };
        category queries { query_log; };
    };

    // Dengarkan hanya pada IP server DNS (opsional)
    listen-on { 172.16.200.34; };

    // Port default DNS
    port 53;
};
```

**Simpan (`Ctrl+O`) dan keluar (`Ctrl+X`).**  

---

## **Langkah 3: Konfigurasi Kebijakan Perutean (`named.conf.local`)**  
Kita akan menggunakan **`views`** untuk membedakan lalu lintas DNS.  

```bash
sudo nano /etc/bind/named.conf.local
```

**Salin konfigurasi berikut:**  
```bind9
// Daftar IP yang akan diteruskan ke DNS 3 (172.16.200.18)
acl "excluded_clients" {
    192.168.1.50;    // Contoh IP 1
    172.16.0.3;      // Contoh IP 2
    10.0.0.14;       // Contoh IP 3
    // Tambahkan IP lain jika diperlukan
};

// Daftar semua klien yang diizinkan (RFC1918)
acl "rfc1918_clients" {
    10.0.0.0/8;
    172.16.0.0/12;
    192.168.0.0/16;
};

// ===== VIEW UNTUK PENGEUALIAN (DNS 3) =====
view "exception_view" {
    // Hanya proses permintaan dari "excluded_clients"
    match-clients { "excluded_clients"; };
    recursion yes;

    // Forward ke DNS 3
    forwarders { 172.16.200.18; };
    forward only;  // Hanya gunakan forwarder, jangan coba resolve sendiri
};

// ===== VIEW DEFAULT (DNS 2) =====
view "default_forwarding_view" {
    // Proses semua klien RFC1918 KECUALI yang sudah di-handle "exception_view"
    match-clients { "rfc1918_clients"; !"excluded_clients"; };
    recursion yes;

    // Forward ke DNS 2
    forwarders { 172.16.200.2; };
    forward only;
};
```

**Simpan dan keluar.**  

---

## **Langkah 4: Pastikan Tidak Ada Zona yang Dimuat**  
BIND9 dalam mode forwarder **tidak boleh mengelola zona apa pun**. Pastikan:  

1. **Hapus zona default** (jika ada):  
   ```bash
   sudo nano /etc/bind/named.conf.default-zones
   ```
   - Komentari semua zona dengan `//`.  

2. **Periksa `named.conf`**:  
   ```bash
   sudo nano /etc/bind/named.conf
   ```
   - Pastikan hanya ada:  
     ```bind9
     include "/etc/bind/named.conf.options";
     include "/etc/bind/named.conf.local";
     ```

---

## **Langkah 5: Buat Direktori Log & Set Izin**  
Aktifkan logging untuk memantau permintaan DNS:  

```bash
sudo mkdir -p /var/log/bind/
sudo chown bind:bind /var/log/bind/
```

---

## **Langkah 6: Validasi & Restart BIND9**  

1. **Periksa sintaks konfigurasi**:  
   ```bash
   sudo named-checkconf
   ```
   - Jika tidak ada output, berarti konfigurasi valid.  

2. **Restart BIND9**:  
   ```bash
   sudo systemctl restart bind9
   ```

3. **Periksa status**:  
   ```bash
   sudo systemctl status bind9
   ```
   - Pastikan status **active (running)**.  

---

## **Langkah 7: Pengujian**  

### **1. Dari Klien yang Dikecualikan (misal: 192.168.1.50)**  
- Setel DNS klien ke `172.16.200.34`.  
- Jalankan:  
  ```bash
  dig @172.16.200.34 google.com
  ```
  - **Harus di-forward ke `172.16.200.18` (DNS 3)**.  

### **2. Dari Klien Biasa (misal: 192.168.1.100)**  
- Setel DNS klien ke `172.16.200.34`.  
- Jalankan:  
  ```bash
  dig @172.16.200.34 example.com
  ```
  - **Harus di-forward ke `172.16.200.2` (DNS 2)**.  

### **3. Pantau Log**  
```bash
sudo tail -f /var/log/bind/query.log
```
- Contoh output:  
  ```
  client 192.168.1.50#1234: query: google.com IN A + (172.16.200.34)
  forwarded to 172.16.200.18
  ```

---

## **Solusi Masalah Umum**  

| **Masalah** | **Solusi** |
|-------------|------------|
| BIND9 gagal start (`systemctl status bind9` menunjukkan error) | Periksa sintaks dengan `sudo named-checkconf`. |
| Permintaan DNS timeout | Pastikan firewall tidak memblokir port 53: `sudo ufw allow 53`. |
| Klien tidak terdaftar di `views` | Tambahkan IP klien ke `acl "excluded_clients"` atau `rfc1918_clients`. |
| Log tidak muncul | Pastikan direktori `/var/log/bind/` ada dan dimiliki oleh user `bind`. |

---

## **Kesimpulan**  
Dengan konfigurasi ini, BIND9 berfungsi sebagai **DNS Forwarder** dengan kebijakan:  
✅ Default: Teruskan ke `172.16.200.2`.  
✅ Pengecualian: Teruskan ke `172.16.200.18` untuk IP tertentu.  

Sekarang server DNS Anda siap digunakan! 🚀  

**💬 Pertanyaan?** Tinggalkan komentar di bawah!  

**🔗 Bagikan panduan ini jika bermanfaat!**  

📌 **Tag**: #DNS #BIND9 #Ubuntu #Networking #SysAdmin
