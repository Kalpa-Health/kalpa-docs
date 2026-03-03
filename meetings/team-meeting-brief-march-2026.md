# Briefing Tim — Kalpa Engineering: Dokumentasi, CI/CD & Standar Repo

**Tanggal**: Maret 2026
**Durasi meeting**: ~75 menit
**Penyaji**: Alex

---

## 1. Apa yang Sudah Dilakukan (dan Kenapa)

Selama beberapa minggu terakhir, kami melakukan **audit teknis menyeluruh** terhadap `wellmed-gateway-go` dan membangun **infrastruktur standar org** yang akan dipakai di semua repo Kalpa-health sekarang dan ke depannya.

### 1.1 Gateway — Audit & Dokumentasi

| Deliverable | Detail |
|-------------|--------|
| **README lengkap** | Arsitektur, 39 module, env vars, request flow — semua dalam satu dokumen |
| **Dokumentasi 6 service utama** | transaction, user, patient, autolist, frontline, role |
| **Diagram arsitektur** | Mermaid flowchart: Browser → Fiber → Auth → gRPC → Backbone |
| **45 automated tests** | 91.8% code coverage pada package yang diuji |
| **Makefile** | `make lint`, `make test`, `make build` — satu perintah untuk segalanya |
| **4 GitHub Actions workflows** | CI otomatis setiap PR: lint, test, build, Claude PR review, doc checker, approval check |
| **Claude AI PR review** | Bot review kode bahasa Indonesia setiap ada PR baru |
| **Branch protection** | develop/staging/main sudah dikonfigurasi dan aktif |
| **Notifikasi approval** | Alert otomatis ke Alex jika merge ke `main` tanpa approval Hamzah |

Sebelumnya: **1 test file, 9 tests, tidak ada CI, tidak ada dokumentasi**.

### 1.2 Integration Auth Layer — Gateway Phase 1 & Backbone Phase 2

Ini adalah fitur baru pertama yang dibangun di atas infrastruktur standar yang baru. Konteks: Jurnal.id (sistem akuntansi Mekari) mengirim webhook ke WellMed saat transaksi diposting. Gateway perlu memverifikasi bahwa webhook itu asli — menggunakan HMAC-SHA256 signature — tanpa menyimpan kredensial sendiri.

| Deliverable | Detail |
|-------------|--------|
| **Gateway PR #5** — webhook HMAC verification | Middleware verifikasi signature Jurnal.id inbound. Caching config per-tenant di Redis (15 menit). Menunggu backbone PR #1 sebelum bisa merge. |
| **Backbone PR #1** — `IntegrationService` gRPC | Endpoint baru `GetTenantIntegrationConfig` — backbone mengambil kredensial dari AWS SSM Parameter Store dan mengembalikan ke gateway. Cache Redis 15 menit di sisi backbone juga. |
| **`apiclient` package** | Klien HTTP authenticated untuk tiga integrasi: Claude (API key), SATU SEHAT (OAuth), Jurnal.id (HMAC outbound signing). |
| **Service-key auth** | Panggilan gateway → backbone untuk endpoint ini menggunakan `x-service-key` header (bukan JWT user) — defence-in-depth di samping SG network controls. |
| **AWS Security Groups** | Port 50051 (gRPC backbone) sudah dikonfigurasi — lihat §5.1 di bawah. |

**Urutan merge yang benar:**
```
Backbone PR #1 merge ke develop dulu
→ Backbone deploy ke dev
→ Gateway PR #5 bisa di-approve dan merge
```

### 1.3 Infrastruktur Org — Standar Baru

| Deliverable | Detail |
|-------------|--------|
| **`wellmed-infrastructure` repo** | Repo baru di Kalpa-health org: script, template CI, konfigurasi AWS |
| **`bootstrap-repo.sh`** | Satu perintah untuk setup repo baru sesuai standar org |
| **Template CI workflows** | Source of truth untuk semua repo: ci.yml, pr-review.yml, main-approval-check.yml |
| **`wellmed-backbone` di-bootstrap** | Backbone sekarang punya branch structure, CI, dan branch protection yang sama |
| **HOW-TO.md** | Dokumentasi lengkap cara pakai bootstrap di `wellmed-infrastructure` |

---

## 2. Standar Repo Baru — Bootstrap

Mulai sekarang, setiap repo baru di Kalpa-health di-setup menggunakan **satu perintah**:

```bash
cd ~/Projects/WellMed/infrastructure/scripts
./bootstrap-repo.sh Kalpa-health/<nama-repo>
```

### 2.1 Apa yang Dilakukan Bootstrap Secara Otomatis

