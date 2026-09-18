# 新增后台页面

*[English](04-admin-add-page.md)*

新增一个页面需要四块内容，跟后端那几章的结构是一样的：**路由 + 菜单项**、**API client 模块**、
**页面组件**，以及**三个语言的 i18n 文案**。跟《[新增一个后台 API 端点](02-backend-add-admin-endpoint.zh-CN.md)》
一样，这一章走一遍 Product 页面（`admin/src/pages/Product/index.tsx`）实际是怎么搭出来的。

## 1. 路由和菜单项

`admin/src/router/menu.tsx` 是侧边栏菜单、路由表和面包屑的唯一数据来源——加一个页面只需要在这里
加一条，不用同时改三个地方：

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

`label` 是一个 i18n key（渲染时会去 `menu` 这个命名空间里解析），不是写死的文字——见第 4 步。真正
的 React Router 路由还需要单独在 `App.tsx` 里注册：

```tsx
// admin/src/App.tsx
import ProductPage from './pages/Product'
// ...
<Route path="/product" element={<ProductPage />} />
```

## 2. API client 模块

每个资源在 `admin/src/api/` 下有一个文件，都建立在 `client.ts` 里那个共享的 `api` 工具上（统一
处理 bearer token 和错误规范化，页面代码从来不直接调 `fetch`）：

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

`Product` 这个接口跟后端的 `productDTO` 是逐字段对应的——后端 DTO 改了之后，这两边要手动同步
（这里 Go 和 TypeScript 之间没有共享 schema 或代码生成）。

## 3. 页面组件

重活都交给 Ant Design 的组件（`Table`、`Modal`、`Form` 等等）；页面本身主要是一些 state 加三个
处理函数——加载、提交（`editing` 有没有值决定是新建还是编辑）、删除：

```tsx
// admin/src/pages/Product/index.tsx（节选）
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
  // ... columns、Table，以及用来新建/编辑的 Modal+Form
}
```

注意这里页面自己的文案用的是 `t('product')`，但共享的文案用的是带命名空间前缀的
`t('common:saveSuccess')`——见第 4 步。错误统一走 `apiErrorMessage(e, fallback)`
（`src/api/errors.ts`），它会把后端稳定的错误 `code` 映射成翻译好的文案（如果有对应翻译的话），
没有的话再用你传的兜底字符串——绝对不要直接把 `e.message` 显示出来，那个永远是英文，跟当前语言
无关。

## 4. i18n：三个语言的文件，自动发现

`admin/src/i18n/index.ts` 用 `import.meta.glob('./locales/*/*.json')` 自动加载每一个
`locales/<语言>/<命名空间>.json` 文件——**给已有语言加一个新的命名空间文件就够了，`index.ts`
里什么都不用改。** 每个页面都有一个以自己名字命名的命名空间。三个语言要一起加，key 保持一致：

```
admin/src/i18n/locales/en-US/product.json
admin/src/i18n/locales/zh-CN/product.json
admin/src/i18n/locales/ja-JP/product.json
```

```json
// zh-CN/product.json
{
  "pageTitle": "产品管理",
  "createButton": "新建产品",
  "table": { "name": "名称", "description": "描述" },
  "deleteConfirm": "删除这个产品？已经用这个 pid 的设备不受影响——只是不再显示产品名称。",
  "modal": {
    "editTitle": "编辑产品",
    "createTitle": "新建产品",
    "pidExtraLocked": "创建后不可修改",
    "pidExtraHint": "必须和设备上报的 pid 一致"
  },
  "message": { "loadFailed": "加载产品失败" }
}
```

同时把菜单项需要的这一个新 key，加进**三个** `menu.json` 文件：

```json
// zh-CN/menu.json
"product": "产品",
```

并且优先复用两个共享命名空间，而不是自己重新写一遍：**`common`** 存放通用的按钮/提示文案
（`common:save`、`common:cancel`、`common:saveSuccess`、`common:deleteSuccess`、
`common:actions` 等），**`errors`** 存放后端错误码的翻译（只有你的端点真的引入了一个新的错误码
时才需要动这个）。

> **注意：** 三个语言是英文、简体中文、日文（`i18n/index.ts` 里的 `SUPPORTED_LANGUAGES`）——每次
> 都要三个一起加，不是只加英文。整个漏掉某个语言的文件会回退到 `en-US`（`fallbackLng`），不会
> 报错，所以很容易漏加而没发现——不要指望这个回退机制替代真正的翻译。

## 检查清单

1. 在 `router/menu.tsx` 里加菜单项，在 `App.tsx` 里加 `<Route>`。
2. 在 `api/` 下加一个 API client 模块，手动跟后端 DTO 保持一致。
3. 用 Ant Design 搭页面组件，共享文案用 `common`/`errors`，页面专属文案用自己的命名空间。
4. 把命名空间的 JSON 文件加进**三个**语言，菜单那一个新 key 也加进三个 `menu.json`。
5. 验证：`cd admin && npm run lint && npm run build`。

## 下一步

[固件：接入新传感器](05-firmware-add-sensor.zh-CN.md)。
