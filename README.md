# ecaptureQ

一个基于 Tauri + React + TypeScript 构建的跨平台网络数据包捕获和分析工具，支持桌面端（Linux）和移动端（Android）
目前在构建中

## 功能特性

- 🌐 **实时数据包监控** - 通过 WebSocket 实时接收和显示 HTTP 数据包
- 📱 **跨平台支持** - 支持 Linux 桌面平台和 Android 移动平台
- 🎨 **现代化 UI** - 基于 Tailwind CSS 的响应式设计，支持暗黑模式
- 🔧 **可配置设置** - 灵活的配置选项，包括 WebSocket 连接、显示设置等
- 📊 **详细分析** - 支持查看请求/响应详情、Headers、Body 等信息
- 🚀 **高性能** - Rust 后端 + React 前端，保证流畅的用户体验

## 技术架构

### 前端技术栈
- **React 18** - 用户界面框架
- **TypeScript** - 类型安全的 JavaScript
- **Tailwind CSS** - 实用优先的 CSS 框架
- **Vite** - 快速的前端构建工具
- **Lucide React** - 现代化图标库

### 后端技术栈
- **Tauri 2.x** - 跨平台应用程序框架
- **Rust** - 系统级编程语言，保证性能和安全性
- **WebSocket** - 实时数据传输

### 核心组件
```
src/
├── components/          # React 组件
│   ├── ConnectionStatus.tsx    # 连接状态显示
│   ├── PacketCard.tsx         # 数据包卡片
│   ├── PacketDetailModal.tsx  # 数据包详情弹窗
│   └── PacketList.tsx         # 数据包列表
├── windows/            # 窗口组件
│   ├── MainWindow.tsx         # 主窗口
│   └── SettingsWindow.tsx     # 设置窗口
├── services/           # 服务层
│   └── websocketService.ts    # WebSocket 服务
├── hooks/              # React Hooks
│   └── useConfig.ts           # 配置管理
├── types/              # TypeScript 类型定义
│   └── index.ts
└── utils/              # 工具函数
    └── httpParser.ts          # HTTP 解析工具
```

## 开发环境要求

### 系统要求
- **Node.js** >= 18.0.0
- **Rust** >= 1.77.0 (通过 [rustup](https://rustup.rs/) 安装)
- **PNPM** >= 8.0.0 (推荐) 或 npm/yarn

### 移动端开发额外要求
- **Android Studio** (用于 Android 开发)
- **Java JDK** >= 11

## 快速开始

### 1. 克隆项目
```bash
git clone <repository-url>
cd ecaptureQ
```

### 2. 安装依赖
```bash
# 使用 PNPM (推荐)
pnpm install

# 或使用 npm
npm install
```

### 3. 开发模式

#### 桌面端开发
```bash
# 启动开发服务器
pnpm tauri dev

# 或者分步启动
pnpm dev          # 启动前端开发服务器
pnpm tauri dev    # 启动 Tauri 应用
```

#### Android 开发
```bash
# 首次运行需要初始化 Android 项目
pnpm tauri android init

# 启动 Android 开发模式
pnpm tauri android dev
```

### 4. 生产构建

#### 桌面端构建
```bash
# 构建桌面应用
pnpm tauri build
```
构建产物位于 `src-tauri/target/release/bundle/` 目录下。

#### Android 构建
```bash
# 构建 Android APK (Debug)
pnpm tauri android build

# 构建 Android APK (Release)
pnpm tauri android build --release
```
APK 文件位于 `src-tauri/gen/android/app/build/outputs/apk/` 目录下。

## 配置说明

### WebSocket 连接
应用通过 WebSocket 连接接收数据包数据。默认配置：
- **开发环境**: `ws://localhost:8080/packets`
- **生产环境**: 根据实际部署情况配置

### 配置项
- `websocketUrl`: WebSocket 服务器地址
- `autoScroll`: 是否自动滚动到最新数据包
- `showTimestamp`: 是否显示时间戳
- `maxPackets`: 最大保存的数据包数量
- `filterEnabled`: 是否启用过滤器

## 项目结构说明

### 前端路由
- **主界面**: 显示数据包列表和连接状态
- **设置界面**: 应用配置管理
- **移动端**: 使用 Hash 路由实现单页面应用

### 数据流
1. WebSocket 服务接收网络数据包
2. 数据通过 WebSocket 传输到前端
3. React 组件实时更新显示
4. 用户可以查看详情、配置设置等

## 开发说明

### 样式系统
项目使用 Tailwind CSS，支持响应式设计和暗黑模式：
- 使用 `@layer components` 定义公共组件样式
- 支持暗黑模式切换 (`dark:` 前缀)
- 响应式断点设计 (移动端优先)

### 字体配置
使用思源黑体作为主要字体，支持中英文显示：
```css
font-family: 'Source Han Sans CN', -apple-system, BlinkMacSystemFont, ...
```

### 构建优化
- **字体文件**: 仅保留 regular 权重，减少包体积
- **代码分割**: 通过动态导入优化加载性能
- **资源压缩**: Vite 自动处理资源优化

## 故障排除

### 常见问题

1. **端口占用错误**
   ```bash
   # 检查端口占用
   lsof -i :1422
   # 或更换端口配置
   ```

2. **Android 开发模式连接问题**
   - 确保 Vite 开发服务器监听 `0.0.0.0`
   - 检查防火墙设置
   - 确认设备与开发机在同一网络

3. **依赖安装失败**
   ```bash
   # 清除缓存重新安装
   pnpm store prune
   pnpm install
   ```

### 调试技巧
- 使用浏览器开发者工具调试前端
- 查看 Tauri 控制台输出调试后端
- 使用 `console.log` 追踪数据流

## 贡献指南

1. Fork 项目
2. 创建功能分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开 Pull Request

## 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情。

## 更新日志

### v0.1.0
- 初始版本发布
- 支持基本的数据包捕获和显示功能
- 跨平台支持（桌面端 + Android）
- 现代化 UI 设计

---

## 相关链接

- [Tauri 官方文档](https://tauri.app/)
- [React 官方文档](https://react.dev/)
- [Tailwind CSS 文档](https://tailwindcss.com/)
- [Vite 官方文档](https://vitejs.dev/)
