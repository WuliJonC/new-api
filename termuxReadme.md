# 在 Termux/Zerotermux 中编译和部署 New API 指南

本文档旨在提供在 Android 的 Termux 或 Zerotermux 环境中编译和部署 New API 项目的详细步骤。这些步骤基于对项目结构和通用 Go、Node.js 应用构建流程的分析。

## 目录
1.  [前提条件](#前提条件)
2.  [详细步骤](#详细步骤)
    *   [阶段一：环境准备与源码获取](#阶段一环境准备与源码获取)
    *   [阶段二：构建前端应用 (动态版本号)](#阶段二构建前端应用-动态版本号)
    *   [阶段三：构建后端 Go 应用 (生成可执行文件)](#阶段三构建后端-go-应用-生成可执行文件)
    *   [阶段四：部署和运行](#阶段四部署和运行)
3.  [更新现有部署](#更新现有部署)
4.  [重要注意事项](#重要注意事项)
5.  [关于 Makefile](#关于-makefile)
6.  [简单故障排查](#简单故障排查)

## 前提条件

*   您的 Termux/Zerotermux 环境已正确安装和配置。
*   您有稳定的网络连接以下载依赖项和源码。
*   对 Linux 命令行有基本了解。

## 详细步骤

### 阶段一：环境准备与源码获取

1.  **更新并安装必要的工具:**
    在 Termux/Zerotermux 终端中执行以下命令。

    ```bash
    pkg update && pkg upgrade
    ```
    *   **解释:**
        *   `pkg update`: 更新可用软件包列表。
        *   `pkg upgrade`: 升级所有已安装的软件包到最新版本。
        *   `&&`: 逻辑与操作符，表示只有前一个命令成功执行后，才会执行后一个命令。

    ```bash
    pkg install git golang nodejs-lts
    ```
    *   **解释:**
        *   `pkg install`: Termux 的包管理器命令，用于安装新软件包。
        *   `git`: 分布式版本控制系统，用于克隆项目源代码。
        *   `golang`: Go 编程语言的编译器和工具链，用于编译后端应用。
        *   `nodejs-lts`: Node.js 的长期支持 (LTS) 版本，包含 npm (Node Package Manager)，用于构建前端应用。如果 `nodejs-lts` 不可用，可以尝试 `pkg install nodejs`。

2.  **获取项目源代码:**

    ```bash
    git clone https://github.com/Calcium-Ion/new-api.git
    ```
    *   **解释:**
        *   `git clone <URL>`: 从指定的 URL 克隆一个 Git 仓库到本地。

    ```bash
    cd new-api
    ```
    *   **解释:**
        *   `cd <directory>`: 更改当前工作目录到指定的 `<directory>`。此处进入刚克隆的项目根目录。

### 阶段二：构建前端应用 (动态版本号)

为了确保前端应用能显示正确的、与代码库状态一致的版本号，我们将从 Git 获取版本信息。

1.  **从 Git 获取版本信息并写入 `VERSION` 文件:**
    在项目根目录 (`new-api/`) 执行：

    ```bash
    git describe --tags --dirty --always > VERSION 2>/dev/null || echo "v0.0.0-unknown" > VERSION
    ```
    *   **解释:**
        *   `git describe --tags --dirty --always`:
            *   `--tags`: 查找当前提交历史上最近的标签。如果当前提交是标签，则显示标签名 (如 `v0.7.0`)。否则，显示类似 `v0.7.0-3-g123abc` (表示在 `v0.7.0` 标签后的第3个提交，提交哈希简写 `g123abc`)。
            *   `--dirty`: 如果工作目录有未提交的修改，会在版本号后附加 `-dirty`。
            *   `--always`: 如果找不到标签，则回退到显示当前提交的唯一简短哈希。
        *   `> VERSION`: 将 `git describe` 命令的输出重定向并覆盖写入到名为 `VERSION` 的文件中。
        *   `2>/dev/null`: 抑制 `git describe` 可能产生的错误输出。
        *   `|| echo "v0.0.0-unknown" > VERSION`: 如果 `git describe` 命令因错误失败 (例如，当前目录不是 Git 仓库，或没有任何提交/标签)，则执行 `echo` 命令，将默认版本号 "v0.0.0-unknown" 写入 `VERSION` 文件。
    *   *此步骤确保 `VERSION` 文件包含动态生成的版本信息，供后续前端构建使用。*

2.  **构建前端资源:**

    ```bash
    cd web
    ```
    *   **解释:** 进入前端项目的子目录。

    ```bash
    npm install
    ```
    *   **解释:**
        *   `npm install`: 读取 `web` 目录下的 `package.json` 文件，并下载安装其中声明的所有前端依赖项到 `node_modules` 目录。如果 `package-lock.json` 或 `yarn.lock` 存在，它会尝试安装精确版本。

    ```bash
    NODE_OPTIONS=--max-old-space-size=4096 DISABLE_ESLINT_PLUGIN='true' VITE_REACT_APP_VERSION=$(cat ../VERSION) npm run build
    # 如果您的项目使用 pnpm 作为包管理器，请将 npm run build 替换为 pnpm build
    ```
    *   **解释:**
        *   `DISABLE_ESLINT_PLUGIN='true'`: 设置一个环境变量，可能用于在构建过程中禁用 ESLint 插件，以避免某些 linting 错误中断构建。
        *   `VITE_REACT_APP_VERSION=$(cat ../VERSION)`: 设置一个名为 `VITE_REACT_APP_VERSION` 的环境变量。
            *   `$(cat ../VERSION)`: 读取项目根目录下 (上一级目录 `../`) 的 `VERSION` 文件的内容 (即我们通过 `git describe` 生成的版本号)。
            *   这个环境变量会被 Vite (前端构建工具) 用来将版本号嵌入到前端应用代码中。
        *   `npm run build`: 执行 `package.json` 中 `scripts` 部分定义的 `build` 命令，通常这会调用 Vite 或其他构建工具来编译和打包前端资源 (HTML, CSS, JavaScript)。
    *   构建完成后，前端的静态文件通常会生成在 `web/dist` 目录中。

    ```bash
    cd ..
    ```
    *   **解释:** 返回到项目的根目录。

### 阶段三：构建后端 Go 应用 (生成可执行文件)

1.  **编译 Go 后端程序:**
    确保您在项目的根目录 (`new-api/`)。

    ```bash
    go build -o new-api main.go
    ```
    *   **解释:**
        *   `go build`: Go 语言的编译命令。
        *   `-o new-api`: 指定输出的可执行文件的名称为 `new-api`。如果省略 `-o`，则默认输出文件名与包名或 `main.go` 所在目录名一致 (在 Unix-like 系统中通常是 `main`，Windows 上是 `main.exe`)。
        *   `main.go`: 指定包含 `main` 函数的 Go 源文件作为编译入口。
    *   此命令会编译整个 Go 项目，并将所有依赖链接到一个独立的可执行文件 `new-api` 中。

2.  **优化编译 (可选):**
    如果您希望生成的可执行文件更小，并移除调试信息，可以使用以下命令：

    ```bash
    go build -ldflags="-s -w" -o new-api main.go
    ```
    *   **解释:**
        *   `-ldflags`: 向链接器传递参数。
        *   `"-s -w"`:
            *   `-s`: 省略符号表 (symbol table)。
            *   `-w`: 省略 DWARF 调试信息。
        *   这可以显著减小最终可执行文件的大小，但会使得调试更加困难。

3.  **禁用 CGO 编译 (可选):**
    如果在编译过程中遇到与 C 语言编译器 (GCC/Clang) 或 C 库相关的错误，并且您确定项目不依赖 CGO (C Go bindings)，可以尝试禁用 CGO 进行编译：

    ```bash
    CGO_ENABLED=0 go build -ldflags="-s -w" -o new-api main.go
    ```
    *   **解释:**
        *   `CGO_ENABLED=0`: 设置环境变量，指示 Go 编译器在编译时不使用 CGO。
    *   *注意：如果项目确实需要 CGO，此选项会导致编译失败或运行时错误。*

### 阶段四：部署和运行

1.  **运行编译好的后端程序:**
    完成上述步骤后，您应该在项目根目录 (`new-api/`) 下看到一个名为 `new-api` 的可执行文件。

    ```bash
    ./new-api
    ```
    *   **解释:**
        *   `./new-api`: 执行当前目录下的 `new-api` 文件。
    *   程序启动后，它会开始监听配置的端口 (通常是 `3000`，具体请查阅项目文档或启动时的日志输出)。为了使其在后台运行，您可以使用 `nohup ./new-api &` 或 `screen` / `tmux` 等工具。

2.  **访问应用:**
    *   在您的设备浏览器中访问 `http://localhost:3000` (假设端口是 `3000`)。
    *   您应该能看到 New API 的前端界面，并且版本号应为您在 `VERSION` 文件中通过 `git describe` 生成的值。

## 更新现有部署

如果您之前已经按照上述步骤部署了 New API，并希望更新到最新版本，请按照以下步骤操作：

1.  **停止当前运行的应用 (如果正在运行):**
    *   如果您是直接在终端运行 `./new-api`，按下 `Ctrl+C` 即可停止。
    *   如果您使用了 `nohup` 或 `screen`/`tmux`，您需要找到对应的进程并结束它。
        *   查找进程ID (PID): `ps aux | grep new-api` (或者您给可执行文件起的名字)
        *   结束进程: `kill <PID>` (其中 `<PID>` 是您找到的进程号)

2.  **进入项目目录:**
    确保您在 `new-api` 项目的根目录下。

    ```bash
    cd /path/to/your/new-api # 替换为您的实际项目路径
    ```

3.  **拉取最新代码:**

    ```bash
    git pull origin main # 或者项目使用的主要分支，例如 master
    ```
    *   **解释:**
        *   `git pull`: 从远程仓库获取最新更改并尝试合并到当前本地分支。
        *   `origin main`: 指定从名为 `origin` 的远程仓库的 `main` 分支拉取。
    *   **处理冲突:** 如果 `git pull` 过程中出现合并冲突 (merge conflicts)，您需要手动解决这些冲突，然后提交更改 (`git add .` 和 `git commit`)，之后才能继续。

4.  **检查依赖更新 (可选但推荐):**
    *   **Go 依赖:**
        ```bash
        go mod tidy
        ```
        *   **解释:** `go mod tidy` 会确保 `go.mod` 文件与源代码中的依赖项匹配，移除未使用的依赖并添加任何缺失的依赖。
    *   **前端依赖:**
        ```bash
        cd web
        npm install # 或者 yarn install，取决于项目使用的包管理器
        cd ..
        ```
        *   **解释:** 重新运行 `npm install` 可以确保前端依赖根据 `package.json` 和 `package-lock.json` (如果存在) 更新到最新兼容版本。

5.  **重新构建应用 (重复阶段二和阶段三):**

    *   **更新 `VERSION` 文件 (自动获取最新 git 版本):**
        ```bash
        git describe --tags --dirty --always > VERSION 2>/dev/null || echo "v0.0.0-unknown" > VERSION
        ```

    *   **重新构建前端:**
        ```bash
        cd web
        # 注意：如果前端依赖没有变化，npm install 可能很快完成或什么都不做
        # npm install # 如果在上一步已经执行过，这里可以考虑跳过，除非构建失败
        NODE_OPTIONS=--max-old-space-size=4096 DISABLE_ESLINT_PLUGIN='true' VITE_REACT_APP_VERSION=$(cat ../VERSION) npm run build
        # 如果您的项目使用 pnpm 作为包管理器，请将 npm run build 替换为 pnpm build
        cd ..
        ```

    *   **重新构建后端:**
        ```bash
        go build -o new-api main.go
        # 或者使用优化编译命令:
        # go build -ldflags="-s -w" -o new-api main.go
        ```

6.  **重启应用:**
    与“阶段四：部署和运行”中的步骤1相同。

    ```bash
    ./new-api
    ```
    *   或者使用 `nohup ./new-api &` 或在 `screen`/`tmux` 会话中启动。

7.  **验证更新:**
    访问应用，检查功能是否正常，并确认前端显示的版本号是否已更新。

## 重要注意事项

*   **前端静态文件服务:**
    Go 后端程序 (通常在 `main.go` 或相关的路由配置如 `router/web-router.go` 中定义) 必须被正确配置以服务 `web/dist` 目录下的前端静态文件。如果前端无法加载或显示404错误，请检查这部分的配置。许多 Go Web 框架提供了内置的静态文件服务功能。

*   **资源消耗:**
    编译过程，尤其是前端的 `npm install` 和 `npm run build`，可能会消耗较多的设备资源 (CPU 和内存)。请确保您的 Termux 环境有足够的资源分配，并且设备性能足以支撑。编译大型项目在资源受限的移动设备上可能会非常缓慢或失败。

*   **项目特定配置:**
    *   **环境变量:** 查阅项目的官方 `README.md` (特别是“环境变量配置”和“部署”部分) 和 [官方Wiki](https://docs.newapi.pro/)，了解项目运行时可能需要的环境变量。例如，`SESSION_SECRET`, `CRYPTO_SECRET`, `SQL_DSN` (如果使用外部数据库) 等。
    *   **数据库:** 项目默认可能使用 SQLite。如果使用 Docker 部署，`README.md` 提到需要挂载 `/data` 目录。在 Termux 中直接运行时，确保程序有权限在工作目录或指定路径创建和写入 SQLite 数据库文件。如果配置为使用 MySQL 或 PostgreSQL，确保数据库服务可访问。更新时，数据库迁移可能是需要考虑的一个方面，但这通常由项目本身提供迁移脚本或指南。
    *   **端口:** 默认端口通常是 `3000`。如果需要更改，请查阅项目文档或代码，看是否可以通过配置文件或环境变量进行修改。

*   **依赖问题:**
    在 `npm install` 或 `go build` 过程中，可能会遇到特定于 Termux 环境的依赖问题或缺失的系统库。请仔细阅读错误信息，并尝试在网络上搜索针对 Termux 的解决方案。

## 关于 Makefile

项目根目录下的 `makefile` 文件为 `make` 工具提供了构建指令。
*   `make build-frontend`: 执行前端构建，与本文档阶段二的步骤类似。
*   `make start-backend`: 使用 `go run main.go` 启动后端，这主要用于开发，它会即时编译并运行，但不生成独立的可执行文件。

本文档中的步骤旨在创建一个更适合“部署”的场景，即生成一个独立的后端可执行文件 (`go build`)，并详细解释了每个步骤，包括动态版本号的生成。您可以将本文档的步骤视为 `makefile` 中指令的更细化和针对 Termux 部署的解释与扩展。

## 简单故障排查

*   **查看日志:** 程序启动后，关注终端输出的日志信息，它们通常会指示程序是否成功启动、监听哪个端口以及任何潜在的错误。
*   **权限问题:** 确保您对项目目录有读写权限，并且编译生成的可执行文件有执行权限 (`chmod +x ./new-api` 如果需要)。
*   **端口占用:** 如果提示端口已被占用，请检查是否有其他程序在使用该端口，或修改 New API 的监听端口。

希望这份详细的指南能帮助您成功在 Termux/Zerotermux 上编译、部署和更新 New API！