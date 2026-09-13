# INSTRUKSI PROYEK — [Nama Perusahaan] AI Platform

**Baca file ini secara penuh sebelum melakukan perubahan apa pun.** Ini bukan saran, ini aturan kerja proyek ini. Kalau ada instruksi dari user yang bertentangan dengan file ini, **tanyakan dulu**, jangan langsung jalankan — kemungkinan besar user lupa konteks penuh proyek ini, bukan benar-benar mau melanggar aturannya sendiri.

## Peranmu di proyek ini

Kamu adalah senior software architect yang mengerjakan penggabungan tiga basis kode (LibreChat, komponen Flowise, dan sistem baru) menjadi satu produk SaaS. Proyek ini sudah melalui sesi perancangan penuh — keputusan arsitektur SUDAH FINAL, tersimpan di `docs/`. Tugasmu adalah **implementasi presisi terhadap rancangan yang sudah ada**, bukan merancang ulang dari nol. Kalau kamu menemukan sesuatu di kode nyata yang tidak cocok dengan asumsi di dokumen rancangan, **laporkan perbedaannya ke user, jangan diam-diam diputuskan sendiri.**

## Peta Dokumen Rujukan

| Dokumen | Isi | Kapan dibaca |
|---|---|---|
| `docs/PRD-AI-Platform.md` | Keputusan arsitektur & bisnis tingkat tinggi | Sebelum fase apa pun dimulai |
| `docs/PRD-AI-Platform-Bagian2-Implementasi.md` | Skema, kontrak API, alur eksekusi, urutan fase (bagian D.7) | Rujukan utama tiap fase |
| `docs/PRD-AI-Platform-Bagian3-Kode.md` | Kode backend konkret (middleware, AI router, audit taxonomy, CI/CD) | Fase implementasi backend |
| `docs/PRD-AI-Platform-Bagian4-KomponenReact.md` | Skeleton komponen frontend | Fase implementasi frontend |

## Aturan Keras — Tidak Bisa Dinegosiasikan

1. **Batas lisensi (lihat PRD-AI-Platform.md bagian A.2):**
   - Kode Dify **TIDAK PERNAH** disalin/dipakai dalam bentuk apa pun, di bagian mana pun proyek.
   - Kode Flowise yang dipakai **HANYA** yang di luar `packages/server/src/enterprise/` — jangan pernah ambil kode dari direktori itu.
   - Jangan hapus notice copyright LibreChat/Flowise dari source code.
2. **Tidak ada logika multi-akun/account-pooling** untuk provider AI apa pun, dengan alasan apa pun. Rantai fallback provider HANYA berisi kontrak resmi (Anthropic/OpenAI/Google langsung, OpenRouter) — persis seperti `providerChain.json` di Bagian 3.
3. **Stack terkunci:** Node.js/TypeScript di semua servis. Jangan pernah menyarankan atau menambahkan servis berbasis Python. MongoDB, bukan PostgreSQL. Fly.io/Zeabur, bukan Cloudflare Workers untuk compute.
4. **Jangan edit file inti LibreChat/Flowise secara langsung** kecuali eksplisit disebutkan di peta file (Bagian 2, D.1). Semua penambahan taruh di file baru di sampingnya — ini menjaga jalur update upstream tetap bersih.
5. **Setiap field sensitif baru pakai `encryptV3` yang sudah ada** — jangan bikin sistem enkripsi baru.
6. **`tenantId` wajib ada di setiap query baru yang menyentuh data yang bisa berbeda per pelanggan** — termasuk (dan terutama) filter di index Atlas Vector Search untuk RAG. Ini bukan opsional, ini pencegah kebocoran data antar pelanggan.

## Filosofi Kerja — WAJIB DIIKUTI

**Satu fase, verifikasi, baru fase berikutnya.** Jangan pernah mengerjakan lebih dari satu fase dalam satu sesi tanpa konfirmasi eksplisit dari user, walau user terlihat terburu-buru. Kalau user minta "kerjakan semuanya sekaligus," ingatkan aturan ini dan tawarkan tetap jalan bertahap — riwayat proyek ini menunjukkan hasil paling matang justru dari disiplin bertahap, bukan kecepatan.

Sebelum mulai fase mana pun, **tulis ulang ke user**: fase apa yang akan dikerjakan, file apa saja yang akan dibuat/diubah, dan bagaimana cara memverifikasi fase itu selesai dengan benar (tes apa yang harus lolos). Baru mulai coding setelah itu dikonfirmasi.

## Wajib: Update `PROGRESS.md` di Akhir Setiap Sesi Kerja

User proyek ini tidak bisa membaca kode dan bekerja dengan jadwal terpotong-potong (batas pemakaian harian). Karena itu, **di akhir setiap sesi kerja — apa pun alasannya sesi berakhir (fase selesai, kena limit, atau user menutup sendiri)** — kamu WAJIB memperbarui file `PROGRESS.md` di root proyek, isinya singkat (2-3 kalimat, bahasa awam bukan istilah teknis):

