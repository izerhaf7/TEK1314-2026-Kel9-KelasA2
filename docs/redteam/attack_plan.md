# Red Team Attack Plan — SMILE (Smart Soil Monitoring System)


---

## 1. Tujuan dan Ruang Lingkup

Tujuan latihan Red Team ini adalah menguji ketahanan SMILE terhadap serangan yang **mengancam integritas keputusan irigasi dan fertigasi**, bukan sekadar membuktikan sebuah celah dapat dieksploitasi. SMILE adalah sistem *cyber-physical*: data sensor yang salah atau perintah aktuator yang tidak sah dapat berujung pada penyiraman atau pemupukan yang keliru pada tanaman nyata. Karena itu, fokus utama serangan adalah jalur **sensor → server → basis data → aktuator**, bukan hanya pencurian data.

Ruang lingkup yang diuji:

- Jalur telemetri node ESP32-S3 → *endpoint ingest* backend (HTTP POST).
- *Actuator command queue* dan mekanisme *polling* perintah oleh node.
- Lapisan Wi-Fi node di lapangan (node memakai mode *multi-SSID priority reconnect*).
- Akses ke basis data **PostgreSQL lokal** (*staging*) dan jalur backend/app yang mengaksesnya.
- Permukaan publik backend FastAPI (domain *staging* di balik Cloudflare Tunnel).

Di luar lingkup: serangan ke infrastruktur pihak ketiga (Supabase sebagai layanan, Open-Meteo, Cloudflare), rekayasa sosial terhadap anggota tim, dan serangan fisik destruktif terhadap perangkat lapangan.

---

## 2. Aset yang Dilindungi

Daftar ini menjawab pertanyaan dosen: *apa yang sedang kami lindungi.*

| Aset | Mengapa kritis | Dampak bila jatuh |
|---|---|---|
| Integritas data telemetri (kelembapan, suhu, EC, hujan) | Menjadi dasar keputusan irigasi/fertigasi | Keputusan penyiraman salah pada tanaman nyata |
| Integritas *command queue* aktuator | Mengendalikan pompa air dan pompa peristaltik | Penyiraman/fertigasi berlebih, pemborosan air, pompa cepat rusak |
| Kredensial dan akses basis data (kredensial PostgreSQL, port DB, jalur query backend) | Pintu masuk ke seluruh data lahan | Kebocoran atau manipulasi data lahan |
| Ketersediaan node di lapangan | Perekaman data berkelanjutan | Hilangnya kontinuitas data; menguji klaim *store-and-forward* |
| Kerahasiaan konfigurasi (kredensial Wi-Fi, *endpoint*, kunci API dalam *firmware*/APK) | Menjadi kunci ke seluruh rantai | Serangan lanjutan yang lebih dalam |

---

## 3. Arsitektur dan Permukaan Serangan

Ringkasan alur data SMILE dan titik-titik yang menjadi sasaran:

```
[Sensor 7-in-1 RS485 / SHT31 / Rain gauge]
            │  (Modbus RTU, lokal)
            ▼
     [Node ESP32-S3] ──SD card (log lokal)
            │  (A) HTTP POST telemetri
            │  (B) Polling actuator command queue
            ▼
   [Backend FastAPI / ingest server]  ◄── Open-Meteo (cuaca)
            │  (C) baca/tulis   (D) query DB
            ▼
      [PostgreSQL lokal (staging)]
            ▲
            │  app membaca data lewat backend
      [Aplikasi Mobile Flutter]
```

| Kode | Permukaan serangan | Properti keamanan yang diuji |
|---|---|---|
| A | Endpoint ingest telemetri | Autentikasi node, integritas data, anti-*replay* |
| B | Command queue aktuator | Otorisasi perintah, autentikasi dua arah node↔server |
| C | Backend FastAPI (domain *staging*) | Hardening *endpoint*, dokumentasi API yang terekspos |
| D | PostgreSQL lokal + jalur query backend | Kerahasiaan kredensial DB, eksposur port, ketahanan terhadap injeksi |
| Wi-Fi | Koneksi nirkabel node | Ketahanan *reconnect*, risiko *evil twin*, ketersediaan |

---

## 4. Aturan Main dan Keamanan Simulasi

