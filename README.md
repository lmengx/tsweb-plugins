# tsweb-plugins —— TSWeb 插件发行仓库

本仓库是 **TSWeb（cordis 版）的插件发行通道**：只存放**发行包与元数据**，不存放插件源码。

- 插件源码在宿主仓库 `tsweb-cordis` 中开发与维护
- 本仓库的用途是让面板/终端能下载并安装插件：`market-list` / `market-install` / `market-update`

## 当前发行兼容性（2026-10-25）

当前索引提供 27 个适配 Cordis/Vue 宿主的插件，包含市场自身；基础插件仍随 Windows / Windows-lite 发行。
新包声明 `tsweb.accessApi=1`；带前端的包同时声明 `tsweb.clientApi=2`，使用编译后的 Vue 客户端。
权限声明须与实际 `auth.access.policyApp` 路由实现一致，不能仅给旧包补字段。
这些包面向已完成新架构升级的宿主，旧宿主应先升级。旧发行文件保留用于追溯，默认索引仅选择每个插件的最高版本。

- 安装 `config-anything` 前，先安装并启用 `file-manager`；游戏服也需要对应的 `FileManager.dll`。
- 游戏服插件的部署使用新版 TsWebHost 与已有 TShock REST；部署 token 需要 `tshock.rest.manage`，不需要 7979 端口。
- 本次进行了隔离宿主安装、启停及浏览器初始化验证；未将其视为真实游戏服业务与长期运行验收。
- 更新索引后，在面板插件市场点击“刷新索引”。下载仍会验证包的 SHA-256；不要覆盖同版本包来绕过更新流程。

## 目录结构

```
market.json                  集中索引（由发布脚本自动生成，请勿手改）
packages/<name>-<ver>.tgz    发行包
README.md
```

## 发行包格式

每个 `.tgz` 顶层为 `<name>/`，内容为**运行所需**，不含源码：

```
<name>/
├── package.json        { name, version, tsweb: { accessApi: 1, clientApi: 2, autoload, bundled, client, dll } }
├── index.js            cordis 插件对象（export const name + export async function apply）
├── dist/               编译后的 Vue 前端（可选）
└── plugin/*.dll        C# 游戏服端 DLL（可选，安装后下发到 TShock 服务器热加载）
```

约定：`package.json` 的 `name` 必须与目录名、索引中的 `name` 三者一致——宿主加载器与注册表都以插件名为键。

## market.json（schema 2）

```jsonc
{
  "schema": 2,
  "repo": "lmengx/tsweb-plugins",
  "branch": "main",
  "updated": "2026-09-26T14:19:41.259Z",
  "mirrors": [
    { "id": "github",  "base": "https://raw.githubusercontent.com/{repo}/{branch}" },
    { "id": "ghproxy", "base": "https://gh-proxy.com/https://raw.githubusercontent.com/{repo}/{branch}" }
  ],
  "packages": [
    {
      "name": "online-stats",
      "version": "0.2.0",
      "description": "…",
      "author": null,             // 可选
      "tags": ["数据统计"], // 分类标签：插件在 package.json 用 tsweb.tags 声明，发布时写入索引
      "accessApi": 1,          // 当前权限契约
      "bundled": false,          // 是否属于"自带插件"（自带插件随宿主发行，一般不走本仓库）
      "hasClient": true,         // 是否含前端半
      "dll": ["./plugin/*.dll"], // C# 半声明（安装后用于下发到游戏服）
      "requires": ["world-preview"], // 插件依赖：安装/更新时缺依赖会先弹窗确认再下载（无依赖省略）
      "deploy": "all",           // DLL 下发模式（三态，缺省 all）：all=自动全量 / ask=安装时弹窗选 / manual=静默不下发
      "url": "packages/online-stats-0.2.0.tgz",
      "sha256": "…",             // 整包哈希，客户端下载后强制校验
      "size": 27865,
      "publishedAt": "…"
    }
  ]
}
```

客户端（`market` 插件）按 `mirrors` 顺序逐个尝试下载，失败自动切换并记住成功来源；下载后校验 `sha256`，不符即拒绝安装。

`requires` 是插件依赖：市场在安装/更新前解析依赖闭包，本机缺依赖时**不直接下载**，而是返回 409 让面板弹窗
列出依赖包与大小；管理员确认后先装依赖（含自动启用已装但禁用的依赖），再装目标。
目前 `history`、`house` 依赖 `world-preview`（世界预览，约 9.6 MB，含图格贴图），
`world-preview` 自 base 2.4.0 起从 base 拆出，独立发布。

## 如何发布新版本

1. 在 `tsweb-cordis` 仓库内改插件源码，提升 `package.json` 的 `version`
2. 打包并上架（一条命令，自动复制发行包并重建 `market.json`）：

