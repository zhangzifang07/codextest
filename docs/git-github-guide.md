# Git 与 GitHub 实战入门指南

这份文档的目标是让你不只是“会上传一次文件”，而是能比较熟练地完成这些日常操作：

- 本地初始化 Git 仓库
- 把本地文件推送到 GitHub
- GitHub 已有空仓库时完成首次上传
- GitHub 还没有仓库时创建仓库并完成首次上传
- 本地与远程同步
- 拉取、推送、删除、重命名、忽略文件
- 遇到常见报错时知道怎么处理

本文默认你在 Windows 的 PowerShell 中操作。

## 1. 先理解 Git 和 GitHub

- `Git`：本地版本管理工具，负责记录文件变化。
- `GitHub`：远程代码托管平台，负责保存远程仓库。
- `本地仓库`：你电脑上的项目目录。
- `远程仓库`：GitHub 上的仓库。
- `commit`：一次本地版本记录。
- `push`：把本地提交上传到远程仓库。
- `pull`：把远程仓库最新内容拉回本地。

你可以把最常见的工作流理解成：

1. 先在本地改文件。
2. 用 Git 记录这次改动。
3. 再推送到 GitHub。

## 2. 先准备好环境

## 2.1 安装 Git

先确认 Git 是否已安装：

```powershell
git --version   # version = 版本，查看 Git 版本
```

如果能看到类似 `git version 2.x.x`，说明已经安装。

如果没有安装，到官网下载安装：

