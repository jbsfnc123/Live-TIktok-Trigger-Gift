# PRD — Pasar Malam: TikTok Live Gift → Mesin Capit (Arsitektur Lokal)

| Item | Nilai |
|---|---|
| Produk | **Pasar Malam** — kumpulan mini-game yang dikendalikan gift penonton TikTok Live |
| Game pertama | **Capit 2D** (claw machine) |
| Versi dokumen | 1.1 — fokus arsitektur & infrastruktur **lokal** |
| Tanggal | 2026-10-18 |
| Lingkungan | Berjalan 100% di **PC streamer** (Windows/macOS/Linux), tanpa server cloud |
| Biaya | **Rp 0** — seluruh teknologi open source / gratis |

---

## 1. Ringkasan

Satu aplikasi lokal bernama **Pasar Malam Server** berjalan di PC streamer, di samping OBS / TikTok LIVE Studio. Aplikasi ini:

1. **Menangkap gift TikTok Live secara otomatis** dari room streamer (tanpa input manual).
2. Mengonversi gift menjadi koin: **1 koin TikTok = 1 percobaan**.
3. Mengantrekan percobaan, **memutuskan hasil (menang/kalah)** dengan logika arcade (peluang 1:100), dan
4. **Menjalankan mesin capit 2D** di halaman overlay yang ditampilkan OBS sebagai Browser Source, secara realtime (< 1 detik dari gift ke reaksi di layar).

Internet hanya dipakai untuk dua hal: siaran live itu sendiri dan koneksi pembaca event TikTok. Semua logika game, antrean, database, dan tampilan berjalan di `localhost`.

### Non-goals
- Tidak ada hadiah uang / cash-out — hadiah bersifat **virtual** (shout-out, badge, leaderboard) agar tidak dikategorikan perjudian oleh kebijakan TikTok.
- Tidak ada server cloud pada MVP (sinkronisasi online = opsional, lihat Roadmap).

---

## 2. Gambaran Arsitektur Lokal

```
                ┌──────────────────────────────┐
                │  TikTok LIVE (room streamer) │
                └──────────────┬───────────────┘
                               │ (1) WebSocket "webcast" (protobuf)
                               │     signature URL dari sign server Euler Stream
                               ▼
┌──────────────────── PC STREAMER : Pasar Malam Server (Node.js 20, 1 proses) ────────────────────┐
│                                                                                                 │
│  [1] GIFT LISTENER ──► [2] EVENT BUS ──► [3] QUEUE MANAGER ──► [4] GAME ENGINE (Claw)           │
│   tiktok-live-          EventEmitter       antrean FIFO,         state machine sesi,            │
│   connector             internal           cap/user, saldo       Payout RNG + pity              │
│   normalisasi,                                   │                     │                        │
│   dedupe, combo                                  ▼                     ▼                        │
│                                         [5] STORAGE  SQLite (better-sqlite3, mode WAL)          │
│                                             plays · audit · saldo · config · msgId              │
│                                                                                                 │
│  [6] HTTP + WEBSOCKET SERVER  (Fastify)   http://127.0.0.1:3000                                 │
│        /overlay  → halaman game Phaser 3     /admin → dashboard     /ws → event realtime        │
└───────────────────────────────┬───────────────────────────────────────────┬─────────────────────┘
                                │ ws://127.0.0.1:3000/ws                    │
                                ▼                                           ▼
             ┌─────────────────────────────────────┐      ┌──────────────────────────────────┐
             │ OBS Browser Source → /overlay       │      │ Browser streamer → /admin        │
             │ (1080×1920, animasi mesin capit)    │      │ status, kontrol, simulator gift  │
             └──────────────────┬──────────────────┘      └──────────────────────────────────┘
                                │ (2) video hasil komposisi OBS
                                ▼
                       Siaran TikTok LIVE (penonton melihat capit bergerak)
```

### 2.1 Tanggung Jawab Modul

