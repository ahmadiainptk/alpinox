# Business Requirements Document (BRD)
## Project: Offline-First SPA Blog Engine — "Alpinox"

---

### 1. Tujuan Proyek (Objective)

Membangun platform blog berbasis Single Page Application (SPA) yang mengutamakan kecepatan akses, ketahanan terhadap koneksi tidak stabil (*offline-first*), serta efisiensi biaya infrastruktur pada **Cloudflare Workers Free Tier** (100.000 request/hari). Aplikasi dirancang agar **pengunjung blog tidak mengonsumsi Worker invocation sama sekali** — seluruh konten publik di-render statis saat build time dan disajikan dari CDN.

---

### 2. Persona & User Stories

| Persona | Story |
|---------|-------|
| **Pembaca** | Membaca daftar artikel dan isi artikel dengan navigasi instan SPA, dapat membaca ulang artikel yang sudah dikunjungi meskipun offline. |
| **Pembaca (returning)** | Mendapatkan artikel terbaru secara otomatis saat membuka blog kembali tanpa perlu reload penuh (delta sync). |
| **Admin** | Membuat, mengedit, menghapus, dan mempublikasikan artikel melalui dashboard admin berbasis SPA. |

---

### 3. Fitur Utama (Core Features)

#### 3.1 Blog Publik (Pembaca)
- **Instant SPA Navigation:** Transisi antar halaman mulus tanpa *full reload* via Astro ViewTransitions.
- **Offline-First Reading:** Artikel yang sudah pernah dibaca disimpan di IndexedDB browser. Dapat diakses kembali saat offline.
- **Delta Synchronization:** Saat pengunjung kembali, client hanya meminta artikel yang `updated_at`-nya lebih baru dari `last_sync_timestamp` lokal. Bukan mengunduh ulang seluruh database.
- **Client-Side Search & Filter:** Pencarian dan penyaringan artikel dilakukan sepenuhnya di browser menggunakan IndexedDB — **nol request ke server**.
- **Zero Worker Cost for Readers:** Seluruh halaman blog (listing, detail, about) di-render statis saat `astro build`. Tidak ada API call pembaca yang mengenai Worker.

#### 3.2 Dashboard Admin
- **Autentikasi Admin:** Login via token sederhana yang disimpan di environment variable Cloudflare.
- **CRUD Post:** Membuat, membaca, mengedit, menghapus artikel melalui antarmuka SPA admin (`/admin`).
- **Status Publikasi:** Artikel dapat disimpan sebagai `draft` atau `published`.
- **Preview:** Pratinjau artikel sebelum publikasi.
- **Trigger Redeploy:** Setelah perubahan konten, admin dapat memicu build ulang agar halaman statis diperbarui.

#### 3.3 Optimisasi Free Tier
- **SSG-first:** Blog publik = static HTML (CDN). **0 Worker invocation** per page view.
- **Minimal API Endpoints:** Hanya 3 endpoint Worker: `/api/auth` (login), `/api/posts` (CRUD admin), `/api/posts/sync` (delta sync pembaca).
- **Client-Side Processing:** Semua operasi pencarian, filter, sorting dilakukan di browser.
- **Debounce & Lock:** Mekanisme pencegahan request ganda saat sync berjalan.

---

### 4. Arsitektur & Batasan Teknis (Technical Scope)

| Layer | Teknologi |
|-------|-----------|
| Frontend Framework | AstroJS (static shell + router) + AlpineJS (state & reactivity) |
| Styling | Tailwind CSS v4 |
| Hosting & Backend | Cloudflare Workers / Pages Functions |
| Database | Cloudflare D1 (relational, SQLite-compatible) |
| ORM | Drizzle ORM |
| Package Manager | pnpm |
| Build Target | Static Generation (SSG) untuk halaman publik |

#### 4.1 Alur Data

```
[Build Time]
  D1 ──fetch all posts──▶ Astro SSG ──generate──▶ Static HTML/CSS/JS ──deploy──▶ Cloudflare CDN
  
[Runtime - Pembaca]
  Browser ◀──static HTML── Cloudflare CDN (0 Worker hit)
  Browser ◀──delta sync── Worker API ──query── D1 (hanya saat sync)

[Runtime - Admin]
  Browser (SPA /admin) ◀──CRUD── Worker API ──query── D1
  Admin trigger redeploy ──▶ Cloudflare Deploy Hook
```

---

### 5. Data Model (D1 Schema)

#### Tabel: `posts`

