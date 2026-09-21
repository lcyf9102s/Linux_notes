# 手动更新手机上的 LTS 内核 —— 逐步流程

**核对于 2026-09-19。** 每条路径、版本号、断言都是当天在真机/真仓库上查过的,不是凭记忆写的。

## 起点(已核实)

| | |
|---|---|
| 工作树 | `~/pixel4-kernel/lts/wt-fedora-lts`,分支 `fedora-lts`,工作区干净 |
| 当前版本 | **6.18.50**,commit `de1033564` |
| 上游最新 longterm | **6.18.52**(2026-09-14) |
| 推送目标 | remote **`gitlab-personal`** = `https://gitlab.postmarketos.org/lcyf9102s/sm8150-linux.git` |
| NixOS 侧 | `nixos-flame2/kernel.nix` 的 `lines.lts`,用 `fetchgit` 按 `rev`+`hash` 钉死 |
| LOCALVERSION | `config/lts.config:43` `CONFIG_LOCALVERSION="-sm8150-lts"` |
| 构建位置 | **手机上**(`deploy-update.sh` 已弃用) |

`origin` 是 postmarketOS 的上游 fork,**不要往那里推**。

---

## 步骤 1 — 确认目标版本

```sh
curl -s https://www.kernel.org/releases.json \
  | python3 -c "import json,sys;[print(r['moniker'],r['version'],r['released']['isodate']) for r in json.load(sys.stdin)['releases'] if r['version'].startswith('6.18')]"
```

## 步骤 2 — 应用增量补丁

⛔ **必须用增量补丁 `incr/patch-A-B.xz`,不能用累积的 `patch-6.18.52.xz`。**
累积补丁是相对 **6.18 基线**的 diff,打在 6.18.50 上必然一地 `.rej`。

从 6.18.50 到 6.18.52 要**按顺序走两步**:

```sh
cd ~/pixel4-kernel/lts/wt-fedora-lts
git status --porcelain            # 必须为空再开始

for step in 50-51 51-52; do
  curl -fL -o /tmp/p-$step.xz \
    "https://cdn.kernel.org/pub/linux/kernel/v6.x/incr/patch-6.18.$step.xz" || break
  xzcat /tmp/p-$step.xz | patch -p1 --forward 2>&1 | tee /tmp/patch-$step.log
  echo "--- $step: .rej=$(find . -name '*.rej' | wc -l)  可疑行=$(grep -icE 'skipping|reversed|FAILED|hunks? ignored' /tmp/patch-$step.log)"
done
```

### 判据 —— 两个都要看

1. **`.rej` 的数量**(不是 `patch` 的退出码)。多 commit 的补丁文件上
   `patch --dry-run` 会报假失败,见 [[feedback-patch-dry-run-lies-on-multicommit]]。
2. ⭐ **日志里有没有 `skipping` / `reversed` / `hunks ignored`。**

**第 2 条不是冗余的。** `patch` 只要判定某个文件"已应用",就会**整个文件跳过**,
而那条提示在几千行滚屏里极易错过:

```
Reversed (or previously applied) patch detected!  Skipping patch.
13 out of 13 hunks ignored -- saving rejects to file drivers/.../a6xx_gmu.c.rej
```

**"13 个全部忽略"不代表那 13 个都是重复的** —— 它可能只凭第一个 hunk 判定,
然后把另外 12 个真正的新改动一起丢掉。2026-09-19 的 6.18.52 就是这样,详见下一节。

有任何一条命中就停下来,**不要继续下一步**。

### ⚠️ 清掉 `.orig`

`patch` 只要有一个 hunk 带偏移就会留下 `.orig` 备份,**`git add -A` 会把它们一起提交进内核树**(以前发生过):

```sh
find . -name '*.orig' -print -delete
find . -name '*.rej'  -print        # 应该什么都不输出
```

### 确认版本真的变了

```sh
make -s kernelversion        # 应该输出 6.18.52
```

---

## 步骤 2.5 — 遇到 reject 怎么办

最常见的一类是 **"我们的本地 backport 被上游收编了"**。
2026-09-19 从 6.18.50 升 6.18.52 时就是这个:

```
Reversed (or previously applied) patch detected!  Skipping patch.
13 out of 13 hunks ignored -- saving rejects to drivers/gpu/drm/msm/adreno/a6xx_gmu.c.rej
```

### 先判别,别急着删 `.rej`

⛔ **"全部 hunk 被忽略"绝不等于"全部 hunk 都是重复的"。**
那次 13 个 hunk 里**只有 2 个是我们的**,另外 11 个是上游的新改动,
一共约 100 行,**被一起丢掉了**。

判别方法 —— 把上游前后两版的同一个文件拉下来直接对比:

