# “小程序敲木鱼”技术架构设计文档

**文档版本**: 1.0
**创建日期**: 2026-01-27

## 1. 产品形态分析：小程序 vs. 游戏

### 1.1. 结论

经过对产品需求和用户场景的分析，我们明确建议将此项目定位为 **“小程序”**，而非纯粹的“游戏”。

### 1.2. 核心理由

1.  **产品本质与用户心智 (Product Essence & User Mentality):**
    *   **小程序**: 强调“用完即走”的工具属性。用户打开“敲木鱼”的核心诉求是快速获得一个**解压、静心**的工具，而非进入一个复杂、沉浸的游戏世界。小程序的轻量化、便捷性与此完美契合。
    *   **游戏**: 通常意味着更重的交互、更长的用户在线时长、更复杂的游戏系统。将“敲木鱼”做成重度游戏，反而可能违背了用户追求“宁静”、“简单”的初衷，增加了用户的认知负担。

2.  **技术选型与开发成本 (Technology & Development Cost):**
    *   **小程序 (使用 uniapp):**
        *   **跨平台优势**: 一套代码可以发布到微信小程序、支付宝小程序、App 等多个平台，极大地降低了开发和维护成本。
        *   **生态成熟**: `uniapp` 基于 Vue.js，前端开发者上手快，社区活跃，有大量现成的UI库和插件可用。
        *   **性能满足**: 对于PRD中描述的交互（点击、滑动、简单3D），现代小程序引擎的性能足以支撑，可以实现流畅的用户体验。
    *   **游戏 (使用游戏引擎):**
        *   **开发门槛高**: 游戏引擎（如 Cocos, Laya）的学习曲线更陡峭。
        *   **性能过剩**: 为了实现“敲木鱼”的核心功能，动用重型游戏引擎如同“杀鸡用牛刀”，会导致项目包体过大，启动变慢。

3.  **用户获取与传播 (User Acquisition & Virality):**
    *   **小程序**: 依托于微信等超级App的社交生态，极易通过好友分享、社群传播等方式获取用户，传播路径短，获客成本低。
    *   **游戏**: 获客成本更高，周期更长。

### 1.3. 关键技术点考量：3D渲染

PRD中明确要求3D渲染。在小程序框架下，我们有成熟的方案应对：
*   **方案**: 在 `uniapp` 中使用 `renderjs` 模块，它允许我们在视图层运行一个独立的、可操作DOM的 `three.js` 实例。或者，直接利用小程序平台提供的 `WebGL` 画布组件。
*   **优势**: 此方案能将3D渲染的复杂计算与小程序的逻辑层解耦，保证UI渲染和业务逻辑的流畅性，完全能满足本项目对3D木鱼模型展示和交互的需求。

---

## 2. 技术架构与选型

### 2.1. 核心框架
*   **跨端框架**: **`uni-app`**
    *   **原因**: 基于 Vue.js 语法，学习成本低，生态丰富。一次开发，多端发布（iOS, Android, H5, 以及各种小程序），完美契合项目快速迭代和广泛分发的需求。

### 2.2. 状态管理
*   **状态管理库**: **`Pinia`**
    *   **原因**: 作为 Vue 3 官方推荐的状态管理库，`Pinia` 提供了极简的 API、完整的 TypeScript 支持和出色的开发工具集成。相比 Vuex，它更轻量、更直观，非常适合中小型项目的状态管理。

### 2.3. 3D渲染
*   **3D引擎**: **`Three.js`**
    *   **原因**: 轻量而强大的 WebGL 库，在Web和小程序领域拥有最广泛的社区支持和最丰富的文档资源。足以实现本项目所需的模型加载、光影效果、材质渲染和交互动画。将结合 `uniapp` 的 `renderjs` 功能来实现。

### 2.4. UI与样式
*   **UI组件库**: **不引入大型第三方UI库**
    *   **原因**: 本项目UI界面相对简单，核心是3D交互区域。引入大型UI库（如uView）会增加包体积。建议基于 `uniapp` 内置组件，手动封装少量必要的业务组件，保持项目的轻量化。
*   **CSS预处理器**: **`SCSS`**
    *   **原因**: 提供了变量、嵌套、混合（Mixin）等强大功能，可以极大地提高样式代码的可维护性和复用性。

