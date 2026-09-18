# Add a New Admin Console Page

*[中文](04-admin-add-page.zh-CN.md)*

Four pieces go into a new page, same as the backend chapters: a **route + menu entry**, an
**API client module**, the **page component**, and **i18n strings for all three locales**. As in
[Add a New Admin API Endpoint](02-backend-add-admin-endpoint.md), this walks through how the
Product page (`admin/src/pages/Product/index.tsx`) was actually built.

## 1. Route and menu entry

`admin/src/router/menu.tsx` is the single source of truth for the sider menu, the route table, and
the breadcrumb — adding a page means adding one entry here, not touching three separate places:

```tsx
// admin/src/router/menu.tsx
{
  key: 'device-management',
  label: 'deviceManagement',
  path: '/device-management',
  icon: <HddOutlined />,
  children: [
    { key: 'device', label: 'device', path: '/device', icon: <UnorderedListOutlined /> },
    { key: 'product', label: 'product', path: '/product', icon: <AppstoreOutlined /> },
  ],
},
```

`label` is an i18n key (resolved against the `menu` namespace at render time), not literal text —
see step 4. The actual React Router route still needs registering separately in `App.tsx`:

```tsx
// admin/src/App.tsx
import ProductPage from './pages/Product'
// ...
<Route path="/product" element={<ProductPage />} />
```

## 2. API client module

One file per resource under `admin/src/api/`, built on the shared `api` helper in `client.ts`
(handles the bearer token and error normalization so page code never calls `fetch` directly):

```ts
// admin/src/api/product.ts
import { api } from './client'

export interface Product {
  id: number
  pid: string
  name: string
  description: string
  created_at: number
}

export function listProducts() {
  return api.get<{ list: Product[] }>('/api/admin/products')
}

export function createProduct(input: { pid: string; name: string; description?: string }) {
  return api.post<Product>('/api/admin/products', input)
}

export function updateProduct(id: number, input: { name: string; description?: string }) {
  return api.patch<{ message: string }>(`/api/admin/products/${id}`, input)
}

export function deleteProduct(id: number) {
  return api.del<{ message: string }>(`/api/admin/products/${id}`)
}
```

The `Product` interface mirrors the backend's `productDTO` field-for-field — keep these two in
sync by hand whenever the backend DTO changes (there's no shared schema/codegen between Go and
TypeScript here).

## 3. The page component

Ant Design components (`Table`, `Modal`, `Form`, ...) do the heavy lifting; the page itself is
mostly state plus three handlers — load, submit (create or edit, depending on whether `editing` is
set), delete:

```tsx
// admin/src/pages/Product/index.tsx (abridged)
export default function ProductPage() {
  const { t } = useTranslation('product')
  const [rows, setRows] = useState<Product[]>([])
  const [editing, setEditing] = useState<Product | null>(null)
  // ...

  const load = async () => {
    const res = await listProducts()
    setRows(res.list)
  }

  const onSubmit = async (values: { pid: string; name: string; description?: string }) => {
    if (editing) {
      await updateProduct(editing.id, { name: values.name, description: values.description })
    } else {
      await createProduct(values)
    }
    message.success(t('common:saveSuccess'))
    load()
  }
  // ... columns, the Table, and a Modal+Form for create/edit
}
```

Notice `t('product')` for this page's own strings but `t('common:saveSuccess')` (namespace-prefixed)
for a shared one — see step 4. Errors go through `apiErrorMessage(e, fallback)` (`src/api/errors.ts`),
which maps the backend's stable error `code` to a translated message where one exists, falling
back to the string you pass otherwise — never show `e.message` directly, it's always in English
regardless of the active language.

## 4. i18n: three locale files, auto-discovered

`admin/src/i18n/index.ts` uses `import.meta.glob('./locales/*/*.json')` to load every
`locales/<lang>/<namespace>.json` file automatically — **adding a new namespace file for an
existing language is enough; nothing in `index.ts` needs to change.** Every page gets its own
namespace named after it. Add all three locales together, keeping the same keys:

```
admin/src/i18n/locales/en-US/product.json
admin/src/i18n/locales/zh-CN/product.json
admin/src/i18n/locales/ja-JP/product.json
```

```json
// en-US/product.json
{
  "pageTitle": "Product Management",
  "createButton": "New Product",
  "table": { "name": "Name", "description": "Description" },
  "deleteConfirm": "Delete this product? Devices already using this pid are unaffected — they just stop showing a product name.",
  "modal": {
    "editTitle": "Edit Product",
    "createTitle": "New Product",
    "pidExtraLocked": "Cannot be changed after creation",
    "pidExtraHint": "Must match the pid your devices report"
  },
  "message": { "loadFailed": "Failed to load products" }
}
```

Also add the one new key the menu entry needs, to **all three** `menu.json` files:

```json
// en-US/menu.json
"product": "Products",
```

And reuse the two shared namespaces rather than duplicating strings into your own: **`common`**
for generic buttons/toasts (`common:save`, `common:cancel`, `common:saveSuccess`,
`common:deleteSuccess`, `common:actions`, ...) and **`errors`** for backend error-code
translations (only touch this one if your endpoint introduces a genuinely new error code).

> **Note:** the three locales are English, Simplified Chinese, and Japanese
> (`SUPPORTED_LANGUAGES` in `i18n/index.ts`) — all three, every time, not just English. Missing a
> locale file entirely falls back to `en-US` (`fallbackLng`) rather than erroring, so it's easy to
> forget one; don't rely on that fallback as a substitute for actually translating it.

## Checklist

1. Add the menu entry in `router/menu.tsx` and the `<Route>` in `App.tsx`.
2. Add an API client module under `api/`, mirroring the backend DTO by hand.
3. Build the page component with Ant Design, using `common`/`errors` for shared strings and your
   own namespace for page-specific ones.
4. Add the namespace JSON file to **all three** locales, plus the one new `menu.json` key to all
   three.
5. Verify: `cd admin && npm run lint && npm run build`.

## Next

[Firmware: Wire In a New Sensor](05-firmware-add-sensor.md).
