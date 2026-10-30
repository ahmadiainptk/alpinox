# Developer Guidelines & Workflow
## Project: Alpinox — Arch Linux | pnpm | AI-Agent Enabled

Dokumen ini adalah instruksi utama bagi AI Agent untuk mengembangkan aplikasi dengan standar penulisan kode yang ketat, pencegahan duplikasi request, dan optimalisasi Cloudflare Workers Free Tier.

---

### 1. Struktur Proyek (Target)

```
alpinox/
├── src/
│   ├── assets/                  # Font, gambar static
│   │   └── fonts/
│   ├── components/              # Komponen Astro reusable
│   │   ├── admin/               # Komponen khusus dashboard admin
│   │   │   ├── AdminLayout.astro
│   │   │   ├── PostEditor.astro
│   │   │   └── PostList.astro
│   │   ├── BaseHead.astro
│   │   ├── Footer.astro
│   │   ├── FormattedDate.astro
│   │   ├── Header.astro
│   │   └── HeaderLink.astro
│   ├── content/                 # Konten blog (hanya fallback / seed data)
│   │   └── blog/
│   ├── content.config.ts        # Konfigurasi Astro Content (jika masih digunakan)
│   ├── consts.ts                # Konstanta global (site title, dll)
│   ├── db/                      # Drizzle ORM schema & migrations
│   │   ├── schema.ts            # Definisi tabel D1
│   │   └── migrations/          # File migrasi auto-generated
│   ├── layouts/
│   │   ├── BlogPost.astro
│   │   └── BaseLayout.astro
│   ├── lib/                     # Utility & helper
│   │   ├── api.ts               # Wrapper fetch() dengan debounce/lock
│   │   └── db.ts                # IndexedDB wrapper (open, read, write, sync)
│   ├── pages/
│   │   ├── admin/               # Dashboard admin (SPA)
│   │   │   └── index.astro
│   │   ├── api/                 # API endpoints (Cloudflare Workers)
│   │   │   ├── auth.ts
│   │   │   └── posts.ts
│   │   ├── blog/
│   │   │   ├── index.astro      # Daftar artikel (SSG)
│   │   │   └── [...slug].astro  # Detail artikel (SSG)
│   │   ├── about.astro
│   │   ├── index.astro
│   │   └── rss.xml.js
│   ├── stores/                  # AlpineJS global stores
│   │   ├── auth.ts              # Store: auth state, login/logout
│   │   ├── posts.ts             # Store: daftar post, CRUD, delta sync
│   │   └── ui.ts                # Store: loading, toast, pull-to-refresh
│   └── styles/
│       └── global.css
├── public/
│   ├── favicon.ico
│   └── favicon.svg
├── astro.config.mjs
├── drizzle.config.ts            # Konfigurasi Drizzle Kit
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── wrangler.jsonc
├── BRD.md
└── Developer.md
```

---

### 2. Pengaturan Lingkungan Kerja (Environment Setup)

#### 2.1 Wajib
- **Package Manager:** `pnpm` — tidak boleh ada `package-lock.json` atau `yarn.lock`.
- **Node.js:** `>=22.12.0` (sesuai `engines` di `package.json`).
- **OS Target:** Arch Linux (development), Cloudflare Workers (production).

#### 2.2 Environment Variables
Buat file `.dev.vars` untuk development lokal (tidak di-commit):

```
ADMIN_TOKEN=your-secret-admin-token
D1_DATABASE_ID=alpinox-db
```

Untuk production, set via `wrangler secret put`:
```bash
pnpm wrangler secret put ADMIN_TOKEN
```

#### 2.3 Instalasi Dependensi Tambahan

Drizzle ORM dan driver D1 perlu diinstal secara eksplisit:

```bash
pnpm add drizzle-orm @libsql/client
pnpm add -D drizzle-kit @types/node
```

---

### 3. Konfigurasi Drizzle + Cloudflare D1

#### 3.1 Buat Database D1
```bash
pnpm wrangler d1 create alpinox-db
```
Catat `database_id` dari output, masukkan ke `wrangler.jsonc`.

#### 3.2 `wrangler.jsonc` — Tambahkan Binding D1
```jsonc
{
  "compatibility_date": "2026-06-04",
  "name": "alpinox",
  "main": "@astrojs/cloudflare/entrypoints/server",
  "assets": { "directory": "./dist", "binding": "ASSETS" },
  "observability": { "enabled": true },
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "alpinox-db",
      "database_id": "<ID_DARI_OUTPUT_WRANGLER>"
    }
  ]
}
```