| # | Modul | Tanggung jawab |
|---|---|---|
| 1 | **Gift Listener** | Terhubung ke room TikTok Live berdasarkan `@username`, menerima event gift, menormalisasi data, menghapus duplikat, menghitung combo dengan benar. |
| 2 | **Event Bus** | Saluran internal antar-modul (`gift.received` → `coin.credited` → `play.queued` → …). Membuat modul saling lepas: sumber gift bisa diganti tanpa mengubah game. |
| 3 | **Queue Manager** | Mengubah koin menjadi percobaan, menyusun antrean FIFO, membatasi 20 percobaan aktif per user (sisanya jadi saldo), mengaktifkan mode turbo saat antrean panjang. |
| 4 | **Game Engine – Claw Controller** | Mengambil percobaan berikutnya, **memutuskan hasil sebelum animasi** (RNG + pity), membuat "skrip gerakan", mengirimnya ke overlay, menunggu konfirmasi selesai, mencatat hasil. Server = sumber kebenaran tunggal. |
| 5 | **Storage (SQLite)** | Menyimpan antrean & saldo (tahan restart), counter pity, riwayat & audit hasil, konfigurasi, daftar `msgId` yang sudah diproses. File tunggal `data/pasarmalam.db`. |
| 6 | **HTTP + WS Server** | Menyajikan file statis overlay & admin, serta WebSocket untuk broadcast event realtime ke overlay dan menerima perintah dari admin. |
| — | **Overlay (Phaser 3)** | Hanya visualisasi: memainkan animasi capit sesuai skrip dari server, sensor lubang, efek suara, panel antrean & leaderboard. |

### 2.2 Prinsip Desain
- **Satu proses, satu perintah** (`pnpm start`) — mudah dijalankan streamer non-teknis.
- **Server otoritatif** — hasil diputuskan di server, overlay tidak bisa memengaruhi hasil; beberapa overlay (OBS + preview) selalu menampilkan hasil yang sama.
- **Event-driven** — setiap langkah adalah event yang tercatat, sehingga bisa diaudit dan di-debug.
- **Tahan gangguan** — reconnect otomatis, antrean tersimpan di SQLite, tidak ada gift yang hilang atau terhitung ganda.

---

## 3. Cara Kerja: Menangkap Gift Secara Otomatis

### 3.1 Bagaimana koneksi ke TikTok Live terjadi

TikTok tidak menyediakan API publik resmi untuk event gift. Aplikasi memakai library open source **`tiktok-live-connector`** (Node.js), yang meniru cara aplikasi web TikTok menerima event live:

```
 Gift Listener                 TikTok                          Sign server (Euler Stream)
      │  1. GET halaman live @username ─►│                                   │
      │◄─ roomId + status live ──────────│                                   │
      │  2. minta signed WebSocket URL ──────────────────────────────────────►│
      │◄─ URL bertanda tangan ───────────────────────────────────────────────│
      │  3. buka WebSocket webcast ─────►│                                   │
      │◄═ stream pesan protobuf ═════════│  (gift, chat, like, follow, ...)  │
      │  4. decode protobuf → event JS `gift`                                │
```

- **Syarat**: akun `@username` harus **sedang LIVE**. Jika belum live, koneksi ditolak (`user offline`).
- **Tidak perlu login** akun TikTok; cukup username publik.
- Sign server Euler Stream gratis (dengan rate limit); API key gratis opsional untuk kuota lebih longgar.

### 3.2 Siklus hidup koneksi (otomatis)

```
 STARTING ─► WAITING_FOR_LIVE ──(live terdeteksi)──► CONNECTING ─► CONNECTED
                 ▲   (cek tiap 30 dtk)                                │
                 │                                                    │ putus / error
                 └──── stream berakhir ◄── RECONNECTING (backoff 1s→2s→4s… maks 30s)
```
- Saat aplikasi dinyalakan sebelum live, listener **menunggu** dan mengecek status tiap 30 detik, lalu otomatis terhubung begitu streamer mulai live.
- Setiap perubahan status dikirim sebagai event `listener.status` ke dashboard admin (indikator hijau/kuning/merah).

### 3.3 Normalisasi & penghitungan koin

Payload gift mentah dari library dinormalisasi ke bentuk internal:

