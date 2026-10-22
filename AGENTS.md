# AGENTS.md — icons-harmony

本仓把上游官方 SVG 同步进 HAR 资源，产出两个独立 ohpm 包：`@isfk/octicons`（`OctIcon`，填充型 24px）与 `@isfk/lucide`（`LucideIcon`，描边型 24×24）。

**这份文件是 agent 的操作手册**：维护流程、构建验收、发布、约定、排错都在这里。
面向使用者的文档是 `README.md`（仓库门面：包一览 / 安装 / 用法）和 `<模块>/README.md`（随 HAR 发布到 ohpm）——那两份不写流程细节；反过来这里也不写安装用法。改文档时守住这条分工线。

## 触发词：更新版本

主人说「更新版本」「同步图标」「同步 octicons / lucide」，把下面的流程走完一遍。
**两套图标集流程完全相同，只换脚本与目标模块**：

| 图标集 | 同步脚本 | 模块 | 发版号 |
| --- | --- | --- | --- |
| Octicons（填充型，24px） | `scripts/sync-octicons.mjs` | `octicons/` | `octicons/oh-package.json5` |
| lucide（描边型，24×24） | `scripts/sync-lucide.mjs` | `lucide/` | `lucide/oh-package.json5` |

下面以 octicons 为例写命令，换成 lucide 就把 `sync-octicons` → `sync-lucide`、`octicons` → `lucide`。

1. **预检** — `node scripts/sync-octicons.mjs --dry-run`
   完成标准：报出**上游版本 + 新增 / 更新 / 未变 / 本地多余**五个数字，并点名「更新」的图标（这些是上游改版，图形会变，要单独提醒主人）。
   `新增 0 / 更新 0` 即已最新——汇报后停下。
2. **同步** — `node scripts/sync-octicons.mjs`
   完成标准：新增 + 更新的 svg 已落盘，`Index.ets` 与 `octicons/README.md` 的「已同步 vX」都跟着更新。
   走默认参数，保持「原封同步」。
3. **发版号** — `octicons/oh-package.json5` 版本号只递增最后一位（`1.0.4` → `1.0.5`），主人另有指定再按指定。
   完成标准：`octicons/CHANGELOG.md` 顶部新增一条 `# vX.Y.Z Release` + 一行说明（同步的上游版本、新增图标数、改版图标）。
4. **校验** — `node scripts/sync-octicons.mjs --check`
   完成标准：退出码 `0`；且 `ls octicons/src/main/resources/base/media/*.svg | wc -l` 等于 `grep -c '^export const' octicons/Index.ets`。
5. **构建** — `hvigorw --mode module -p module=octicons@default -p product=default -p buildMode=release assembleHar --no-daemon`
   完成标准：`BUILD SUCCESSFUL`，且 `octicons/build/default/outputs/default/octicons.har` 是刚生成的。
   要带 `-p buildMode=release`：否则包内 `debug:true` 且带 `ets/sourceMaps.map`，ohpm 会警告「contains source code」。
6. **验收** — 拆 HAR 核对（HAR 是 **gzip + tar** 的 `package/...` 布局，`unzip` 打不开）
   ```shell
   gunzip -c octicons/build/default/outputs/default/octicons.har > /tmp/o.har.tar
   tar -tf /tmp/o.har.tar | grep -c 'base/media/.*\.svg$'              # == Index.ets 常量数
   tar -tf /tmp/o.har.tar | grep -cE '\.DS_Store|sourceMaps\.map'       # == 0
   tar -xOf /tmp/o.har.tar package/oh-package.json5                    # 版本号是新版、debug:false、repository 是本仓地址
   ```
   完成标准：svg 数量与常量数相等、新图标 0 缺失、无 macOS 垃圾与 sourcemap、包内版本号是新版、`repository` 是 `https://github.com/shenapps/icons-harmony`。
