# Git 日常使用标准化实战手册

> 本手册覆盖单人开发、团队协作、版本控制、冲突解决与错误自救等日常高频标准场景。所有命令均遵循现代 Git 规范（如优先使用 `switch` / `restore`）。

---

## 一、核心架构与状态模型

理解 Git 的四个逻辑区域，是正确使用所有命令的前提：

```text
┌──────────────┐   git add   ┌──────────────┐  git commit  ┌──────────────┐   git push   ┌──────────────┐
│    工作区    │ ──────────> │    暂存区    │ ───────────> │  本地版本库  │ ───────────> │  远程仓库    │
│ (Working Dir)│             │ (Stage/Index)│              │ (Local Repo) │              │ (Remote Repo)│
│  编辑中的文件│ <────────── │ 待提交的清单 │              │ 永久历史快照 │ <─────────── │ GitHub/Gitee │
└──────────────┘ git restore └──────────────┘              └──────────────┘   git fetch  └──────────────┘
```

### 文件的 4 种基础状态
- **Untracked（未跟踪）**：新建文件，尚未纳入 Git 管理。
- **Modified（已修改）**：已被 Git 跟踪的文件发生变更，但未进入暂存区。
- **Staged（已暂存）**：通过 `git add` 加入暂存区，等待下一次提交。
- **Committed（已提交）**：通过 `git commit` 永久写入本地版本库快照。

---

## 二、初始化与全局配置规范

### 1. 身份与核心行为配置
```bash
# 身份信息（每次提交必带）
git config --global user.name "YourName"
git config --global user.email "your_email@example.com"

# 现代默认主分支名
git config --global init.defaultBranch main

# Windows 跨平台换行符规范（提交转 LF，检出转 CRLF）
git config --global core.autocrlf true

# 解决 git status / log 中中文路径显示为 \344\270\255 等转义字符的问题
git config --global core.quotepath false

# 查看所有生效配置及其来源文件
git config --list --show-origin
```

### 2. 推荐全局效率别名（Alias）
```bash
git config --global alias.st "status -sb"
git config --global alias.cm "commit -m"
git config --global alias.sw "switch"
git config --global alias.lg "log --oneline --graph --decorate --all"
```

### 3. `.gitignore` 标准规则与避坑
常见忽略规则示例（**必须使用 UTF-8 无 BOM 编码保存**）：
```gitignore
# 忽略日志文件与环境变量密钥
*.log
.env
.env.local

# 忽略依赖与构建产物目录
node_modules/
dist/
build/

# 忽略 IDE 与系统缓存文件
.vscode/
.idea/
.DS_Store
```

> **常见坑：文件已经被 commit 过，再写入 `.gitignore` 不会生效！**
> 解决方法（仅从 Git 索引中移除跟踪，保留本地物理文件）：
> ```bash
> git rm --cached <文件名>
> git commit -m "chore: 停止跟踪敏感或缓存文件"
> ```

---

## 三、团队协作与命名标准

### 1. 分支命名规范（Branch Naming）
| 分支类型 | 命名格式 | 用途说明 |
| :--- | :--- | :--- |
| 主分支 | `main` | 随时可发布到生产环境的稳定代码，禁止直接提交 |
| 开发主干 | `develop` | 集成测试分支（中型/大型团队常用） |
| 功能分支 | `feat/<功能名>` | 如 `feat/user-login`，开发完毕后合并进主干并删除 |
| 缺陷修复 | `fix/<问题描述>` | 如 `fix/order-total-calc`，日常 bug 修复 |
| 紧急热修 | `hotfix/<补丁名>` | 如 `hotfix/security-patch`，基于 `main` 拉出并紧急合回 |

### 2. 提交信息规范（Conventional Commits）
标准格式：`<type>(<可选范围>): <简明描述>`

| Type 类型 | 含义 | 示例 |
| :--- | :--- | :--- |
| `feat` | 新增功能 | `feat(auth): 增加微信扫码登录` |
| `fix` | 修复 Bug | `fix(cart): 修复商品数量为负数的异常` |
| `docs` | 仅修改文档 | `docs: 更新项目安装指南` |
| `style` | 代码格式调整（不影响逻辑） | `style: 统一缩进与分号规范` |
| `refactor` | 代码重构（既非新增功能也非修 Bug） | `refactor(api): 抽离通用请求拦截器` |
| `test` | 增加或修改测试 | `test: 补充登录模块单元测试` |
| `chore` | 构建配置、依赖升级、工具脚本等杂项 | `chore: 更新 .gitignore 规则` |

---

## 四、日常高频标准工作流（SOP）