```ts
type GiftEvent = {
  msgId: string;        // ID unik pesan TikTok → kunci idempotensi
  userId: string;
  uniqueId: string;     // @handle
  nickname: string;
  avatarUrl: string;
  giftId: number;
  giftName: string;     // "Rose"
  diamondCount: number; // nilai koin per 1 gift
  repeatCount: number;  // jumlah combo saat ini
  repeatEnd: boolean;   // combo selesai?
  streakable: boolean;  // giftType === 1
};
```
> Nama field mentah berbeda antar versi library; pemetaan dilakukan di satu file adapter (`sources/tiktok.ts`) dan versi library di-**pin**.

**Masalah combo**: gift streakable (mis. Rose ×10) memancarkan *banyak* event selama combo berjalan dengan `repeatCount` yang terus naik (1, 2, 3 … 10). Jika setiap event dihitung penuh, koin akan terhitung berlipat.

**Solusi — hitung delta secara langsung** (respons cepat, tidak menunggu combo selesai):

```ts
// key = userId + giftId + groupId combo
const last = streak.get(key) ?? 0;
const delta = ev.repeatCount - last;          // hanya tambahan baru
if (delta > 0) credit(ev.userId, delta * ev.diamondCount);
streak.set(key, ev.repeatCount);
if (ev.repeatEnd || !ev.streakable) streak.delete(key);
```
- Gift non-streakable (gift mahal) dihitung sekali: `diamondCount × repeatCount`.
- **Idempotensi**: `msgId` yang sudah diproses disimpan di SQLite (24 jam) → event ganda saat reconnect diabaikan.
- **Konversi**: `koin → percobaan` dengan rasio 1:1, lalu event `coin.credited { userId, coins }` dipancarkan ke Event Bus.

### 3.4 Sumber gift yang dapat diganti (`GiftSource`)

```ts
interface GiftSource {
  start(): Promise<void>;
  stop(): Promise<void>;
  on(event: "gift", cb: (g: GiftEvent) => void): void;
  on(event: "status", cb: (s: ListenerStatus) => void): void;
}
```
Implementasi: `TikTokConnectorSource` (default), `SimulatorSource` (testing tanpa live), dan cadangan bila library rusak, mis. menerima webhook lokal dari aplikasi pihak ketiga (TikFinity) di `POST /hooks/gift`.

---

## 4. Cara Kerja: Menjalankan Mesin Capit

### 4.1 Alur end-to-end (dari gift sampai hasil)

```
Penonton      TikTok    Gift Listener   Queue Mgr    Claw Controller        Overlay (OBS)
   │ kirim Rose ×3 │          │             │               │                     │
   │──────────────►│ webcast  │             │               │                     │
   │               │─────────►│ normalisasi │               │                     │
   │               │          │ +3 koin ───►│ +3 percobaan  │                     │
   │               │          │             │── play.queued ┼────────────────────►│ "Budi +3"
   │               │          │             │   (giliran)   │                     │
   │               │          │             │──────────────►│ decideOutcome()     │
   │               │          │             │               │ (RNG+pity, audit)   │
   │               │          │             │               │── play.started ────►│ animasi
   │               │          │             │               │   {outcome, script} │ ~8–10 dtk
   │               │          │             │               │◄──── play.ack ──────│ sensor lubang
   │               │          │             │               │── claw.result ─────►│ MENANG/KALAH
   │               │          │             │◄── next ──────│                     │
```

### 4.2 Tahapan permainan

| Tahap | Apa yang terjadi |
|---|---|
| **Start / Insert Coin** | Giliran user tiba → nama & avatar tampil, animasi koin masuk. 1 koin = 1 sesi capit. |
| **Positioning** | Capit bergerak horizontal ke `targetX` yang **acak setiap sesi** (dari server), dengan gerakan "ragu" agar natural. |
| **Grab & Retract** | Capit turun ke dasar, menutup cakar, mencengkeram **objek terdekat**, lalu ditarik ke atas. |
| **Payout (Grip Strength)** | **Weak Grip (default)**: objek terangkat lalu cengkeraman sengaja dikendurkan di tengah perjalanan. **Strong Grip (win)**: cengkeraman ditahan penuh sampai di atas lubang hadiah. |
| **Return to Chute** | Capit kembali otomatis ke titik awal di atas lubang hadiah. |
| **Release & Evaluation** | Cakar membuka. Hadiah melewati **sensor lubang** → **MENANG**. Hadiah lepas sebelum lubang atau capit kosong → **KALAH**. |

