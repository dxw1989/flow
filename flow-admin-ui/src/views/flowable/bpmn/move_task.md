# Flowable 前端设计器迁移说明

## 1. 管理端现状概览

### 1.1 流程模型列表页
- `/flowable/bpmn/modelInfo/index.vue` 通过左侧 `FlowCategoryTree` 加载分类，右侧 `BasicTable` 载入流程模型分页数据，并在工具栏提供“新增”入口。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/index.vue†L1-L107】
- `createActions` 在操作列挂载“预览 / 发布 / 停用 / 修改 / 删除”等动作，所有动作都依赖 `/@/api/flowable/bpmn/modelInfo` 提供的 REST 接口；迁移时需要逐项确认新页面的交互仍调用这些接口或替换为新 API。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/index.vue†L119-L167】
- 分类筛选通过 `handleSelect` 与 `reload({ searchInfo })` 联动；迁移到新前端时需保留该筛选参数以兼容现有查询接口。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/index.vue†L205-L236】

### 1.2 流程设计入口
- `/flowable/bpmn/designer/index.vue` 仅负责生成嵌入式 `FramePage`，在开发模式下指向 `/flow-bpmn-front/index.html/#/bpmn/designer?modelId=...`，生产模式指向 `/flow-bpmn/index.html` 构建产物；迁移时需要同步更新 iframe 目标地址及缓存逻辑（`useFrameKeepAlive`）。其中 `flow-bpmn-front` 对应本仓库 `public/flow-bpmn-front` 目录内的调试版静态资源，方便 `pnpm dev` 时直接从 Vite 静态目录读取；`flow-bpmn` 则对应后端 `flow-admin` 工程内 `/static/flow-bpmn` 的发布版资源，生产打包时会随后端一起部署。虽然两个目录下的 `js/app.js`、`js/chunk-vendors.js` 是 webpack 打包后的结果，但仍可通过替换这些静态文件来自定义：开发态在 `flow-admin-ui/public/flow-bpmn-front` 下覆盖，生产态需把同一套构建产物同步到 `flow-admin/src/main/resources/static/flow-bpmn`。【F:flow-admin-ui/src/views/flowable/bpmn/designer/index.vue†L1-L49】【F:flow-admin-ui/public/flow-bpmn-front/index.html†L1-L24】【F:flow-admin-ui/public/flow-bpmn/index.html†L1-L24】【F:flow-admin/src/main/resources/static/flow-bpmn/index.html†L1-L24】【F:flow-admin/src/main/resources/static/flow-bpmn-front/index.html†L1-L24】

### 1.3 模型弹窗 `ModelInfoModal`
- 弹窗顶部使用 `RadioGroup` 在“表单设计 / 流程设计 / 扩展设置”之间切换，并分别加载 `formDesignerUrl` 与 `flowDesignerUrl` 的 iframe；迁移时需保证新的设计器同样暴露 `FramePage` 兼容的入口。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L1-L35】【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L236-L249】
- `window.submitFormInfo` 会在表单设计器内被调用：它先保存流程模型（`saveFlowInfo`），再保存表单定义（`saveFormInfo`），并在成功后把回写数据赋值给 `flowBaseInfo` 与 `formBaseInfo`；迁移新设计器时必须维持这一路径或提供新的双向通信方式。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L110-L148】
- 当首次切换到“流程设计”页签时才懒加载 `/flow-bpmn-front` 资源并强制刷新 iframe；这一逻辑可避免旧设计器残留缓存，迁移时应确认新的静态资源同样支持延迟加载。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L236-L248】
- “扩展设置”页签绑定 `BasicForm`，其中 `modelInfoFormSchema` 负责渲染扩展字段；若迁移后扩展字段有所调整，应同步更新表单 schema。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L27-L33】【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L196-L223】

## 2. 嵌入式 Flowable 设计器应用

