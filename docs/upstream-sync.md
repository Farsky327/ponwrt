# PonWrt 上游同步指南

本文件记录将 `Farsky327/ponwrt` 与上游 `pbs05/ponwrt`（及更上游的 ImmortalWrt）同步的完整操作流程、已知坑与预案。

## 仓库结构

| Remote | 地址 | 用途 |
|---|---|---|
| `origin` | `git@github.com:Farsky327/ponwrt.git` | 自己的 fork，日常 push 目标 |
| `upstream` | `git@github.com:pbs05/ponwrt.git` | 上游（pbs05），只读拉取 |

同步链路：ImmortalWrt core → pbs05/ponwrt（parent）→ Farsky327/ponwrt。

## 方式一：本地 rebase（推荐，保持线性历史）

### 1. 拉取上游

```bash
cd /f/OS/App/ponwrt-ci/ponwrt
git fetch upstream
```

### 2. 检查落后情况

```bash
# 本地没有的上游提交（远程跟踪引用被 IDE 锁挡住时用 FETCH_HEAD）
git log --oneline HEAD..FETCH_HEAD
git rev-list --count HEAD..FETCH_HEAD
```

### 3. 冲突预判（重要，先看再动手）

```bash
# 上游这批提交改了哪些文件
git diff --stat HEAD..FETCH_HEAD
# 本地侧相对旧 base 改过哪些文件
git diff --name-only <旧上游base>..HEAD
```

两个清单有交集即预示冲突，需先想好解决方式。无交集则可放心 rebase。

### 4. 执行 rebase

正常环境：

```bash
git rebase FETCH_HEAD          # 或 git rebase upstream/master
```

**IDE（CodeBuddy / VS Code 的 git 集成）占锁时**（报 `failed to finalize '.git/rebase-merge/git-rebase-todo'` / `Permission denied`），`git rebase` 无法直接运行，用底层等价手法重放本地提交：

```bash
# 场景：上游只改了少量文件、与本地零冲突时
cd /f/OS/App/ponwrt-ci/ponwrt
export GIT_INDEX_FILE=/tmp/sync-idx

# 上游有改动的文件（按实际增删），取上游新版本 blob
DTS=target/linux/airoha/dts/an7581-h3c-hm2004-du.dts
DTS_NEW=$(git rev-parse FETCH_HEAD:$DTS)

PARENT=8ef1bd9d58            # FETCH_HEAD（上游 master）
for c in $(git rev-list --reverse <旧上游base>..HEAD); do
  git read-tree $c
  # 上游改动的每个文件逐一替换成上游版本
  git update-index --add --cacheinfo 100644,$DTS_NEW,$DTS
  TREE=$(git write-tree)
  git log -1 --format=%B $c > /tmp/sync-msg
  GIT_AUTHOR_NAME="$(git log -1 --format=%an $c)" \
  GIT_AUTHOR_EMAIL="$(git log -1 --format=%ae $c)" \
  GIT_AUTHOR_DATE="$(git log -1 --format=%aI $c)" \
  GIT_COMMITTER_NAME="Farsky327" \
  GIT_COMMITTER_EMAIL="58379211+Farsky327@users.noreply.github.com" \
  NEW=$(git commit-tree $TREE -p $PARENT -F /tmp/sync-msg)
  PARENT=$NEW
done
```

上游改了多个文件时，在循环里对每个文件重复一行 `git update-index --add --cacheinfo <mode>,<blob>,<path>`（blob 用 `git rev-parse FETCH_HEAD:<path>` 取）。有实质冲突时不要用此法，改到临时 clone 里正常 rebase：

```bash
git clone --shared /f/OS/App/ponwrt-ci/ponwrt /tmp/ponwrt-sync
cd /tmp/ponwrt-sync
git remote add upstream git@github.com:pbs05/ponwrt.git
git fetch upstream && git rebase upstream/master   # 冲突在 clone 里解决
git remote set-url origin git@github.com:Farsky327/ponwrt.git
git push --force origin master:master
```