### 场景 1：新功能标准开发闭环
```bash
# 1. 切回主分支并同步最新远程代码
git switch main
git pull

# 2. 创建并切换到独立功能分支
git switch -c feat/payment

# 3. 编写代码后查看变动并提交
git status
git diff
git add .
git commit -m "feat(pay): 完成支付宝回调接口"

# 4. 推送功能分支到远程（用于备份或发起 Pull Request）
git push -u origin feat/payment

# 5. 代码评审合入主分支后，清理本地与远程分支
git switch main
git pull
git branch -d feat/payment              # 删除本地分支
git push origin --delete feat/payment   # 删除远程分支
```

### 场景 2：`git fetch` vs `git pull`
- **`git fetch`**：只从远程拉取最新提交记录到本地追踪分支（如 `origin/main`），**不改动工作区和当前分支**。适合先审查差异：
  ```bash
  git fetch origin
  git log main..origin/main --oneline   # 查看远程比本地多了哪些提交
  git merge origin/main                 # 确认无误后再合并
  ```
- **`git pull`**：等价于 `git fetch` + `git merge`，适合确认本地无复杂分叉时快速同步。

### 场景 3：紧急切换分支时的现场暂存（`git stash`）
当当前分支代码写到一半无法提交，却必须切分支修 Bug 时：
```bash
# 1. 暂存当前工作区与暂存区改动（带备注，含未跟踪新文件可加 -u）
git stash push -u -m "开发支付功能到一半"

# 2. 查看暂存列表
git stash list

# 3. 切去别的分支修完 Bug 后切回，恢复并删除最近一条现场
git switch feat/payment
git stash pop

# 仅恢复现场但不从储藏列表删除：git stash apply stash@{0}
# 手动丢弃某条储藏记录：        git stash drop stash@{0}
```

### 场景 4：合并冲突解决标准流程（SOP）
当执行 `git merge` 或 `git pull` 触发 `CONFLICT` 时：
1. **查看冲突清单**：执行 `git status`，找到标为 `both modified` 的文件。
2. **手工编辑裁决**：打开文件定位冲突标记：
   ```text
   <<<<<<< HEAD
   当前分支的代码
   =======
   待合并分支的代码
   >>>>>>> feat/xxx
   ```
   根据业务需求保留正确代码（或融合两者），**必须彻底删除 `<<<<<<<`、`=======`、`>>>>>>>` 三行标记**。
3. **验证与完成提交**：
   ```bash
   git add <已解决冲突的文件>
   git commit                          # 使用 Git 自动生成的合并信息或加上 -m "..."
   ```
4. **后悔退出**：如果冲突解决乱了想重来，执行 `git merge --abort` 即可瞬间回到合并前状态。

---

## 五、信息检索与排查命令

```bash
# 差异比对
git diff                         # 工作区 vs 暂存区
git diff --staged                # 暂存区 vs 最新提交
git diff main..feat/xxx          # 两个分支最新快照之间的代码差异

# 历史日志检索
git log --oneline -n 10          # 最近 10 条简洁日志
git log -p -n 2                  # 查看最近 2 次提交的具体代码改动
git log --author="vvay"          # 按作者过滤提交
git log --grep="login"           # 按提交说明关键字搜索
git log -- README.md             # 查看某一个文件的全部演变历史

# 单点追踪
git show <commit-id>             # 查看某一次提交的详情与代码变动
git blame <文件名>               # 逐行查看每一行代码最后是谁在何时提交的
```

---

## 六、后悔药决策矩阵（按阶段精准撤销）

| 当前所处阶段 | 想实现的目标 | 标准命令 | 安全等级 |
| :--- | :--- | :--- | :---: |
| **修改在工作区**<br>（未 `add`） | 放弃对某文件的全部修改，恢复原样 | `git restore <文件>` | 🟡 丢弃未保存修改 |
| **已放入暂存区**<br>（已 `add` 未 `commit`） | 撤出暂存区，保留工作区代码继续修改 | `git restore --staged <文件>` | 🟢 安全 |
| **刚完成提交**<br>（仅在本地，未 `push`） | 修改上次提交说明，或把漏掉的文件补进上次提交 | `git commit --amend` | 🟢 安全 |
| **已提交到本地**<br>（未 `push`） | 撤销最近一次提交，改动保留在暂存区 | `git reset --soft HEAD~1` | 🟢 安全 |
| **已提交到本地**<br>（未 `push`） | 撤销最近一次提交并移出暂存区，改动保留在工作区 | `git reset HEAD~1`（默认 `--mixed`） | 🟢 安全 |
| **已提交到本地**<br>（未 `push`） | 彻底删除最近一次提交及对应所有改动 | `git reset --hard HEAD~1` | 🔴 **高危（丢代码）** |
| **已推送到公共分支**<br>（团队共享） | 安全抵消某次旧提交的效果（新增反向提交，不改写历史） | `git revert <commit-id>` | 🟢 **团队协作首选** |
| **误操作导致提交消失**<br>（如误用 `--hard`） | 从 Git 本地操作黑匣子找回丢失的 `commit-id` | `git reflog` + `git reset --hard <哈希>` | 🟢 **终极救命神器** |