```sh
F=drivers/gpu/drm/msm/adreno/a6xx_gmu.c
B=https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/plain
curl -fsL -o /tmp/old.c "$B/$F?h=v6.18.50"     # 你的起点版本
curl -fsL -o /tmp/new.c "$B/$F?h=v6.18.52"     # 目标版本

# 上游这次到底改了多少
diff -u /tmp/old.c /tmp/new.c | grep -c '^@@'

# 我们在这个文件上有多少私货(应该只有你认得出的那几处)
git show HEAD:$F > /tmp/ours.c
diff -u /tmp/old.c /tmp/ours.c
```

- **我们的差异 ⊂ 上游的新版本** → 本地 backport 已被收编,按下面处理。
- **有上游没有的内容** → 那部分是真私货,必须手工合进去,不能整体覆盖。

### 处理:整体换成上游的版本

确认我们那几处上游都有了之后,直接拿目标版本的文件覆盖:

```sh
F=drivers/gpu/drm/msm/adreno/a6xx_gmu.c
curl -fL -o $F "$B/$F?h=v6.18.52"
rm -f $F.rej $F.orig
cmp $F /tmp/new.c && echo "✅ 与上游逐字节一致"
```

即使后一步补丁在这个文件上**部分**应用过,整体覆盖也能拉回正确状态。

### 更干净的办法:先 revert 再打补丁

如果还没开始打补丁,**先把那个冗余的本地 commit revert 掉**,两步补丁就会零 reject:

```sh
git reset --hard <起点 commit 的完整 40 位 sha>
git revert --no-edit 9e835d100     # 那个已被上游收编的 backport
# 然后正常走步骤 2
```

⭐ **零 reject 本身就是"没有东西被跳过"的证明**,比事后验证可靠。

提交时在 message 里写清楚,免得以后再查一遍:

```
Linux 6.18.52 (SM8150/Google Pixel 4)

Drops the local backport of d9108bfdb746 ("drm/msm/a6xx: Fix stale rpmh
votes after suspend"): 6.18.52 carries it upstream, byte-identical at the
same two sites, so a6xx_gmu.c is now plain upstream.
```

---

## 步骤 3 — 提交

沿用这条线既有的 commit 标题格式(`git log --oneline -3` 看得到):

```sh
git add -A
git commit -m "Linux 6.18.52 (SM8150/Google Pixel 4)"
git log --oneline -2
```

---

## 步骤 4 — 推送,并**独立验证**推成功了

```sh
git push gitlab-personal fedora-lts
```

⛔ **不要只看命令没报错就认为推成功了。** 管道会遮住退出码,而且 git 推送失败
(`send-pack: unexpected disconnect`)可能被报成 exit 0 —— 以前就这样漏过一次。
用一条**独立的只读查询**确认:

```sh
LOCAL=$(git rev-parse HEAD)
REMOTE=$(git ls-remote gitlab-personal refs/heads/fedora-lts | cut -f1)
echo "local =$LOCAL"
echo "remote=$REMOTE"
[ "$LOCAL" = "$REMOTE" ] && echo "✅ 推送已确认" || echo "❌ 没推上去"
```

### 推不上去时

这个仓库有过两次"推送把服务器打挂"的记录(见 `kernel.nix` 里 mainline/rc 两段注释):
本地历史和远端**没有共同祖先**时 git 要传几 GB,服务器会直接断开。
LTS 这条线是连续的 fast-forward,正常不会遇到;**真遇到了就不要硬推**,
改成"把同一棵树 commit 到远端已有的 commit 之上",再核对 tree 哈希一致。

---

## 步骤 5 — 取新的 `hash`

`kernel.nix` 用 SRI 哈希。两种取法,任选:

**方法 A(推荐,不用额外工具)—— 故意写错让构建告诉你**

先把 `rev` 改成新值、`hash` 随便改一个字符,跑构建,错误信息里的
`got: sha256-...` 就是正确值。

**方法 B —— `nix-prefetch-git`**

```sh
nix-shell -p nix-prefetch-git --run \
  "nix-prefetch-git --url https://gitlab.postmarketos.org/lcyf9102s/sm8150-linux.git \
                    --rev $(git rev-parse HEAD) --quiet" | grep hash
```

---

## 步骤 6 — 改 `nixos-flame2/kernel.nix`

`lines.lts` 里**四个字段要一起改**:

```nix
    lts = {
      version = "6.18.52";                        # ← 改
      modDirVersion = "6.18.52-sm8150-lts";       # ← 改(见下)
      configfile = ./config/lts.config;           #   不动
      rev = "<新的完整 40 位 sha>";                # ← 改
      hash = "sha256-<步骤 5 得到的>";             # ← 改
    };
```

### `modDirVersion` 的两个坑

1. **`-sm8150-lts` 来自 `config/lts.config:43` 的 `CONFIG_LOCALVERSION`**,不是凭空拼的。
2. ⛔ **末尾不能有 `+`。** 在 git 工作树里跑 `make kernelrelease` 会打印
   `6.18.52-sm8150-lts+` —— 那个 `+` 是 `scripts/setlocalversion` 发现
   "有 git 树且 commit 超过 tag" 时加的。而 `fetchgit` 默认 `leaveDotGit = false`,
   Nix 构建里**没有 .git,所以没有 `+`**。写错了会构建成功但**找不到模块**。