### 4.3 State machine (dijalankan di overlay, dikendalikan skrip server)

```
IDLE → COIN_INSERTED → POSITIONING → DESCEND → GRAB → RETRACT
          ┌───────────────────────────────────────────┴──────────────┐
     grip = weak                                               grip = strong
   DROP_MIDWAY (lepas di dropAt 30–80%)                         HOLD
          └──────────────► RETURN_TO_CHUTE ◄─────────────────────────┘
                                 → RELEASE → EVALUATE (sensor) → RESULT → IDLE
```

### 4.4 Skrip gerakan dari server

Server memutuskan hasil lebih dulu, lalu mengirim skrip deterministik. Overlay menjalankan fisika Matter.js dengan `seed` yang sama sehingga animasi konsisten di semua tampilan.

```json
{
  "type": "play.started",
  "data": {
    "playId": "p_000123",
    "user": { "nickname": "Budi", "avatarUrl": "..." },
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

Variasi kalah (dipilih acak, bobot dapat diatur) agar tidak monoton:
- `empty` — capit meleset, tidak dapat objek (≈ 25%)
- `dropped_midway` — objek terangkat lalu jatuh di perjalanan (≈ 65%)
- `near_miss` — objek jatuh di bibir lubang lalu memantul keluar (≈ 10%)

Untuk `outcome = win`, objek dikunci ke cakar (constraint fisika) sampai RELEASE, lalu jatuh melewati **sensor lubang** (Matter sensor body) → overlay mengirim `play.ack { sensorTriggered: true }`. Bila terjadi selisih, **hasil server tetap berlaku** dan selisih dicatat untuk debugging.

### 4.5 Pengaturan tempo
- Durasi normal **8–10 detik** per percobaan.
- **Turbo**: antrean > 30 percobaan → animasi 2× lebih cepat (≈ 4–5 detik).
- Percobaan beruntun milik user yang sama digabung dalam satu sesi (intro tidak diulang).
- **Watchdog**: jika `play.ack` tidak datang dalam 20 detik (OBS ditutup/overlay crash), server menyelesaikan play memakai hasil yang sudah diputuskan dan lanjut ke antrean berikutnya. Jika tidak ada overlay terhubung, antrean di-**pause** otomatis agar penonton tidak "kehilangan" giliran tanpa terlihat.

---

## 5. Payout Logic — Hybrid RNG + Pity (Peluang 1:100)

| Parameter | Default | Keterangan |
|---|---|---|
| `baseWinChance` | **0,6%** | Peluang strong grip per percobaan |
| `pityThreshold` | **150** | Percobaan ke-150 tanpa pemenang → dijamin menang |
| Peluang efektif | **≈ 1 : 99** | `E = (1 − 0,994^150) / 0,006 ≈ 99,1` percobaan per kemenangan |
| Lingkup pity | per room (global) | Bukan per user |

```ts
function decideOutcome(state: PayoutState, cfg: PayoutConfig) {
  state.sinceLastWin += 1;
  const roll = cryptoRandomFloat();                 // crypto.randomInt / getRandomValues
  const pity = state.sinceLastWin >= cfg.pityThreshold;
  const win = pity || roll < cfg.baseWinChance;
  if (win) state.sinceLastWin = 0;
  return { win, roll, pity, sinceLastWin: state.sinceLastWin };
}
```
- `sinceLastWin` disimpan di SQLite → tidak ter-reset saat aplikasi restart.
- Setiap keputusan dicatat di tabel `audit` (`playId, userId, roll, pity, sinceLastWin, outcome, configVersion, ts`).
- Kemenangan terasa acak (bisa terjadi di percobaan ke-3), tetapi tidak pernah lebih dari 150 percobaan tanpa pemenang (≈ 41% siklus berakhir lewat pity).
- Unit test: simulasi **1.000.000 percobaan** → rata-rata harus 1:95–1:105 dan tidak ada gap > 150.

---

## 6. Aturan Gift → Percobaan & Antrean

| Aturan | Default |
|---|---|
| Konversi | `percobaan = diamondCount × jumlah gift (delta combo)` |
| Gift dihitung | Semua gift (whitelist/blacklist gift ID dapat diatur) |
| Cap per user | 20 percobaan aktif di antrean; kelebihan menjadi **saldo** dan otomatis masuk saat slot tersedia |
| Batas antrean | 500 percobaan (sisanya tetap tersimpan sebagai saldo) |
| Urutan | FIFO; round-robin antar user saat turbo agar adil |
| Like/follow/share | Hanya efek visual, tidak memberi percobaan |

---

## 7. Kontrak Event WebSocket Lokal (`ws://127.0.0.1:3000/ws`)

