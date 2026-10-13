# Open With

PI-Desktop 插件 `io.github.eureka-0x01.open-with`：输入框底栏左侧一颗贴合宿主风格的**分裂按钮**
`[📁 Open With][▾]`（全部使用宿主 `--ds-*` 设计令牌，自动跟随明暗主题）：

- **主区点击** → 用系统文件管理器（资源管理器）打开当前工作目录；
- **▾ 点击** → 应用管理弹层（`pi.ui.openLayer`）：应用列表跟随按钮下拉，
  添加 / 编辑表单居中显示——**添加时直接粘贴 exe 完整路径**，
  保存前校验必须以 `.exe` 结尾（Windows）。

资源管理器是**内置默认**：硬编码 `locked: true`，任何机器都有，不可编辑、不可删除。

## 数据模型

自定义应用存插件 settings 的 `apps` 数组（`pi.plugin.getSettings` / `setSettings`）：

```json
{ "id": "vscode", "label": "VSCode", "command": "D:/apps/Code.exe", "args": ["{{dir}}"], "icon": "data:image/png;base64,…" }
```

- 表单只需 **名称 + exe 路径** 两项：粘贴 exe 完整路径，主进程校验
  `\.exe$`（大小写不敏感，仅 Windows；`.bat`/`.cmd`/裸命令如 `code` 会被拒绝）
  与**文件存在**，并**自动清洗粘贴带入的隐形字符**（方向控制符/零宽字符/包裹引号
  ——从网页复制路径时常见的"打不开"根因）；
- **保存时提取该 exe 的图标**（PowerShell + System.Drawing，32×32 PNG 存为
  data URI），应用列表里直接显示程序本尊图标；提取失败自动回退默认图标；
  历史条目在下次编辑保存时会补上图标；
- `args` 对用户不可见，默认 `["{{dir}}"]`（用该程序打开当前项目目录）；
  数据层仍兼容完整 args（数组 / 字符串简写 / JSON 数组字符串）供 API 调用；
- 名称 ≤ 40 字符，最多 32 个应用，id 自动生成（撞号追加 `-2`/`-3`）。

## 交互流

```
分裂按钮主区 ─plugin.call("open",{id:"explorer"})→ 主进程 spawn(explorer,[dir])
▾          ─pi.ui.openLayer()→ 应用列表跟随 ▾ 按钮下拉；添加/编辑表单居中
  点应用行  ─plugin.call("open",{id})→ 主进程按 settings.apps 拉起
  添加/编辑 ─plugin.call("add"/"update",{label,command})→ 校验（.exe 结尾）+ 落盘
  删除     ─plugin.call("remove",{id})→ 校验（内置拒绝）+ 落盘
```

增删改与打开的结果 toast 由主进程统一发出；校验错误（非 .exe、空名称等）
内联显示在表单上方的错误行里。

> 为什么没有"浏览选 exe"？宿主给插件的唯一选择框 `pi.fs.requestDirectory`
> 写死了只能选文件夹（`openDirectory`），没有文件选择框；早期版本用
> "选文件夹 → 列出里面的 exe"绕行，后按需求改为直接粘贴路径 + `.exe` 校验。

## 文件结构

| 文件 | 角色 |
| --- | --- |
| `manifest.json` | 唯一权限 `renderer.extension`；`plugin.call` 白名单：open / list / add / update / remove |
| `main.js` | 主进程：应用列表（内置 + settings.apps）、`.exe` 校验、跨平台拉起、toast |
| `renderer/index.mjs` | 渲染模块：分裂按钮 + openLayer 弹层（React + react-dom，宿主 import map 提供） |
| `package.json` | `{"type":"commonjs"}`：防工作区 ESM 工程误判 main.js |

## 加载（查看效果）

1. PI-Desktop → 插件页 → **从本地目录加载** → 选 `D:\dev\code\other\open-with`；
   （注册表已有 dev 条目时，重启 PI-Desktop 同样生效）
2. 批准 `renderer.extension`（高危，唯一权限）；
3. 输入框底栏左侧出现 `[📁 Open With][▾]`：点主区开资源管理器，点 ▾ 管理应用。

## 分发

安装包位于 `dist/io.github.eureka-0x01.open-with-0.6.0.piplug`（store-only 未压缩
zip）。把这个文件发给对方，对方在 PI-Desktop 插件页**安装 .piplug** 即可——
不走插件中心、无需账号与审核。
重新打包：改代码后 commit + `git tag v<新版本>`，再用 PluginPack 出包
（manifest 的 `version` 与 changelog 记得同步更新）。

日后要上架插件中心（plugins.aiuo.net）：id 需改为 `io.github.<用户名>.open-with`
（当前 `local.` 前缀仅限本地/私发），源码推公开 GitHub 仓库并打 tag，控制台用
GitHub App 绑定仓库后上传。

## 开发循环（热重载）

- 从目录加载的插件**保存即热重载**：改 `main.js` / `renderer/index.mjs` 立即生效；
- 例外：扩大 `permissions` 不随热重载生效，需在插件页手动重载
  （本版权限是收窄，重载/重启即生效）。

## 后续迭代方向

1. 每个应用一条命令面板条目（`contributes.commands` 动态注册）；
2. 自定义应用排序 / 设默认（当前默认固定为资源管理器）；
3. 若仍想要图形化选 exe：扫开始菜单 `.lnk` 生成应用列表（绕开文件夹对话框）；
4. 打包发布：PluginPack 出 `.piplug`；上架 plugins.aiuo.net 需要打 tag 的源码仓库 +
   GitHub App 绑定 + 新版本号与发布说明。

## 权限与安全

- `renderer.extension`（高，唯一权限）：按钮与弹层画进主窗口，只调用白名单内的
  `plugin.call`；
- 保存时校验用 Node `existsSync` 确认文件存在；为显示图标，会调用系统
  PowerShell（绝对路径，一次、带超时）读取**该 exe 的图标资源**转为 PNG 存储，
  除此之外不读任何磁盘文件；
- 自定义打开方式只执行用户自己粘贴并保存的 exe 路径（保存前清洗 + 校验），
  detached 拉起即返回；
- 不联网、不碰剪贴板。
