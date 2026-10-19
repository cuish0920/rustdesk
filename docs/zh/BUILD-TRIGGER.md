# 构建触发说明（RemoteDesk 定制版）

本文档说明：改完代码后，怎样触发 GitHub Actions 构建出安装包，以及中间那些容易踩的坑。

适用仓库：`cuish0920/rustdesk`（本地工作目录 `G:\pr\rustdesk-hhit`）

---

## 一、一句话版本

```bash
git add -A
git commit -m "你的改动"
git push origin master

# 关键：把版本标签移到新提交上，然后推送这个标签
git tag -f 1.5.0 <新提交的 sha>
git push origin refs/tags/1.5.0 --force
```

**推送标签这一步会「自动」触发构建，不需要去网页上点 Run workflow。**

---

## 二、为什么是「推标签」触发

相关文件：`.github/workflows/flutter-tag.yml`

```yaml
name: Flutter Tag Build

on:
  workflow_dispatch:          # 允许手动点按钮触发
  push:
    tags:                     # 推送符合下面格式的标签时自动触发
      - 'v[0-9]+.[0-9]+.[0-9]+'
      - '[0-9]+.[0-9]+.[0-9]+'
      - 'v[0-9]+.[0-9]+.[0-9]+-[0-9]+'
      - '[0-9]+.[0-9]+.[0-9]+-[0-9]+'

jobs:
  run-flutter-tag-build:
    uses: ./.github/workflows/flutter-build.yml   # 真正的构建逻辑在这个可复用工作流里
    secrets: inherit
    with:
      upload-artifact: true
      upload-tag: ${{ github.ref_name }}          # ← 这一行是关键
```

所以触发条件只有两个：

| 方式 | 说明 |
|------|------|
| 推送 `1.5.0` 这类标签 | **推荐**。自动触发，无需人工操作 |
| 网页上点 `Run workflow` | 可以，但**要选对 ref**，见下面的陷阱 |

---

## 三、★ 最重要的陷阱：`github.ref_name` 是「二合一」的

`upload-tag: ${{ github.ref_name }}` 里的 `github.ref_name`，同时决定了两件事：

1. **编译哪个提交**（选 tag 就是编译标签指向的那个提交）
2. **发布到哪个 Release**（`TAG_NAME` 就是它）

这就带来两个坑：

### 坑 A：在 master 上点 Run workflow → Release 名字变成 `master`
因为 `github.ref_name` 会是 `master`，构建出来的包会被发布到一个叫 `master` 的 Release 里，很乱。

### 坑 B：选标签点 Run workflow → 编译的是「标签当前指向的旧提交」
如果你改了代码但**没有移动标签**，标签还指着旧提交，那么这次构建编出来的是**旧代码**。
包能正常下载、能正常安装，就是没有你的改动 —— 极难察觉。

> 这个坑实际踩过：修复代码已推到 master，但 `1.5.0` 标签还指向更早的提交，
> 构建产物里根本没有修复，装上去症状照旧。

### 正确做法
**先把标签移到目标提交，再推标签**（而不是在网页上点按钮）：

```bash
git tag -f 1.5.0 <目标提交sha>
git push origin refs/tags/1.5.0 --force
```

标签一推上去，`push: tags` 触发器就自动开构建，`github.ref_name` 正好是 `1.5.0`，
提交和 Release 名两边都对。

---

## 四、`VERSION` 与 `TAG_NAME` 是两套东西

`.github/workflows/flutter-build.yml` 顶部：

```yaml
env:
  TAG_NAME: "${{ inputs.upload-tag }}"   # 来自标签名 → 决定发到哪个 Release
  VERSION: "1.5.0"                       # 硬编码 → 决定产物文件名
```

| 变量 | 来源 | 作用 |
|------|------|------|
| `TAG_NAME` | 标签名（或手动输入） | Release 的 tag |
| `VERSION` | 工作流里**写死** | 产物文件名，如 `RemoteDesk-1.5.0-x86_64.msi` |

**所以：只改标签不改 `VERSION`，产物文件名还是 1.5.0。**
升级版本号要**同时**改：

- `.github/workflows/flutter-build.yml` 的 `VERSION`
- `Cargo.toml` 的 `version`
- `libs/portable/Cargo.toml` 的 `version`
- `res/rpm-flutter*.spec` 的 `Version:`
- `res/PKGBUILD` 的 `pkgver`
- `build.py` / `appimage/AppImageBuilder-*.yml` 里可能写死的地方

否则包名和包内版本号会对不上。

---

## 五、完整操作流程（含验证）

### 1. 改代码并提交

```bash
cd G:\pr\rustdesk-hhit
git add -A
git commit -m "改了什么"
```

### 2. 推送 master

```bash
git push origin master
```

> 本机需要代理，且 `git` 要设成 HTTP/1.1，否则会卡死：
> ```bash
> export HTTPS_PROXY=http://127.0.0.1:7897
> git config http.version HTTP/1.1
> ```
> 偶尔会遇到一次 `schannel: failed to receive handshake`，**重试即可**。

### 3. 移动标签并推送（= 触发构建）

```bash
git tag -f 1.5.0 $(git rev-parse HEAD)
git push origin refs/tags/1.5.0 --force
```

### 4. 确认构建真的跑起来了

```bash
gh run list --repo cuish0920/rustdesk --limit 5
```

没有 `gh` 的话，用 API 查（把 sha 换成你刚推的）：

```bash
curl -s "https://api.github.com/repos/cuish0920/rustdesk/actions/runs?per_page=5" \
  | grep -E '"name"|"head_sha"|"status"'
```

