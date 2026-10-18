# PRD — Pasar Malam: TikTok Live Gift → Realtime Game

| Item | Nilai |
|---|---|
| Produk | **Pasar Malam** — kumpulan mini-game interaktif yang dikendalikan gift penonton TikTok Live |
| Game pertama | **Capit 2D** (claw machine) |
| Versi dokumen | 1.0 (draft) |
| Tanggal | 2026-10-18 |
| Status | Disetujui untuk MVP |
| Prinsip infrastruktur | **100% free tier** — tanpa biaya bulanan |

---

## 1. Ringkasan

Pasar Malam adalah overlay game yang tampil di siaran TikTok Live (via OBS / TikTok LIVE Studio). Setiap gift dari penonton dikonversi menjadi **koin game**: **1 koin TikTok = 1 percobaan**. Game merespons secara realtime (< 2 detik dari gift ke animasi), menampilkan nama/avatar gifter, dan menyimpan riwayat serta leaderboard.

Game pertama adalah **Capit 2D**: capit turun, mencengkeram hadiah, lalu membawanya ke lubang hadiah. Logika payout mengikuti standar arcade (weak grip / strong grip) dengan peluang menang efektif **1:100**.

### 1.1 Tujuan
1. Meningkatkan jumlah gift & durasi tonton dengan interaksi langsung yang terlihat di layar.
2. Arsitektur **modular**: game baru di Pasar Malam cukup menambah modul tanpa mengubah pipeline gift.
3. Biaya operasional **Rp 0** dengan memanfaatkan free tier (Cloudflare, GitHub).
4. Hasil permainan **adil, dapat diaudit, dan dapat dikonfigurasi**.

### 1.2 Non-goals (di luar cakupan)
- **Tidak ada hadiah uang/barang bernilai tunai atau cash-out.** Hadiah bersifat virtual (shout-out, badge, posisi leaderboard, request lagu, dll.) agar tidak dikategorikan sebagai perjudian dan tetap patuh pada kebijakan komunitas TikTok.
- Tidak membangun aplikasi mobile; client game berbasis web.
- Tidak menggunakan API resmi TikTok berbayar/terbatas (belum tersedia publik untuk event gift).

---

## 2. Persona & User Stories

| Persona | Kebutuhan |
|---|---|
| **Streamer** (pemilik akun) | Menyalakan game dengan cepat, overlay stabil, bisa pause/atur win-rate. |
| **Penonton / Gifter** | Mengirim gift → langsung melihat giliran & hasil dengan namanya di layar. |
| **Admin / Moderator** | Memantau koneksi, antrean, audit hasil, simulasi gift untuk testing. |

User stories utama:
- **US-01** Sebagai gifter, saat saya mengirim Rose (1 koin), saya mendapat 1 percobaan dan nama saya muncul di antrean.
- **US-02** Sebagai gifter, saat saya mengirim gift 5 koin dengan combo ×3, saya mendapat 15 percobaan (dibatasi cap per user).
- **US-03** Sebagai penonton, saya melihat capit bergerak, mencengkeram, dan hasil MENANG/KALAH dengan jelas.
- **US-04** Sebagai streamer, saya bisa pause game, mengubah win-rate, dan reset antrean dari dashboard.
- **US-05** Sebagai admin, saya bisa mengirim "gift palsu" (simulator) untuk tes tanpa live.
- **US-06** Sebagai streamer, jika koneksi internet putus sesaat, game otomatis tersambung kembali tanpa kehilangan antrean.

---

## 3. Arsitektur Sistem

### 3.1 Diagram

