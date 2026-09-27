# tsweb-plugins —— TSWeb 插件发行仓库

本仓库是 **TSWeb（cordis 版）的插件发行通道**：只存放**发行包与元数据**，不存放插件源码。

- 插件源码在宿主仓库 `tsweb-cordis` 中开发与维护
- 本仓库的用途是让面板/终端能下载并安装插件：`market-list` / `market-install` / `market-update`

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
├── package.json        { name, version, tsweb: { autoload, bundled, client, dll, deploy } }
├── index.js            cordis 插件对象（export const name + export async function apply）
├── client/             前端半（可选）
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
    { "id": "gitee",   "base": "https://gitee.com/{repo}/raw/{branch}" },
    { "id": "ghproxy", "base": "https://gh-proxy.com/https://raw.githubusercontent.com/{repo}/{branch}" }
  ],
  "packages": [
    {
      "name": "players",
      "version": "1.0.0",
      "description": "…",
      "bundled": false,          // 是否属于"自带插件"（自带插件随宿主发行，一般不走本仓库）
      "hasClient": true,         // 是否含前端半
      "dll": ["./plugin/*.dll"], // C# 半声明（安装后用于下发到游戏服）
      "deploy": "all",           // DLL 下发模式（三态，缺省 all）：all=自动全量 / ask=安装时弹窗选 / manual=静默不下发
      "url": "packages/players-1.0.0.tgz",
      "sha256": "…",             // 整包哈希，客户端下载后强制校验
      "size": 39514,
      "publishedAt": "…"
    }
  ]
}
```

客户端（`market` 插件）按 `mirrors` 顺序逐个尝试下载，失败自动切换并记住成功来源；下载后校验 `sha256`，不符即拒绝安装。

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
| `github` | 原始仓库（raw.githubusercontent.com） |
| `gitee` | Gitee 镜像仓库（如需启用，在 Gitee 侧建立同名仓库并同步） |
| `ghproxy` | 公共前缀代理（gh-proxy.com） |

`market.json` 内的哈希与版本不随镜像变化（内容相同哈希相同），因此切换镜像不会破坏校验。

## 安全

- 客户端强制 `sha256` 校验，篡改/传输损坏一律拒绝安装
- 安装落位前校验包名与目标一致、拒绝非向前版本、解压拒绝路径穿越
- 更新采用"备份旧目录 → 原子替换 → 失败回滚"三段式，插件状态数据不在插件目录内，更新不影响数据
