# icons-harmony

HarmonyOS NEXT（ArkUI）用的 ohpm 图标库：把上游官方 SVG 同步进 HAR 资源，对外提供**图标组件 + 图标名常量**。

两套图标集各自独立成包，互不依赖：

| 包 | 模块 | 图标集 | 风格 | 图标数 | 组件 | 许可 |
| --- | --- | --- | --- | --- | --- | --- |
| `@isfk/octicons` | `octicons/` | Primer [octicons](https://primer.style/octicons) `v19.41.0` | 填充型，24px | 366 | `OctIcon` | Apache-2.0 |
| `@isfk/lucide` | `lucide/` | [lucide](https://lucide.dev) `v1.61.0` | 描边型，24×24 | 2171 | `LucideIcon` | ISC |

## 安装

```shell
ohpm install @isfk/octicons
ohpm install @isfk/lucide
```

## 使用

两个组件都是 `@ComponentV2` 的 struct，参数只有两个：`icon`（图标名，`OctIcon` 默认 `home_24`、`LucideIcon` 默认 `lucide_house`）和 `color`（`ResourceColor`，默认 `Color.Black`）。
`color` 支持 `Color.Red` / `'#FF6699'` / `$r('app.color.xxx')`。尺寸由调用方给 `width` / `height`。

```ts
// octicons
import { AppsIcon, OctIcon } from '@isfk/octicons';

OctIcon({ icon: AppsIcon, color: Color.Red }).width(20).height(20)
```

```ts
// lucide
import { HouseIcon, LucideIcon } from '@isfk/lucide';

LucideIcon({ icon: HouseIcon, color: '#FF6699' }).width(20).height(20)
```

图标名常量就是资源名（`AppsIcon === 'apps_24'`、`HouseIcon === 'lucide_house'`），也可以直接传字符串：`OctIcon({ icon: 'person_24' })`、`LucideIcon({ icon: 'lucide_heart' })`。

### 着色机制（两者不同，容易踩）

| | octicons | lucide |
| --- | --- | --- |
| svg 形态 | `fill` 填充 | `fill="none"` + `stroke` |
| 着色方式 | `Image.fillColor()` | `Image.colorFilter()`（`BlendMode.SRC_IN`） |
| 原因 | 填充路径可直接换填色 | `fillColor` 对 `fill="none"` 不生效，只能整图混合着色 |

对使用者来说一致：传 `color` 即可，颜色落在填充 / 描边上。差异只在内部实现。

## 图标清单

- 全量列表：`octicons/Index.ets`、`lucide/Index.ets`（一行一个常量，按字母序，可直接搜索）
- 各模块的安装、用法与样式说明：`octicons/README.md`、`lucide/README.md`
- 版本变化（新增 / 改版图标）：`octicons/CHANGELOG.md`、`lucide/CHANGELOG.md`

## 仓库结构

```text
octicons/    @isfk/octicons 模块：Index.ets 导出 + src/main/ets/components/OctIcon.ets + 366 个 svg
lucide/      @isfk/lucide 模块：Index.ets 导出 + src/main/ets/components/LucideIcon.ets + 2171 个 svg
entry/       演示 App，不参与发布
scripts/     同步 / 校验脚本（引擎在 scripts/lib/，新图标集只加一个 profile）
AGENTS.md    agent 操作手册：维护流程、约定、排错
```

## 维护者：不要手改图标

`<模块>/src/main/resources/base/media/*.svg` 和 `<模块>/Index.ets` 都是**同步脚本的产物**，手改会在下次同步时被覆盖。改图标就调上游版本或同步范围，然后重跑脚本：

```shell
node scripts/sync-octicons.mjs --dry-run   # 先看差异
node scripts/sync-octicons.mjs             # 落盘：svg + Index.ets + README 版本号
node scripts/sync-octicons.mjs --check     # 校验（有差异退出码 2）
```

lucide 换成 `scripts/sync-lucide.mjs`。完整的发版流程（预检 → 同步 → 发版号 → 校验 → 构建 → 拆 HAR 验收 → 发布 → 汇报）、约定与排错见 [AGENTS.md](./AGENTS.md)。

## 许可

各包许可见 [octicons/LICENSE](./octicons/LICENSE)、[lucide/LICENSE](./lucide/LICENSE)；图标本身版权归上游项目所有。
