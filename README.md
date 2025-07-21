# ecaptureQ

一个使用 Tauri 和 React 构建的跨平台网络抓包与分析工具。

## 🏛️ 项目架构 (Architecture)

`ecaptureQ` 采用 **Tauri** 框架，将高性能的 **Rust** 后端与现代化的 **React** 前端相结合。核心设计思想是：由 Rust 负责繁重的数据捕获和处理任务，并将结果高效地传递给 React UI 进行展示。

-----

### 1\. 后端架构 (Rust)

后端是整个应用的大脑，负责启动抓包、处理数据和响应前端请求。它主要由以下几个模块构成：

#### **核心服务 (`src-tauri/src/services`)**

这些是独立的后台任务，负责与外部进程和数据源交互。

  * **`CaptureManager`**:

      * **职责**: 管理核心抓包程序（在开发阶段为 `mock_ws.go` 编译的二进制文件）的生命周期。
      * **流程**:
        1.  从应用内释放二进制文件到应用数据目录。
        2.  赋予其可执行权限 (`0o755`)。
        3.  通过 `su` 权限启动子进程。
        4.  监听 `shutdown` 信号，通过发送 `SIGINT` 来优雅地终止子进程。

  * **`WebsocketService`**:

      * **职责**: 连接由核心抓包程序提供的 WebSocket 服务，接收实时数据包。
      * **流程**:
        1.  连接到 `ws://127.0.0.1:18088/ws`。
        2.  异步接收 JSON 格式的数据包。
        3.  为了性能，它会将数据包缓存起来，当达到一定数量 (`BATCH_SIZE`) 或超时 (`FLUSH_TIMEOUT`) 后，批量发送给 `DataFrameActor` 处理。

#### **数据核心 (`src-tauri/src/core`)**

这是数据处理和存储的中心，采用了 **Actor 模型** 来保证高性能和线程安全。

  * **`DataFrameActor`**:
      * **职责**: 作为一个独立的异步任务，专门负责管理一个 **Polars `DataFrame`**。`DataFrame` 是一个高性能的内存数据表。
      * **设计**: 使用 Actor 模型（通过 `tokio::mpsc` channel）可以避免多线程直接访问 `DataFrame` 带来的数据竞争和锁开销。所有对 `DataFrame` 的读写操作都通过消息传递进行。
      * **消息类型 (`ActorMessage`)**:
          * `UpdateBatch`: 从 `WebsocketService` 接收批量数据包，并将其追加到 `DataFrame` 中。
          * `QuerySql`: 接收来自前端的 SQL 查询请求，使用 Polars 内置的 SQL 引擎查询 `DataFrame` 并返回结果。

#### **Tauri 桥接层 (`src-tauri/tauri_bridge`)**

这是 Rust 后端与 JavaScript 前端的沟通桥梁。

  * **`commands.rs`**:
      * 定义了所有暴露给前端的接口，例如 `start_capture`、`stop_capture`、`get_all_data`、`get_incremental_data`。
      * 这些命令通过 Tauri 的 `invoke` 机制被前端调用。
  * **`state.rs`**:
      * 定义了全局共享的应用状态 `AppState`，由 Tauri 管理。
      * `AppState` 持有 `DataFrameActorHandle` (用于与 Actor 通信) 和后台任务的句柄，方便在不同命令之间共享状态和控制任务。

-----

### 2\. 前端架构 (React)

前端使用 `React`、`TypeScript` 和 `Tailwind CSS` 构建，负责用户交互和数据可视化。

#### **UI 组件 (`src/components`)**

  * **`NewPacketList`**: 列表是应用的核心。为了处理大量数据包而不卡顿，这里使用了 `react-window` 库实现了**虚拟列表**，只渲染视口内可见的列表项。
  * **`NewPacketCard`**: 单个数据包的卡片展示。
  * **`Controls`**: 包含“开始/停止/清空”等功能的控制栏。
  * **`DetailModal`**: 点击数据包后弹出的详情模态框，展示解码后的 `payload` 等信息。

#### **状态管理 (`src/hooks/useAppState.ts`)**

  * 整个应用的客户端状态由一个自定义 Hook `useAppState` 集中管理，这简化了状态逻辑。
  * 它负责管理 `isCapturing` (是否在抓包)、`packets` (数据包列表) 等核心状态。
  * **数据同步**: 当用户点击 "Start Capture" 后，`useAppState` 会首先调用 `getAllData` 加载一次全量数据，然后**启动一个定时器 (`setInterval`)，周期性地调用 `getIncrementalData` 来拉取增量数据**，实现近乎实时的数据更新。

#### **API 服务 (`src/services/apiService.ts`)**

  * 这是一个封装层，将对 Tauri 后端命令的调用 (`invoke`) 包装成易于使用的异步函数 (`startCapture`, `stopCapture` 等)。这使得业务逻辑与底层的 Tauri API 解耦。

-----

### 3\. 完整数据流 (从抓包到显示)