### 2.1 布局入口 `BpmnDesigner`
- `/public/flow-bpmn-front/js/app.js` 中的 `BpmnDesigner` 模板负责组合流程画布组件 `my-process-designer`、右侧属性面板 `my-properties-panel` 与“偏好设置”抽屉；迁移时需要保留同样的布局骨架或在新设计器里提供等效功能（键盘快捷键、流程 ID/名称、引擎前缀切换等）。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L609-L684】
- `my-process-designer` 双向绑定流程 XML（`xmlString`），并通过 `element-click` / `init-finished` 事件驱动属性面板刷新；确保新画布同样暴露事件给外层管理界面。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L609-L640】
- 偏好设置抽屉允许修改流程 ID、名称、引擎前缀、是否启用模拟、禁用双击重命名等参数并触发 `reloadProcessDesigner` / `changeLabelVisibleStatus` 等函数；迁移时若改造为新配置面板，需要同步这些能力。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L640-L684】

### 2.2 全局交互约定
- 设计器通过 `window.bpmnInstances` 暴露 `modeler`、`modeling`、`moddle` 等对象，供属性面板更新 BPMN 元素；迁移到新的前端框架时必须继续导出这些全局实例或封装兼容层。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L190-L238】
- 旧设计器的表单调用 `window.submitFormInfo` 与父窗口通信（见 1.3），并依赖 `modelId` 查询后台模型数据；新设计器若改用 postMessage 或 API，需要在 `ModelInfoModal` 中同步调整。

## 3. 右侧属性面板拆解

### 3.1 面板框架
- 属性面板 `my-properties-panel` 提供展开/收起按钮（`.process-properties-bar`）及 `el-collapse` 折叠面板。迁移时应复刻该交互以便用户在窄屏下隐藏面板。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】
- 折叠项按节点类型动态展示：常规信息、任务、自由审批配置、子流程结构、监听器、表单、消息与信号、流转条件、扩展属性、边界事件、中间捕获事件、描述信息等。迁移时需要基于节点类型条件渲染对应子组件，以免在无效节点上显示冗余设置。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】

### 3.2 常规信息（`ElementBaseInfo`）
- 负责读取当前 `businessObject`，判断是否为第一个用户任务节点，从而允许“设置为提交人”；并在更新属性时调用 `window.bpmnInstances.modeling.updateProperties` 或 `updateModdleProperties` 同步到图形与 DI 对象。迁移时应保留这些更新方式以确保流程图 ID 与图形元素同步。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L190-L238】
- 提供 `handleSetToInitiator`、`updateBaseInfo` 等方法设置节点名称、跳过表达式和参与者信息；新面板需要实现同样的快捷操作避免回归体验倒退。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L190-L238】

### 3.3 任务配置（`ElementTask` 及其子组件）
- `ElementTask` 汇总用户任务、接收任务等子组件（`task-components/UserTask.vue`、`ReceiveTask.vue` 等），用于配置候选人、任务服务、审批方式等；迁移时需逐个确认子组件内部对 `window.bpmnInstances` 的依赖并在新框架中重建。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】
- 用户任务子组件会处理候选人（`candidateUsers`/`candidateGroups`）字段、办理人选人弹窗及会签配置；这些字段与流程运行时权限密切相关，迁移时不可丢失。

### 3.4 其他关键面板
- **自由审批配置** (`custom-approve-setting`)：扩展 Flowable 运行时的自由审批节点，需要迁移定制字段和 API 对接逻辑。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】
- **结构设置** (`element-structure`)：仅在 `CallActivity` 可见，用于配置子流程引用。
- **监听器** (`element-listeners`)：绑定执行/任务监听器集合，需同步 `extensionElements` 的写入逻辑。
- **表单设置** (`element-form`)：绑定表单 Key、表单变量等字段；需与模型表单库联动。
- **消息与信号 / 流转条件** (`signal-and-massage`, `flow-condition`)：依赖 `updateExtensionElement` 写入 `flowable:Condition`、`flowable:ErrorRef` 等扩展属性；迁移时注意 Flowable 自定义命名空间。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】
- **扩展属性列表** (`element-properties`)：提供扩展属性的增删改 UI，与流程变量、业务键关联。
- **边界事件 / 中间捕获事件** (`element-boundary-info`, `intermediate-catch-event`)：根据事件类型展示定时器、条件、错误等字段并通过 `updateExtensionElement` 写入 BPMN 模型。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L190-L238】【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L633-L684】
- **描述信息** (`element-other-config`)：承载节点备注、帮助文本等元信息。

