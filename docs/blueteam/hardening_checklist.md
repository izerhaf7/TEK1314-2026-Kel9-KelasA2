# Checklist Implementasi Keamanan Siber Server

## 1. SECURE SSH ACCESS (Pengamanan Akses SSH)

### A. Checklist Status
- [✓] Mengubah port SSH standar (misalnya ke `2222`)
- [✓] Menonaktifkan root login (`PermitRootLogin no`)
- [✓] Mengaktifkan autentikasi berbasis kunci (`PubkeyAuthentication yes`)
- [✓] Menonaktifkan autentikasi password (`PasswordAuthentication no`)
- [✓] Membatasi pengguna yang diizinkan (`AllowUsers admin deploy`)
- [✓] Menerapkan konfigurasi baru (`sudo systemctl restart sshd`)

## 2. CONFIGURE FIREWALL (Konfigurasi UFW)

### A. Checklist Status
- [✓] Mengatur kebijakan default (`deny incoming`, `allow outgoing`)
- [✓] Membuka port SSH custom (misalnya `2222/tcp` atau `22/tcp`)
- [✓] Membuka port web/HTTP (`80/tcp`)
- [✓] Mengaktifkan firewall (`sudo ufw enable`)
- [✓] Memeriksa status UFW (`sudo ufw status`)

## 3. FAIL2BAN PROTECTION (Proteksi Brute-Force)

### A. Checklist Status
- [✓] Menginstal paket Fail2Ban
- [✓] Mengaktifkan dan menjalankan service Fail2Ban
- [✓] Memeriksa status aktif Fail2Ban Client

## 4. KEEP SYSTEM UPDATED (Pembaruan dan Pemeliharaan)

### A. Checklist Status
- [✓] Memperbarui daftar repositori dan paket sistem
- [✓] Membersihkan paket/dependensi yang tidak terpakai
- [✓] Mengatur pembaruan keamanan otomatis
- [✓] Melakukan reboot server jika diperlukan

## 5. ENABLE LOGGING & MONITORING (Audit dan Logging)

### A. Checklist Status
- [✓] Menginstal layanan `auditd`
- [✓] Mengaktifkan dan menjalankan `auditd`
- [✓] Memantau percobaan login yang gagal
- [✓] Meninjau log sistem secara berkala
- [✓] Mengonfigurasi log rotation