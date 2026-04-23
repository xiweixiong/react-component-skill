---
name: stoly-react-component
description: >-
  基于 antd 6.x 封装的 React 组件库 @stoly/react-component 的开发与使用指南。
  当任务涉及 RcTable、RcTableForm、RcModal、RcLayoutTabPage、RcExcel 或该库的
  Hooks（useSimpleReducer、useOnlyAsync、usePage、useRenderRef）时使用。
  触发场景：使用 @stoly/react-component 构建表格查询页面、弹窗、布局、Excel 导出，
  或在项目中引用该库的组件和类型。
---

# @stoly/react-component 使用指南

基于 antd 6.x + React 19 + styled-components 6.x 的业务组件库，提供表格、查询表单、弹窗、布局等开箱即用的组件。

## 核心概念：Query 数据流

所有表格相关组件围绕 `SearchQuery<T>` 协议协作：

```
useSearchQuery → query → RcTableForm(set) → RcTable(get/refresh)
```

`SearchQuery<T>` 接口：
- `get()` — 获取当前查询条件
- `set(q)` — 合并更新查询条件，触发表格重新请求
- `refresh()` — 使用当前条件重新请求

## 组件速查

### RcTable — 数据表格

```tsx
import { Table } from 'antd'
import { RcTable, RcTableForm } from '@stoly/react-component'

// 1. 创建查询条件
const query = RcTableForm.useQuery<MyQuery>({ pageNum: 1, pageSize: 20 })

// 2. 使用表格（推荐声明式 <Table.Column /> 写法，render 可直接访问外部闭包）
<RcTable<Row, Params, Result, MyQuery>
  query={query}
  api={fetchList}
  queryFormat={(q) => ({ page: q.pageNum, size: q.pageSize, ...q })}
  dataFormat={(res) => ({ rows: res.data.list, total: res.data.total })}
  showIndex          // 序号列
  exported           // 右键导出 Excel（简单表头时使用）
  rowKey="id"
>
  <Table.Column<Row> title="姓名" dataIndex="name" key="name" />
  <Table.Column<Row>
    {...RcTable.OperateColumnProps}
    key="operate"
    render={(_, record) => <a onClick={() => modal.open({ id: record.id })}>编辑</a>}
  />
</RcTable>

// 也支持传统 columns 数组（二选一）：
<RcTable ... columns={columns} />
```

> **列声明推荐使用 `<Table.Column />` 子组件形式**：更贴近 JSX 习惯、render 无需声明 record 类型、避免在组件外部维护 `columns` 变量。`RcTable` 内部会把 children 转换为列配置，与数组写法行为一致。

**Props（继承 antd TableProps，移除 dataSource/loading/components/onChange）：**

| Prop | 类型 | 说明 |
|------|------|------|
| `query` | `SearchQuery<Q>` | 查询条件（必传） |
| `api` | `(params: P) => Promise<R>` | 数据请求接口（必传） |
| `queryFormat` | `(q: Q) => P` | 查询条件 → 接口参数转换（必传） |
| `dataFormat` | `(res: R) => DataFormat<T>` | 接口响应 → `{ rows, total }` 转换（必传） |
| `showIndex` | `boolean` | 显示序号列 |
| `exported` | `boolean` | 启用右键导出 Excel |

**Ref 方法：** `getCurrentData()` 返回 `{ rows, total }`

**静态属性：**
- `RcTable.OperateColumnProps` — 操作列默认配置 `{ align:'center', title:'操作', width:120, fixed:'right' }`
- `RcTable.Context` — 通过 render props 获取表格数据

**内置功能：** 列可见性切换、列拖拽排序、密度切换、回到顶部、右键菜单（刷新/复制单元格/复制行/导出）

### RcTableForm — 查询表单

```tsx
const query = RcTableForm.useQuery<MyQuery>()
const controller = RcTableForm.useController()

<RcTableForm<MyQuery> query={query} controller={controller}>
  <Flex gap={12}>
    <Form.Item name="name" label="姓名"><Input /></Form.Item>
    <Form.Item name="age" label="年龄"><Input /></Form.Item>
  </Flex>
</RcTableForm>
```

**Props（继承 antd FormProps，移除 form）：**

| Prop | 类型 | 说明 |
|------|------|------|
| `query` | `SearchQuery<T>` | 查询条件（必传） |
| `controller` | `TableFormController` | 表单控制器（必传） |
| `customButtons` | `ReactNode` | 替换默认查询/重置按钮 |
| `extra` | `ReactNode` | 按钮区额外内容 |
| `defaultShowMore` | `boolean` | 默认展开筛选 |