1.  **启动**: 用户点击 "Start" 按钮。
2.  **前端**: `useAppState` -\> `apiService.startCapture()` -\> `invoke('start_capture')`。
3.  **后端**: `start_capture` 命令被触发。
4.  **服务启动**: 后端创建并启动 `CaptureManager` 和 `WebsocketService` 两个异步任务。
5.  **模拟器运行**: `CaptureManager` 启动 `mock_ws.go` 编译的二进制文件。该程序开始通过其内置的 WebSocket 服务器广播模拟的数据包。
6.  **数据接收**: `WebsocketService` 连接到模拟器的 WebSocket，开始接收 JSON 格式的数据包。
7.  **数据入库**: `WebsocketService` 将数据包批量发送给 `DataFrameActor`。
8.  **数据处理**: `DataFrameActor` 将这批数据高效地写入 Polars `DataFrame`。
9.  **前端轮询**: 与此同时，前端的 `useAppState` Hook 中的定时器不断调用 `getIncrementalData` 命令。
10. **数据查询**: 该命令向 `DataFrameActor` 发送一个 SQL 查询（如 `SELECT * FROM packets OFFSET ...`），只获取新到达的数据。
11. **UI 更新**: 新数据返回到前端，`useAppState` 更新 `packets` 状态，`React` 自动渲染出新的数据包卡片。
12. **停止**: 用户点击 "Stop"，后端发送 `shutdown` 信号，所有服务和模拟器进程优雅退出，前端轮询停止。

-----

## 🚀 开发与编译 (Development & Build)

### 使用模拟数据进行开发

为了方便前端开发和后端数据处理逻辑的测试（无需依赖真实的 `eCapture` 环境），项目提供了一个 Go 语言编写的 WebSocket 模拟数据服务器。

  * **文件**: `scripts/mock_ws.go`
  * **功能**:
      * 启动一个和 `eCapture` 接口完全相同的 WebSocket 服务器 (`ws://127.0.0.1:18088/ws`)。
      * 模拟真实网络环境中的“突发”和“平静”阶段，以不规则的模式持续发送符合 `PacketData` 结构的数据。
      * 这使得开发者可以在任何机器上进行端到端的应用功能测试。

### 编译流程 (Android)

编译流程现在包含将 Go 模拟器交叉编译为 Android 二进制文件，并将其打包到最终的 APK 中。

#### **先决条件**

  * **Go** 编译器 (\>= 1.18)
  * **Rust** 工具链 (rustc, cargo)
  * **pnpm** 包管理器
  * **Android SDK 和 NDK** (可通过 Android Studio 安装)
  * 配置好 `ANDROID_HOME` 和 `NDK_HOME` 环境变量

##### Android  证书签名
为了在 Android 上发布应用，你需要一个签名证书。可以使用 `keytool` 命令生成一个新的密钥库文件。
```shell
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

参考：https://v2.tauri.app/zh-cn/distribute/sign/android/
#### **编译步骤**

1.  **编译 Go 模拟服务器**:
    打开终端，执行以下命令，将 Go 源码交叉编译为 Android ARM64 架构的二进制文件，并放置到 `src-tauri/binaries` 目录下。

    ```bash
    cd ./scripts
    CGO_ENABLED=0 GOOS=android GOARCH=arm64 go build -o ./../src-tauri/binaries/android_test-aarch64-linux-android ./scripts/mock_ws.go
    ```

      * `GOOS=android GOARCH=arm64`: 指定目标平台为 Android ARM64。
      * `CGO_ENABLED=0`: 生成静态链接的二进制文件，不依赖 C 库。
      * `-o ...`: 指定输出路径和文件名，这个路径和文件名在 Rust 代码 (`src-tauri/src/services/capture.rs`) 中被硬编码使用。

2.  **安装前端依赖**:

    ```bash
    pnpm install
    ```

3.  **初始化 Android 项目 (首次运行时需要)**:

    ```bash
    pnpm tauri android init
    ```

4.  **构建最终的 Android 应用**:
    这个命令会把 Tauri 的 Rust 核心、前端代码和你在第一步中编译的 Go 二进制文件一起打包成一个 APK。

    ```bash
    pnpm tauri android build --target aarch64
    ```

-----

## 📁 目录结构

```
.
├── public/                  # 静态资源
├── scripts/
│   ├── log.sh               # 用于查看 Android logcat 的辅助脚本
│   └── mock_ws.go           # Go 语言编写的 WebSocket 模拟服务器
├── src/                     # 前端 React 源码
│   ├── components/          # React 组件
│   ├── hooks/               # 自定义 Hooks (如 useAppState)
│   ├── services/            # 前端服务 (如 apiService)
│   ├── types/               # TypeScript 类型定义
│   └── App.tsx              # 主应用组件
├── src-tauri/               # 后端 Rust 源码
│   ├── binaries/            # 嵌入的二进制文件 (由 mock_ws.go 编译而来)
│   ├── capabilities/        # Tauri 权限配置
│   ├── src/
│   │   ├── core/            # 核心数据处理 (Actor, Polars)
│   │   ├── services/        # 后台服务 (Capture, WebSocket)
│   │   └── tauri_bridge/    # 与前端交互的命令和状态
│   │   └── lib.rs           # Rust lib 的主入口
│   ├── Cargo.toml           # Rust 依赖配置
│   └── tauri.conf.json      # Tauri 应用配置
└── package.json             # 前端项目和脚本配置
```