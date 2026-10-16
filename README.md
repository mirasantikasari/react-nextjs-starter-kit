This is a [Next.js](https://nextjs.org) app template, bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app) and extended with a standard Redux Toolkit + Axios architecture.

## Getting Started

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) to see the result.

Set your backend URL in `.env.local`:

```bash
NEXT_PUBLIC_API_URL=https://your-api.example.com
```

## Folder Structure

```
app/         Routes (Next.js App Router). One folder per route, e.g. app/sign-in/page.tsx
hooks/       Reusable React hooks (useAppDispatch/useAppSelector, useNavigate, ...)
layouts/     Page shells that wrap routes (e.g. AppLayout)
services/    Shared API clients (axios instance with auth header injection)
store/       Redux Toolkit state
  controller/  createAsyncThunk API calls, grouped by domain
  slices/      createSlice reducers + extraReducers wired to controllers
  types/       Request/response and shared types (e.g. ApiErrorResponse)
  store.ts     configureStore, combines all slices
  StoreProvider.tsx  Client component wrapping <Provider store={store}>
```

Adding a new domain (e.g. "product") means adding:
`store/types/ProductType.ts` → `store/controller/productController.ts` → `store/slices/productSlice.ts`, then registering the reducer in `store/store.ts`.

## Redux data flow

Each domain follows the same pattern, using `auth` as the reference:

1. **Controller** (`store/controller/authController.ts`) — a `createAsyncThunk` per API call. Every thunk wraps its request in try/catch and calls `rejectWithValue(error.response?.data ?? { message: error.message })` on failure, so the real API error body reaches the slice instead of getting swallowed.
2. **Slice** (`store/slices/authSlice.ts`) — `extraReducers` handles `.pending` / `.fulfilled` / `.rejected` for each thunk, setting `loading` and `error` accordingly.
3. **Page** (`app/**/page.tsx`) — reads state with `useAppSelector`, dispatches thunks with `useAppDispatch` (both from `hooks/useStore.ts`), and renders the `error.message` when present.

## Examples included

- **`app/sign-in/page.tsx`** — login form wired to `handleActionLogin`, stores the returned token in `localStorage`, redirects on success, shows the API error message on failure.
- **`app/example/page.tsx`** — GET and POST round trip against the public [JSONPlaceholder](https://jsonplaceholder.typicode.com) API via `store/controller/exampleController.ts`, demonstrating loading states and error display. Replace `exampleApi`'s baseURL with your own `services/api.ts` instance once you have a real endpoint.

## Auth

- `services/api.ts` attaches `Authorization: Bearer <token>` from `localStorage["auth-key"]` to every request.
- `layouts/AppLayout.tsx` fetches the current profile (`getProfile`) on mount when `checkAuth` is true and an `auth-key` exists. Pass `checkAuth={false}` for public pages (sign-in, sign-up, etc.).

## Deploy ke Cloudflare Workers

Project ini di-deploy ke Cloudflare Workers memakai [OpenNext](https://opennext.js.org/cloudflare) (`@opennextjs/cloudflare`) lewat GitHub Actions ([`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)). Setiap push ke branch `master` otomatis build dan deploy.

### Setup sekali di awal

1. **Ganti nama Worker** di [`wrangler.jsonc`](wrangler.jsonc): field `"name"` masih placeholder `nama-proyek-kamu`. Ini juga menentukan URL `https://<name>.<subdomain>.workers.dev`.
2. **Buat API Token Cloudflare**: dashboard Cloudflare → *My Profile* → *API Tokens* → *Create Token* → template **Edit Cloudflare Workers**.
3. **Ambil Account ID**: dashboard Cloudflare → *Workers & Pages* (ada di sidebar kanan).
4. **Isi GitHub Secrets**: repo GitHub → *Settings* → *Secrets and variables* → *Actions* → *New repository secret*:

   | Secret | Isi |
   | --- | --- |
   | `CLOUDFLARE_API_TOKEN` | token dari langkah 2 |
   | `CLOUDFLARE_ACCOUNT_ID` | account ID dari langkah 3 |
   | `NEXT_PUBLIC_API_URL` | URL backend produksi, mis. `https://api.example.com` |

   `NEXT_PUBLIC_API_URL` dipakai saat **build** (nilainya ditanam ke bundle client), jadi kalau diganti harus deploy ulang.
5. Push ke `master`, lalu pantau di tab **Actions** di GitHub.

### Deploy manual dari lokal

Butuh **Node 22+** (syarat wrangler).

```bash
npm ci
npm run build:cf          # hasil build ada di .open-next/
npx wrangler login        # sekali saja
npx wrangler deploy
```

`NEXT_PUBLIC_API_URL` harus tersedia saat `build:cf` (mis. di `.env.local` atau diset di shell), karena nilainya dibaca saat build.

### Troubleshooting

| Error | Penyebab / solusi |
| --- | --- |
| `No open-next.config.ts file was found` | File [`open-next.config.ts`](open-next.config.ts) harus ada di root dan ter-commit. |
| `EBADENGINE ... required: { node: '>=22.0.0' }` | Pakai Node 22+ (lokal dan di `setup-node` pada workflow). |
| `ERESOLVE ... peer next` saat install | Versi `next` harus cocok dengan peer dependency `@opennextjs/cloudflare`. Cek `npm view @opennextjs/cloudflare peerDependencies`. |
| `Refusing to fetch ... remote` saat `npm ci` | `package-lock.json` berisi URL `registry.npmmirror.com`. Hapus lockfile lalu `npm install` ulang memakai registry default (`https://registry.npmjs.org`), lalu commit. |
| Error 1101 `Worker threw exception` | Ada dua penyebab yang pernah terjadi: (1) `wrangler.jsonc` tidak punya binding `ASSETS` / `WORKER_SELF_REFERENCE`. (2) Versi `next` terlalu baru dan belum didukung OpenNext (mis. 16.4.0 → `Unexpected loadManifest(...preview-props.json)`). Versi `next` di-pin ke `16.3.8`; naikkan hanya setelah `@opennextjs/cloudflare` mendukungnya, dan tes dulu dengan `npm run build:cf && npx wrangler dev`. |
| Deploy gagal `Authentication error` | Cek `CLOUDFLARE_API_TOKEN` dan `CLOUDFLARE_ACCOUNT_ID` di GitHub Secrets. |
| API call ke `undefined/...` di produksi | Secret `NEXT_PUBLIC_API_URL` belum diisi (harus ada sebelum build). |

## Learn More

- [Next.js Documentation](https://nextjs.org/docs)
- [Redux Toolkit Documentation](https://redux-toolkit.js.org)