Bagian ini menjelaskan **mengapa simulasi ini aman bagi proyek IoT yang sesungguhnya** — persyaratan eksplisit dari dosen. Seluruh skenario WAJIB mematuhi aturan berikut.

1. **Lingkungan terpisah.** Jalankan terhadap salinan *staging*: **PostgreSQL di server lokal** yang diisi data sintetis (tanpa koneksi ke proyek Supabase produksi), instans FastAPI lokal/*staging*, dan satu ESP32 uji — **bukan** node ladang yang terpasang pada tanaman.
2. **Aktuator di-*stub*.** Ganti pompa dengan feedback lain di software untuk menyederhanakan arsitektur.
3. **Jaringan milik sendiri.** Serangan Wi-Fi (*deauth*, *evil twin*) hanya boleh dilakukan pada *access point* lab terisolasi yang dikendalikan sendiri (mis. SSID `SMILE-LAB`). **Dilarang** melakukan *deauth*/DoS terhadap Wi-Fi kampus, Wi-Fi kebun, atau jaringan publik mana pun — selain tidak etis, itu melanggar hukum.
4. **Data sintetis.** Tidak memakai data petani/pengguna nyata. Isi basis data uji dengan data *dummy*.
5. **Dapat dipulihkan.** Ambil *snapshot* basis data *staging* sebelum uji dan pulihkan setelahnya.
6. **Terdokumentasi.** Setiap skenario dicatat pada `docs/purple-team-log.md` mengikuti 5 tahap Purple Team Loop (Bagian X).

---

## 5. Skenario Serangan

Lima skenario disusun; **S1–S3 wajib dijalankan** (memenuhi syarat ≥3), S4–S5 opsional sebagai pengayaan. Setiap skenario dipetakan ke kategori STRIDE dan diakhiri dengan rujukan ke mitigasi di `docs/blueteam/hardening_checklist.md`.

### S1 — Injeksi Data Sensor Palsu (*False Data Injection*)

| | |
|---|---|
| **Kategori STRIDE** | *Spoofing* + *Tampering* |
| **Target** | A — endpoint ingest telemetri backend (*staging*) |
| **Tools** | `curl`, Python `httpx`/`requests`, Burp Suite (*repeater*) |

**Hipotesis.** Jika *endpoint* ingest menerima POST tanpa autentikasi node yang kuat, penyerang dapat menyuntikkan telemetri palsu seolah-olah berasal dari node sah.

**Langkah (tingkat rencana).**
1. Amati struktur satu payload telemetri sah dari node uji (format JSON, field, *node id*).
2. Susun payload tiruan dengan nilai ekstrem, mis. kelembapan jauh di bawah ambang siram.
3. Kirim payload ke *endpoint* *staging* tanpa menyertakan kredensial node yang valid.
4. Amati apakah server menerima, menyimpan, dan meneruskannya ke mesin keputusan.

**Dampak bila berhasil.** Mesin keputusan dapat memerintahkan penyiraman/fertigasi berdasarkan kondisi yang tidak nyata — serangan *cyber-physical* paling langsung terhadap SMILE.

**Deteksi (Blue Team).** Lonjakan data yang tidak konsisten dengan laju fisik tanah, data dari *node id* yang tidak dikenal, atau tanpa *signature* yang sah.

**Mitigasi → hardening.** Autentikasi node (kunci per-node/HMAC pada payload), validasi rentang nilai fisik di sisi server, TLS wajib. (Ref. `hardening_checklist.md`.)

---

### S2 — Perintah Aktuator Tidak Sah (*Unauthorized Actuation*)

| | |
|---|---|
| **Kategori STRIDE** | *Tampering* + *Elevation of Privilege* |
| **Target** | B — *actuator command queue* / jalur *polling* node |
| **Tools** | `curl`/Postman, Python, Burp Suite |

**Hipotesis.** Jika perintah ke *command queue* dapat disisipkan tanpa otorisasi, atau jika node mengeksekusi perintah tanpa memverifikasi keaslian server, penyerang dapat menyalakan pompa dari jarak jauh.

**Langkah (tingkat rencana).**
1. Pelajari skema *command queue* (field perintah, target node, durasi) di *staging*.
2. Sisipkan satu perintah "nyalakan pompa" ke antrean uji tanpa otorisasi yang sah.
3. Biarkan node uji (dengan aktuator di-*stub* LED/ember) melakukan *polling*.
4. Amati apakah node mengeksekusi perintah tersebut.