**静态方法：**
- `RcTableForm.useQuery<T>(initValues?)` — 创建 SearchQuery，默认 `{ pageNum:1, pageSize:20 }`
- `RcTableForm.useController()` — 创建表单控制器，可调用 `submit()` / `clear(fields?)`

### RcModal — 可拖拽弹窗

```tsx
const modal = RcModal.useModal<Params>()

<Button onClick={() => modal.open({ id: 1 })}>编辑</Button>
<RcModal title="编辑" open={modal.visable} onCancel={modal.close} fullScreenModal>
  <EditForm modal={modal} query={query} />
</RcModal>
```

**Props（继承 antd ModalProps，移除 modalRender）：**

| Prop | 类型 | 说明 |
|------|------|------|
| `fullScreenModal` | `boolean` | 显示全屏按钮 |
| `autoFullScreen` | `boolean` | 默认全屏 |
| `onFullScreenChange` | `(fullScreen: boolean) => void` | 全屏状态回调 |

**弹窗页面 Props 接口：**

```tsx
interface RcModalProps<P = unknown> {
  modal: ModalProps<P>   // { visable, params, open, close }
  query?: SearchQuery<QueryData>
}
```

**静态方法：**
- `RcModal.useModal<P>()` — 创建弹窗状态 `{ visable, params, open, close }`
- `RcModal.confirm(opts, options?)` — 确认操作弹窗，成功后自动刷新/翻页

**confirm 用法：**

```tsx
// 对象参数形式（推荐）
RcModal.confirm({
  api: deleteApi,
  params: { id: record.id },
  query,
  tableData,  // 可选，用于删除后自动翻页
}, { title: '删除', content: '确认删除？' })
```

### RcExcel — Excel 导出

```tsx
import { RcExcel } from '@stoly/react-component'

RcExcel.export({
  header: [{ header: '姓名', key: 'name' }, { header: '年龄', key: 'age' }],
  data: [{ name: '张三', age: 18 }],
}, '文件名')
```

### 布局组件

- `RcLayoutHeader` — 头部布局，左右 flex 排列，props: `children`, `actions`
- `RcLayoutSider` — 侧栏，内置 Menu + Scrollbars，props: `menuProps`, `header`, `footer`, `trigger`, `scrollBarProps`
- `RcLayoutTabPage` — 标签页布局，支持 Sider/Header 两种模式
  - `.TabLabel` — 可关闭的 tab 标签
  - `.TabContextMenu` — tab 右键菜单
  - `.PageContainer` — 页面容器（Card + Scrollbars）

## Hooks

| Hook | 签名 | 说明 |
|------|------|------|
| `useSimpleReducer` | `(init: T) => [T, dispatch, reset]` | 轻量 state 合并更新 + reset |
| `useOnlyAsync` | `(fn: AsyncFunc) => AsyncFunc` | 防止异步函数并发执行 |
| `usePage` | `(init?) => [pager, dispatch]` | 分页状态 `{ page, size, total }` |
| `useRenderRef` | `(init?) => [ref, setRef]` | ref + 触发重渲染的更新 |

## 共享类型

```typescript
interface BaseResult<T> { success: boolean; data?: T }
type BaseResultData<T, K> = NonNullable<T[K]>
interface ListPager { page: number; size: number; total?: number }
interface QueryData { pageNum: number; pageSize: number; sort?: Sort }
interface Sort { field?: string; order?: 'asc' | 'desc' }
interface DataFormat<T> { rows: T[]; total: number }
```

## 典型页面模板

推荐将"列表页"与"编辑弹窗"拆分为两个文件：`List.tsx` + `Edit.tsx`。编辑弹窗作为独立组件通过 `RcModalProps<P>` 接收 `modal` 与 `query`，实现关注点分离。

### 脚手架文件

创建列表/编辑页面时，**先读取并复制**以下两个模板文件，再按业务替换字段、接口、类型：

- `skills/stoly-react-component/template/List.tsx.tmp` → 列表页（查询表单 + 表格 + 操作列）
- `skills/stoly-react-component/template/Edit.tsx.tmp` → 编辑弹窗（新增/编辑共用，按 `id` 判断态）

> 模板基于当前 skill 目录的相对路径。使用时读取原文件内容，去掉 `.tmp` 后缀后作为目标文件写入业务目录（例如 `src/pages/user/List.tsx`、`src/pages/user/Edit.tsx`）。

### 必须替换的占位

复制模板后按实际业务替换以下内容，保留结构不变：

| 占位 | 说明 |
|------|------|
| `Query` / `Row` | 查询条件类型 / 表格行类型 |
| `Params` / `Result` | 列表接口的入参 / 返回类型 |
| `fetchList` / `deleteApi` / `detailApi` / `createApi` / `updateApi` | 真实业务接口 |
| `EditParams` / `FormType` | 弹窗入参类型 / 表单数据类型 |
| `field1` / `field2` | 查询表单与编辑表单的业务字段 |
| 列定义、`columns` | 按业务补齐 `dataIndex` / `render` |