7. **发布** — 先自检，**再**发布：
   `node scripts/check-publish-env.mjs`
   完成标准：退出码 `0` → 直接 `ohpm publish`（`key_passphrase` 已配置，agent 可免交互发布）。
   退出码 `2` → 密码未配置，此时**停手**：让主人自己在交互式终端发，agent 硬发会触发 ohpm 的死循环刷日志直到 OOM。
   退出码 `1` → 环境有问题，先修。
   ```shell
   ohpm publish octicons/build/default/outputs/default/octicons.har
   ```
   「发布」指把 HAR 推到 ohpm registry，与 git `commit`/`push` 是两件事（后者始终由主人手动）。
   完成标准：命令返回成功即可。**注意：官方要审核** —— 提交后 `latest` 可能在一段时间内仍是上一版，**别当成失败去重发**。
   核实用（**只认包文档**；`/ohpm/<包>/<版本>` 这个端点对本仓所有版本都返回 `404`，不能拿它当判据）：
   `curl -s https://ohpm.openharmony.cn/ohpm/@isfk/<包> | grep -o '"latest":"[^"]*"'`
   目标版本已在 registry 上时**先改版本号**，ohpm 不能覆盖已发布版本。
8. **汇报** — 列出改了什么、HAR 的 `shasum -a 256`、以及 commit 信息建议（不提交）。

## 同步脚本速查

两个脚本选项几乎一致（lucide 没有 `--size`，固定 24），完整清单看 `node scripts/sync-<集>.mjs --help`：

| 选项 | 说明 |
| --- | --- |
| `-v, --version <x.y.z>` | 指定上游版本，默认取 registry latest |
| `--size <n>` | 图标尺寸，默认 24（octicons 可选 12 / 16 / 24） |
| `--dry-run` | 只汇报差异，不改动项目文件（仅写 tarball 缓存） |
| `--check` | 有差异时退出码 `2`，无差异 `0` |
| `--prune` | 删掉上游已移除的本地图标（默认只提示不删） |
| `--from-tarball <p>` / `--from-dir <p>` | 离线：读本地 `.tgz` 或已解压目录 |
| `--refresh` | 忽略缓存重新下载（缓存 `.octicons-cache/`、`.lucide-cache/`，已 gitignore） |
| `--proxy <url>` / `--downloader curl` | 直连不通时走代理；`curl` 天然尊重 `http_proxy` / `socks5` 等环境变量 |
| `--timeout <sec>` | 单次网络超时，默认 30 |
| `--media-dir` / `--index` / `--readme` | 覆盖默认输出路径，本仓一般不用 |
| `--no-index` / `--no-readme` | 不重生成 `Index.ets` / 不改 README 版本号 |
| `--sanitize` | 应急备用，**本仓不用**：去掉 `fill="currentColor"` / 根节点 `fill="none"` |
| `--json` / `-q` | JSON 输出 / 只输出错误 |

退出码：`0` 正常、`1` 出错、`2` `--check` 发现差异。

## 约定