#### 3.3 `drizzle.config.ts` — Buat File Konfigurasi
```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/db/schema.ts',
  out: './src/db/migrations',
  dialect: 'sqlite',
  driver: 'd1-http',
  dbCredentials: {
    accountId: process.env.CLOUDFLARE_ACCOUNT_ID!,
    databaseId: process.env.CLOUDFLARE_DATABASE_ID!,
    token: process.env.CLOUDFLARE_API_TOKEN!,
  },
});
```

#### 3.4 `src/db/schema.ts` — Definisi Tabel
```ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const posts = sqliteTable('posts', {
  id: text('id').primaryKey(),
  title: text('title').notNull(),
  slug: text('slug').notNull().unique(),
  description: text('description').notNull(),
  body: text('body').notNull(),
  heroImage: text('hero_image'),
  status: text('status').notNull().default('draft'),
  createdAt: integer('created_at').notNull(),
  updatedAt: integer('updated_at').notNull(),
});
```

#### 3.5 Perintah Migrasi
```bash
# Generate file migrasi dari perubahan schema
pnpm drizzle-kit generate

# Terapkan migrasi ke D1 lokal
pnpm wrangler d1 migrations apply alpinox-db --local

# Terapkan migrasi ke D1 production
pnpm wrangler d1 migrations apply alpinox-db --remote
```

---

### 4. Aturan Penulisan Kode (Coding Standards)

#### 4.1 Astro & AlpineJS Integration
- Astro digunakan untuk merender struktur HTML statis dan menangani transisi halaman (`<ViewTransitions />`).
- **State global dikelola murni oleh AlpineJS** (`Alpine.store()`). Tidak boleh mencampur vanilla JS DOM manipulation jika bisa ditangani direktif AlpineJS (`x-bind`, `x-model`, `x-for`, `x-show`, `x-on`).
- Halaman publik (blog listing, detail, about) menggunakan `getStaticPaths()` dan prerender. **Tidak menggunakan `Astro.request`** di halaman publik.
- Halaman admin (`/admin`) adalah client-side SPA penuh — semua interaksi via AlpineJS store + fetch ke API.
- File `.astro` untuk API endpoint (di `src/pages/api/`) menggunakan `export const prerender = false`.

#### 4.2 Mekanisme Anti-Duplicate Request (Debounce & Lock)
- Setiap store yang melakukan `fetch()` harus memiliki properti `isFetching: boolean`.
- Sebelum fetch, periksa `isFetching`. Jika `true`, batalkan eksekusi.
- Setelah fetch selesai (sukses/gagal), reset `isFetching` ke `false`.
- Contoh pola di `src/lib/api.ts`:

```ts
let isFetching = false;

export async function fetchWithLock(url: string, options?: RequestInit) {
  if (isFetching) return null;
  isFetching = true;
  try {
    const res = await fetch(url, options);
    return await res.json();
  } finally {
    isFetching = false;
  }
}
```

#### 4.3 IndexedDB Wrapper (`src/lib/db.ts`)
- Abstraksi sederhana untuk operasi baca/tulis post ke IndexedDB.
- Method minimal:
  - `openDB(): Promise<IDBDatabase>` — buka/koneksi database
  - `savePosts(posts: Post[]): Promise<void>` — simpan batch post
  - `getAllPosts(): Promise<Post[]>` — ambil semua post
  - `getPostBySlug(slug: string): Promise<Post | undefined>` — ambil satu post
  - `getLastSyncTimestamp(): Promise<number>` — baca timestamp terakhir
  - `setLastSyncTimestamp(ts: number): Promise<void>` — simpan timestamp
- Gunakan single database `alpinox` dengan object store `posts` (keyPath: `slug`).

#### 4.4 Tailwind CSS v4 Guidelines
- Gunakan utility class standar Tailwind v4.
- **Prefiks yang valid di v4:** `size-*` (width + height), `inset-*`, `transform`, dsb.
- Hindari `@apply` — gunakan utility class langsung di markup atau komponen.
- Gunakan CSS custom properties di `global.css` untuk theming (warna, font).

#### 4.5 TypeScript
- Gunakan strict mode. Jangan gunakan `any` kecuali terpaksa.
- Ekspor tipe dari file schema Drizzle untuk digunakan di store.
- Gunakan `zod` (sudah include di Astro) untuk validasi API request body.

---

### 5. Arsitektur AlpineJS Store