### 5. 推送

```bash
git push --force-with-lease origin master
```

**若被锁挡到整体失败**（输出只有 `error: update_ref failed for ref 'refs/remotes/origin/master'`、没有 `master -> master` 行，远程未收到），改用：

```bash
git push --force origin master:master
```

判断是否真成功：输出里有 `To github.com:... <旧sha>...<新sha> master -> master` 或 `(forced update)` 即成功；仅尾部的 `refs/remotes/origin/master.lock` 报错是 IDE 占位的已知误报，远程实际已一致。

### 6. 同步本地工作树

rebase 后本地 HEAD 需指向新提交，工作树需更新上游改动的文件：

```bash
# 本地引用直写（绕开 refs/heads/master.lock）
git rev-parse <新HEAD> > .git/refs/heads/master

# 工作树文件逐个更新（只写上改动的文件）
git show HEAD:<path> > <path>

# 索引重建
export GIT_INDEX_FILE=/tmp/final-idx && git read-tree HEAD && cp /tmp/final-idx .git/index
git status        # 应显示 nothing to commit, working tree clean
```

### 7. 验证

```bash
# rebase 语义自检：两个 diff 必须完全一致
git diff <旧上游base>..<旧HEAD> > /tmp/before.diff
git diff FETCH_HEAD..<新HEAD> > /tmp/after.diff
diff /tmp/before.diff /tmp/after.diff

# 远程一致性（PAT 鉴权，替代被 IDE 锁挡住的 git fetch 验证）
curl -sS -H "Authorization: token $TOK" \
  "https://api.github.com/repos/Farsky327/ponwrt/commits/master"
```

## 方式二：GitHub 网页 Sync fork（快速通道）

适合上游小更新、无冲突、不在乎 merge commit 的场景：

1. 打开 https://github.com/Farsky327/ponwrt
2. 文件列表上方点 **Sync fork** 下拉菜单（仅落后于上游时出现）
3. 确认提交列表后点 **Update branch**

注意：网页 Sync fork 是 **merge 语义**，会在历史里留下 `Merge branch 'master' of ...` 合并提交；与上游有冲突时网页无法完成，会提示本地解决。要保持线性历史或改动有冲突，用方式一。

## 已知坑与预案

| 坑 | 现象 | 预案 |
|---|---|---|
| IDE 占 `.git/index.lock`、`.git/rebase-merge/*.lock`、`.git/refs/**/*.lock` | rebase / commit / reset 全部 Permission denied | 提交走「临时 `GIT_INDEX_FILE` + `commit-tree` + 直写 `.git/refs/heads/master`」；rebase 用第 4 节底层手法或临时 clone |
| `git push` 后跟踪引用锁报错 | 输出同时有成功行和 lock 报错 | 看有无 `master -> master` 行：有=成功（误报）；无=真失败，改 `--force` |
| **libffi add/add 冲突** | pbs05 同步 ImmortalWrt 后带上 `package/libs/libffi`，与 fork vendored 的（`ac74a40f6`，为解决 python3 编译失败而保留）同路径冲突 | rebase 前先 `git rm package/libs/libffi/Makefile` 用上游那份；当前上游已无此文件，暂留本地 vendored 版 |
| HTTPS 代理 502 | `git fetch https://github.com/...` 失败 | remote 一律用 SSH（`git@github.com:...`），绕开代理 |
| 远程跟踪引用陈旧 | `git log origin/master` 显示旧值 | IDE 占锁导致 fetch 无法更新跟踪引用，用 GitHub API 查远程真实 sha |

## 发版联动

同步后如需发版，走 `release.yml` 的 workflow_dispatch（tag 格式 `ponwrt-YYYYMMDDHH`，prerelease 默认 false），并先在 CHANGELOG.md 写对应版本段——详见 CHANGELOG.md 与仓库记忆。
