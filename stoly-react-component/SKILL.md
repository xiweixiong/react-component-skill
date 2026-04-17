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
import { RcTable, RcTableForm } from '@stoly/react-component'

// 1. 创建查询条件
const query = RcTableForm.useQuery<MyQuery>({ pageNum: 1, pageSize: 20 })

// 2. 使用表格
<RcTable<Row, Params, Result, MyQuery>
  query={query}
  api={fetchList}
  queryFormat={(q) => ({ page: q.pageNum, size: q.pageSize, ...q })}
  dataFormat={(res) => ({ rows: res.data.list, total: res.data.total })}
  columns={columns}
  showIndex          // 序号列
  exported           // 右键导出 Excel（简单表头时使用）
  rowKey="id"
/>
```

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
  <Form.Item name="name" label="姓名"><Input /></Form.Item>
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

```tsx
import { RcTable, RcTableForm, RcModal } from '@stoly/react-component'

interface Query extends RcQueryData { name?: string }
interface Row { id: number; name: string }

const ListPage = () => {
  const query = RcTableForm.useQuery<Query>()
  const controller = RcTableForm.useController()
  const modal = RcModal.useModal<Row>()
  const tableRef = useRef<RcTableRef<Row>>(null)

  const columns = [
    { title: '姓名', dataIndex: 'name', key: 'name' },
    {
      ...RcTable.OperateColumnProps,
      render: (_: unknown, record: Row) => (
        <Space>
          <a onClick={() => modal.open(record)}>编辑</a>
          <a onClick={() => RcModal.confirm({ api: deleteApi, params: record.id, query })}>删除</a>
        </Space>
      ),
    },
  ]

  return (
    <>
      <RcTableForm<Query> query={query} controller={controller}>
        <Form.Item name="name" label="姓名"><Input /></Form.Item>
      </RcTableForm>
      <RcTable<Row, Params, Result, Query>
        ref={tableRef}
        query={query}
        api={fetchList}
        queryFormat={(q) => q}
        dataFormat={(res) => ({ rows: res.data.list, total: res.data.total })}
        columns={columns}
        rowKey="id"
      />
      <RcModal title="编辑" open={modal.visable} onCancel={modal.close}>
        <EditForm modal={modal} query={query} />
      </RcModal>
    </>
  )
}
```

## 注意事项

- 所有接口返回值必须符合 `BaseResult<T>` 格式（包含 `success` 字段）
- `RcTable` 的 `exported` 仅适用于简单表头，复杂嵌套表头请自行调用 `RcExcel`
- `RcModal.confirm` 传入 `tableData` 时会自动处理"删除最后一条数据后翻页"逻辑
- 主题通过 styled-components `ThemeProvider` 注入 antd `GlobalToken` + `prefixCls`