### 模板要点

- 列表页通过 `modal.open(params)` 传递参数（通常为 `{ id }`），弹窗内部按 `id` 判断"新增/编辑"态
- 弹窗组件统一使用 `RcModalProps<P>` 作为 props 类型，保证 `modal` / `query` 协议一致
- 提交成功后调用 `query?.refresh()` 刷新列表；删除走 `RcModal.confirm` 并传入 `tableData` 以自动翻页
- `loadDetail` / `handleClose` / `handleSubmit` 用 `useRef` 包裹，避免闭包过期与重复创建

## 注意事项

- 所有接口返回值必须符合 `BaseResult<T>` 格式（包含 `success` 字段）
- `RcTable` 的 `exported` 仅适用于简单表头，复杂嵌套表头请自行调用 `RcExcel`
- 主题通过 styled-components `ThemeProvider` 注入 antd `GlobalToken` + `prefixCls`

## 强制规则

1. **所有弹窗尽量都使用 `RcModal.useModal`** 管理显隐状态，避免自行用 `useState` 维护 `visible` 与 `params`。通过 `modal.open(params)` / `modal.close()` / `modal.visable` / `modal.params` 统一调用协议，弹窗组件 props 统一使用 `RcModalProps<P>`。

   ```tsx
   // 推荐
   const modal = RcModal.useModal<EditParams>()
   <Button onClick={() => modal.open({ id: record.id })}>编辑</Button>
   <EditModal modal={modal} query={query} />

   // 不推荐
   const [visible, setVisible] = useState(false)
   const [current, setCurrent] = useState<Row>()
   ```

2. **所有需要用户二次确认再调用接口的操作尽量都用 `RcModal.confirm`**（如删除、禁用、批量操作、状态切换等），避免手动拼 `Modal.confirm` + 调用接口 + 刷新列表的样板代码。传入 `query` 自动刷新，传入 `tableData` 自动处理"删除最后一条数据后翻页"。

   ```tsx
   // 推荐
   <a
     onClick={() =>
       RcModal.confirm(
         { api: deleteApi, params: record.id, query, tableData: tableRef.current?.getCurrentData() },
         { title: '删除', content: '确认删除该条数据？' },
       )
     }
   >
     删除
   </a>

   // 不推荐
   Modal.confirm({
     title: '删除',
     onOk: async () => {
       const res = await deleteApi(record.id)
       if (res.success) query.refresh()
     },
   })
   ```

3. **简单详情页面直接复用 `Edit` 弹窗**（仅展示表单字段、无复杂布局/Tab/多区块时），通过给 `Form` 设置只读态（例如统一 `disabled`、或按 `params.mode` 区分）实现"查看/编辑"共用同一组件。避免为只读展示单独新建一个弹窗。

   ```tsx
   // 推荐：复用 Edit，通过 mode 区分
   export type EditParams = { id?: string; mode?: 'view' | 'edit' }
   const modal = RcModal.useModal<EditParams>()
   <a onClick={() => modal.open({ id: record.id, mode: 'view' })}>查看</a>
   <a onClick={() => modal.open({ id: record.id, mode: 'edit' })}>编辑</a>

   // 在 Edit.tsx 内
   const readonly = modal.params?.mode === 'view'
   <Form form={form} disabled={readonly}>...</Form>
   ```

4. **复杂详情页面独立一个 `Detail.tsx` 弹窗**（涉及 Descriptions、Tabs、多区块、关联子表、图表等非表单展示时），同样使用 `RcModal.useModal` + `RcModalProps<P>` 协议，与 `Edit.tsx` 并列放置。弹窗内通常只有 `onCancel`，不需要 `onOk`。

   ```tsx
   // pages/user/Detail.tsx
   export type DetailParams = { id: string }
   const DetailModal: React.FC<RcModalProps<DetailParams>> = ({ modal }) => {
     // 加载详情、渲染 Descriptions / Tabs / 子表等
     return (
       <RcModal open={modal.visable} title="详情" footer={null} onCancel={modal.close} fullScreenModal>
         {/* 复杂内容 */}
       </RcModal>
     )
   }

   // pages/user/List.tsx
   const detailModal = RcModal.useModal<DetailParams>()
   const editModal = RcModal.useModal<EditParams>()
   <a onClick={() => detailModal.open({ id: record.id })}>详情</a>
   <a onClick={() => editModal.open({ id: record.id })}>编辑</a>
   <DetailModal modal={detailModal} />
   <EditModal modal={editModal} query={query} />
   ```