```bash
cd tsweb-cordis
node scripts/publish-plugin.mjs <name>          # 单个插件
node scripts/publish-plugin.mjs --all-market    # 全部非自带插件
node scripts/publish-plugin.mjs --reindex       # 只重建索引（不打包）
```

3. 提交并推送本仓库（发布脚本会打印下一步命令）：

```bash
cd tsweb-plugins
git add -A
git commit -m "release: <name> <version>"
git push
```

4. 面板或终端执行 `market-refresh` 即可看到新版本；`market-update <name>` 完成更新（失败自动回滚到旧版本）。

## 镜像

| id | 说明 |
|----|------|
| `cloudflare` | **唯一来源**：自建 R2 直链 `https://cdn.plugin.tlry.me`（桶 `tlry-plugins`，键同本仓库目录） |

`github` 与 `ghproxy` 已于 2026-10-30 弃用：前者在国内解析到黑洞地址（TCP 不通），后者只是它的公共前缀代理（同一上游）。`market.json` 的 `mirrors[]` 现在只保留 `cloudflare`；市场插件侧同样只剩这一个候选，老配置里残留的 id 会被过滤掉，无需迁移。
`market.json` 内的哈希与版本与镜像无关（内容相同哈希相同）。
发布后需把 `packages/*.tgz` 与重建的 `market.json` 上传到 R2（见 `tsweb-cordis/plugins/market/README.md`）。

## 宿主前端（`web` 包）

索引里的 `web` 条目不是普通业务插件：它的 `public/` 就是整个面板前端，由打包器在发版时用 `buildHost` 现构建进包。面板不会用注册表「更新」它（`canUpdate('web')` 为 false），而是走市场页的**「更新前端」**——只替换静态资源、失败回滚，刷新页面即生效。详见 `tsweb-cordis/plugins/market/README.md` 的「宿主前端投放」一节。

## 安全

- 客户端强制 `sha256` 校验，篡改/传输损坏一律拒绝安装
- 安装落位前校验包名与目标一致、拒绝非向前版本、解压拒绝路径穿越
- 更新采用"备份旧目录 → 原子替换 → 失败回滚"三段式，插件状态数据不在插件目录内，更新不影响数据

## 2026-10-25 初始化扩展更新

- `registration` 2.0.4：由插件声明注册方式、已有账号密码验证的初始化步骤。
- `anti-cheat` 2.1.6：由插件声明物品检测、弹幕检测和恶性 Bug 修复步骤。
- `market` 2.0.1：修正安装状态、版本比较和自更新结果反馈。
- 配套当前基础宿主的 `ctx.initStep`、`UiInitChoices` 扩展；Base 提供独立 SSC API，向导支持失败步骤重新读取和确认，保留已保存步骤。
- 源码提交：`d476674545c80485d6662e4b16919a04862553aa`。提交快照 217 项相关回归通过；隔离市场安装验证为 27 个市场条目、34 个后端 ready、26 个浏览器插件加载。没有接入真实游戏服验证游戏规则。
- 详细证据：`verification/2026-10-25-plugin-init.json`。


## 2026-10-25 本机初始化重启配套更新

27 个市场插件从当前源码重新构建，补丁版本递增，旧包保留。Market 2.0.2、registration 2.0.5、anti-cheat 2.1.7、gameplay-settings 1.3.1、history 2.4.2。

配套 Windows 基础发行包含独立正式登录界面和本机扫描初始化的一键保存重启按钮；这些宿主与 init 改动需使用新版基础发行，单独更新市场插件不能替代宿主升级。

本轮通过实际市场安装器、34 个后端就绪、插件启停与 26 个前端加载验收。记录见 verification/2026-10-25-local-init-restart.json。

## 2026-10-31 接收端热替换与连接收敛配套更新

- `init` 0.2.0：一键接入改为“卸载运行中的旧版接收端并装载配套新版”；接入成功后返回 `replaced` / `retiredReceivers`，只有替换没完成（回执仍是旧 MVID）才提示重启一次。
- `web` 2.0.4：添加服务器向导同步新文案（接收端不配套时说明“接入时卸旧装新”，成功后展示旧版 MVID 与替换说明）。
- `market` 2.2.1 / `base` 2.4.2 以及前几轮已构建未推送的包（`market` 2.2.0、`web` 2.0.2/2.0.3、`market` 2.1.2~2.1.5、`init` 0.1.9）随本次索引一并推送。
- 配套基础发行（接收端 2.3.5、`src/plugin-sync.js` 连接即收敛）必须使用新版 Windows 基础发行；单独更新市场插件不能替代宿主升级。
- 验收：32 个市场包、36 个后端就绪、26 个浏览器前端加载与逐插件停用/启用全部通过；报告见 `verification/2026-10-31.json`。未接入真实游戏服。