```
┌────────────────┐   Webcast WS (unofficial)   ┌──────────────────────────────────┐
│  TikTok LIVE   │ ──────────────────────────► │  BRIDGE (Node.js) @ PC streamer  │
│ (room streamer)│                             │  - tiktok-live-connector         │
└────────────────┘                             │  - normalisasi & dedupe event    │
                                               │  - buffer offline (SQLite lokal) │
                                               └───────────────┬──────────────────┘
                                                               │ WSS + token HMAC
                                                               ▼
                         ┌──────────────────────────────────────────────────────────┐
                         │  CLOUDFLARE WORKER  (router, auth, REST admin)           │
                         │      └─► DURABLE OBJECT "Room:{streamerId}"              │
                         │           - otoritatif: antrean, RNG, pity, state game   │
                         │           - SQLite storage (state room, idempotency)     │
                         │           - WebSocket Hibernation API (broadcast)        │
                         │      └─► D1 (riwayat play, audit RNG, leaderboard)       │
                         └───────────────┬───────────────────────────┬──────────────┘
                                         │ WSS broadcast             │ HTTPS + WSS
                                         ▼                           ▼
                    ┌────────────────────────────────┐   ┌───────────────────────────┐
                    │ GAME CLIENT (Phaser 3) @ Pages │   │ ADMIN DASHBOARD @ Pages   │
                    │  → OBS Browser Source 1080x1920│   │  dilindungi CF Access     │
                    └────────────────────────────────┘   └───────────────────────────┘
```

### 3.2 Komponen & Tanggung Jawab

| Komponen | Teknologi | Tanggung jawab |
|---|---|---|
| **Bridge** | Node.js 20 LTS, `tiktok-live-connector` (versi di-pin), `ws`, `better-sqlite3` | Terhubung ke room TikTok Live berdasarkan `@username`, menerima event gift/like/follow/chat, menormalisasi ke skema internal, dedupe combo, mengirim ke Room DO. Menyimpan buffer event saat cloud tidak terjangkau lalu replay. |
| **Worker (API gateway)** | Cloudflare Workers (TypeScript) | Routing `/ws/bridge`, `/ws/game`, `/ws/admin`, `/api/*`; verifikasi token; meneruskan ke Durable Object yang tepat. |
| **Room Durable Object** | Cloudflare Durable Objects (SQLite backend) | **Single source of truth** per streamer: antrean play, penentuan hasil (RNG + pity), state machine sesi, broadcast ke semua client. Single-threaded → bebas race condition. |
| **Database** | Cloudflare D1 | Riwayat play, audit log RNG, leaderboard harian/total, konfigurasi game. |
| **Game Client** | Phaser 3 + Matter.js (fisika), Vite, TypeScript | Memvisualisasikan play sesuai instruksi server (hasil sudah ditentukan), animasi capit, efek suara, antrean & leaderboard overlay. |
| **Admin Dashboard** | Vite + Preact/React (ringan), Cloudflare Pages | Status koneksi, konfigurasi win-rate/pity, simulator gift, pause/resume, audit. |
| **Shared package** | TypeScript | Tipe event, skema validasi (zod), logika payout & antrean (diuji unit test, dipakai Worker & client). |

### 3.3 Alasan Pemilihan

- **Bridge di PC streamer**: koneksi ke TikTok Live adalah koneksi persisten (long-lived WebSocket) dari library unofficial. Menjalankannya di PC streamer = gratis, latency rendah, dan menggunakan **IP residensial** yang jauh lebih jarang diblokir TikTok dibanding IP datacenter. Streamer memang sudah menyalakan PC untuk OBS.
- **Durable Objects sebagai hub otoritatif**: hasil ditentukan di server, bukan di browser/bridge, sehingga tidak bisa dimanipulasi dan bisa diaudit. DO menyediakan WebSocket + storage dalam satu unit dengan konsistensi kuat, dan **Hibernation API** membuat koneksi idle tidak menghabiskan kuota durasi.
- **Cloudflare Pages** untuk game & admin: hosting statis gratis tanpa batas bandwidth, CDN global, deploy otomatis dari GitHub.
- **Phaser 3**: engine 2D matang, gratis, performa baik di OBS Browser Source (Chromium).

### 3.4 Mode Operasi