Amplop pesan: `{ "v": 1, "type": "...", "id": "...", "ts": 1760000000000, "data": {} }`

| Arah | Event | Isi / kegunaan |
|---|---|---|
| Server → semua | `room.state` | Snapshot (status, antrean, play aktif, leaderboard) saat client connect/reconnect |
| Server → semua | `listener.status` | `waiting_for_live` / `connected` / `reconnecting` / `error` |
| Server → semua | `gift.received` | Ticker "Budi mengirim Rose ×3" |
| Server → semua | `play.queued` | Penambahan percobaan & posisi antrean |
| Server → overlay | `play.started` | `outcome` + `script` (lihat 4.4) |
| Overlay → server | `play.ack` | Animasi selesai + `sensorTriggered` |
| Server → semua | `claw.result` | `win`/`lose`, `reason`, `prizeId` |
| Admin → server | `admin.*` | `pause`, `resume`, `skip`, `queue.clear`, `config.update`, `gift.simulate` (wajib token admin) |

Semua pesan masuk divalidasi dengan **zod**; skema berada di `packages/shared` dan dipakai bersama server, overlay, dan admin.

---

## 8. Tech Stack

| Lapisan | Teknologi | Alasan |
|---|---|---|
| Runtime | **Node.js 20 LTS** + **TypeScript** | Ekosistem library TikTok Live paling matang ada di Node; satu bahasa untuk server & client |
| Penangkap gift | **tiktok-live-connector** (versi di-pin) | Library open source paling aktif untuk event TikTok Live (gift, combo, chat, like) |
| Sign server | **Euler Stream** (free tier, bawaan library) | Diperlukan library untuk membuat URL WebSocket bertanda tangan |
| Web server | **Fastify** + **@fastify/websocket** + **@fastify/static** | Ringan, cepat, satu port untuk HTTP & WebSocket |
| Database | **SQLite** via **better-sqlite3** (mode WAL) | Tanpa instalasi server DB, satu file, sinkron & sangat cepat, tahan restart |
| Validasi | **zod** | Skema event bersama, mencegah data rusak |
| Game engine | **Phaser 3** + **Matter.js** (bawaan Phaser) | Engine 2D matang; fisika untuk cakar, objek, sensor lubang; lancar di OBS Browser Source (Chromium) |
| Admin UI | **Vite** + Preact (atau HTML + Alpine.js) | Ringan, build statis disajikan Fastify |
| Build & tooling | **pnpm** workspace, **Vite**, **tsx**, **Vitest**, ESLint/Prettier | Monorepo sederhana, test cepat |
| Logging | **pino** (bawaan Fastify) → file `logs/` | Log terstruktur untuk debugging live |
| Distribusi (opsional) | `start.bat` / `start.sh`; Node SEA atau `pkg` untuk satu file `.exe` | Sekali klik untuk streamer |
| Aset | Kenney.nl / OpenGameArt (CC0), audio freesound (CC0) | Gratis untuk komersial |

---

## 9. Infrastruktur Lokal & Kebutuhan PC