---

## 七、进阶协作操作

### 1. `git merge` vs `git rebase`
- **`git merge`**：保留真实的时间线与分支拓扑结构，会生成合并节点。**公共分支合并推荐优先使用**。
- **`git rebase`**：将当前分支的提交“剪下并平移”到目标分支末端，历史呈一条干净的直线。
  ```bash
  # 在自己的私有功能分支上，同步 main 的最新代码保持提交线性
  git switch feat/my-work
  git rebase main
  # 若 rebase 过程解冲突想放弃：git rebase --abort
  ```
- **交互式整理提交（`git rebase -i`）**：合并本地多个零碎提交，保持提交记录整洁：
  ```bash
  git rebase -i HEAD~3   # 在弹出编辑器中将需要合并的提交前的 pick 改为 squash 或 s
  ```

### 2. 精准单点挑摘（`git cherry-pick`）
只把其他分支的某几个特定提交应用到当前分支，而不合并整个分支：
```bash
git cherry-pick <commit-id>               # 挑取单个提交
git cherry-pick <id1> <id2>               # 挑取多个提交
git cherry-pick --abort                   # 遇到冲突放弃本次挑摘
```

### 3. 版本标签管理（Tag & SemVer）
遵循语义化版本号：`v主版本.次版本.修订号`（如 `v1.0.0`）。
```bash
git tag -a v1.0.0 -m "Release 1.0.0 正式版"  # 创建带附注的标签（推荐）
git tag                                      # 查看所有标签
git push origin v1.0.0                       # 推送指定标签到远程
git push origin --tags                       # 推送所有本地未推送的标签
git tag -d v1.0.0                            # 删除本地标签
git push origin --delete v1.0.0              # 删除远程标签
```

---

## 八、⚠️ 高危命令红线清单

以下操作会**永久销毁未提交代码**或**破坏团队历史记录**，执行前务必二次确认：

1. **`git push --force`（`-f`）**：强行用本地历史覆盖远程仓库，会抹掉同事已提交的记录。**公共分支（`main` / `develop`）严禁使用**。
   - *安全替代*：在个人独立分支确需强推时，使用 `git push --force-with-lease`（若远程有别人新推的提交会自动阻止覆盖）。
2. **`git reset --hard`**：不可逆地丢弃所有未提交的工作区与暂存区代码（未提交过的文件无法通过 `reflog` 找回）。
3. **`git clean -fd` / `git clean -fdx`**：永久删除磁盘上所有未跟踪的文件和目录（跳过回收站）。执行前务必先用 `git clean -nd` 预览将被删除的清单！
4. **`git branch -D <分支>`**：无视合并状态强制删除分支。若分支未合并，独有提交会立即成为游离状态。
5. **对已推送的公共分支执行 `git rebase`**：会更改公共提交的哈希值，导致所有协作者本地同步全面报错。

---

## 九、常见报错与 Windows 避坑速查

| 现象 / 报错信息 | 原因分析 | 标准解决方案 |
| :--- | :--- | :--- |
| `Binary files a/xxx and b/xxx differ` 或 `.gitignore` 规则不生效 | Windows PowerShell 使用 `>` / `>>` 默认写入 **UTF-16 LE**，含 `0x00` 空字节导致 Git 误判为二进制 | 改用 VS Code 等编辑器保存为 **UTF-8（无 BOM）**，或 PowerShell 加 `-Encoding utf8` |
| `Permission denied (publickey)` | 本地 SSH 私钥未生成、未启动，或对应公钥未正确添加到 GitHub / Gitee | 用 `ssh -T git@github.com` 排查；确认 `~/.ssh/id_ed25519.pub` 已完整粘贴到平台 SSH Keys |
| `! [rejected] main -> main (non-fast-forward)` | 远程仓库存在你本地尚未拉取的新提交 | 先执行 `git pull`（或 `git pull --rebase`）同步合并远程代码，再 `git push` |
| `Please commit your changes or stash them before you merge/switch` | 目标分支修改了你当前工作区尚未保存的文件 | 先执行 `git stash` 暂存或 `git commit` 提交后再切换/合并 |
| `You are in 'detached HEAD' state`（游离状态） | 直接 `checkout` 了某个历史 commit 哈希或 tag，当前不处于任何分支上 | 若要基于此处继续开发，执行 `git switch -c <新分支名>`；若只是查看完想回去，直接 `git switch main` |
