# Briefing Tim — Kalpa Engineering: Dokumentasi, CI/CD & Standar Repo

**Tanggal**: Maret 2026
**Durasi meeting**: ~60 menit
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

### 1.2 Infrastruktur Org — Standar Baru

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
2. **Tambah secret Anthropic**:
   ```bash
   gh secret set ANTHROPIC_API_KEY -r Kalpa-health/<nama-repo>
   ```

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

### Untuk semua developer (Ingarso, Raka, Fajri, Hamzah)

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

---

## 5. Dampak pada Deployment

**Sekarang**: Ingarso rsync langsung ke production. Ini akan diganti.

**Setelah**: Deploy hanya terjadi via merge ke `main` (workflow CD akan ditambahkan di fase berikutnya). Untuk saat ini, alur deploy manual tetap bisa jalan — tapi kodenya harus lewat `main` dulu.

**Cutover**: Disepakati bersama di meeting ini. Rekomendasi: 1 minggu setelah meeting.

---

## 6. PR Terbuka untuk Didiskusikan

**PR #5 — Webhook Integration (feature/webhook-integration → develop)**
👉 https://github.com/Kalpa-Health/wellmed-gateway-go/pull/5

| Topik | Detail |
|-------|--------|
| **HMAC signature verification** | Middleware verifikasi Mekari/Jurnal.id HMAC-SHA256 — apakah spec sudah sesuai? |
| **IntegrationRpc** | gRPC client fetch config per-tenant dari backbone — backbone siap? |
| **Phase 1 scope** | Handler sekarang hanya log + return 200. Phase 2 (forward ke backbone) kapan? |

**PR #2 — develop → staging**
👉 https://github.com/Kalpa-Health/wellmed-gateway-go/pull/2

| Topik | Detail |
|-------|--------|
| **Perubahan schema People/Patient** | Beberapa field jadi non-nullable. Pastikan data existing di staging tidak pecah. |
| **Env vars staging** | Cek apakah env vars di staging sudah match dengan field proto yang baru. |

---

## 7. Demo Live (5 menit)

1. Buat branch: `git checkout -b test/demo-pipeline`
2. Ubah satu baris komentar di file mana saja
3. `git push origin test/demo-pipeline`
4. Buka PR ke `develop` di GitHub
5. Tonton: CI trigger → Claude posting review → doc checker komentar

---

## 8. Pertanyaan yang Mungkin Muncul

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