| Mode | Keterangan |
|---|---|
| **Cloud (default)** | Bridge → Worker/DO → Game. Mendukung admin jarak jauh, leaderboard persisten, multi-overlay. |
| **Local fallback** | Jika Cloudflare tidak terjangkau > 30 dtk, Bridge menjalankan hub WS lokal (`ws://localhost:7777`) dengan logika `packages/shared` yang sama; game client otomatis pindah ke endpoint lokal. Hasil disinkronkan ke D1 saat online kembali. |
| **Simulasi** | Tanpa TikTok: event gift berasal dari simulator admin/CLI. Untuk development & QA. |

---

## 4. Infrastruktur Gratis

| Layanan | Kegunaan | Batas free tier (perkiraan) | Mitigasi |
|---|---|---|---|
| Cloudflare Workers Free | API gateway | 100.000 request/hari, 10 ms CPU/request | Event via WebSocket (1 koneksi, bukan request per event). |
| Cloudflare Durable Objects (SQLite) | Room hub & state | Tersedia di plan Free; kuota request & durasi harian | WebSocket Hibernation, batch broadcast tiap 50 ms. |
| Cloudflare D1 | Riwayat, audit, leaderboard | 5 GB storage, 5 juta baris dibaca/hari, 100 ribu tulis/hari | Batch insert per 1 dtk; retensi audit 90 hari; agregasi leaderboard. |
| Cloudflare Pages | Hosting game & admin | Unlimited bandwidth, 500 build/bulan | Build hanya dari branch `main`. |
| Cloudflare Access (Zero Trust Free) | Proteksi admin | ≤ 50 user | Login email OTP. |
| GitHub + GitHub Actions | Repo, CI test, deploy `wrangler` | Unlimited untuk repo publik; 2.000 menit/bulan untuk privat | Cache pnpm, jalankan test hanya pada path yang berubah. |
| Euler Stream (sign server untuk tiktok-live-connector) | Signature koneksi webcast | Free tier dengan rate limit | Koneksi sekali per sesi live; backoff saat limit; opsi API key gratis. |
| Workers Logs / Sentry Free | Logging & error tracking | Sentry 5k error/bulan | Sampling error di client. |
| UptimeRobot Free | Monitor endpoint `/health` | 50 monitor, interval 5 menit | — |

> Catatan: angka kuota dapat berubah oleh penyedia; verifikasi ulang saat setup (milestone M0).

Estimasi beban: live 4 jam dengan 2.000 gift → ~2.000 pesan WS masuk, ~10.000 broadcast, ~2.000 baris D1 — jauh di bawah semua kuota.

---

## 5. Kontrak Event (Protocol)

Semua pesan berupa JSON dengan amplop standar:

```json
{ "v": 1, "type": "<event.type>", "id": "<uuid/msgId>", "ts": 1760000000000, "data": { } }
```

### 5.1 Bridge → Room

**`gift.received`**
```json
{
  "type": "gift.received",
  "id": "tt-7412345678901234567",
  "data": {
    "userId": "6812345",
    "uniqueId": "budi_santoso",
    "nickname": "Budi",
    "avatarUrl": "https://...",
    "giftId": 5655,
    "giftName": "Rose",
    "diamondCount": 1,
    "repeatCount": 3,
    "coins": 3
  }
}
```
Aturan normalisasi di Bridge:
- Gift streakable (combo) memancarkan banyak event selama combo berlangsung. Bridge **hanya menghitung delta** `repeatCount` atau menunggu `repeatEnd = true`, sehingga koin tidak terhitung ganda.
- `coins = diamondCount × repeatCount(delta)`.
- `id` = `msgId` dari TikTok → **idempotency key**; Room menolak `id` yang sudah diproses (disimpan 24 jam).

Event lain (opsional, untuk efek visual): `like.received`, `follow.received`, `chat.received`, `bridge.status`.

### 5.2 Room → Game/Admin