- [Git for Windows](https://git-scm.com/download/win)

## 2.2 配置用户名和邮箱

首次使用建议配置一次：

```powershell
git config --global user.name "你的名字"          # config = 配置，设置全局用户名
git config --global user.email "你的GitHub邮箱"  # email = 邮箱，设置全局邮箱
```

检查配置：

```powershell
git config --global --list   # list = 列表，查看当前全局配置
```

## 2.3 GitHub 远程地址有两种：HTTPS 和 SSH

同一个 GitHub 仓库通常会同时提供两种地址。

### HTTPS 地址

```text
https://github.com/你的用户名/你的仓库名.git
```

### SSH 地址

```text
git@github.com:你的用户名/你的仓库名.git
```

你可以这样记：

- `https://` 开头的是网页风格地址
- `git@github.com:` 开头的是 SSH 风格地址

### 两者区别

- `HTTPS`：更容易上手，复制地址就能用，但首次推送时通常需要登录或输入 Token。
- `SSH`：更适合长期使用，配置好 SSH Key 后一般不用反复登录。

如果你现在是新手，可以先用 HTTPS；如果你想长期高频使用 GitHub，建议尽快学会 SSH。

## 2.4 如果你想使用 SSH，需要先配置 SSH Key

先生成 SSH Key：

```powershell
ssh-keygen -t ed25519 -C "你的GitHub邮箱"   # keygen = key generate，生成密钥；-C = comment 注释
```

一路回车后，公钥一般会在：

```text
C:\Users\你的用户名\.ssh\id_ed25519.pub
```

把这个 `.pub` 文件里的内容复制出来，然后到 GitHub：

1. `Settings`
2. `SSH and GPG keys`
3. `New SSH key`
4. 粘贴公钥并保存

测试 SSH 是否可用：

```powershell
ssh -T git@github.com   # test = 测试，验证是否可以通过 SSH 连到 GitHub
```

如果看到欢迎信息，说明 SSH 配置成功。

## 2.5 HTTPS 登录说明

如果你使用 HTTPS 地址推送到 GitHub，通常不会再直接使用 GitHub 密码，而是使用下面任一方式：

- 浏览器授权登录
- Personal Access Token，简称 `PAT`
- Git Credential Manager 自动保存凭据

如果推送时弹出登录窗口，按提示完成即可。

## 3. 先记住最核心的命令

```powershell
git status                    # status = 状态，查看当前仓库状态
git add .                     # add = 添加，把改动加入暂存区
git commit -m "说明本次修改"   # commit = 提交，-m = message 提交说明
git push                      # push = 推送，把本地提交传到远程
git pull                      # pull = 拉取，把远程更新拉回本地
```

它们分别表示：

- `git status`：查看当前改了什么
- `git add .`：把当前改动加入暂存区
- `git commit -m "..."`：生成一次本地提交
- `git push`：上传到 GitHub
- `git pull`：先从远程拉最新内容到本地

## 4. 常见单词全拼，帮助记命令

把这些单词记住，你会更容易理解 Git 命令：

- `init` = `initialize`，初始化
- `status` = 状态
- `add` = 添加
- `commit` = 提交
- `push` = 推送
- `pull` = 拉取
- `clone` = 克隆，把远程仓库复制到本地
- `remote` = 远程
- `origin` = 来源，Git 默认给主远程仓库起的名字
- `branch` = 分支
- `main` = 主分支
- `fetch` = 获取
- `merge` = 合并
- `rebase` = 变基，把你的提交重新接到最新提交后面
- `remove` = 删除
- `rm` = `remove` 的缩写
- `move` = 移动
- `mv` = `move` 的缩写
- `cached` = 已缓存、已跟踪
- `upstream` = 上游，表示当前分支默认跟踪的远程分支
- `verbose` = 详细的

## 5. `git pull` 到底是什么意思

这是新手最容易“会用但不懂”的命令。

`git pull` 本质上约等于两步：

1. `git fetch`：先从远程把最新内容取回来
2. `git merge`：再把这些内容合并到当前分支

你可以这样记：

- `fetch`：只取货，不合并
- `pull`：拉回来并合并

### 最常见的两种写法

```powershell
git pull           # 拉取远程更新，并直接合并 merge
git pull --rebase  # 拉取远程更新，并把本地提交接到最新提交后面
```

### 它们的区别

- `git pull`：简单直接，但有时会多出一条 merge 记录
- `git pull --rebase`：提交历史通常更整洁

如果是你自己维护项目，平时更推荐：

```powershell
git pull --rebase   # 先同步远程，再把本地提交“顺到后面”
```

### 带分支名的写法是什么意思

```powershell
git pull origin main   # 从 origin 这个远程仓库，拉取 main 分支
```

这里可以这样记：

- `origin`：远程仓库默认名字
- `main`：远程主分支名字

## 6. 场景一：GitHub 上已经有一个空仓库，本地文件上传到 GitHub

这个场景最常见。

假设：

- 你本地已经有一个项目文件夹
- GitHub 上已经新建好了一个空仓库
- 远程仓库里没有 README、没有 LICENSE、没有 `.gitignore`

如果你还没创建空仓库，可以在 GitHub 网页上：

1. 点击右上角 `New repository`
2. 输入仓库名
3. 选择公开或私有
4. 不勾选 `Add a README file`
5. 不勾选 `.gitignore`
6. 不勾选 `License`
7. 点击 `Create repository`

## 6.1 进入本地项目目录

```powershell
cd D:\你的项目目录   # cd = change directory，进入项目目录
```

## 6.2 初始化本地 Git 仓库

```powershell
git init   # initialize，初始化当前目录为 Git 仓库
```

## 6.3 查看当前状态

```powershell
git status   # 查看哪些文件是新增、修改、删除状态
```

如果你看到 `Untracked files`，意思是这些文件还没被 Git 跟踪。

## 6.4 把文件加入暂存区

```powershell
git add .   # 把当前目录下所有改动加入暂存区
```

如果只想添加一个文件：

```powershell
git add README.md   # 只添加指定文件
```

## 6.5 创建第一次提交

```powershell
git commit -m "feat: initial commit"   # initial = 初始的，第一次提交
```

## 6.6 绑定 GitHub 远程仓库

### 用 HTTPS 绑定

```powershell
git remote add origin https://github.com/你的用户名/你的仓库名.git   # remote = 远程，add = 添加
```

### 用 SSH 绑定

```powershell
git remote add origin git@github.com:你的用户名/你的仓库名.git   # 用 SSH 地址绑定远程仓库
```

查看绑定结果：

```powershell
git remote -v   # -v = verbose，显示详细远程地址
```

## 6.7 首次推送到 GitHub

```powershell
git branch -M main       # branch = 分支，把当前分支重命名为 main
git push -u origin main  # push 到 origin 的 main；-u = upstream，建立默认跟踪关系
```

成功后，以后通常只需要：

```powershell
git push   # 因为已经建立了跟踪关系，所以不用每次都写 origin main
```

## 6.8 以后每次修改后的标准上传流程

```powershell
git status                    # 看改了什么
git add .                     # 把改动加入暂存区
git commit -m "fix: 修改说明"  # 生成本地提交
git pull --rebase             # 先同步远程，减少冲突
git push                      # 再推送到 GitHub
```

## 7. 场景二：GitHub 上还没有仓库，先创建仓库，再上传本地文件

这个场景和场景一本质一样，只是多了“先建远程仓库”这一步。

## 7.1 方法一：先在 GitHub 网页创建仓库

这是最推荐的方式。

步骤：

1. 登录 GitHub
2. 点击右上角 `New repository`
3. 填写仓库名
4. 选择 `Public` 或 `Private`
5. 如果你是上传本地已有项目，建议不要勾选 `Add a README file`
6. 点击 `Create repository`

然后执行：

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

如果你已经配置了 SSH，也可以把远程地址改成：

```powershell
git remote add origin git@github.com:你的用户名/你的仓库名.git   # 使用 SSH 地址
```

## 7.2 方法二：先在本地初始化，再去 GitHub 建仓库

你也可以先在本地执行：

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
```

然后去 GitHub 创建空仓库，再执行其中一种：

### HTTPS

```powershell
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

### SSH

```powershell
git remote add origin git@github.com:你的用户名/你的仓库名.git
git push -u origin main
```

## 8. 以后如何同步本地与 GitHub

真正熟练使用 Git，不只是会第一次上传，而是会长期同步。

## 8.1 本地修改后上传到 GitHub

```powershell
git status                     # 查看状态
git add .                      # 添加改动
git commit -m "docs: 更新文档"  # 本地提交
git push                       # 推送到远程
```

## 8.2 GitHub 上有新内容，拉回本地

```powershell
git pull   # 从远程拉回最新内容，并直接合并
```

更推荐：

```powershell
git pull --rebase   # 拉回远程内容，并把本地提交放到后面
```

## 8.3 最稳妥的同步顺序

如果你可能在多台电脑工作，或者 GitHub 上可能已有更新，建议固定使用：

```powershell
git status                   # 先确认当前仓库状态
git add .                    # 暂存本地改动
git commit -m "你的提交说明"  # 生成本地提交
git pull --rebase            # 拉取远程最新内容并整理提交顺序
git push                     # 最后推送到 GitHub
```

## 9. 文件删除、重命名、修改后如何同步

## 9.1 删除文件并同步到 GitHub

如果你直接在资源管理器里删了文件，也没问题：

```powershell
git status                         # 查看哪些文件被删除了
git add .                          # 把删除动作也加入暂存区
git commit -m "chore: 删除文件"    # 提交删除记录
git push                           # 推送到 GitHub
```

你也可以直接用 Git 删除：

```powershell
git rm old.txt                     # rm = remove，直接用 Git 删除文件
git commit -m "chore: remove old.txt"
git push
```

## 9.2 删除文件夹并同步

```powershell
git rm -r old-folder               # -r = recursive，递归删除整个目录
git commit -m "chore: 删除旧目录"
git push
```

## 9.3 重命名文件

你可以直接在资源管理器改名，然后执行：

```powershell
git add .                          # Git 通常能识别出重命名
git commit -m "refactor: 重命名文件"
git push
```

也可以直接用 Git：

```powershell
git mv old-name.txt new-name.txt   # mv = move，用 Git 直接移动或重命名
git commit -m "refactor: rename file"
git push
```

## 9.4 修改文件内容后同步

```powershell
git add .                          # 添加修改
git commit -m "fix: 修改文件内容"   # 提交修改
git push                           # 推送修改
```

## 10. 新电脑或新目录如何把 GitHub 仓库拉到本地

如果远程仓库已经存在，你想在另一台电脑继续开发，用 `clone`。

### HTTPS 克隆

```powershell
git clone https://github.com/你的用户名/你的仓库名.git   # clone = 克隆，复制远程仓库到本地
```

### SSH 克隆

```powershell
git clone git@github.com:你的用户名/你的仓库名.git   # 用 SSH 地址克隆
```

进入目录：

```powershell
cd 你的仓库名   # 进入 clone 下来的项目目录
```

以后在这里继续：

```powershell
git pull                     # 先同步远程最新内容
git add .                    # 添加本地改动
git commit -m "你的说明"      # 本地提交
git push                     # 再推送回 GitHub
```

## 11. `.gitignore` 是什么

有些文件不应该上传，比如：

- 编译产物
- 临时文件
- 日志
- 密钥
- 本地编辑器配置

这时应该创建 `.gitignore` 文件。

示例：

```gitignore
node_modules/
dist/
*.log
.env
.vscode/
```

注意两点：

- `.gitignore` 只对“还没有被 Git 跟踪”的文件生效
- 如果某文件已经提交过，仅写入 `.gitignore` 不够

如果一个已经上传过的文件以后想忽略，需要先取消跟踪：

```powershell
git rm --cached .env                    # --cached = 只取消跟踪，不删除本地文件
git commit -m "chore: stop tracking .env"
git push
```

## 12. 如何查看或修改远程仓库地址

查看当前远程仓库：

```powershell
git remote -v   # 查看 origin 对应的远程地址
```

如果你想把 HTTPS 改成 SSH：

```powershell
git remote set-url origin git@github.com:你的用户名/你的仓库名.git   # set-url = 修改地址
```

如果你想把 SSH 改成 HTTPS：

```powershell
git remote set-url origin https://github.com/你的用户名/你的仓库名.git
```

## 13. 常见报错与处理

## 13.1 `remote origin already exists`

意思是：`origin` 这个远程名字已经存在了。

先查看：

```powershell
git remote -v   # 看看当前 origin 指向哪里
```

如果地址错了，直接改：

```powershell
git remote set-url origin https://github.com/你的用户名/你的仓库名.git
```

如果你就是想重新绑定，也可以先删再加：

```powershell
git remote remove origin                                             # remove = 删除旧远程定义
git remote add origin https://github.com/你的用户名/你的仓库名.git    # 重新添加正确地址
```

如果你想用 SSH，也可以写成：

```powershell
git remote set-url origin git@github.com:你的用户名/你的仓库名.git
```

## 13.2 `failed to push some refs`

通常表示远程仓库比你本地更新，先拉取再推送：

```powershell
git pull --rebase origin main   # 先把远程 main 拉回来
git push origin main            # 再推送本地 main
```

## 13.3 `rejected` 或 `non-fast-forward`

本质和上一条类似，表示远程有你本地没有的提交。

处理方式：

```powershell
git pull --rebase   # 先同步远程
git push            # 再推送
```

## 13.4 首次推送提示需要认证

这通常是正常的：

- 用 HTTPS：按提示登录 GitHub 或输入 Token
- 用 SSH：确认是否已经正确配置 SSH Key

可以用下面命令检查 SSH：

```powershell
ssh -T git@github.com   # 如果 SSH 有问题，这里通常会暴露出来
```

## 13.5 GitHub 仓库不是空仓库

如果你创建仓库时勾选了 `README`、`.gitignore` 或 `LICENSE`，远程就不是空仓库。

此时首次推送可能失败，处理方式：

```powershell
git pull --rebase origin main   # 先把远程已有文件拉下来
git push origin main            # 再推送本地内容
```

如果发生冲突，按 Git 提示解决冲突后，再继续 `add`、`commit`、`push`。

## 14. 最推荐你记住的完整工作流

## 14.1 第一次上传本地项目到 GitHub

### HTTPS 版本

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

### SSH 版本

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
git remote add origin git@github.com:你的用户名/你的仓库名.git
git push -u origin main
```

## 14.2 以后每次修改后的上传流程

```powershell
git status                   # 看状态
git add .                    # 添加改动
git commit -m "你的修改说明"  # 本地提交
git pull --rebase            # 先同步远程
git push                     # 再推送远程
```

## 14.3 删除文件后的同步流程

```powershell
git add .                    # 让 Git 识别删除动作
git commit -m "chore: 删除文件"
git push
```

或者：

```powershell
git rm 文件名                # 直接用 Git 删除
git commit -m "chore: 删除文件"
git push
```

## 15. 帮助记忆的一句话口诀

### 第一次上传

`init -> add -> commit -> remote -> push`

对应命令：

```powershell
git init
git add .
git commit -m "first commit"
git remote add origin 仓库地址
git push -u origin main
```

### 日常更新

`status -> add -> commit -> pull -> push`

对应命令：

```powershell
git status
git add .
git commit -m "说明"
git pull --rebase
git push
```

### 删除同步

`delete -> add -> commit -> push`

对应命令：

```powershell
git add .
git commit -m "删除说明"
git push
```

## 16. 最适合你的练习方式

建议你按这个顺序实操一遍：

1. 在本地新建一个测试目录，放两个文本文件。
2. 去 GitHub 创建一个空仓库。
3. 用 HTTPS 完成一次首次推送。
4. 再把远程地址改成 SSH。
5. 修改一个文件，再做一次 `add`、`commit`、`pull --rebase`、`push`。
6. 删除一个文件，再推送一次。
7. 在 GitHub 网页上改一个文件，然后回本地执行 `git pull`。

如果这几步你能独立做完，就已经具备日常使用 Git 和 GitHub 的基础能力了。