```
Step 1: Branch
  ├── Buat branch develop (dari main)
  └── Buat branch staging (dari main)

Step 2: Workflow files (di-push ke .github/workflows/)
  ├── ci.yml              — lint + unit test + build
  ├── pr-review.yml       — Claude AI code review
  └── main-approval-check.yml — notifikasi approval

Step 3: Branch protection
  ├── main    — 2 approval wajib, enforce admins: aktif
  ├── staging — 2 approval wajib
  └── develop — 1 approval wajib
```

Semua status check (Lint, Unit Test, Build) harus hijau sebelum bisa merge di ketiga branch.

### 2.2 Opsi Bootstrap

| Flag | Default | Kapan Diubah |
|------|---------|-------------|
| `--go-version VERSION` | `1.25.6` | Repo pakai Go versi berbeda |
| `--binary-name NAME` | Nama repo minus `wellmed-` | Binary name tidak sama dengan repo name |
| `--cmd-path PATH` | `./cmd` | Entry point bukan di `./cmd` |
| `--dry-run` | Off | Preview semua aksi tanpa mengeksekusi |

### 2.3 Setelah Bootstrap — 2 Langkah Manual

1. **Update konteks arsitektur** di `.github/workflows/pr-review.yml` (cari `TODO:ARCH`) — isi dengan deskripsi arsitektur repo ini
//AK COMMENT - bisa minta bantuan dari bang claude


