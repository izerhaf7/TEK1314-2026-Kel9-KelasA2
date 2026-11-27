# **PURPLE TEAM LOG** 

Tanggal: 25 November 2026 

Target: Ubuntu Linux — 172.16.9.1 Attacker: Kali Linux — 172.16.9.150 Monitoring: Security Onion — 172.16.9.100 

## **A. SCENARIO 1 — Information Disclosure melalui Dokumentasi API** 

## **1. PLAN** 

Mengidentifikasi kemungkinan kebocoran informasi melalui dokumentasi API FastAPI yang dapat diakses tanpa autentikasi. Dokumentasi /docs berpotensi memberikan informasi mengenai struktur endpoint, metode HTTP, dan parameter API. 

## **2. EXECUTE** 

Kali Linux melakukan pengaksesan terhadap endpoint dokumentasi API: 

```
curl -k -i https://172.16.9.1/docs
```

## **3. DETECT** 

- a. Mengamati lalu lintas HTTP/HTTPS menggunakan Wireshark. 

- b. Memeriksa access log Caddy dan log aplikasi. 

- c. Mengidentifikasi respons server terhadap permintaan /docs. 

- d. Memastikan apakah dokumentasi Swagger UI menampilkan informasi endpoint API. 

## **4. GAP ANALYSIS** 

Backend masih mengaktifkan dokumentasi Swagger UI (/docs) tanpa pembatasan akses sehingga informasi struktur API dapat diketahui oleh pihak yang tidak berwenang. 

## **5. IMPROVE** 

- a. Menonaktifkan dokumentasi /docs, /redoc, dan /openapi.json pada konfigurasi FastAPI. 

- b. Memastikan dokumentasi hanya tersedia pada lingkungan pengembangan jika memang diperlukan. 

## **6. RETEST** 

Kali Linux mencoba kembali mengakses: 

```
curl -k -i https://172.16.9.1/docs
```

## **7. RESULT** 

PASSED apabila dokumentasi API tidak lagi dapat diakses oleh pihak yang tidak berwenang, sedangkan endpoint API yang sah tetap berfungsi. 

## **B. SCENARIO 2 — False Data Injection melalui POST API** 

## **1. PLAN** 

Menguji kemungkinan pengiriman data telemetry palsu oleh pihak yang tidak berwenang melalui endpoint ingest API sehingga data tersebut masuk ke database PostgreSQL 

## **2. EXECUTE** 

Kali Linux mengirimkan data telemetry palsu melalui endpoint: 

```
POST /v1/ingest/readings
```

Contoh pengujian: 

```
curl -k -X POST https://172.16.9.1/v1/ingest/readings
\
-H "Content-Type: application/json" \
-d
'{"node_id":"FAKE-NODE-ATTACKER","temperature":85.5,"
humidity":5.0,"soil_moisture":99.9,"soil_ph":2.0}'
```

## **3. DETECT** 

- a. Mengamati komunikasi jaringan menggunakan Wireshark. 

- b. Memeriksa access log Caddy dan log FastAPI. 

- c. Memeriksa tabel sensor_readings pada PostgreSQL. 

- d. Mengidentifikasi data yang berasal dari node tidak dikenal. 

## **4. GAP ANALYSIS** 

Endpoint ingest API belum menerapkan mekanisme autentikasi sehingga pihak yang tidak memiliki otorisasi dapat mengirimkan data telemetry dan menyimpannya ke database. 

## **5. IMPROVE** 

- a. Menerapkan autentikasi berbasis API Key menggunakan header X-API-Key. 

b. Memvalidasi identitas node sebelum menerima data telemetry. 

- c. Menerapkan validasi rentang nilai sensor. 

- d. Mencatat aktivitas pengiriman data yang mencurigakan. 

## **6. RETEST** 

Kali Linux mengulangi pengiriman POST menggunakan identitas node palsu tanpa API Key yang valid. Kemudian dilakukan pengujian menggunakan API 