### ⭐ 有个安全网

`pkgs/os-specific/linux/kernel/build.nix:360-364` 会断言:

```
Error: modDirVersion 6.18.51-sm8150-lts specified in the Nix expression is wrong,
it should be: 6.18.52-sm8150-lts
```

**写错了不会静默过去,会直接失败并告诉你正确值。**

---

## 步骤 7 — 在手机上构建

```sh
# 先 diff,别整文件覆盖 —— 手机上的配置可能比 PC 上这份新
sudo nixos-rebuild switch --flake /etc/nixos#p4nixos
```

⛔ **推送 `kernel.nix` 之前先和手机上的现有文件 diff、确认删除行数为 0**
(见 [[feedback-diff-before-every-push]])。

### 构建耗时

手机原生编内核约 1 小时量级。可以派给树莓派(aarch64 原生,约 36 分钟),
见 [[reference-pi5-remote-builder]] —— 但注意**不要开
`builders-use-substitutes`**,派的代理到 cache.nixos.org 只有 29 B/s。

---

## 步骤 8 — 上机核对

```sh
uname -r                                  # 应为 6.18.52-sm8150-lts
ls /run/booted-system/kernel-modules/lib/modules/   # 目录名要和 modDirVersion 一致
```

⚠️ **别只看 `uname -r`。** 模块目录名不匹配时内核照样能起来,
但所有外部模块都加载不了,表现成"WiFi/蓝牙/音频莫名其妙没了"。

---

## 一件容易忽略的事:新的 CONFIG 符号

`build.nix:356` 跑的是 **`make oldconfig`**,构建环境里 stdin 是空的,
所以 **6.18.52 新引入的配置符号会静默取内核默认值**,不会提示你。

多数情况无所谓。但如果这次稳定版引入了和这台设备相关的新选项
(GPU、UFS、电源管理一类),默认值未必是你要的。要检查就对比构建产物里的 `.config`:

```sh
diff <(sort nixos-flame2/config/lts.config) \
     <(sort /run/current-system/kernel-modules/lib/modules/*/build/.config) | head -40
```

⭐ 相关教训:[[feedback-compare-module-lists-not-sizes]] —— 配置镜像不全会
**静默丢驱动**,代价是没音频、没振动。判据要用 `.ko` 名字列表对比,不是包大小。

---

## 附:`fedora-lts` 上的本地改动 —— 哪些会撞,哪些不会

⭐ **判据只有一个:这个文件上游有没有。** 上游没有的文件,稳定版补丁永远碰不到,
本地改多少都不会 reject。核于 2026-09-19(逐个查 `git.kernel.org` v6.18.52):

### ⚠️ 会撞的(上游文件,我们在上面有本地改动)

| 文件 | 来自 |
|---|---|
| `drivers/tty/serial/qcom_geni_serial.c` | 设备补丁系列 0001-0008(蓝牙 UART 的 IRQ ack 修复) |
| `drivers/misc/Kconfig` | 0001-0008(cs40l25a 振动马达) |
| `drivers/misc/Makefile` | 同上 |
| `scripts/package/kernel.spec` | `8c1f43ec4` / `9ae600983` 两个打包 commit |

### ✅ 不会撞的(下游独有,上游没有这个文件)

| 文件 | 来自 |
|---|---|
| `drivers/input/touchscreen/fts_touch/fts.c` | `1a65a94ca` `1e59e0591` `6a01a3029` `fb47640b6` 四个触摸屏 commit |
| `arch/arm64/boot/dts/qcom/sm8150-google-flame.dts` | `5258e23a3`(RTC)+ 0001-0008 |
| `drivers/misc/cs40l25a-probe.c` | 0001-0008 |
| `sound/soc/qcom/sm8150.c` | 0001-0008 |

⚠️ **这张表推翻过一次直觉判断。** 触摸屏那 4 个 commit 挤在同一个驱动里,
看起来风险最高 —— 实际上 `fts_touch/` 是 postmarketOS fork 独有的目录,
**上游没有,所以风险是零**。判断"哪里会撞"不能看本地改了多少,
要看**上游会不会碰到同一个文件**。

`9e835d100`(GPU fix)原本在这张表的第一类,6.18.52 之后已消失 —— 上游收编了。

---

## 回滚

三层都能退:

1. **最快** —— 开机时用音量键在 systemd-boot 菜单里选上一个 generation
2. **配置层** —— 把 `kernel.nix` 的四个字段改回旧值,重新 `nixos-rebuild`
3. **仓库层** —— `git push gitlab-personal +<旧sha>:fedora-lts`(强推,慎用)

旧版本的 `rev`/`hash` 在 git 历史里,`git log -p nixos-flame2/kernel.nix` 找得到。