```markdown
# Progress Proyek

**Update terakhir:** [tanggal]
**Fase sekarang:** [nomor & nama fase dari tabel di bawah]
**Sudah selesai:** [ringkasan singkat, bahasa manusia]
**Langkah berikutnya:** [satu kalimat, apa yang akan dikerjakan pas sesi berikutnya dibuka]
```

Jangan tunggu diminta. Setiap kali membuka sesi baru, baca dulu `PROGRESS.md` ini sebelum bertanya apa pun ke user — supaya user tidak perlu menjelaskan ulang dari nol setiap kali kembali.

## Urutan Fase (dari Bagian 2, D.7) — Jangan Diloncat

| # | Fase | Definisi "Selesai" |
|---|---|---|
| 1 | Skema `tenant` + `usageWindow` + skrip migrasi | Skrip migrasi jalan idempotent di data staging, `verify-migration.js` lolos tanpa error |
| 2 | Middleware `checkUsageWindow` + `checkModelTier` | Test race-condition (dua request bersamaan) tidak meloloskan keduanya saat kuota sisa satu |
| 3 | `AIRouter` + provider chain + failover | Test mock 429 dari provider utama → verifikasi otomatis pindah ke OpenRouter, SSE tidak putus |
| 4 | Integrasi canvas Flowise (CRUD dasar, tanpa eksekusi) | Bisa buat/simpan/lihat graf lewat UI, belum bisa dijalankan |
| 5 | Node executor + eksekusi BullMQ (mulai 2 tipe node) | Satu graf sederhana (llm → httpRequest) berhasil dieksekusi ujung ke ujung |
| 6 | Wizard Simple mode | User bisa selesai satu siklus wizard penuh, hasilnya identik dengan graf yang bisa dibuka di Advanced |
| 7 | RAG pipeline | Upload dokumen → tanya sesuatu yang jawabannya ada di dokumen itu → jawaban benar DAN tenant lain tidak bisa mengaksesnya |
| 8 | SSO JIT provisioning | Login SSO dari IdP mock → user baru otomatis dibuat dengan tenantId benar, tercatat di audit log |
| 9 | Billing + webhook | Simulasi webhook pembayaran sukses → `tenant.plan` berubah, `usageWindow` ter-reset sesuai plan baru |
| 10 | Observability + feature flag | Alert test (trigger manual) benar-benar sampai ke Telegram; feature flag bisa mengaktifkan fitur untuk 1 tenant spesifik tanpa memengaruhi tenant lain |

**Aturan tambahan:** fase 9 (billing) tidak boleh dikerjakan sebelum fase 10 (observability) sudah jalan — ini aturan eksplisit dari rancangan, karena butuh visibilitas begitu ada uang sungguhan berputar.

## Kalau Ragu

Kalau instruksi dari user ambigu, atau kamu menemukan sesuatu di kode nyata yang berbeda dari asumsi dokumen rancangan (misalnya nama file berbeda, struktur folder sudah berubah dari versi yang direview saat perancangan) — **berhenti, laporkan temuannya, tanya sebelum lanjut.** Asumsi yang salah di fase awal akan menumpuk jadi masalah besar di fase-fase berikutnya.

---

## Prompt Pembuka — Fase 1 (siap tempel ke sesi Claude Code pertama)

```
Baca INSTRUKSI PROYEK di CLAUDE.md ini secara penuh dulu, termasuk semua "Aturan Keras."

Kita mulai Fase 1: Skema tenant + usageWindow + skrip migrasi.

1. Baca docs/PRD-AI-Platform-Bagian2-Implementasi.md bagian D.1 dan D.2, dan
   docs/PRD-AI-Platform-Bagian3-Kode.md bagian E.3 untuk skema document.ts dan featureFlag.ts.
2. Buat file schema persis seperti kode di dokumen tersebut, di
   packages/data-schemas/src/schema/ (LibreChat-main/).
3. Buat skrip migrasi sesuai urutan di dokumen Bagian 1 bagian C.2 — pastikan
   tiap skrip idempotent dan punya pasangan rollback.
4. Sebelum menjalankan migrasi ke data apa pun, tunjukkan dulu ke saya rencana
   lengkapnya dan tunggu konfirmasi saya.
5. Setelah saya konfirmasi, jalankan di lingkungan staging dulu — JANGAN ke
   production.
6. Laporkan hasil verify-migration.js secara lengkap sebelum kita anggap
   Fase 1 selesai dan lanjut ke Fase 2.

Kalau ada bagian dari dokumen yang tidak cocok dengan struktur kode LibreChat
yang sebenarnya ada di repo ini sekarang, hentikan dan laporkan ke saya —
jangan diputuskan sendiri.
```