### 2.5. 架构图
```mermaid
graph TD
    subgraph 用户界面 (View Layer)
        A[Pages & Components] -- 使用 --> B(uni-app内置组件)
        A -- 样式 --> C(SCSS)
        D[3D Canvas via Render.js] -- 渲染 --> E(Three.js)
    end

    subgraph 逻辑层 (Logic Layer)
        F[Vue 3 / uni-app API]
        G[Pinia Store] -- 管理状态 --> F
        H[API Service] -- 请求 --> I[Backend API]
        F -- 调用 --> H
        F -- 交互 --> D
    end

    A -- 数据绑定/事件 --> F
```

---

## 3. 目录结构规范

一个清晰的目录结构是项目可维护性的基石。我们采用功能化和模块化相结合的方式组织代码。

```
/
├── api/                  # API 请求模块
│   ├── user.js
│   └── resource.js
├── assets/               # 静态资源 (本地图片、音频、3D模型等)
│   ├── images/
│   ├── audio/
│   └── models/
├── components/           # 全局通用组件
│   ├── BaseIcon.vue
│   └── LoadingSpinner.vue
├── pages/                # 主包页面
│   ├── index/            # 首页 (敲木鱼主页)
│   │   ├── index.vue
│   │   └── components/   # 页面级私有组件
│   │       └── Fish3D.vue
│   └── settings/         # 设置页
│       └── settings.vue
├── static/               # 静态资源 (会被直接拷贝到根目录)
├── store/                # Pinia 状态管理
│   ├── index.js          # 主入口
│   └── modules/
│       ├── user.js
│       └── settings.js
├── styles/               # 全局样式
│   ├── _variables.scss   # SCSS变量
│   ├── _mixins.scss      # SCSS混合
│   └── global.scss       # 全局样式表
├── utils/                # 工具函数
│   ├── request.js        # HTTP请求封装
│   └── index.js
├── App.vue               # 应用入口文件
├── main.js               # Vue初始化入口
├── manifest.json         # uniapp配置文件
├── pages.json            # 页面路由配置
└── uni.scss              # uniapp内置的全局样式变量
```

---

## 4. 编码规范

### 4.1. 代码风格
*   **Linter/Formatter**: 强制使用 **`ESLint`** 和 **`Prettier`**。
    *   `ESLint` 负责代码质量检查（如未使用的变量、潜在的逻辑错误）。
    *   `Prettier` 负责代码格式化（如缩进、分号、引号），保证团队风格一致。
    *   将在项目配置 `pre-commit hook`，在提交代码前自动格式化和检查。

### 4.2. 命名规范
*   **文件**: 大驼峰命名法 (PascalCase)，如 `UserInfo.vue`。
*   **组件**:
    *   组件名应为多个单词，避免与HTML标签冲突。
    *   基础通用组件以 `Base` 开头，如 `BaseButton.vue`。
    *   页面级私有组件应放在对应页面的 `components` 目录下。
*   **变量/函数**: 小驼峰命名法 (camelCase)，如 `const userName = '...';`。
*   **常量**: 全大写，下划线分隔，如 `const MAX_COUNT = 100;`。

### 4.3. Vue/uniapp 规范
*   **v-for 必须带 :key**: 且 `key` 必须是唯一且稳定的值。
*   **Props 定义**: 必须明确类型 (`type`)，最好提供默认值 (`default`)。
*   **组件通信**: 遵循单向数据流。父到子通过 `props`，子到父通过 `emits`。跨层级或兄弟组件通信使用 `Pinia`。
*   **逻辑复用**: 优先使用 Vue 3 的 **Composition API (`<script setup>`)**，将相关逻辑封装在 `composables` (或 `hooks`) 目录中。

### 4.4. 注释规范
*   **组件/页面文件**: 顶部应有注释块，说明组件用途、作者、创建日期。
*   **复杂函数**: 函数上方应有注释，解释其功能、参数和返回值。
*   **复杂逻辑**: 在代码块内部，对难以理解的逻辑行进行行内或行上注释。

```javascript
/**
 * @description 获取用户信息
 * @param {string} userId - 用户ID
 * @returns {Promise<Object>} 用户信息
 */
async function getUserInfo(userId) {
  // ...
}
```