| Event | Data penting | Kegunaan |
|---|---|---|
| `room.state` | status (`running`/`paused`), antrean ringkas, play aktif, leaderboard top 10, config publik | Snapshot saat client connect/reconnect |
| `play.queued` | `playId`, user, posisi antrean, jumlah percobaan ditambahkan | Notifikasi "Budi +3 percobaan" |
| `play.started` | `playId`, user, `game: "claw"`, **`script`** (lihat 6.3), `outcome`, `turbo` | Client menjalankan animasi sesuai skrip |
| `claw.result` | `playId`, `outcome` (`win`/`lose`), `prizeId`, `reason` (`dropped_midway`/`empty`/`chute_sensor`) | Overlay MENANG/KALAH, update leaderboard |
| `play.ack` (Game → Room) | `playId`, `sensorTriggered` | Konfirmasi animasi selesai & sensor lubang; memicu play berikutnya |

### 5.3 Admin → Room (REST `/api/rooms/:id/*` atau WS admin)
`config.update`, `game.pause`, `game.resume`, `queue.clear`, `gift.simulate`, `pity.reset`.

---

## 6. Spesifikasi Game #1 — Capit 2D

### 6.1 Alur Permainan

| Tahap | Deskripsi |
|---|---|
| **Start / Insert Coin** | 1 koin gift = 1 koin game. Saat giliran user tiba, sesi dimulai: tampil nama + avatar, animasi koin masuk. |
| **Positioning** | Capit bergerak horizontal ke posisi X **random per sesi** (ditentukan server via `script.targetX`) dengan sedikit gerakan "ragu" agar terlihat natural. |
| **Grab & Retract** | Capit turun ke dasar, menutup cakar, mencengkeram **objek terdekat** dari titik turun, lalu ditarik kembali ke atas. |
| **Payout Rate Logic** | **Weak Grip (default)**: objek terangkat, lalu cengkeraman sengaja dikendurkan di tengah perjalanan → objek jatuh. **Strong Grip (win state)**: cengkeraman penuh dipertahankan sampai di atas lubang hadiah. |
| **Return to Chute** | Capit bergerak otomatis kembali ke titik awal di atas lubang hadiah. |
| **Release & Evaluation** | Cakar membuka. Hadiah jatuh melewati **sensor lubang** → **MENANG**. Hadiah lepas sebelum lubang atau capit kosong → **KALAH**. |

### 6.2 State Machine

```
IDLE
 └─(play.started)─► COIN_INSERTED
     └─► POSITIONING (gerak ke targetX random)
         └─► DESCEND
             └─► GRAB (tutup cakar, attach objek terdekat / kosong)
                 └─► RETRACT
                     ├─[grip = weak]──► DROP_MIDWAY (lepas di dropAt 30–80% perjalanan)
                     │                     └─► RETURN_TO_CHUTE (kosong)
                     └─[grip = strong]─► HOLD
                                           └─► RETURN_TO_CHUTE (membawa hadiah)
                                                 └─► RELEASE
                                                     └─► EVALUATE (sensor lubang)
                                                         └─► RESULT (overlay 2 dtk)
                                                             └─► IDLE / play berikutnya
```

Variasi kalah agar tidak monoton (dipilih server secara acak, bobot configurable):
- `empty` — cakar meleset/tidak mendapat objek (≈ 25%).
- `dropped_midway` — objek terangkat lalu jatuh saat retract/perjalanan (≈ 65%).
- `near_miss` — objek jatuh tepat di bibir lubang dan memantul keluar (≈ 10%) — meningkatkan ketegangan.

### 6.3 Hasil Ditentukan Server, Client Hanya Memvisualisasikan

Agar adil, konsisten di semua overlay, dan tidak bisa dicurangi, **Room DO memutuskan outcome sebelum animasi** dan mengirim skrip deterministik:

```json
{
  "type": "play.started",
  "data": {
    "playId": "p_01J...",
    "user": { "nickname": "Budi", "avatarUrl": "..." },
    "game": "claw",
    "outcome": "lose",
    "script": {
      "seed": 918273645,
      "targetX": 0.62,
      "grip": "weak",
      "loseVariant": "dropped_midway",
      "dropAt": 0.47,
      "prizeId": "boneka_kucing"
    },
    "turbo": false
  }
}
```
- Fisika (Matter.js) dijalankan dengan `seed` sehingga animasi sama di semua client.
- Untuk `outcome = win`, objek dikunci ke cakar (constraint) sampai RELEASE, lalu diarahkan ke lubang; **sensor lubang** (Matter sensor body) mengirim `play.ack { sensorTriggered: true }` sebagai verifikasi. Jika terjadi mismatch (sangat jarang), **hasil server tetap berlaku** dan mismatch dicatat untuk debugging.

### 6.4 Durasi & Antrean
- Target durasi per percobaan **8–10 detik** (normal).
- **Mode turbo** otomatis jika antrean > 30 percobaan: animasi dipercepat 2× (≈ 4–5 detik).
- Percobaan beruntun dari user yang sama dibundel dalam satu "sesi" (tanpa jeda intro ulang).
- Watchdog: jika `play.ack` tidak diterima dalam 20 dtk (overlay tertutup/crash), Room menyelesaikan play secara otomatis berdasarkan outcome server dan melanjutkan antrean.

### 6.5 Aset & Tampilan
- Resolusi canvas **1080×1920 (portrait)**, latar transparan opsional untuk OBS.
- Elemen UI: mesin capit bertema pasar malam (lampu, warna neon), panel antrean (5 teratas), ticker "Budi +3 percobaan", leaderboard pemenang, counter "percobaan sejak menang terakhir" (opsional, bisa disembunyikan).
- Aset gratis/CC0 (Kenney.nl, OpenGameArt) atau dibuat sendiri; audio CC0 (freesound).

---

## 7. Payout Logic — Hybrid RNG + Pity

### 7.1 Parameter Default

| Parameter | Default | Keterangan |
|---|---|---|
| `baseWinChance` | **0,6%** (0.006) | Peluang strong grip per percobaan |
| `pityThreshold` | **150** | Percobaan ke-150 tanpa menang → dijamin menang |
| Peluang efektif | **≈ 1 : 99** (1,01%) | Memenuhi target 1:100 |
| `pityScope` | `room` | Counter pity berlaku global per room (bukan per user) |

### 7.2 Perhitungan

Dengan peluang dasar *p* dan jaminan menang pada percobaan ke-*N*, rata-rata percobaan per kemenangan:

```
E = (1 − (1 − p)^N) / p
  = (1 − 0,994^150) / 0,006
  ≈ 99,1 percobaan per kemenangan   →  ≈ 1 : 100
```
Peluang sebuah siklus mencapai pity (149 kali kalah berturut) ≈ 40,8%; artinya kemenangan terasa acak (bisa menang di percobaan ke-3), namun tidak pernah lebih dari 150 percobaan tanpa pemenang.

### 7.3 Algoritma (di Room DO)

```ts
function decideOutcome(state: PayoutState, cfg: PayoutConfig): Decision {
  state.sinceLastWin += 1;
  const roll = cryptoRandomFloat(); // crypto.getRandomValues → [0,1)
  const pity = state.sinceLastWin >= cfg.pityThreshold;
  const win = pity || roll < cfg.baseWinChance;
  if (win) state.sinceLastWin = 0;
  return { win, roll, pity, sinceLastWin: state.sinceLastWin };
}
```
- RNG: `crypto.getRandomValues` (CSPRNG) di server.
- State `sinceLastWin` disimpan di SQLite DO (bertahan saat restart/deploy).
- Setiap keputusan dicatat ke **audit log D1**: `playId, userId, roll, pity, sinceLastWin, outcome, configVersion, ts`.
- Semua parameter dapat diubah lewat admin (perubahan diberi `configVersion` dan tercatat).
- Fungsi ini berada di `packages/shared` dan diuji dengan **simulasi 1.000.000 percobaan** di unit test (assert rata-rata 1:95–1:105, maksimum gap = 150).

---

## 8. Aturan Gift → Percobaan & Antrean