| Kebutuhan | Spesifikasi |
|---|---|
| OS | Windows 10/11, macOS 12+, atau Linux |
| CPU / RAM | Setara Intel i5 gen-8 / Ryzen 5, RAM 8 GB (aplikasi ini memakai ±150 MB RAM; sisanya untuk OBS & encoding) |
| Software | Node.js 20 LTS, pnpm, OBS Studio (atau TikTok LIVE Studio yang mendukung sumber browser/window capture) |
| Jaringan | Internet stabil (upload ≥ 5 Mbps untuk siaran); koneksi keluar HTTPS/WSS ke TikTok & Euler Stream |
| Port lokal | `3000` (dapat diubah), **bind ke 127.0.0.1** — tidak terbuka ke jaringan luar, tidak perlu setting firewall/port forwarding |
| Penyimpanan | ± 200 MB (aplikasi + aset + database) |

| Komponen | Lokal / Internet | Biaya |
|---|---|---|
| Logika game, antrean, payout, database, overlay, admin | Lokal | Gratis |
| Koneksi pembaca event TikTok Live | Internet | Gratis |
| Sign server Euler Stream | Internet | Gratis (rate limit) |
| Siaran live (OBS → TikTok) | Internet | Gratis |

---

## 10. Cara Menjalankan (Runbook Streamer)

**Sekali saja (instalasi):**
```bash
# 1. Install Node.js 20 LTS (nodejs.org) lalu pnpm
npm install -g pnpm
# 2. Ambil proyek & install dependency
git clone <repo> pasar-malam && cd pasar-malam
pnpm install
# 3. Salin konfigurasi
cp .env.example .env
```

**Isi `.env`:**
```env
TIKTOK_USERNAME=akun_streamer      # tanpa @
PORT=3000
ADMIN_TOKEN=ganti-dengan-rahasia
EULER_API_KEY=                     # opsional
GIFT_SOURCE=tiktok                 # tiktok | simulator
```

**Setiap mau live:**
1. Jalankan `pnpm start` (atau klik dua kali `start.bat`).
2. Buka `http://127.0.0.1:3000/admin` → status listener **"Menunggu live"**.
3. Di OBS: *Sources → + → Browser* → URL `http://127.0.0.1:3000/overlay`, ukuran **1080×1920**, centang *Shutdown source when not visible* = **off**.
4. Mulai LIVE di TikTok → listener otomatis berubah **"Terhubung"** dalam ≤ 30 detik.
5. Gift penonton langsung menggerakkan mesin capit.

**Testing tanpa live:**
- `GIFT_SOURCE=simulator pnpm start`, lalu kirim gift palsu dari tombol **Simulator** di `/admin`, atau
- `pnpm simulate --user budi --gift Rose --count 5`.

---

## 11. Struktur Folder & Ekstensibilitas

```
pasar-malam/
├── apps/
│   ├── server/                 # Pasar Malam Server (Node.js)
│   │   ├── src/sources/        # GiftSource: tiktok.ts, simulator.ts, webhook.ts
│   │   ├── src/core/           # event-bus, queue-manager, storage (SQLite)
│   │   ├── src/games/claw/     # ClawController (server side)
│   │   └── src/http/           # Fastify routes + WebSocket hub
│   ├── overlay/                # Phaser 3: shell Pasar Malam + scene per game
│   │   └── src/games/claw/     # ClawScene, state machine, aset
│   └── admin/                  # dashboard
├── packages/shared/            # tipe & skema event (zod), decideOutcome, aturan antrean + test
├── data/                       # pasarmalam.db (dibuat otomatis)
├── .env.example · start.bat · start.sh
└── PRD.md
```

Game Pasar Malam berikutnya cukup mengimplementasikan:
```ts
interface GameModule {
  id: string;                                  // "claw", "lempar-gelang", "tembak-balon"
  costPerPlay: number;                         // default 1 koin
  decide(ctx: DecideContext): PlayScript;      // server: hasil + skrip
  createScene(): Phaser.Scene;                 // overlay: visualisasi
}
```
Gift Listener, Event Bus, antrean, payout, audit, dan WebSocket dipakai ulang tanpa perubahan.

---

## 12. Kebutuhan Non-Fungsional