Dokumentasi lengkap: [`wellmed-infrastructure/HOW-TO.md`](https://github.com/Kalpa-Health/wellmed-infrastructure/blob/main/HOW-TO.md)

---

## 3. Workflow Developer — 3 Langkah

```
Feature Branch → Pull Request ke develop → Merge setelah CI hijau + 1 approval
```

### 3.1 Alur Normal (sehari-hari)

```bash
git checkout develop
git pull origin develop
git checkout -b feature/nama-fitur

# ... kerjakan ...

make lint && make test    # cek lokal dulu
git push origin feature/nama-fitur
# buka PR ke develop di GitHub
```

### 3.2 Yang Terjadi Otomatis Saat PR Dibuka

1. **CI berjalan** — lint + test + build (±5 menit)
2. **Claude AI posting komentar** — review diff dalam Bahasa Indonesia, flag risiko HIJAU/KUNING/MERAH
3. **Doc checker** — cek apakah ada module baru yang belum punya dokumentasi

### 3.3 Aturan Merge per Branch

| Branch | Approval dibutuhkan | CI harus hijau |
|--------|---------------------|----------------|
| `develop` | 1 | Ya |
| `staging` | 2 | Ya |
| `main` | 2 (termasuk CTO) | Ya |

---

## 4. Perubahan Per Anggota Tim

<<<<<<< Local Changes
### Untuk semua developer (Abdul, Fajri, Hamzah)
||||||| Old File
## 4. Perubahan Per Anggota Tim
=======
### Untuk semua developer (Abdul, Raka, Fajri, Hamzah)
>>>>>>> External Changes

**Yang berubah:**
- [ ] Semua kode masuk via PR ke `develop` — tidak ada push langsung ke `main`
- [ ] Jalankan `make lint && make test` sebelum push (butuh ±30 detik)
- [ ] Update git remote ke repo baru di org:
  ```bash
  git remote set-url origin https://github.com/Kalpa-Health/wellmed-gateway-go.git
  git fetch origin
  git remote -v  # verifikasi
  ```
- [ ] Rename folder lokal jika masih pakai nama lama: `kalpa-gateway-main` → `wellmed-gateway-go`

**Yang tidak berubah:**
- Cara menulis kode Go — tidak ada perubahan arsitektur
- Local development flow — masih sama
- Kebebasan push ke feature branch sendiri

### Untuk Alex (CTO / reviewer) — sudah selesai ✅

- [x] Repo dipindah ke GitHub Organization (`Kalpa-Health/wellmed-gateway-go`)
- [x] Branch structure: `main`, `develop`, `staging` aktif di gateway dan backbone
- [x] Branch protection aktif di ketiga branch (semua repo)
- [x] `ANTHROPIC_API_KEY` dikonfigurasi di GitHub Secrets (gateway)
- [x] `wellmed-infrastructure` repo dibuat — bootstrap tooling tersedia
- [x] `wellmed-backbone` di-bootstrap sesuai standar org
- [x] Integration auth layer Phase 2 dibangun di backbone (PR #1 dibuka)
- [x] SG port 50051 dikonfigurasi — prod dan dev/staging (lihat §5.1)
- [x] `BACKBONE_SERVICE_KEY` di-generate, disimpan di SSM, dan ditambahkan ke `.env` di dev-app-go
- [ ] Migrasi semua env vars Go services ke SSM Parameter Store (lihat §5.2 — Alex ambil task ini)

---

## 5. Dampak pada Deployment

**Sekarang**: Abdul rsync langsung ke production. Ini akan diganti.
**Setelah**: Deploy hanya terjadi via merge ke `main` (workflow CD akan ditambahkan di fase berikutnya). Untuk saat ini, alur deploy manual tetap bisa jalan — tapi kodenya harus lewat `main` dulu.

**Cutover**: Disepakati bersama di meeting ini. Rekomendasi: 1 minggu setelah meeting.

// AK Comment - lebih baik kita sync repo -> VM jadi nggak butuh aktivitas manual dalam??

---

## 5.1 AWS Infrastructure — Port 50051 (gRPC Backbone)

Ini sudah dikonfigurasi. Catatan untuk referensi tim dan onboarding environment baru.

| Environment | SG Rule | Status |
|-------------|---------|--------|
| **Production** | Gateway fe SG → Backbone app SG, port 50051 inbound | ✅ Done |
| **Dev / Staging** | Gateway fe SG → Backbone dev-app SG, port 50051 inbound | ✅ Done |
| **Staging Backbone** *(belum ada)* | Perlu ditambah saat staging app backbone dibuat | ⏳ Future |

> **Catatan untuk saat staging backbone app ditambah:** Cukup clone konfigurasi SG dari dev-app backbone, lalu tambah satu inbound rule port 50051 source = staging gateway fe SG. Tidak ada yang lain yang perlu diubah di sisi infrastruktur.

## 5.2 Pending Infra — Migrasi Env Vars Go Services ke SSM

**Context:** Saat ini semua env vars backbone dan gateway (DB password, JWT secret, Redis password, AWS keys, dll.) disimpan sebagai plaintext di `/var/www/wellmed-api/.env` pada server. Ini sudah umum tapi bisa diperbaiki.

**Pola yang sudah ada** (dari integration auth layer): SSM Parameter Store SecureString → di-inject ke container via `.env` saat startup atau via script pull. Infrastruktur SSM sudah terpakai dan dipahami.

**Note:** PHP/Laravel di `.70` tidak perlu dimigrasikan — akan deprecated dalam ±30 hari.

**Yang perlu dilakukan (Alex — sudah familiar dengan pola SSM ini):**
- [ ] Inventarisasi semua secrets di `/var/www/wellmed-api/.env` pada dev dan prod
- [ ] Simpan ke SSM sebagai SecureString dengan path convention: `/wellmed/{env}/go/{service}/{key}`
- [ ] Buat script pull-and-inject sederhana yang dijalankan saat deploy (sebelum `docker compose up`)
- [ ] Hapus plaintext values dari `.env` — ganti dengan placeholder atau generate dari script

**Scope:** Go services saja (`backbone-service`, `gateway-service`, `pos-service`) pada `16-dev-plus-app-go-arm` dan `04-prod-app-arm`.

//inefficient if there way more repeat config data vs. the secret keys, good to have 2? one standard per MS kalpa, and one specific per service (secrets)?

---

## 6. PR Terbuka untuk Didiskusikan

### Backbone PR #1 ← **Baru, perlu review & merge duluan**
**`feature/integration-auth-layer` → `develop`**
👉 https://github.com/Kalpa-Health/wellmed-backbone/pull/1

| Topik | Detail |
|-------|--------|
| **`IntegrationService` gRPC** | Endpoint baru untuk gateway. Ambil kredensial dari SSM, cache Redis 15 menit. |
| **Service-key auth** | Env var baru: `BACKBONE_SERVICE_KEY` — harus di-set di ECS task definition backbone **dan** gateway (nilai sama). |
| **`apiclient` package** | Belum dipakai oleh handler mana pun — foundation untuk integrasi outbound ke Claude / SATU SEHAT / Jurnal.id. |
| **CI** | Harus hijau (Lint + Unit Test + Build) sebelum bisa merge ke develop. |

### Gateway PR #5 ← **Menunggu backbone PR #1**
**`feature/webhook-integration` → `develop`**
👉 https://github.com/Kalpa-Health/wellmed-gateway-go/pull/5

| Topik | Detail |
|-------|--------|
| **Status** | Kode selesai dan sudah diperbarui (x-service-key injection ditambah). Tidak bisa merge sebelum backbone PR #1 deploy ke dev. |
| **HMAC verification** | Middleware verifikasi signature Jurnal.id — spec sesuai dokumen Mekari. |
| **Env var baru** | `BACKBONE_SERVICE_KEY` — sama dengan yang di backbone. |

### Gateway PR #2 — develop → staging
👉 https://github.com/Kalpa-Health/wellmed-gateway-go/pull/2

| Topik | Detail |
|-------|--------|
| **Perubahan schema People/Patient** | Beberapa field jadi non-nullable. Pastikan data existing di staging tidak pecah. |
| **Env vars staging** | Cek apakah env vars di staging sudah match dengan field proto yang baru. |

---

## 7. Diskusi: Status Backbone & Rencana Dekomposisi

**Konteks**: Backbone saat ini adalah monolith terstruktur — 37 domain module dalam satu binary, semua traffic masuk melalui satu `BACKBONE_GRPC_ADDRESS`. Ini adalah *current state yang disengaja* dan sudah didokumentasikan di `wellmed-system-architecture.md §2.3.2`. Bukan drift yang tidak diketahui.

**Yang bagus**: Struktur internal sudah benar. Setiap module sudah punya `handler/`, `service/`, `repository/`, `dto/` sendiri. Schema PostgreSQL juga sudah dipartisi per service (`emr`, `cashier`, `pharmacy`, `backbone` schema). Jalur dekomposisi adalah ekstrak module ke binary baru — bukan untangle spaghetti.

**Mapping module ke service target:**

| Target Service | Module di Backbone Sekarang |
|----------------|----------------------------|
| **Tetap di Backbone** | `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`, `people` |
| **→ EMR** | `patient`, `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `data_sync`, `family_relationship` |
| **→ Cashier** | `billing`, `invoice`, `transaction`, `item`, `item_rent`, `service_price`, `card_stock` |
| **→ Pharmacy** | `pharmacy`, `pharmacy_sale`, `medicine` |

**3 keputusan yang harus disepakati sebelum bisa mulai dekomposisi:**

1. **Saga ownership setelah split.** Saga framework sekarang ada di Backbone. Visit creation saga saat ini menyentuh `patient`, `visit_patient`, `pharmacy_sale`, dan `billing` — semua dalam satu proses. Setelah split, siapa yang orkestrasi cross-service saga? Opsi: (a) EMR panggil Cashier langsung via gRPC dalam saga steps-nya, (b) Backbone tetap jadi saga-coordinator tipis sementara service lain implementasi domain logic-nya, (c) async via RabbitMQ untuk cross-service steps. **Ini keputusan paling penting — menentukan segalanya.**

2. **Module mana yang route melalui Backbone vs. langsung ke service-nya?** EMR adalah kandidat split pertama. Apakah semua EMR call langsung `Gateway → EMR`, atau ada workflow tertentu yang masih perlu lewat Backbone (misalnya untuk saga coordination)?

3. **Perubahan gateway routing.** Sekarang satu env var: `BACKBONE_GRPC_ADDRESS`. Setelah setiap split: tambah `EMR_GRPC_ADDRESS`, `CASHIER_GRPC_ADDRESS`, dst. Gateway perlu satu RPC client struct baru per service. Pekerjaan gateway-nya straightforward — tapi harus disequence dengan backbone split, bukan dilakukan sendiri.

**Rekomendasi urutan**: Mulai dari EMR — paling besar, paling domain-specific, dan sudah terdefinisi sebagai Lite-tier service di product spec. Tapi keputusan (1) harus dijawab dulu sebelum sprint planning apapun.

---

## 8. Demo Live (5 menit)

1. Buat branch: `git checkout -b test/demo-pipeline`
2. Ubah satu baris komentar di file mana saja
3. `git push origin test/demo-pipeline`
4. Buka PR ke `develop` di GitHub
5. Tonton: CI trigger → Claude posting review → doc checker komentar

---

## 9. Pertanyaan yang Mungkin Muncul

**"Ini memperlambat kerja saya?"**
> Tidak. CI jalan paralel saat kamu kerjakan hal lain. Yang nambah adalah waktu tunggu 1 approval — tapi ini juga berarti ada orang lain yang tahu kodenya sebelum masuk.

**"Kalau urgent / hotfix?"**
> Buat branch dari `main`, fix, PR langsung ke `main` dengan 2 approval. Lalu merge `main` kembali ke `develop`.

**"Test-nya saya harus tulis sendiri?"**
> Untuk sekarang, hanya package yang sudah ada test-nya yang diukur coverage-nya. Kamu tidak wajib menulis test untuk semua kode baru — tapi CI akan mengingatkan kalau coverage turun.

**"Repo baru dibuat, siapa yang setup?"**
> Alex jalankan `bootstrap-repo.sh` — selesai dalam 1-2 menit. Lihat HOW-TO di `wellmed-infrastructure`.

**"Di mana repo-nya sekarang?"**
> Semua repo ada di GitHub Organization: `github.com/Kalpa-Health`. Update remote-mu dengan perintah di atas.

---

*Dokumen ini dikelola di `kalpa-docs/meetings/`. Untuk brief berikutnya, buat file baru di folder yang sama.*