| Kolom | Tipe | Deskripsi |
|-------|------|-----------|
| `id` | TEXT (PK) | UUID |
| `title` | TEXT NOT NULL | Judul artikel |
| `slug` | TEXT UNIQUE NOT NULL | URL-friendly identifier |
| `description` | TEXT NOT NULL | Deskripsi singkat / meta description |
| `body` | TEXT NOT NULL | Konten artikel dalam format MDX |
| `hero_image` | TEXT | URL gambar sampul (opsional) |
| `status` | TEXT NOT NULL DEFAULT 'draft' | `draft` atau `published` |
| `created_at` | INTEGER NOT NULL | Unix timestamp (detik) |
| `updated_at` | INTEGER NOT NULL | Unix timestamp (detik) |

---

### 6. API Contract

#### 6.1 `POST /api/auth`
**Purpose:** Autentikasi admin.

| | |
|---|---|
| Request Body | `{ "token": "string" }` |
| Response `200` | `{ "ok": true }` |
| Response `401` | `{ "ok": false, "error": "Invalid token" }` |

Token divalidasi terhadap `ADMIN_TOKEN` di environment variable.

#### 6.2 `GET /api/posts/sync`
**Purpose:** Delta sync untuk pembaca (public, no auth).

| | |
|---|---|
| Query Param | `since` — Unix timestamp (detik) |
| Response `200` | `{ "posts": [...], "server_timestamp": number }` |
| Behavior | Hanya mengembalikan post `published` dengan `updated_at > since` |

#### 6.3 `GET /api/posts`
**Purpose:** Admin: mengambil semua post (termasuk draft).

| | |
|---|---|
| Header | `Authorization: Bearer <token>` |
| Response `200` | `{ "posts": [...] }` |
| Response `401` | `{ "error": "Unauthorized" }` |

#### 6.4 `POST /api/posts`
**Purpose:** Admin: membuat post baru.

| | |
|---|---|
| Header | `Authorization: Bearer <token>` |
| Request Body | `{ "title", "slug", "description", "body", "hero_image"?, "status" }` |
| Response `201` | `{ "post": {...} }` |

#### 6.5 `PUT /api/posts/:id`
**Purpose:** Admin: mengedit post.

| | |
|---|---|
| Header | `Authorization: Bearer <token>` |
| Request Body | `{ "title"?, "slug"?, "description"?, "body"?, "hero_image"?, "status"? }` |
| Response `200` | `{ "post": {...} }` |

#### 6.6 `DELETE /api/posts/:id`
**Purpose:** Admin: menghapus post.

| | |
|---|---|
| Header | `Authorization: Bearer <token>` |
| Response `200` | `{ "ok": true }` |

---

### 7. Client-Side Caching Strategy

| Storage | Konten | Tujuan |
|---------|--------|--------|
| **IndexedDB** | Daftar post + isi post (JSON body) | Offline reading, client-side search/filter |
| **localStorage** | `last_sync_timestamp` | Acuan untuk delta sync berikutnya |
| **Service Worker Cache** | Static assets (HTML, CSS, JS, font) | Offline app shell |

#### Alur Caching Pembaca:
1. Pertama kali buka blog → halaman statis dimuat dari CDN.
2. Background: `GET /api/posts/sync?since=0` → simpan semua post ke IndexedDB, simpan `server_timestamp` ke localStorage.
3. Kunjungan berikutnya: `GET /api/posts/sync?since=<last_sync_timestamp>` → hanya terima post baru/update, merge ke IndexedDB.
4. Search/filter: query langsung ke IndexedDB, **tanpa network request**.
5. Offline: baca IndexedDB + Service Worker cache.

---

### 8. Kriteria Keberhasilan (Success Criteria)

| Metrik | Target |
|--------|--------|
| **Worker Invocations per Reader** | 0 (semua halaman publik static dari CDN) |
| **Worker Invocations per Sync** | 1 request per kunjungan ulang |
| **First Contentful Paint** | < 1 detik pada jaringan 3G/4G |
| **Offline Readability** | Artikel yang sudah di-cache dapat dibaca penuh tanpa koneksi |
| **Admin Operations** | CRUD post berfungsi penuh via `/admin` SPA |
| **Daily Worker Limit** | < 100.000 request (Free Tier), mayoritas dari admin + delta sync |

---

### 9. Batasan & Risiko

- **SSG Redeploy:** Setelah admin mengubah konten, harus build ulang agar publik melihat perubahan. Bisa diotomatisasi via Cloudflare Deploy Hook.
- **Delta Sync Granularity:** Pembaca tidak melihat artikel baru sampai mereka membuka blog kembali (pull-based, bukan push/real-time).
- **Free Tier:** 100.000 request/hari cukup untuk skala personal blog selama pola penggunaan terjaga.

(End of file)