#### 5.1 `src/stores/auth.ts` — Store Autentikasi
```ts
Alpine.store('auth', {
  isAuthenticated: false,
  token: localStorage.getItem('admin_token') || '',

  async login(token: string) { /* POST /api/auth, simpan ke localStorage */ },
  logout() { /* hapus token, reset state */ },
  checkAuth() { /* verifikasi token saat load halaman admin */ },
});
```

#### 5.2 `src/stores/posts.ts` — Store Post & Delta Sync
```ts
Alpine.store('posts', {
  items: [] as Post[],          // Semua post di memory
  isFetching: false,            // Lock anti-duplicate
  isSyncing: false,             // Lock khusus sync

  async syncFromServer() {     // GET /api/posts/sync?since=...
    if (this.isSyncing) return;
    this.isSyncing = true;
    // fetch delta, merge ke IndexedDB, update memory
  },
  async loadFromCache() {      // Baca IndexedDB → memory
  },
  search(query: string) {      // Client-side search di this.items
  },
  filterByStatus(status: string) { // Client-side filter
  },
});
```

#### 5.3 `src/stores/ui.ts` — Store UI
```ts
Alpine.store('ui', {
  isLoading: false,
  toasts: [] as { message: string; type: 'success' | 'error' }[],
  pullToRefresh: { /* state untuk gesture */ },
  showToast(message: string, type: 'success' | 'error') { /* ... */ },
});
```

---

### 6. Spesifikasi API Endpoint

Semua file di `src/pages/api/` harus:
- Menggunakan `export const prerender = false`
- Mengakses D1 binding via `Astro.locals.runtime.env.DB` (Cloudflare adapter)
- Menggunakan `drizzle-orm` untuk query

#### 6.1 `src/pages/api/auth.ts`
- Method: `POST`
- Validasi `ADMIN_TOKEN` dari `env` vs request body
- Return `{ ok: true }` atau 401

#### 6.2 `src/pages/api/posts.ts`
- Method: `GET`, `POST`
- Auth required (Bearer token header)
- `GET` → semua post (termasuk draft)
- `POST` → buat post baru, generate UUID slug jika tidak disediakan

#### 6.3 `src/pages/api/posts/sync.ts`
- Method: `GET`
- **No auth** (public endpoint)
- Query param `since` (timestamp)
- Hanya return post dengan `status = 'published'` dan `updated_at > since`
- Return `{ posts, server_timestamp: Date.now() }`

#### 6.4 `src/pages/api/posts/[id].ts`
- Method: `PUT`, `DELETE`
- Auth required
- `PUT` → update post by ID
- `DELETE` → hapus post by ID

---

### 7. Service Worker & Offline Strategy

- Astro dengan adapter Cloudflare **tidak otomatis** menangani Service Worker.
- Buat file `public/sw.js` secara manual atau gunakan library seperti Workbox.
- **Caching strategy:**
  - **App Shell** (HTML, CSS, JS, font): Cache-first, pre-cache saat install.
  - **Halaman blog** (static HTML): Network-first, fallback ke cache.
  - **API response** (sync): Disimpan ke IndexedDB, bukan Service Worker cache.

```js
// public/sw.js — kerangka dasar
const CACHE_NAME = 'alpinox-v1';
const APP_SHELL = ['/offline/', '/styles/global.css'];

self.addEventListener('install', (e) => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(APP_SHELL)));
});

self.addEventListener('fetch', (e) => {
  if (e.request.mode === 'navigate') {
    e.respondWith(
      fetch(e.request).catch(() => caches.match('/offline/'))
    );
  }
});
```

---

### 8. Alur Pengembangan (Development Phases)

#### Fase 1: Setup Proyek & Dependencies
1. Inisialisasi proyek Astro dengan adapter Cloudflare (✅ sudah)
2. Setup Tailwind v4 (✅ sudah)
3. Install AlpineJS (✅ sudah)
4. Install Drizzle ORM + `drizzle-kit`
5. Konfigurasi `drizzle.config.ts`
6. Konfigurasi binding D1 di `wrangler.jsonc`

#### Fase 2: Database & Schema
1. Buat database D1 via wrangler
2. Definisikan schema di `src/db/schema.ts`
3. Generate & apply migrasi awal

#### Fase 3: API Endpoints
1. `src/pages/api/auth.ts` — endpoint login admin
2. `src/pages/api/posts.ts` — CRUD admin posts
3. `src/pages/api/posts/sync.ts` — delta sync publik
4. `src/pages/api/posts/[id].ts` — update/delete single post