- **图标与 `Index.ets` 都归脚本管**：`<模块>/src/main/resources/base/media/*.svg` 和 `<模块>/Index.ets` 是同步脚本的产物。要加图标、删图标、改图标，就调同步范围或上游版本再跑脚本，手改会在下次同步时被覆盖。
- **手写的只有这些**：两个组件（`octicons/src/main/ets/components/OctIcon.ets`、`lucide/src/main/ets/components/LucideIcon.ets`）、引擎与配置（`scripts/lib/icon-sync.mjs`、`scripts/lib/profiles.mjs`）、入口（`scripts/sync-{octicons,lucide}.mjs`）、发布自检（`scripts/check-*.mjs`）、文档（`README.md`、本文件、`<模块>/README.md`、`<模块>/CHANGELOG.md`）。新图标集就新加一个 profile（~60 行），不要复制引擎。
- **模块 README 只有「已同步 vX」那一行归脚本**：`profiles.mjs` 的 `readmeVersionRe` 按正则改版本号，其余内容手写。所以可以放心编辑模块 README 的其它段落，别动那行版本号。
- **仓库改过名（octicons-harmony → icons-harmony）**：源码与元数据已无旧名，但 `build/`、`.preview/`、`.hvigor/` 下的缓存二进制里可能残留旧地址 / 旧绝对路径。它们都在 gitignore 内且**不进 HAR**，不影响发布；只有怀疑增量构建异常时才考虑清理，且清理前先问主人。
- **资源名前缀规则（避免撞名）**：HAR 的资源都并进同一个 `app.media.*`，两套图标集必须能共存。octicons 用裸名 + 尺寸后缀（`alert_24`），lucide 统一加 `lucide_` 前缀（`lucide_arrow_down`）。新图标集必须自带前缀。
- **Octicons 原封同步**：svg 内容与上游 npm 包**逐字节一致**，脚本只把文件名的 `-` 换成 `_`。所以上游 `alert-fill-24.svg` 落盘为 `alert_fill_24.svg`，导出 `AlertFillIcon`。`--sanitize` 是应急开关，本仓不用。
- **lucide 有两处最小归一化**（其余原样）：① 去掉文件头 HTML 许可注释（许可见 `lucide/LICENSE`）；② `stroke="currentColor"` → `stroke="#000000"`（currentColor 初始值即黑，等价；避免渲染器不认 currentColor 时图标全透明）。颜色统一交给 `LucideIcon` 的 `colorFilter(SRC_IN)`，因为 lucide 是 `fill="none"` 的描边图标，`fillColor` 对它**不生效**。
- **lucide 导出名消歧**：`arrow-down-01` 与 `arrow-down-0-1` 是**两张不同图标**，PascalCase 后会撞成同一个常量，所以相邻纯数字段之间保留下划线（`ArrowDown01Icon` / `ArrowDown0_1Icon`）。引擎会在同步时直接报导出名冲突，不会拖到构建才炸。
- **本仓约定不改写 fill**：`flag_24`（根节点 `fill="none"` + 路径 `fill="currentColor"`）和 `vscode_24`（`fill="currentColor"`）按官方原样保留，最终颜色由 ArkUI 解析决定；其余 7 个带 `fill-rule` 的图标官方文档确认支持。
- **删文件先问**：`--prune` 默认关闭，本地多余图标只汇报不删。
- **`element/` 里不留占位资源**：`octicons/src/main/resources/base/element/` 曾有模板残留（`page_text_font_size` / `page_show`），会和 App、旧版依赖、hypium 撞名，报 `WARN: 'xxx' conflict, first declared` —— HAR 的资源会并进使用方资源表，库侧一个占位都不该有。注意资源数组**不允许为空**（`The array or object node 'float' cannot be empty`），所以只能**整个文件删掉**，不能清空数组。
- **发布凭据**：`~/.ohpm/.ohpmrc` 里的 `publish_id` + `key_path` + `key_passphrase`，键值都不要打印或写入任何文件/日志。私钥**必须带密码**（PKCS#1 PEM：`ssh-keygen -t rsa -b 4096 -m PEM -f <file>`；无密码的私钥 ohpm 直接报 `NotSupportPrivateKey`）。密码丢了只能重新生成密钥对，并到 ohpm 网站「OHPM公钥」里**新增公钥**（`publish_id` 是账号级的，不用换，见上节）。
- **发布 ≠ 提交**：`ohpm publish` 是本流程的最后一步，由 agent 执行；`git add`/`commit`/`push`/`tag` 始终由主人手动完成（全局规则 1）。
- **`author` 保持对象形态**：`<模块>/oh-package.json5` 里 `author` 必须是 `{ "name": ..., "url": ... }`（服务端要求 url 或 email，参照 `@ohos/axios` 只给 url 即可），写成字符串会被 400 拒。改了元数据必须**重新构建** —— 它是打进 HAR 的。
- `entry/` 是演示 App，不参与发布；它依赖的 `@isfk/octicons` 版本号与本地开发不同步属正常。

## 换发布密钥（官方步骤）

ohpm 只支持**加密密钥**认证 —— 生成时必须输入密码；无密码的私钥它直接拒（`NotSupportPrivateKey`）。密码丢失只能重来一遍：

```shell
# 1. 生成公私钥（会提示输入密码，必填）
ssh-keygen -m PEM -t RSA -b 4096 -f ~/.ssh/ohpm_key_v2

# 2. 配置私钥路径
ohpm config set key_path ~/.ssh/ohpm_key_v2

# 3. 配置发布码（个人中心 → OHPM公钥管理 里拿到的 publish_id）
ohpm config set publish_id <your-publishId>
```

**4. 在个人中心 → OHPM公钥管理里添加公钥**，粘贴 `~/.ssh/ohpm_key_v2.pub` 文件的内容（OpenSSH 单行格式，**不是** `openssl rsa -pubout` 的多行 PEM）。入口：`https://ohpm.openharmony.cn/#/cn/personalCenter` → 左侧「OHPM公钥」→ 右上「新增」→ 填「标题」+「将公钥粘贴在这里」→ 提交。

