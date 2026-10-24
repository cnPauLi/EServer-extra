# EServer 运行时资源

本仓库保存 EServer 桌面端打包所需的**运行时二进制资源**，已从主仓库中拆分出来独立维护。

主仓库不包含本目录内容，**构建前必须先执行 `npm run fetch:extra`**，把本仓库拉到主仓库根目录的 `extra/` 下。主仓库通过 `.gitignore` 忽略 `extra/`，本地改动不会被主仓库跟踪。

## 目录结构

```
extra/
├── darwin/
│   ├── arm64/Core/          Apple Silicon 运行时
│   │   ├── Library/         第三方动态库（openssl、icu、curl、zlib 等）
│   │   ├── bin/mkcert       本地 HTTPS 证书工具
│   │   ├── config/          子应用清单 childApp.json 与图标
│   │   ├── script/          PHP 扩展编译脚本
│   │   ├── initFiles/       首次安装下发的 www、custom 模板
│   │   ├── update/          版本升级包与 update.json
│   │   └── installOrUpdate  首次安装 / 覆盖安装标记
│   └── x64/Core/            Intel Mac 运行时（结构同上）
└── win32/
    ├── EServerService.exe   Windows 服务程序
    ├── windowsapi.dll       Windows API 封装
    └── core/                Windows 运行时（结构同 macOS）
```

## 关键约定

1. **`Library` 不可删减**。php-fpm、nginx 等二进制以 `/Applications/EServer/Library/<lib>/lib/*.dylib` 的绝对路径链接动态库，缺少任意一个都会导致服务无法启动。
2. **`mkcert` 必须存在**。应用启动时会用 `Core/bin/mkcert` 检查并安装本地根证书；缺失会直接抛错。
3. **PHP 扩展编译依赖 `Library` 的 `include/`、`lib/pkgconfig` 与 `zlib`**，例如 `swoole`（`--enable-openssl`）、`xlswriter`（`--with-zlib-dir`）、`imagick`。
4. **php / nginx / mysql / phpMyAdmin / Composer 等子应用安装包不在本仓库**。它们由下载服务 `http://dl.eserver.devcn.top/childApp/**` 按需分发。
5. **`initFiles/downloads` 不入库**。离线安装包体积大且与下载服务重复，已在 `.gitignore` 中排除。
6. **运行时数据目录不入库**。`settings.json`、`childApp`、`www`、`temp`、`downloads`、`database`、`etc`、`custom` 均为运行期生成，`bin/*` 只保留 `mkcert`。

## 在主仓库中拉取

```bash
# 首次拉取 / 同步到默认源（gitee，私有仓库）
npm run fetch:extra

# 只校验本地 extra 是否完整，不联网
npm run check:extra
```

`build:mac`、`build:win`、`build:linux` 会自动先执行 `fetch:extra`。

### 环境变量

| 变量 | 说明 |
| --- | --- |
| `ESERVER_EXTRA_REPO` | 优先使用的仓库地址，覆盖默认 gitee 源 |
| `ESERVER_EXTRA_FALLBACK_REPO` | 备源地址，前面的来源都失败时使用 |
| `ESERVER_EXTRA_BRANCH` | 分支名，默认 `main` |
| `ESERVER_EXTRA_TOKEN` | HTTPS 访问令牌，用于 CI 拉取私有仓库 |
| `ESERVER_EXTRA_SKIP_FETCH=1` | 跳过同步，仅校验完整性（离线构建用） |

### 脚本参数

```bash
node scripts/fetch-extra.mjs --check     # 仅校验
node scripts/fetch-extra.mjs --force     # extra 存在但不是 Git 仓库时，删除后重新克隆
```

## 远端仓库

本仓库同时维护在两个地址，内容一致，均为私有：

| 远端名 | 地址 | 说明 |
| --- | --- | --- |
| `gitee` | `git@gitee.com:dragonlhp/EServer-extra.git` | 主源，构建默认优先从这里拉取 |
| `github` | `git@github.com:cnPauLi/EServer-extra.git` | 镜像备份 |

主仓库的 `fetch:extra` 会按 gitee → github 的顺序尝试，任一可用即可构建。

## 更新本仓库

`extra` 内的改动只提交到本仓库，不进入主仓库。**两个远端都要推送**：

```bash
git -C extra add -A
git -C extra commit -m "update: 说明改动内容"
git -C extra push gitee main
git -C extra push github main
```

注意 `git -C extra push`（不带远端名）只会推上游 `gitee`，github 需要显式推送。

主仓库执行 `npm run fetch:extra` 时会 `fetch` 后 `reset --hard`，**本地未推送的 extra 改动会被覆盖**，请先提交并推送。

### macOS 可执行权限

`Library/**/bin` 下的可执行文件需要可执行位。在主仓库根目录执行：

```bash
./libraryAddExec.sh
```

该脚本会 `chmod +x` 并在本仓库索引中记录 `100755`。

## 分支与版本

- 主分支为 `main`，与主仓库发布节奏保持一致。
- 不按平台拆分为多个仓库，`darwin/arm64`、`darwin/x64`、`win32` 同仓维护。
- 许可与主仓库保持一致，见主仓库 `LICENSE`。
