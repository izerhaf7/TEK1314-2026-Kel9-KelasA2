# 🌐 Skema Pembagian IP — Skenario Kelompok 9

> Pembagian IP yang kami susun didasari dari kebutuhan skenario kami, baik IP privat maupun IP publik yang kami rancang keduanya memiliki **_identifier_ unik** di oktet ke-3 berupa **nomor kelompok** kami.

---

## 🧩 Topologi Utama

Topologi ini meliputi IP privat dan IP publik yang disusun sedemikian rupa mendekati studi kasus _design system_ pada dunia nyata, sekaligus merepresentasikan topologi dari sistem **IoT (Internet of Things)** kami.

| Hostname | IP Address | OS Direncanakan |
|:---|:---|:---:|
| **Attacker Node** | Eth0: `10.0.9.10/24` | Kali Linux |
| **Monitoring Node** | Eth0: `172.16.9.10/24` | Security Union |
| **Target Node** | Eth0: `192.168.9.10/24` | Metasploitable |
| **RServer** | Eth0: `192.168.9.1/24`<br>Eth1: `200.200.9.2/30` | Linux Debian Server |
| **Rclient** | Eth0: `172.16.9.1/24`<br>Eth1: `200.200.9.6/30` | Linux Debian Server |
| **RAttacker** | Eth0: `10.0.9.1/24`<br>Eth1: `200.200.9.10/30` | Linux Debian Server |
| **ISP** | Eth0: `200.200.9.1/30`<br>Eth1: `200.200.9.5/30`<br>Eth2: `200.200.9.9/30` | Linux Debian Server |

---

## 🔁 Topologi Alternatif

> Topologi ini disiapkan **hanya sebagai cadangan** apabila topologi utama tidak memenuhi syarat praktikum.

| Hostname | IP Address | OS Direncanakan |
|:---|:---|:---:|
| **Attacker Node** | `172.16.9.150` | Kali Linux |
| **Monitoring Node** | `172.16.9.100` | Security Union |
| **Target Server** | `172.16.9.1` | Metasploitable |