| Aturan | Default |
|---|---|
| Konversi | `percobaan = diamondCount × repeatCount` (1 koin = 1 percobaan) |
| Gift yang dihitung | Semua gift (whitelist/blacklist gift ID dapat dikonfigurasi) |
| Cap per user di antrean | 20 percobaan aktif; kelebihan disimpan sebagai **saldo kredit** user dan otomatis masuk antrean saat slot tersedia (tidak hangus) |
| Batas total antrean | 500 percobaan; di atas itu kredit tetap tersimpan sebagai saldo |
| Urutan | FIFO per sesi user (round-robin antar user bila turbo aktif agar adil) |
| Masa berlaku saldo | Sampai live berakhir + 24 jam (dapat dikonfigurasi) |

Like/follow/share **tidak** memberi percobaan (hanya efek visual), agar mekanisme tetap berbasis gift.

---

## 9. Admin Dashboard

Fitur MVP:
1. **Status**: koneksi Bridge ↔ TikTok, Bridge ↔ Cloud, jumlah overlay terhubung, latency.
2. **Kontrol**: Pause / Resume, Clear antrean, Skip play aktif, Toggle turbo.
3. **Konfigurasi**: `baseWinChance`, `pityThreshold`, cap antrean, bobot varian kalah, daftar hadiah virtual.
4. **Simulator gift**: kirim gift palsu (nama, gift, combo) untuk testing.
5. **Audit & statistik**: total percobaan, total menang, rasio aktual vs target, riwayat 100 play terakhir, export CSV.
6. **Leaderboard**: top gifter & top pemenang (harian/total).

Akses dilindungi **Cloudflare Access** (OTP email) + token admin.

---

## 10. Kebutuhan Non-Fungsional

| Kategori | Target |
|---|---|
| Latency gift → `play.queued` tampil | < 1 detik (p95) |
| Latency gift → animasi mulai (antrean kosong) | < 2 detik (p95) |
| FPS overlay di OBS | ≥ 50 fps pada PC streamer menengah |
| Ketersediaan selama live | ≥ 99% (dengan local fallback) |
| Reconnect | Bridge & client: exponential backoff 1s → 2s → 4s … maks 30s; client meminta `room.state` saat reconnect |
| Idempotensi | Tidak ada gift dihitung ganda (dedupe `msgId` + delta combo) |
| Durabilitas | Antrean & saldo kredit bertahan saat deploy/restart (SQLite DO) |
| Keamanan | Token Bridge (HMAC, rotasi via admin), token overlay read-only, CORS terbatas ke domain Pages, admin via CF Access, validasi skema (zod) di semua input |
| Privasi | Hanya menyimpan `userId`, `uniqueId`, `nickname`, `avatarUrl` publik; retensi 90 hari |
| Observability | Log terstruktur (Workers Logs), endpoint `/health`, metrik ringkas di admin |

---

## 11. Struktur Repository (Monorepo pnpm)

```
Live-TIktok-Trigger-Gift/
├── apps/
│   ├── bridge/          # Node.js: tiktok-live-connector → Room DO; local fallback hub; CLI simulator
│   ├── worker/          # Cloudflare Worker + Durable Object Room + D1 migrations (wrangler.toml)
│   ├── game/            # Phaser 3 overlay (Pasar Malam shell + modul game)
│   │   └── src/games/claw/   # Capit 2D
│   └── admin/           # Dashboard admin
├── packages/
│   └── shared/          # tipe event, skema zod, payout (RNG+pity), aturan antrean, tests
├── .github/workflows/   # CI (lint, test) & deploy (wrangler, Pages)
├── PRD.md
└── README.md
```

### 11.1 Interface Modul Game (ekstensibilitas Pasar Malam)

```ts
interface GameModule {
  id: string;                         // "claw", "lempar-gelang", "tembak-balon", ...
  decide(ctx: DecideContext): PlayScript;      // server: tentukan outcome + skrip (dipakai Room DO)
  createScene(): Phaser.Scene;                 // client: visualisasi
  costPerPlay: number;                         // default 1 koin
}
```
Game baru cukup mengimplementasikan `GameModule`; pipeline gift, antrean, payout, dan audit dipakai ulang.