**Dampak bila berhasil.** Penyiraman atau fertigasi berlebih, pemborosan air/pupuk, dan keausan pompa — kerusakan fisik tanpa menyentuh perangkat.

**Deteksi (Blue Team).** Perintah aktuator di luar jendela keputusan pagi, tanpa korelasi dengan keputusan mesin, atau dari sumber tak dikenal.

**Mitigasi → hardening.** Otorisasi perintah di sisi server, autentikasi dua arah node↔server, *safety limit* lokal pada ESP32 (batas durasi/volume per siraman — sesuai desain "rem" pada node). (Ref. `hardening_checklist.md`.)

---

### S3 — Akses Tidak Sah ke Basis Data PostgreSQL

| | |
|---|---|
| **Kategori STRIDE** | *Information Disclosure* + *Tampering* |
| **Target** | D — PostgreSQL lokal (*staging*) dan jalur backend yang mengaksesnya |
| **Tools** | `nmap` (cek port 5432), `jadx`/`apktool` (cari kredensial ter-*hardcode* di APK uji), `sqlmap` (uji SQL injection pada *endpoint* backend) |

**Hipotesis.** Basis data dapat terbuka bila kredensial PostgreSQL ter-*hardcode* di kode, port 5432 terekspos ke jaringan, atau *endpoint* backend menyusun kueri tanpa *parameterized query* sehingga rentan SQL injection.

**Langkah (tingkat rencana).**
1. Pindai host *staging* untuk memeriksa apakah port 5432 terbuka ke jaringan, bukan terbatas ke *localhost*.
2. Dekompilasi *build staging* APK dan telusuri kode backend uji untuk mencari kredensial DB yang ter-*hardcode*.
3. Uji satu *endpoint* backend yang menerima parameter dengan masukan yang dirancang untuk memicu injeksi; amati apakah kueri dapat dimanipulasi.
4. Bila kredensial/koneksi diperoleh, uji apakah peran DB yang dipakai backend memiliki hak melebihi kebutuhan (*least privilege*).

**Dampak bila berhasil.** Pembacaan atau perubahan langsung pada data lahan, kalibrasi, dan keputusan — merusak seluruh rantai inferensi SMILE.

**Deteksi (Blue Team).** Koneksi ke 5432 dari host tak dikenal, pola kueri anomali/error SQL pada log backend, lonjakan baca dari satu koneksi.

**Mitigasi → hardening.** Kredensial lewat *environment variable* (bukan *hardcode*), batasi *bind* PostgreSQL ke *localhost*/jaringan internal, *parameterized query*/ORM, peran DB *least-privilege* untuk backend. (Ref. `hardening_checklist.md`.)

> **Catatan [CEK]:** Uji ini memakai PostgreSQL lokal demi isolasi. Pada SMILE **produksi** yang memakai Supabase, lapisan *anon key* dan RLS tetap perlu diaudit terpisah karena tidak tercakup skenario lokal ini.

### S4 — Gangguan Wi-Fi: *Deauth* dan *Evil Twin* (opsional)

| | |
|---|---|
| **Kategori STRIDE** | *Denial of Service* + *Spoofing* (MITM) |
| **Target** | Wi-Fi — node uji pada SSID lab terisolasi milik sendiri |
| **Tools** | *suite* aircrack-ng (`airodump-ng`, `aireplay-ng`), `hostapd` (AP palsu) |

**Hipotesis (a) DoS.** Serangan *deauth* memutus node dari AP; karena node memakai *store-and-forward* (log ke SD), data seharusnya tetap utuh dan tersinkron saat koneksi pulih. Skenario ini **memverifikasi klaim ketahanan** tersebut, bukan sekadar merusak.

**Hipotesis (b) Evil Twin.** Karena *firmware* memakai *multi-SSID priority reconnect*, node dapat tertipu bergabung ke AP palsu dengan SSID yang sudah dikenal, membuka peluang MITM pada telemetri.

**Langkah (tingkat rencana).**
1. Pada SSID lab `SMILE-LAB` milik sendiri, lakukan *deauth* terhadap node uji; catat perilaku *offline* dan hasil sinkronisasi setelah koneksi pulih.
2. Siapkan AP palsu ber-SSID sama; amati apakah node uji otomatis bergabung dan apakah telemetri dapat diintip di jalur palsu.

