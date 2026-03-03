# Brief Tim: Pemisahan Consultation Service dari Backbone

**Tanggal:** 03 Maret 2026
**Dari:** Alex
**Untuk:** Abdul, Raka, Fajri, Hamzah

---

## 1. Keputusan

Kita akan memisahkan modul konsultasi klinis dari Backbone menjadi service sendiri: **`wellmed-consultation`**.

Ini bukan rencana ke depan — ini keputusan yang berlaku sekarang, sebelum fitur frontend consultation selesai dikerjakan.

Dokumen resminya ada di: [`adrs/ADR-002-consultation-service-extraction.md`](../adrs/ADR-002-consultation-service-extraction.md)

---

## 2. Kenapa Keputusan Ini Perlu Dibuat Sekarang

Backbone sekarang berisi 37 domain module dalam satu binary. 13 di antaranya adalah milik domain konsultasi klinis — visit, pemeriksaan, assessment, treatment, tanda tangan dokter. Ini sudah seharusnya jadi service sendiri sejak awal, tapi diputuskan untuk ditunda dulu sampai infrastruktur CI/CD dan dokumentasi selesai dibangun.

Infrastruktur itu sekarang sudah selesai. Dan ada dua kondisi yang membuat keputusan ini tidak bisa ditunda lagi:

1. Abdul sedang menyelesaikan koneksi Consultation ke Frontend. Kita perlu tahu persis arsitektur yang dituju sebelum kode itu selesai.
2. Kalau kita tidak definisikan batasnya sekarang, fitur klinis baru akan terus masuk ke Backbone tanpa panduan yang jelas — dan semakin susah dipisahkan nanti.

---

## 3. Apa yang Berubah: Batas Module

**Module yang pindah ke `wellmed-consultation`:**

| Module | Fungsi |
|--------|--------|
| `visit_patient` | Record kunjungan, lifecycle status |
| `visit_registration` | Registrasi pasien untuk kunjungan |
| `visit_registration_referral` | Rujukan saat registrasi |
| `visit_examination` | Record pemeriksaan |
| `assessment` | Assessment klinis (diagnosis) |
| `treatment` | Record tindakan |
| `examination_treatment` | Hubungan pemeriksaan–tindakan |
| `medical_service_treatment` | Layanan medis dalam tindakan |
| `referral` | Rujukan keluar |
| `practitioner_evaluation` | Evaluasi dan tanda tangan dokter |
| `frontline` | Alur kerja staf frontline |
| `data_sync` | Utilitas sinkronisasi data kunjungan |
| `family_relationship` | Data keluarga/relasi pasien |

**Module yang tetap di Backbone:**

`patient` (demografi — dipakai oleh Cashier, Pharmacy, BPJS), `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`

`patient` tetap di Backbone karena ini adalah data identitas yang dibutuhkan oleh banyak service lain — bukan logika klinis.

---

## 4. Satu Hal Baru: Canonical Visit Record

Saat dokter melakukan tanda tangan akhir (final sign-off), Consultation akan menulis satu record ringkasan ke Backbone — berformat FHIR. Record ini berisi hasil kunjungan saja: diagnosis akhir, tindakan yang diberikan, resep yang ditulis, nama dokter.

Record ini yang dipakai untuk:
- Sinkronisasi ke SATU SEHAT
- Pembuatan invoice tertutup untuk asuransi
- Audit trail

Data kunjungan yang masih in-progress (draft assessment, tindakan belum selesai, form) tetap di Consultation dan tidak terlihat oleh service lain sampai tanda tangan dokter selesai.

Ini membuat alur data jauh lebih bersih: service downstream (SATU SEHAT, Cashier, Reporting) baca dari satu record yang rapi, bukan campuran status kunjungan yang sedang berjalan.

---

## 5. Dua Fase

**Fase 1 — Ekstraksi (dikerjakan sekarang):**
- Buat repo `wellmed-consultation`
- Pindahkan 13 module dari Backbone ke Consultation
- Setup gRPC server di Consultation
- Update Gateway untuk routing ke `CONSULTATION_GRPC_ADDRESS`
- Hapus 13 module dari Backbone

Setelah Fase 1 selesai: dua service berjalan terpisah. Consultation bisa di-deploy sendiri.

**Fase 2 — Saga Koordinasi (direncanakan setelah Fase 1 selesai):**