---

## 12. Roadmap & Milestone

| Milestone | Cakupan | Estimasi |
|---|---|---|
| **M0 — Setup** | Monorepo, akun Cloudflare (Workers, DO, D1, Pages, Access), CI/CD GitHub Actions, verifikasi kuota | 1–2 hari |
| **M1 — Pipeline gift** | Bridge (connect, normalisasi, dedupe combo, buffer offline), Worker + Room DO (idempotensi, antrean, broadcast), simulator gift | 3–4 hari |
| **M2 — Capit 2D MVP** | Payout RNG+pity + test simulasi, state machine & animasi Phaser, sensor lubang, overlay antrean, mode turbo, watchdog | 5–7 hari |
| **M3 — Admin & data** | Dashboard, konfigurasi, audit & statistik, leaderboard D1, local fallback mode | 3–4 hari |
| **M4 — Uji live & polish** | Uji di live sungguhan, tuning animasi & audio, dokumentasi setup streamer (README) | 2–3 hari |
| **M5 — Game Pasar Malam berikutnya** | Modul game baru via `GameModule` (mis. lempar gelang, tembak balon, roda putar) | iteratif |

---

## 13. Risiko & Mitigasi

| Risiko | Dampak | Mitigasi |
|---|---|---|
| Library unofficial (`tiktok-live-connector`) rusak saat TikTok mengubah protokol | Gift tidak terbaca | Pin versi, abstraksi `GiftSource` (mudah ganti ke sumber lain, mis. TikFinity/TikTok LIVE Studio webhook), mode simulasi, monitor rilis library |
| Rate limit sign server (Euler Stream) | Gagal connect | Satu koneksi per sesi, backoff, API key free tier |
| Pemblokiran IP oleh TikTok | Gagal connect | Bridge berjalan di IP residensial streamer |
| Kebijakan TikTok terkait konten menyerupai judi | Akun kena sanksi | Hadiah hanya virtual, tanpa cash-out, tanpa istilah taruhan; tampilkan disclaimer "hiburan, hadiah virtual" |
| Kuota free tier terlampaui | Layanan terhenti | Hibernation, batching, monitoring kuota di admin, local fallback |
| Overlay crash/ditutup saat play | Antrean macet | Watchdog 20 dtk, outcome server tetap berlaku |
| Gift ganda/terlewat saat reconnect | Ketidakadilan | Idempotensi `msgId`, buffer offline Bridge + replay |

---

## 14. Metrik Sukses

- Rasio percobaan:menang aktual dalam rentang **1:95 – 1:105** (rolling 10.000 percobaan).
- 0 kasus gift dihitung ganda / hilang pada uji 1.000 gift simulasi + 1 sesi live.
- p95 latency gift → animasi < 2 detik.
- Biaya infrastruktur bulanan **Rp 0**.
- Peningkatan gift per jam dibanding live tanpa game (diukur oleh streamer).

## 15. Acceptance Criteria — Capit 2D MVP

1. Gift 1 koin menambah tepat 1 percobaan; gift 5 koin combo ×3 menambah 15 (dengan cap & saldo kredit).
2. Setiap percobaan menjalankan urutan: insert coin → positioning random → turun → grab → retract → (drop / hold) → return to chute → release → evaluasi.
3. Weak grip menjatuhkan objek sebelum lubang; strong grip membawa objek hingga melewati sensor lubang.
4. Hasil MENANG hanya terjadi bila server memutuskan `win`; seluruh overlay yang terhubung menampilkan hasil yang identik.
5. Unit test simulasi 1.000.000 percobaan: rata-rata 1:95–1:105 dan tidak ada gap > 150.
6. Admin dapat pause/resume, mengubah win-rate, dan mengirim gift simulasi; perubahan tercatat di audit.
7. Bridge/overlay yang terputus tersambung kembali otomatis tanpa kehilangan antrean.