#### Fase 4: Client-Side Stores & Library
1. `src/lib/db.ts` — IndexedDB wrapper
2. `src/lib/api.ts` — fetch wrapper dengan debounce/lock
3. `src/stores/auth.ts` — Alpine store autentikasi
4. `src/stores/posts.ts` — Alpine store post management + delta sync
5. `src/stores/ui.ts` — Alpine store UI (toast, loading, pull-to-refresh)

#### Fase 5: Halaman Publik (SSG)
1. Ubah `src/pages/index.astro` — halaman beranda kustom
2. Ubah `src/pages/blog/index.astro` — daftar artikel dari D1 (fetched at build time via `getStaticPaths`)
3. Ubah `src/pages/blog/[...slug].astro` — detail artikel dari D1
4. Integrasi `<ViewTransitions />` untuk navigasi SPA
5. Init store AlpineJS di layout utama

#### Fase 6: Dashboard Admin (SPA)
1. `src/pages/admin/index.astro` — layout admin dengan AlpineJS SPA
2. `src/components/admin/AdminLayout.astro` — shell admin (sidebar + header)
3. `src/components/admin/PostList.astro` — tabel daftar post + search client-side
4. `src/components/admin/PostEditor.astro` — form create/edit post dengan MDX editor
5. Integrasi dengan store auth & posts

#### Fase 7: Offline & Service Worker
1. Buat `public/sw.js` — service worker dasar
2. Registrasi SW di layout utama
3. Buat halaman `/offline/` sebagai fallback
4. Integrasi delta sync di store posts

#### Fase 8: Testing & Deployment
1. Tulis unit test untuk store AlpineJS
2. Tulis integration test untuk API endpoint
3. Deploy ke Cloudflare Pages
4. Setup deploy hook untuk trigger redeploy dari admin

---

### 9. Testing

#### 9.1 Unit Test (Vitest)
```bash
pnpm add -D vitest
```
- Test store AlpineJS logic (sync, search, filter)
- Test IndexedDB wrapper

#### 9.2 Integration Test
```bash
pnpm wrangler d1 execute alpinox-db --local --command "SELECT 1"
pnpm astro dev
```
- Manual test API endpoint via `curl` atau REST client
- Test alur admin: login → create post → edit → delete

#### 9.3 E2E (Opsional)
- Playwright untuk test alur pengguna penuh

---

### 10. Deployment

```bash
# Build untuk production (SSG)
pnpm astro build

# Deploy ke Cloudflare Pages
pnpm wrangler pages deploy dist --project-name alpinox

# Atau via git integration di Cloudflare Dashboard
```

#### 10.1 Redeploy Trigger dari Admin
- Buat Deploy Hook di Cloudflare Pages Dashboard
- Simpan URL hook di env `DEPLOY_HOOK_URL`
- Admin dapat memicu redeploy via tombol di dashboard

---

### 11. Git Commit Workflow

- Commit secara berkala untuk setiap perubahan fitur atomik.
- Format pesan commit: `<type>: <description>` (contoh: `feat: add D1 schema and drizzle config`)
- Type: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`
- Bahasa: Indonesia atau Inggris, konsisten.

---

### 12. Catatan Penting untuk AI Agent

1. **Jangan pernah membuat file `package-lock.json` atau `yarn.lock`.**
2. **Setiap `fetch()` ke API Worker harus melalui wrapper `fetchWithLock`** untuk mencegah duplicate request.
3. **Halaman publik (blog) HARUS static generation (SSG)** — jangan fetch data di runtime untuk halaman pembaca.
4. **Jangan gunakan `Astro.request` di halaman publik** — itu memaksa SSR.
5. **Semua pemrosesan data (search, filter, sort) dilakukan di client-side** menggunakan IndexedDB store.
6. **Gunakan Zod untuk validasi API request body** — sudah ter-install sebagai dependency Astro.
7. **Gunakan Tailwind v4 utility class** — semua class modern (`size-*`, `inset-*`, dll) valid.
8. **Gunakan direktif AlpineJS** (`x-data`, `x-bind`, `x-model`, `x-for`, `x-show`, `x-on`) — hindari `document.querySelector` manual.
9. **Jangan hardcode credentials** — gunakan environment variable (`Astro.locals.runtime.env` untuk Worker, `import.meta.env` untuk build-time).
10. **Pastikan file migrasi Drizzle di-commit** — jangan di `.gitignore`.

(End of file)