Visit creation adalah event paling kompleks di sistem. Service line items langsung push ke POS, ada pharmacy hook untuk resep, dan layanan tambahan bisa ditambahkan di tengah kunjungan. Pembatalan kunjungan (misalnya salah dokter atau salah poli) juga harus bisa membatalkan semua itu.

Saga untuk ini butuh desain yang matang — bukan sesuatu yang bisa diselesaikan terburu-buru. Fase 2 akan dimulai dengan sesi scenario planning setelah Fase 1 selesai dan kita punya gambaran nyata dari service yang sudah berjalan terpisah.

---

## 6. Kenapa Bukan Opsi Lain

### Opsi A — Biarkan di Backbone

Argumen utamanya: "lebih simpel karena semua saga ada di satu proses."

Ini benar hari ini. Tapi tanpa batas yang jelas, setiap fitur klinis baru akan masuk ke Backbone karena itulah jalur paling mudah. Backbone jadi semakin besar dan semakin susah dipisahkan. Kita sudah melihat pola ini di kodebase sebelumnya — bukan sesuatu yang mau diulang.

Yang lebih penting: tanpa batas yang terdokumentasi, tidak ada panduan yang bisa dipakai oleh developer ketika ada arah yang berbeda masuk dari berbagai sisi. Dokumen ini dan ADR-nya adalah panduan itu.

### Opsi C — Backbone sebagai Koordinator Tipis (Hybrid)

Idenya: Consultation punya logika klinis, Backbone koordinasi saga lintas service.

Kedengarannya masuk akal, tapi ada masalah teknis di alur request-nya:

- Kalau Gateway → Consultation, lalu Consultation → Backbone untuk koordinasi saga: Consultation memanggil ke atas (upstream). Ini membalik arah dependency dan menciptakan coupling yang susah dikelola.
- Kalau Gateway → Backbone dulu, lalu Backbone → Consultation: Backbone kembali jadi pintu masuk semua request klinis. Bukan "tipis" lagi.

Tidak ada cara yang bersih untuk membuat alur ini bekerja tanpa menciptakan masalah baru. Consultation yang koordinasi saga-nya sendiri adalah satu-satunya pilihan yang arah dependency-nya konsisten.

---

## 7. Yang Berubah untuk Masing-masing

**Abdul:** Pekerjaan frontend consultation yang sedang berjalan tidak perlu diulang dari awal — tapi target service berubah. Setelah Fase 1 selesai, backend consultation ada di `wellmed-consultation`, bukan di Backbone. Plan eksekusi ada di [`plans/consultation-extraction/PLAN.md`](../plans/consultation-extraction/PLAN.md).

**Hamzah (CTO):** Backbone yang sedang dibersihkan akan kehilangan 13 module setelah Fase 1 selesai. Ini justru mempersempit scope Backbone — fokus ke tenant, auth, config, dan referensi. Pola Go yang sedang dipelajari tetap berlaku di Consultation: struktur handler/service/repository yang sama.

**Hamzah:** Akan ada port baru untuk gRPC Consultation (kemungkinan `:50052`). Security Group perlu diupdate seperti yang dilakukan untuk Backbone di port 50051. Detail ada di plan eksekusi.

**Semua developer:** Kalau ada fitur klinis baru (apapun yang berhubungan dengan kunjungan, pemeriksaan, atau tindakan), itu masuk ke `wellmed-consultation` — bukan ke Backbone. Kalau ada yang tidak jelas masuknya ke mana, tanya dulu sebelum koding.

---

## 8. Langkah Selanjutnya

- [ ] Review plan eksekusi: [`plans/consultation-extraction/PLAN.md`](../plans/consultation-extraction/PLAN.md)
- [ ] Diskusi singkat dengan Abdul untuk koordinasi pekerjaan Frontend yang sedang berjalan
- [ ] Bootstrap `wellmed-consultation` repo — Alex jalankan `bootstrap-repo.sh`
- [ ] Mulai Fase 1 sesuai urutan di plan

Pertanyaan atau kekhawatiran tentang keputusan ini? Diskusikan sebelum Fase 1 dimulai. Setelah fase 1 berjalan, batas ini sudah dikunci.

---

*Dokumen ini adalah ringkasan keputusan. Dokumen teknis lengkap ada di [`adrs/ADR-002-consultation-service-extraction.md`](../adrs/ADR-002-consultation-service-extraction.md).*