Key yang sah untuk memastikan perangkat resmi tetap dapat mengirimkan telemetry. 

## **7. RESULT** 

PASSED apabila pengiriman data tanpa autentikasi yang valid ditolak, sedangkan pengiriman data dari node yang terautentikasi berhasil diproses. 

## C. **SCENARIO 3 — Denial of Service (DoS) terhadap Backend IoT SMILE** 

### **1. PLAN** 

Menguji ketahanan dan ketersediaan layanan backend IoT SMILE terhadap peningkatan jumlah request HTTP/HTTPS secara berulang. Pengujian dilakukan untuk mengidentifikasi apakah lonjakan request dapat mengganggu kinerja layanan FastAPI dan Caddy pada Ubuntu Server. 

### **2. EXECUTE** 

Kali Linux melakukan pemeriksaan layanan HTTPS pada Ubuntu Server: 

```
nmap -sV -p 443 172.16.9.1
```

### **Mengukur waktu respons normal endpoint health:** 

```
curl -k -s -o /dev/null -w "HTTP:%{http_code}
TIME:%{time_total}s\n" https://172.16.9.1/v1/health
```

**Kali Linux melakukan simulasi peningkatan request secara terkontrol menggunakan WRK:** 

```
wrk -t1 -c5 -d10s -H "X-Test: DoS-Lab"
https://172.16.9.1/v1/health
```

### **3.  DETECT** 

- a. Mengamati lalu lintas request HTTPS dari Kali Linux menuju Ubuntu Server menggunakan Wireshark. 

- b. Memantau log aktivitas Caddy menggunakan journalctl. 

- c. Memantau log aktivitas FastAPI/Uvicorn. 

- d. Mengamati penggunaan CPU dan RAM pada Ubuntu Server. 

- e. Mengidentifikasi peningkatan request, perubahan response time, dan kemungkinan kegagalan layanan. 

### **4. GAP ANALYSIS** 

Apabila backend menerima request berulang tanpa pembatasan, terdapat risiko penggunaan resource server secara berlebihan yang dapat menyebabkan penurunan performa hingga terganggunya ketersediaan layanan. Kondisi ini menunjukkan perlunya mekanisme pembatasan request untuk melindungi backend dari lonjakan trafik yang tidak terkendali. 

### **5. IMPROVE** 

- a. Mengimplementasikan mekanisme Rate Limiting menggunakan library SlowAPI pada FastAPI. 

- b. Membatasi jumlah request maksimal menjadi 10 request per IP dalam periode 10 detik. 

- c. Mengembalikan HTTP status 429 Too Many Requests apabila jumlah request melebihi batas yang ditentukan. 

- d. Menerapkan rate limiting pada endpoint /v1/health, /v1/ingest/readings, /v1/commands, dan /v1/ingest/actuations. 

- e. Mempertahankan autentikasi API Key dan koneksi PostgreSQL agar mekanisme keamanan yang sudah diterapkan tidak terganggu. 

### **6.  RETEST** 

Kali Linux melakukan pengujian ulang terhadap endpoint health dengan mengirimkan 12 request secara berurutan: 

```
for i in {1..12}; do curl -k -s -o /dev/null -w
"%{http_code}\n" https://172.16.9.1/v1/health; done
```

Kemudian memeriksa respons server untuk memastikan request yang melebihi batas rate limiting mendapatkan HTTP 429. 

### **7.  RESULT** 

PASSED apabila backend IoT SMILE mampu membatasi request berdasarkan alamat IP dengan ketentuan maksimal 10 request dalam 10 detik, memberikan respons HTTP 429 Too Many Requests ketika batas terlampaui, serta tetap dapat melayani request normal setelah periode pembatasan berakhir. Selain itu, endpoint FastAPI tetap berfungsi normal, autentikasi API Key tetap berjalan, dan koneksi PostgreSQL tidak terganggu. 

