# seal-cad-uts CAD 文档预览插件使用手册

Demo：[Gitee](https://gitee.com/twofloor/seal-cad-uts-demo) · [GitHub](https://github.com/lavieAll/seal-cad-uts-demo) · [插件地址](https://ext.dcloud.net.cn/plugin?id=29697)

`seal-cad-uts` 是面向 uni-app x 应用的 CAD 文档预览插件，提供文档打开、图层控制、距离计算和内嵌预览组件。业务页面只需要传入本地文件路径和文档 ID，即可在手机或平板页面中展示 CAD 图纸。

当前 Android 平台支持 DXF、DWG 文档预览，支持内嵌预览、独立全屏预览、单指移动、双指缩放与平移、图层显隐和加载状态提示。下一步计划支持鸿蒙 Next 和 iOS。

👏👏👏欢迎W（BJGFCYY）或Q（2480621579）咨询。

## 支持平台

| 平台 | 状态 | 当前能力 |
| --- | --- | --- |
| Android | 已支持 | DXF / DWG 打开和预览、触摸交互、全屏预览 |
| 鸿蒙 Next | 计划支持 | 计划提供统一的公开 API 和预览组件 |
| iOS | 计划支持 | 计划提供统一的公开 API 和预览组件 |

支持的常用 CAD 图形包括直线、圆弧、圆、椭圆、多段线、样条、块、标注、文字、填充和部分三维面线框。复杂专有对象、外部参照、特殊字体和部分高级 CAD 特性可能无法完全还原，实际显示结果以文档内容为准。

当前支持文本型 DXF，暂不支持二进制 DXF。DWG 支持 R13 至 AutoCAD 2018 格式系列，单文件大小上限为 64 MB。主要用于模型空间预览，暂不提供 CAD 编辑、完整布局打印和三维旋转功能。

## 安装插件

通过 HBuilderX 的 uni-app x 插件方式安装 `seal-cad-uts`。安装完成后，在页面中导入插件 API，并使用 `seal-cad-viewer` 组件。

Android 项目首次安装或更新插件后，需要使用包含该插件的 Android 自定义基座运行。正式发行时，插件会随应用一起打包。

标准基座可以打开 DXF；DWG 预览依赖插件内的原生库，必须使用包含本插件的 Android 自定义基座。

示例中的文件选择、文件读取和预览需要启用 `uni-media`、`uni-fileSystemManager`、`uni-canvas` 模块；使用网络下载示例时还需要 `uni-network`。请在项目的 Android 模块配置中启用所需模块，然后制作自定义基座。

页面中使用组件前，请确保组件容器有明确的宽度和高度。推荐使用 `flex: 1`，或设置固定的 `height`。

## 快速开始

将以下模板、逻辑和样式合并到同一个 `.uvue` 页面，并在 `pages.json` 注册页面。示例文件 `/static/sample.dxf` 需要由应用提供，也可以替换为自己的本地文件。后续示例沿用此处的 `docId`、`revision`、`opening`、`status` 和 `openDocument()`。

### 导入 API

```ts
import { cadApi, CadDocInfo, CadFail, CadOpenOptions, CadPoint } from '@/uni_modules/seal-cad-uts'
```

### 页面模板

```vue
<template>
  <view class="page">
    <seal-cad-viewer
      class="viewer"
      :doc-id="docId"
      :revision="revision"
      :loading="opening"
      :loading-text="loadingText"
    />

    <text class="status">{{ status }}</text>
    <button :disabled="opening" @click="openSample">打开 CAD 文件</button>
  </view>
</template>
```

### 页面逻辑

```vue
<script setup lang="uts">
import { cadApi, CadDocInfo, CadFail } from '@/uni_modules/seal-cad-uts'

const docId = ref('')
const revision = ref(0)
const opening = ref(false)
const loadingText = ref('正在加载 CAD 文件')
const status = ref('请选择 CAD 文件')
let unloaded = false

function openDocument(path: string, fileName: string): void {
  if (opening.value || unloaded) return

  opening.value = true
  loadingText.value = `正在打开 ${fileName}`
  status.value = '正在读取文件'

  const type = fileName.toLowerCase().endsWith('.dwg') ? 'dwg' : 'dxf'
  const options: CadOpenOptions = {
    src: path,
    fileType: type,
    success: (doc: CadDocInfo) => {
      if (unloaded) { cadApi.close(doc.docId); return }
      const previousId = docId.value
      docId.value = doc.docId
      if (previousId.length > 0) cadApi.close(previousId)

      status.value = `已打开 ${fileName} · ${doc.entityCount} 个实体`
      if (doc.unsupported.length > 0) {
        status.value += `\n部分内容未完整显示：${doc.unsupported.join(', ')}`
      }
      if (doc.warnings.length > 0) status.value += '\n' + doc.warnings.join('\n')
    },
    fail: (error: CadFail) => {
      if (!unloaded) status.value = `${error.errCode}：${error.errMsg}`
    },
    complete: () => {
      if (!unloaded) opening.value = false
    }
  }
  cadApi.open(options)
}

function openSample(): void {
  openDocument('/static/sample.dxf', 'sample.dxf')
}

onResize(() => {
  revision.value++
})

onUnload(() => {
  unloaded = true
  if (docId.value.length > 0) cadApi.close(docId.value)
})
</script>
```

### 页面样式

```vue
<style>
.page {
  flex: 1;
  padding: 12px;
}

.viewer {
  flex: 1;
  min-height: 280px;
}

.status {
  margin-top: 12px;
  margin-bottom: 12px;
}
</style>
```

## 打开本地 DXF / DWG 文件

可以使用 uni-app x 文件选择器选择用户设备中的 CAD 文件。选择器返回的文件名用于判断格式，文件路径用于打开文件。

```ts
function chooseCadFile(): void {
  if (opening.value) return

  uni.chooseFile({
    count: 1,
    type: 'all',
    success: (result: ChooseFileSuccess) => {
      if (result.tempFiles.length == 0) return

      const file = result.tempFiles[0]
      const name = file.name.toLowerCase()
      if (!name.endsWith('.dxf') && !name.endsWith('.dwg')) {
        status.value = '请选择 DXF 或 DWG 文件'
        return
      }

      // 使用文件名判断格式，使用 file.path 打开文件。
      openDocument(file.path, file.name)
    },
    fail: (error: ChooseFileFail) => {
      status.value = error.errCode == 1101001
        ? '已取消文件选择'
        : `文件选择失败：${error.errMsg}`
    }
  })
}
```

部分 Android 文件选择器返回的路径是没有扩展名的 URI，因此建议始终显式传入 `fileType`。插件支持以下写法：

```ts
const options: CadOpenOptions = {
  src: file.path,
  fileType: 'dwg',
  success: (doc: CadDocInfo) => {
    docId.value = doc.docId
  },
  fail: (error: CadFail) => {
    status.value = error.errMsg
  }
}
cadApi.open(options)
```

## 打开网络文件

插件接收本地文件路径。网络文件需要先由业务层下载到本地，再将下载后的临时路径传给插件。

```ts
function downloadAndOpen(url: string, fileName: string): void {
  uni.downloadFile({
    url,
    success: (result: DownloadFileSuccess) => {
      if (result.statusCode != 200) {
        status.value = `下载失败：HTTP ${result.statusCode}`
        return
      }
      openDocument(result.tempFilePath, fileName)
    },
    fail: (error: DownloadFileFail) => {
      status.value = `下载失败：${error.errMsg}`
    }
  })
}
```

即使下载路径没有 `.dwg` 或 `.dxf` 后缀，也应根据原文件名传入 `fileType`。

## 直接打开 DXF 文本

如果业务层已经取得 DXF 文本，可以直接调用 `openText()`：

```ts
function openDxfContent(content: string): void {
  cadApi.openText(
    'memory.dxf',
    content,
    (doc: CadDocInfo) => {
      const previousId = docId.value
      docId.value = doc.docId
      if (previousId.length > 0) cadApi.close(previousId)
      status.value = `已打开 DXF · ${doc.entityCount} 个实体`
    },
    (error: CadFail) => {
      status.value = error.errMsg
    }
  )
}
```

`openText()` 只用于文本型 DXF，不用于 DWG 二进制文件。

## 预览组件

```html
<seal-cad-viewer
  :doc-id="docId"
  :revision="revision"
  :loading="opening"
  :loading-text="loadingText"
  class="viewer"
/>
```

组件属性如下：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `doc-id` | `string` | `cadApi.open()` 或 `openText()` 成功后返回的文档 ID |
| `revision` | `number` | 递增后重新绘制，图层变化和布局变化后使用 |
| `loading` | `boolean` | 文件打开期间显示加载遮罩 |
| `loading-text` | `string` | 加载提示文字，默认“正在加载 CAD 文件” |

### 触摸操作

| 操作 | 作用 |
| --- | --- |
| 单指拖动 | 移动图纸 |
| 双指张开 / 捏合 | 放大 / 缩小 |
| 双指同时移动 | 平移图纸 |

首次打开文档时组件会自动适配图纸范围。切换到新的 `docId` 会重新适配；仅更新 `revision` 会保留当前视图位置和缩放比例。

## 全屏预览

全屏预览建议使用单独页面，让 CAD 展示区域占据主要空间。

### 注册页面

在 `pages.json` 添加：

```json
{
  "path": "pages/cad-preview/cad-preview",
  "style": {
    "navigationStyle": "custom",
    "disableScroll": true
  }
}
```

### 全屏页面

```vue
<template>
  <view class="full-page">
    <view class="toolbar">
      <text @click="goBack">返回</text>
      <text>CAD 预览</text>
    </view>
    <seal-cad-viewer class="full-viewer" :doc-id="docId" :revision="revision" />
  </view>
</template>

<script setup lang="uts">
const docId = ref('')
const revision = ref(0)

onLoad((options) => {
  docId.value = options['docId'] ?? ''
})

onReady(() => {
  revision.value++
})

onResize(() => {
  revision.value++
})

function goBack(): void {
  uni.navigateBack()
}
</script>

<style>
.full-page {
  flex: 1;
}

.toolbar {
  flex-direction: row;
  align-items: center;
  padding: 12px;
}

.full-viewer {
  flex: 1;
}
</style>
```

### 跳转到全屏页

```ts
function openFullScreen(): void {
  if (docId.value.length == 0) {
    status.value = '请先打开 CAD 文件'
    return
  }

  uni.navigateTo({
    url: '/pages/cad-preview/cad-preview?docId=' + encodeURIComponent(docId.value)
  })
}
```

进入全屏页时不要立即关闭当前文档。返回首页后，确认不再使用文档时再调用 `cadApi.close()`。

来源页不要在 `onHide` 中释放文档；全屏页只负责展示，返回时不关闭共享文档。自定义导航栏还需根据应用的状态栏和安全区设置留出顶部空间。

## 图层控制

打开成功后可以通过 `get()` 读取图层，再使用 `setLayerVisible()` 控制图层显示状态：

```ts
function toggleFirstLayer(): void {
  const doc = cadApi.get(docId.value)
  if (doc == null || doc.layers.length == 0) {
    status.value = '当前文档没有可用图层'
    return
  }

  const layer = doc.layers[0]
  cadApi.setLayerVisible(doc.docId, layer.name, !layer.visible)
  revision.value++
  status.value = `${layer.name} 图层已${layer.visible ? '显示' : '隐藏'}`
}
```

图层名称来自 CAD 文档，建议在业务页面中使用 `doc.layers` 生成图层列表。

## 距离计算

`measureDistance()` 接受同一图纸坐标系中的两个点，返回两点距离：

```ts
const p1: CadPoint = { x: 0, y: 0 }
const p2: CadPoint = { x: 3, y: 4 }
const distance = cadApi.measureDistance(docId.value, p1, p2)

status.value = `距离：${distance}` // 5
```

当前输入点应为图纸坐标，不是屏幕触摸坐标；返回值单位与图纸坐标一致。

## API 说明

### `cadApi.open(options)`

打开本地 DXF 或 DWG 文件。

```ts
const options: CadOpenOptions = {
  src: '/static/sample.dxf',
  fileType: 'dxf',
  success: (doc: CadDocInfo) => {},
  fail: (error: CadFail) => {},
  complete: () => {}
}
cadApi.open(options)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `src` | `string` | 本地文件路径或可访问的本地 URI |
| `fileType` | `string` | `dxf` 或 `dwg`，建议明确传入 |
| `success` | `(doc: CadDocInfo) => void` | 成功回调 |
| `fail` | `(error: CadFail) => void` | 失败回调 |
| `complete` | `() => void` | 成功或失败后回调 |

### `cadApi.openText(src, content, success?, fail?)`

打开已经读取到内存中的文本型 DXF。

### `cadApi.get(docId)`

根据文档 ID 获取当前文档信息。文档不存在时返回 `null`。

### `cadApi.close(docId)`

关闭并释放当前文档。建议在替换文档或页面销毁时调用。

### `cadApi.setLayerVisible(docId, layer, visible)`

设置指定图层的显示状态。调用后递增组件的 `revision` 以刷新画面。

### `cadApi.measureDistance(docId, p1, p2)`

计算两个图纸坐标点之间的距离。

### `cadApi.hitTest(docId, point)`

当前版本暂不支持实体拾取、捕捉或触摸选点测量，请勿依赖此方法获取实体结果。

## 常用数据类型

公共类型直接从插件导入，无需在业务页面重新定义：

```ts
import { CadPoint, CadOpenOptions, CadDocInfo, CadLayer, CadBBox, CadFail } from '@/uni_modules/seal-cad-uts'
```

| 类型 | 常用字段 | 用途 |
| --- | --- | --- |
| `CadPoint` | `x: number`、`y: number` | 图纸坐标点 |
| `CadOpenOptions` | `src`、`fileType`、`success`、`fail`、`complete` | `cadApi.open()` 参数 |
| `CadLayer` | `name: string`、`visible: boolean`、`colorHex: string` | 图层名称、可见性和颜色 |
| `CadBBox` | `minX`、`minY`、`maxX`、`maxY`，均为 `number` | 图纸范围 |
| `CadFail` | `errCode: CadErrorCode`、`errMsg: string` | 失败信息 |

`CadDocInfo` 常用字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `docId` | `string` | 文档 ID，传给预览组件和后续 API |
| `source` | `string` | 打开时传入的来源 |
| `format` | `string` | `dxf` 或 `dwg` |
| `entityCount` | `number` | 实体数量，不等同于实际显示图形数量 |
| `layers` | `CadLayer[]` | 图层列表 |
| `bbox` | `CadBBox` | 图纸范围 |
| `unit` | `string` | 当前为 `unitless`，请勿直接当作毫米或米 |
| `warnings` | `string[]` | 文档打开提示 |
| `unsupported` | `string[]` | 未完整支持的内容提示 |

`docId` 只用于当前运行期间的文档操作，不要保存到本地作为永久文件标识。`bbox` 是图纸范围，`format` 为 `dxf` 或 `dwg`。

## 加载状态建议

文件打开时，推荐同时控制页面状态和组件 loading：

```html
<seal-cad-viewer
  :doc-id="docId"
  :loading="opening"
  :loading-text="loadingText"
/>
```

在 `success` 中设置新的 `docId`，在 `complete` 中将 `opening` 设为 `false`。用户重复点击时应暂时禁用打开按钮。

## 错误处理

| 错误码 | 常见原因 | 建议 |
| --- | --- | --- |
| `9020001` | 路径为空、文件不存在或无法读取 | 检查文件路径和系统文件访问权限 |
| `9020002` | 文件解析失败、DWG 打开失败或文件内容异常 | 确认文件格式、文件完整性和文件大小 |
| `9020003` | 文件类型不支持 | 只传入 `dxf` 或 `dwg` |

建议始终提供 `fail` 和 `complete` 回调，并在页面显示用户可理解的错误提示。

常见问题：

- **打开 DWG 后按 DXF 处理**：文件选择器返回无扩展名 URI 时，使用文件名判断格式，并显式传入 `fileType: 'dwg'`。
- **画布空白**：确认 `docId` 是成功回调返回的 ID，组件容器有高度，文档没有被提前关闭。
- **切换图层后没有刷新**：调用 `setLayerVisible()` 后递增 `revision`。
- **全屏页没有内容**：不要在跳转全屏时关闭文档，返回后再释放文档。
- **文件部分内容未显示**：检查 `warnings` 和 `unsupported`，并确认原文件中的对象类型。

## 版本适配计划

下一步将围绕统一的 API 和组件用法推进以下适配，减少业务页面的跨平台接入差异：

1. **鸿蒙 Next 适配**：完成 DXF / DWG 文件读取、文档预览组件、全屏页面和手机 / 平板触摸交互。
2. **iOS 适配**：完成 DXF / DWG 文件读取、文档预览组件、全屏页面、安全区适配和 iPhone / iPad 触摸交互。
3. **跨平台一致性验证**：使用相同的 API、示例图纸和交互流程验证三端显示结果。

鸿蒙 Next 和 iOS 适配完成并通过对应平台验证后，将在版本说明中公布可用版本和接入要求。在适配版本发布前，请以 Android 平台状态为准。