**必须核对 `head_sha` 是不是你刚推的提交**，这是唯一能确认「编的是新代码」的办法。

### 5. 等构建完成，下载产物

构建约 **40～90 分钟**。产物统一以 `RemoteDesk-` 开头：

| 平台 | 文件名 |
|------|--------|
| Windows 安装包 | `RemoteDesk-1.5.0-x86_64.msi` |
| Windows 便携版 | `RemoteDesk-1.5.0-x86_64.exe` |
| Windows 32 位 | `RemoteDesk-1.5.0-x86-sciter.exe` |
| macOS | `RemoteDesk-1.5.0-x86_64.dmg` / `-aarch64.dmg` |
| Linux | `RemoteDesk-1.5.0-x86_64.deb` / `.rpm` / `.AppImage` / `.flatpak` |
| Arch | `RemoteDesk-1.5.0-0-x86_64.pkg.tar.zst` |
| Android | `RemoteDesk-1.5.0-<arch>.apk` |

> **Windows 请装 `.msi`，不是 `.exe`**。`.exe` 是便携版（自解压单文件），
> 服务安装、快捷方式、卸载项这些行为与 MSI 不同，很多问题只在 MSI 上能复现。

---

## 六、各平台的产物是「分别发布」的

每个平台作业各自调用 `softprops/action-gh-release` 上传自己的文件。
这意味着：

- **某个平台失败，不影响其他平台已经发布的产物**
- 整个 run 显示 `failure`，但 Windows 的 MSI 可能已经上传成功

所以看到 run 失败**不要急着全盘重来**，先看是哪个 job 挂的：

```bash
curl -s "https://api.github.com/repos/cuish0920/rustdesk/actions/runs/<RUN_ID>/jobs?per_page=100" \
  | python -c "import json,sys; [print(j['name'],'->',j['conclusion']) for j in json.load(sys.stdin)['jobs']]"
```

想定位具体挂在哪一步（**日志需要 token，未认证会 403，但步骤级信息不需要**）：

```bash
curl -s "https://api.github.com/repos/cuish0920/rustdesk/actions/runs/<RUN_ID>/jobs?per_page=100" \
  | python -c "
import json,sys
for j in json.load(sys.stdin)['jobs']:
    if j['conclusion'] not in ('success','skipped',None):
        print(j['name'], j['conclusion'])
        for s in j.get('steps',[]):
            if s['conclusion'] not in ('success','skipped',None):
                print('   step', s['number'], s['name'], s['conclusion'])
"
```

多个 job 挂在**同一个步骤名**上，通常说明是共因（同一个脚本/配置问题）。

---

## 七、常见失败与判据

| 现象 | 原因 | 处理 |
|------|------|------|
| MSI 构建报 `WIX0150: Undefined preprocessor variable '$(var.XXX)'`，**连 `Version`/`Manufacturer`/`UpgradeCode` 都说没定义** | 不是变量没定义，而是 `preprocess.py` 提前退出了，`Includes.wxi` 没被填充 | 往日志上方找 `Error: version ... not found in ...`，说明它找不到 dist 里的 exe。通常是 `--exe-name` 与实际文件名不符 |
| `Build pre-built MSI template` 失败 | 该步骤把 dist 里的 exe 改名成 `RDAPPNAM.exe`，但 `--exe-name` 传错 | MSI template 那次调用必须显式传 `--exe-name RDAPPNAM.exe` |
| `Move-Item ... Path is null` | 上一步没产出 MSI，`$msi` 是 null | 是上游失败的**后果**，不是原因，往上看 |
| `x86_64-apple-darwin` 卡在 `Install vcpkg dependencies` | vcpkg 依赖拉取的**历史遗留**问题，与本项目改动无关 | 忽略；macOS 不是主要交付目标 |
| 装上新包但改动没生效 | **标签指向旧提交**（第三节坑 B） | 核对 run 的 `head_sha` 与 `git rev-parse HEAD` |
| `Rename archlinux package` 失败 | 容器内构建的产物属主可能是 root，普通 `mv` 无权 | 已改为 `sudo mv` 并加诊断输出 |

---

## 八、不想动标签、只想临时编一次？（可选）

在 Actions 页面点 `Flutter Tag Build` → `Run workflow`，
**`Use workflow from` 一定要选 `1.5.0` 这个标签，且确认该标签已指向你要编的提交**。

但这本质和推标签一样，还要多一步手点，所以**推荐始终用推标签的方式**。

---

## 九、需要人工确认的两件事

每次发版前请自己核一下：

1. **run 的 `head_sha` == 你本地 `git rev-parse HEAD`**
   → 确保编的是新代码
2. **产物确实是 `RemoteDesk-*` 而不是旧的 `rustdesk-*`**
   → 说明改名改动生效了

---

## 附：本文档涉及的关键文件

| 文件 | 作用 |
|------|------|
| `.github/workflows/flutter-tag.yml` | 触发器（标签 push / 手动），传入 `upload-tag` |
| `.github/workflows/flutter-build.yml` | 全部平台的构建逻辑；`VERSION` 在此硬编码 |
| `res/msi/preprocess.py` | 生成 MSI 用的 `Includes.wxi` 变量、收集待打包文件 |
| `flutter/windows/CMakeLists.txt` | `BINARY_NAME` = exe 名；核心库 `RENAME RemoteDesk.dll` |
| `src/common.rs` | `get_exe_name()`，Windows 返回 `RemoteDesk.exe` |