> **`publish_id` 是账号级的，换密钥不用换它**：它在个人中心的个人资料页，按钮叫「**复制发布码**」（点一下入剪贴板），本地就是 `ohpm config get publish_id` 那个值。公钥页面可以登记**多把**公钥（页面会显示「您当前的OHPM公钥数」），所以换钥匙只需新增公钥，`publish_id` 保持不动。

发布时 ohpm 会问 `what is your passphrase of the private key:`；想免交互就 `ohpm config set key_passphrase '<密码>'`（密码明文落在 `~/.ohpm/.ohpmrc`，先 `chmod 600 ~/.ohpm/.ohpmrc`）。

## 排错

| 现象 | 处理 |
| --- | --- |
| 同步后构建报 `Identifier 'XXX' has already been declared` | 导出常量重名（上游两张不同图标 PascalCase 后撞名，如 lucide 的 `arrow-down-01` 与 `arrow-down-0-1`）。引擎现在会在**同步阶段**就报错；去 `scripts/lib/profiles.mjs` 的 `select()` 补消歧规则 |
| 命令"卡住" | hvigor 常驻 daemon 是常见元凶：`hvigorw --stop-daemon`，构建时加 `--no-daemon` |
| 下载超时 / 慢 | `--proxy http://127.0.0.1:7890`；脚本也会读 `https_proxy`/`http_proxy` 并用 curl 兜底 |
| 想快点失败 | `--timeout 10` |
| 想复现历史版本 | `--from-tarball octicons-x.y.z.tgz` 或 `--from-dir <解压目录>` |
| 要重新下载 | `--refresh`（缓存 `.octicons-cache/`，已 gitignore） |
| 发布鉴权失败 | 依次查 `ohpm config get publish_id` / `key_path` / `key_passphrase`。此版 ohpm（26.0.0.630）**没有 `login` 命令**，鉴权只有 publish_id + 加密私钥签名这一条路 |
| 发布刷屏 `The content of private key in the key_path error` 然后 V8 崩溃 | 签名拿不到私钥密码（非交互环境无 TTY 可输入），而 ohpm 在签名失败处是**死循环**，会一直刷到 OOM。修：`ohpm config set key_passphrase '<密码>'`，或到交互式终端里发布 |
| 发布成功后 registry 上查不到新版本 | 官方审核期，`latest` 会停在上一版；别重发同版本。判断上架**只认包文档**：`curl -s https://ohpm.openharmony.cn/ohpm/@isfk/<包> \| grep -o '"latest":"[^"]*"'` —— `/ohpm/<包>/<版本>` 端点对**所有**版本（含早已上架的）都返回 `404`，不可用作判据 |
| 发布报 `HttpCode 400 The format of the OHPM package no author url or author email` | `author` 写成了字符串。改成对象 `{ "name": "isfk", "url": "https://github.com/isfk" }` 后**重新构建**（元数据打进 HAR）。`ohpm prepublish` **查不出这一条**，别指望它 |
| 构建报 `WARN: 'xxx' conflict, first declared` | 同名资源被两方声明。查证时先看资源任务是否 `UP-TO-DATE` —— 缓存会骗人，必须强制重合并（临时改一下某个资源文件内容）才能作数。典型声明方：`entry` 自己的模板占位、旧版依赖 `@isfk/octicons@1.0.2`、测试框 `@ohos/hypium` |
| HAR 里混入 `.DS_Store` | 它躲在 `octicons/src/main/resources/base/`（被全局 gitignore 挡住所以 git 看不到），删掉后重新构建 |
| HAR 里 `repository` 还是旧仓库名 | 用的是改名前的构建缓存。确认 `<模块>/oh-package.json5` 已是新地址后按第 5 步重新构建，再拆包核对 `package/oh-package.json5` |

脚本选项速查见上文「同步脚本速查」，完整清单看 `node scripts/sync-<集>.mjs --help`。
使用者视角的文档（安装、用法、图标清单）看 `README.md` 与 `<模块>/README.md`——别把流程细节写进那里。