> **Aturan tegas:** hanya pada AP lab milik sendiri. Dilarang keras terhadap Wi-Fi kampus/kebun/publik.

**Dampak bila berhasil.** Node offline berkepanjangan (bila *store-and-forward* gagal) atau telemetri terbaca/termodifikasi di jalur (bila tanpa TLS).

**Deteksi (Blue Team).** Lonjakan *deauth frame*, kemunculan *dual* AP ber-SSID sama, perubahan BSSID yang dilihat node.

**Mitigasi → hardening.** TLS pada telemetri agar MITM tidak berguna, verifikasi sertifikat/*server pinning* di node, ketahanan *store-and-forward*, peringatan node saat berpindah AP. (Ref. `hardening_checklist.md`.)

---

### S5 — Pemetaan Permukaan Backend (*Recon*) (opsional)

| | |
|---|---|
| **Kategori STRIDE** | *Information Disclosure* (pra-eksploitasi) |
| **Target** | C — domain backend FastAPI (*staging*) |
| **Tools** | `nmap`, `ffuf`/`gobuster`, `nikto`, pemeriksaan `/docs` `/openapi.json` `/redoc` |

**Hipotesis.** FastAPI mengekspos dokumentasi interaktif (`/docs`, `/openapi.json`) secara bawaan. Bila tidak dinonaktifkan di *production*, penyerang memperoleh peta lengkap *endpoint* dan skema sebagai bekal S1–S2.

**Langkah (tingkat rencana).**
1. Pindai port dan layanan domain *staging*.
2. Periksa keberadaan `/docs`, `/openapi.json`, `/redoc`.
3. Petakan *endpoint* dan skema yang terungkap, lalu nilai mana yang mempermudah S1/S2.

**Dampak bila berhasil.** Pengungkapan seluruh permukaan API, mempercepat serangan lain.

**Deteksi (Blue Team).** *Scanning* beruntun, akses ke jalur dokumentasi dari IP tak dikenal.

**Mitigasi → hardening.** Nonaktifkan `/docs` dan `/openapi.json` di *production*, autentikasi pada semua *endpoint* sensitif, *rate limiting*, *security header*. (Ref. `hardening_checklist.md`.)

---

## 6. Pemetaan ke Hardening Checklist dan Purple Team Loop

| Skenario | Properti diuji | Kontrol di `hardening_checklist.md` (Bagian VIII) |
|---|---|---|
| S1 | Integritas telemetri | Autentikasi node, validasi rentang nilai, TLS |
| S2 | Otorisasi aktuator | Otorisasi perintah, auth dua arah, *safety limit* ESP32 |
| S3 | Kerahasiaan & integritas DB | Kredensial via env, *bind localhost*, *parameterized query*, *least-privilege* |
| S4 | Ketersediaan & anti-MITM | TLS + *pinning*, ketahanan *store-and-forward* |
| S5 | Minimnya permukaan | Nonaktifkan dokumentasi API, *rate limiting*, *header* |

Setiap skenario yang dijalankan dicatat pada `docs/purple-team-log.md` mengikuti 5 tahap Purple Team Loop: (1) rencana serangan, (2) eksekusi Red Team, (3) deteksi Blue Team, (4) perbaikan/hardening, (5) uji ulang.

---

## 7. Asumsi dan Catatan [CEK]

- **[CEK]** Nama *endpoint* ingest, skema *command queue*, dan nama tabel PostgreSQL perlu dikonfirmasi dari repo backend (`izerhaf7/smile-backendsrv`) dan app (`Nirbatas-SMILE`) agar langkah tepat.
- **[CEK]** Uji keamanan memakai PostgreSQL lokal; SMILE produksi tetap memakai Supabase, jadi nyatakan ini jujur di `tech_stack_declaration.md` dan audit RLS Supabase secara terpisah.
- **[CEK]** Untuk S3, jangan dekompilasi *build production* yang berisi kredensial/kunci nyata; gunakan *build staging*.
- **[CEK]** Samakan nomor "Bagian IX/VIII/X" dengan buku panduan resmi.
- **Pesan komit yang disarankan:** `docs: add minggu-9 slides, hardening checklist, and red team attack plan`.