### 3.5 获取并覆盖属性面板源码
- `my-properties-panel` 实际对应 `package/refactor/PropertiesPanel.vue`，其脚本在打包产物内完整保留了对各子面板的引用（`ElementBaseInfo`、`CustomApproveSetting`、`ElementBoundaryInfo` 等），只是被 webpack 包装成 `eval` 语句并写入单个 `app.js`。【F:flow-admin/src/main/resources/static/flow-bpmn-front/js/app.js†L170-L332】
- 模板也随同保留在同一 bundle 中，第 633 行附近可以看到 `el-collapse-item` 组合出“自由审批配置”“监听器”“描述信息”等折叠面板，因此运行时依旧渲染出和源码一致的 DOM；如在应用中看不到该面板，多半是因为外层 `BpmnDesigner` 关闭了 `showDesigner` 或缓存的旧 iframe 未更新，需要在父级重新触发 `reloadIndex` 或清空 `useFrameKeepAlive` 的缓存。【F:flow-admin/src/main/resources/static/flow-bpmn-front/js/app.js†L625-L684】【F:flow-admin/src/main/resources/static/flow-bpmn-front/js/app.js†L1061-L1077】
- 如果要定制属性面板，不建议直接在压缩后的字符串中硬改；可以把 `app.js` 中 `PropertiesPanel.vue` 对应的 `eval("...\nexport default {...}")` 片段拷贝到一个新的 Vue SFC，修正 `import` 路径（例如参照 `package/refactor` 目录结构），然后使用 `vue-cli-service build` 或 `vite build` 重新生成 `app.js`/`chunk-vendors.js` 并覆盖到 `public/flow-bpmn-front/js` 与后端 `static/flow-bpmn/js`，从而一次性替换整个面板实现。【F:flow-admin/src/main/resources/static/flow-bpmn-front/js/app.js†L170-L684】
- 若短期内无法还原原始工程，也可以在 `PropertiesPanel.vue` 的模块字符串中搜索关键词（如“自由审批配置”），定位到需要调整的模板或逻辑片段，再配合 `flow-admin-ui/public/flow-bpmn-front/index.html` 手动替换静态资源；只是要注意替换后需同步更新开发态和生产态目录，避免 iframe 仍引用旧缓存。【F:flow-admin-ui/public/flow-bpmn-front/index.html†L13-L22】【F:flow-admin/src/main/resources/static/flow-bpmn-front/index.html†L13-L22】

## 4. 迁移步骤建议
1. **拆分仓库结构**：将旧的 `/public/flow-bpmn-front` 构建产物替换为新设计器资源，同时保证 `ModelInfoModal`/`designer/index.vue` 的 iframe 路径指向新资源，并在 `import.meta.env.DEV` 条件下配置本地调试地址。【F:flow-admin-ui/src/views/flowable/bpmn/designer/index.vue†L31-L48】
2. **保留双向通信协议**：若新设计器不再直接挂载到 `window`, 需要在 iframe 与父窗口之间建立显式的 `postMessage` 协议，以传递 `submitFormInfo`、模型名称回显、属性保存事件等关键操作。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L110-L248】
3. **还原属性面板能力**：按照第 3 章列出的组件逐一迁移，确保常规信息、任务、条件、边界事件等配置项都能写回到 `BpmnModeler`。迁移完成后应联通后台 `/flowable/bpmn/modelInfo` 的保存/发布接口进行端到端验证。【F:flow-admin-ui/public/flow-bpmn-front/js/app.js†L190-L684】
4. **同步表单设计**：表单设计器依旧从 `/form-making/index.html` 加载并通过 `loadFormInfo` 注入表单 JSON，如需替换为新表单系统，请在 `ModelInfoModal` 中更新 iframe URL 与回调逻辑。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/ModelInfoModal.vue†L160-L185】
5. **验证权限与分类**：迁移后需确认分类筛选、应用系统（`appSn`）字段、发布/停用动作与原接口完全兼容，以免影响线上流程模型管理。【F:flow-admin-ui/src/views/flowable/bpmn/modelInfo/index.vue†L52-L167】

> 建议在迁移完成后，编写端到端用例覆盖：模型创建→表单设计→流程设计→属性配置→发布→停用，以确保新页面行为与旧版一致。