| Kategori | Target |
|---|---|
| Latency gift → muncul di antrean overlay | < 1 detik (p95) |
| Latency gift → capit mulai bergerak (antrean kosong) | < 1,5 detik (p95) |
| FPS overlay di OBS | ≥ 50 fps |
| Reconnect | Listener & overlay otomatis (backoff maks 30 dtk); overlay meminta `room.state` saat tersambung kembali |
| Akurasi | 0 gift hilang / ganda (delta combo + idempotensi `msgId`) |
| Durabilitas | Antrean, saldo, counter pity bertahan saat aplikasi/PC restart |
| Keamanan | Server hanya bind `127.0.0.1`; perintah admin wajib `ADMIN_TOKEN`; validasi zod di semua input |
| Privasi | Hanya data publik (`userId`, `@handle`, nickname, avatar); retensi 90 hari |

---

## 13. Risiko & Mitigasi

| Risiko | Mitigasi |
|---|---|
| `tiktok-live-connector` rusak saat TikTok mengubah protokol | Pin versi, adapter tunggal `sources/tiktok.ts`, `GiftSource` cadangan (webhook TikFinity), mode simulator |
| Rate limit / gangguan sign server Euler Stream | Satu koneksi per sesi live, backoff, API key gratis |
| PC/OBS mati = game berhenti | Antrean & saldo tersimpan di SQLite, dilanjutkan saat aplikasi hidup kembali; auto-pause bila overlay tidak terhubung |
| Internet putus sesaat | Reconnect otomatis + dedupe `msgId` |
| Kebijakan TikTok terkait konten menyerupai judi | Hadiah virtual saja, tanpa cash-out, disclaimer "hiburan" di overlay |
| Beban CPU mengganggu encoding OBS | Overlay ringan (sprite atlas, 1 scene aktif), batas FPS 60, turbo saat antrean panjang |

---

## 14. Roadmap

| Milestone | Cakupan | Estimasi |
|---|---|---|
| **M0 — Setup** | Monorepo pnpm, Fastify + SQLite + WebSocket hub, `.env`, start script | 1 hari |
| **M1 — Gift Listener** | `TikTokConnectorSource` (auto-wait live, reconnect), normalisasi, delta combo, idempotensi, `SimulatorSource`, uji di live sungguhan | 2–3 hari |
| **M2 — Mesin Capit MVP** | Queue Manager, `decideOutcome` + test 1 juta simulasi, ClawController, overlay Phaser (state machine, fisika, sensor lubang), turbo, watchdog | 5–7 hari |
| **M3 — Admin** | Status listener, pause/resume/skip, konfigurasi payout, simulator, audit & statistik | 2–3 hari |
| **M4 — Uji live & polish** | Uji live 2–3 sesi, tuning animasi & audio, `start.bat`, panduan streamer | 2 hari |
| **M5 — Opsional** | Game Pasar Malam berikutnya via `GameModule`; sinkronisasi leaderboard online (mis. Cloudflare/Supabase free tier) | iteratif |

---

## 15. Acceptance Criteria (MVP)

1. Menjalankan `pnpm start` sebelum live → status "Menunggu live"; setelah live dimulai, listener terhubung otomatis ≤ 30 detik tanpa tindakan manual.
2. Gift Rose (1 koin) menambah tepat **1** percobaan; Rose combo ×10 menambah tepat **10** (bukan 55); gift 5 koin ×3 menambah **15** (dengan cap 20 & saldo).
3. Setiap percobaan menjalankan urutan lengkap: insert coin → positioning acak → turun → grab → retract → (drop / hold) → return to chute → release → evaluasi sensor.
4. Weak grip menjatuhkan objek sebelum lubang; strong grip membawa objek melewati sensor lubang → overlay menampilkan MENANG.
5. Hasil MENANG hanya terjadi bila server memutuskan `win`; semua overlay yang terhubung menampilkan hasil identik.
6. Test simulasi 1.000.000 percobaan: rata-rata 1:95–1:105, tidak ada gap > 150.
7. Internet diputus 10 detik saat live → listener tersambung kembali otomatis, tidak ada gift ganda.
8. Aplikasi ditutup lalu dibuka kembali → antrean, saldo, dan counter pity tetap utuh.
