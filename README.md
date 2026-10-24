# Performance Test JMeter - Book Store (demoqa.com)

## Isi paket
| File | Fungsi |
|---|---|
| `bookstore_performance_test.jmx` | Test plan utama (default: hanya S1 Baseline yang enabled) |
| `users.csv` | 300 akun unik. Password memenuhi kebijakan (>=8, huruf besar/kecil, angka, simbol), mis. `Pwd@0001xY!` |
| `books.csv` | Daftar ISBN Book Store |
| `generate_jmx.py` | Generator; `python3 generate_jmx.py --only 4` -> JMX dengan hanya S4 enabled |

Simpan `.jmx`, `users.csv`, `books.csv` dalam **satu folder**.

## Struktur test plan
- HTTP Request Defaults (https://demoqa.com, host bisa diganti `-Jhost=`), HTTP Header Manager (JSON), HTTP Cookie Manager (clear tiap iterasi)
- **setUp Thread Group**: registrasi akun dari `users.csv` (`POST /Account/v1/User`; 201 atau 406 "sudah ada" = OK, jadi aman dijalankan berulang)
- Di tiap Thread Group (S1-S6) terdapat flow yang sama, satu **Transaction Controller** per step:
  `T01_Login` (GenerateToken + Login) -> `T02_Browse_Books` -> `T03_Select_Book` -> `T04_Add_Book_to_Collection` -> `T05_Cleanup_Delete_Books`
- Tiap Thread Group: CSV Data Set Config (users & books), Uniform Random Timer (think time 1-3 dtk), JSR223 PreProcessor (mengunci 1 akun unik per virtual user)
- Listener: Summary Report, Aggregate Report (+ View Results Tree, disabled, hanya untuk debug)

## Correlation
- `token` <- JSON Extractor `$.token` (GenerateToken) -> `Authorization: Bearer ${token}`
- `userId` <- JSON Extractor `$.userId` (Login) -> body Add Book & `?UserId=` pada DELETE
- ISBN <- JSON Extractor `$.books[*].isbn` pada Browse (match = random) -> dipakai Select & Add. Opsi `-Jisbn_source=csv` memakai ISBN dari `books.csv`.

## Assertion
- Login: status 200; `$.token` dan `$.userId` tidak kosong
- Browse: status 200; `$.books[0].isbn` ada
- Select: status 200; `$.isbn` == ISBN yang dipilih
- Add Book: status 201; `$.books[0].isbn` == ISBN yang dikirim
- Cleanup: status 204

## Alasan pemilihan Thread Count, Ramp-Up, Loop/Duration
| # | Skenario | Threads | Ramp-Up | Loop/Durasi | Alasan |
|---|---|---|---|---|---|
| 1 | Baseline | 1 | 1 dtk | 5 iterasi | Sesuai soal; tanpa konkurensi untuk validasi script dan RT murni |
| 2 | Load 10 | 10 | 20 dtk | 300 dtk | ~2 dtk/user agar tidak spike; steady state ~280 dtk, tiap user menyelesaikan banyak flow |
| 3 | Load 50 | 50 | 100 dtk | 600 dtk | ~2 dtk/user; steady state cukup panjang untuk melihat tren degradasi |
| 4 | Load 100 | 100 | 200 dtk | 900 dtk | ~2 dtk/user; ~700 dtk steady state untuk analisis failed request |
| 5 | Stress | 100 (naikkan sampai jenuh) | 60 dtk (sesuai soal) | 900 dtk | Ramp agresif (~0.6 dtk/user) untuk melihat RT vs kenaikan load; ulangi dengan `-Js5_users=150/200` |
| 6 | Soak (opsional) | 25 (25% puncak) | 60 dtk | 2700 dtk (45 mnt) | Beban rendah-konstan lama untuk deteksi memory leak / degradasi bertahap |

Semua nilai bisa di-override lewat property, mis. `-Js3_users=50 -Js3_ramp=100 -Js3_duration=600`.
Think time: `-Jthink_min=1000 -Jthink_range=2000` (ms).

## Cara menjalankan (CLI / non-GUI)
Pastikan jumlah akun yang diregistrasi (`reg_users`) >= jumlah thread skenario.

```bash
# 1. Baseline (default enabled)
jmeter -n -t bookstore_performance_test.jmx -Jreg_users=5 -l results/s1.jtl -e -o results/s1_report

# 2-6. Buat JMX per skenario lalu jalankan, contoh S4 (100 users)
python3 generate_jmx.py --only 4
jmeter -n -t bookstore_perf_s4.jmx -Jreg_users=100 -l results/s4.jtl -e -o results/s4_report

# Stress sampai titik jenuh (S5)
python3 generate_jmx.py --only 5
jmeter -n -t bookstore_perf_s5.jmx -Jreg_users=200 -Js5_users=200 -Js5_ramp=120 -l results/s5.jtl -e -o results/s5_report
```
(Alternatif tanpa Python: buka JMX di GUI, klik kanan Thread Group yang diinginkan -> Enable, disable yang lain, lalu jalankan via CLI.)
Setelah akun terdaftar sekali, setUp Thread Group boleh di-disable.

## Catatan
- demoqa.com adalah situs publik pihak ketiga: bisa ada rate-limit / proteksi WAF sehingga muncul 403/429/5xx pada beban tinggi. Catat sebagai temuan, bukan bug script.
- Jangan jalankan test beban besar dari GUI mode; listener GUI menambah overhead.
- Skenario S2-S6 memakai Thread Group berbasis durasi (loop tak terbatas + scheduler) agar "semua user menyelesaikan flow" terpantau berulang kali selama steady state.
