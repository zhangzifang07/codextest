# Git 与 GitHub 实战入门指南

这份文档的目标不是只让你“传一次文件”，而是让你能稳定完成下面这些常见工作：

- 本地创建 Git 仓库
- 把本地文件上传到 GitHub
- GitHub 先有空仓库时完成首次推送
- GitHub 还没有仓库时创建仓库并完成首次推送
- 后续持续同步本地与远程内容
- 删除、重命名、覆盖、拉取更新
- 遇到常见报错时知道怎么处理

本文默认你在 Windows 的 PowerShell 中操作。

## 1. 先理解 Git 和 GitHub

- `Git`：本地版本管理工具，负责记录文件变化。
- `GitHub`：远程代码托管平台，负责保存远程仓库，方便备份、协作和同步。
- `本地仓库`：你电脑上的项目目录。
- `远程仓库`：GitHub 上的仓库。
- `commit`：一次本地版本记录。
- `push`：把本地提交上传到 GitHub。
- `pull`：把 GitHub 上的最新变化拉回本地。

你可以把它理解成：

1. 先在本地改文件。
2. 用 Git 记录这次改动。
3. 再推送到 GitHub。

## 2. 使用前准备

## 2.1 安装 Git

先确认是否已经安装：

```powershell
git --version
```

如果能看到类似 `git version 2.x.x`，说明已经安装。

如果没有安装，到 Git 官网下载安装：

- [Git for Windows](https://git-scm.com/download/win)

安装时大部分选项保持默认即可。

## 2.2 配置用户名和邮箱

首次使用建议配置一次：

```powershell
git config --global user.name "你的名字"
git config --global user.email "你的GitHub邮箱"
```

检查配置：

```powershell
git config --global --list
```

## 2.3 GitHub 登录说明

如果你使用 HTTPS 地址推送到 GitHub，通常不会再直接使用 GitHub 密码，而是使用下面任一方式：

- 浏览器授权登录
- Personal Access Token，简称 `PAT`
- Git Credential Manager 自动保存凭据

如果 Git 在推送时弹出登录窗口，按提示完成即可。

## 3. 最核心的日常命令

先记住这几个：

```powershell
git status
git add .
git commit -m "说明本次修改"
git push
git pull
```

它们分别表示：

- `git status`：查看当前改了什么
- `git add .`：把当前改动加入暂存区
- `git commit -m "..."`：生成一次本地提交
- `git push`：上传到 GitHub
- `git pull`：先从 GitHub 拉最新内容到本地

## 4. 场景一：GitHub 上已经有一个空仓库，本地文件上传到 GitHub

这个场景最常见，也最适合新手。

假设：

- 你电脑里已经有一个项目文件夹
- GitHub 上已经新建好了一个空仓库
- 这个空仓库里没有 README、没有 LICENSE、没有 `.gitignore`

如果你还没创建空仓库，可以在 GitHub 网页上：

1. 点击右上角 `New repository`
2. 输入仓库名
3. 选择公开或私有
4. 不要勾选 `Add a README file`
5. 不要勾选 `.gitignore`
6. 不要勾选 `License`
7. 点击 `Create repository`

这样创建出来的仓库才是真正“空仓库”，最适合本地首次上传。

### 4.1 进入本地项目目录

```powershell
cd D:\你的项目目录
```

### 4.2 初始化本地 Git 仓库

```powershell
git init
```

执行后，当前目录就变成了 Git 仓库。

### 4.3 查看当前状态

```powershell
git status
```

如果目录里原本有很多文件，这里会看到它们显示为 `Untracked files`。

### 4.4 添加文件到暂存区

```powershell
git add .
```

说明：

- `.` 表示当前目录下所有改动
- 如果你只想上传某个文件，也可以写具体文件名

例如：

```powershell
git add README.md
```

### 4.5 创建第一次提交

```powershell
git commit -m "feat: initial commit"
```

### 4.6 绑定 GitHub 远程仓库

在 GitHub 仓库页面复制仓库地址，通常类似：

```text
https://github.com/你的用户名/你的仓库名.git
```

然后执行：

```powershell
git remote add origin https://github.com/你的用户名/你的仓库名.git
```

检查是否绑定成功：

```powershell
git remote -v
```

### 4.7 推送到 GitHub

现在执行首次推送：

```powershell
git branch -M main
git push -u origin main
```

说明：

- `git branch -M main`：把当前主分支统一命名为 `main`
- `git push -u origin main`：首次推送并建立上游分支关系

成功后，以后只需要：

```powershell
git push
```

### 4.8 以后修改文件后的标准流程

每次本地修改后，按这个顺序做：

```powershell
git status
git add .
git commit -m "fix: 修改说明"
git push
```

如果你担心远程有人改过，先执行：

```powershell
git pull --rebase
```

再执行 `git push`。

## 5. 场景二：GitHub 上还没有仓库，先创建仓库，再上传本地文件

这和场景一的本质差不多，只是多了“创建 GitHub 仓库”这一步。

## 5.1 方法一：通过 GitHub 网页创建仓库

这是最推荐的方式，最适合初学者。

操作步骤：

1. 登录 GitHub
2. 点击右上角 `New repository`
3. 填写仓库名
4. 选择 `Public` 或 `Private`
5. 如果你是要上传本地已有项目，建议不要勾选 `Add a README file`
6. 点击 `Create repository`

仓库创建完后，GitHub 会显示一组命令。你可以直接使用“push an existing repository from the command line”那部分。

标准流程如下：

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

## 5.2 方法二：先在本地做完，再去 GitHub 建仓库

也可以先在本地操作：

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
```

然后去 GitHub 创建一个空仓库，再执行：

```powershell
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

## 6. 后续如何同步本地与 GitHub

这部分非常重要。真正熟练使用 Git，不只是会“第一次上传”，而是能长期同步。

### 6.1 本地改完后上传到 GitHub

```powershell
git status
git add .
git commit -m "docs: 更新文档"
git push
```

### 6.2 GitHub 上有新内容，拉回本地

```powershell
git pull
```

更推荐：

```powershell
git pull --rebase
```

这样提交历史通常更整洁。

### 6.3 上传前的稳妥顺序

如果你在多台电脑工作，或者 GitHub 上可能有改动，推荐固定使用下面顺序：

```powershell
git status
git add .
git commit -m "你的提交说明"
git pull --rebase
git push
```

## 7. 文件删除、重命名、修改后的同步方式

## 7.1 删除文件并同步到 GitHub

如果你直接在资源管理器里删除了文件，也没问题：

```powershell
git status
git add .
git commit -m "chore: 删除无用文件"
git push
```

Git 会识别出删除操作，并同步到 GitHub。

你也可以用 Git 删除：

```powershell
git rm 文件名
git commit -m "chore: 删除文件"
git push
```

例如：

```powershell
git rm old.txt
git commit -m "chore: remove old.txt"
git push
```

## 7.2 删除文件夹并同步到 GitHub

```powershell
git rm -r 文件夹名
git commit -m "chore: 删除旧目录"
git push
```

## 7.3 重命名文件

你可以直接在资源管理器改名，然后：

```powershell
git add .
git commit -m "refactor: 重命名文件"
git push
```

或者用 Git：

```powershell
git mv 旧文件名 新文件名
git commit -m "refactor: rename file"
git push
```

## 7.4 修改文件内容并同步

```powershell
git add .
git commit -m "fix: 修改文件内容"
git push
```

## 8. 新电脑或新目录如何把 GitHub 仓库拉到本地

如果远程仓库已经存在，你想在另一台电脑继续开发，用 `clone`：

```powershell
git clone https://github.com/你的用户名/你的仓库名.git
```

执行后会在当前目录生成一个同名文件夹。

进入目录：

```powershell
cd 你的仓库名
```

以后就在这里继续：

```powershell
git pull
git add .
git commit -m "你的说明"
git push
```

## 9. `.gitignore` 是什么

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

注意：

- `.gitignore` 只对“还没有被 Git 跟踪”的文件生效
- 如果某文件已经提交过，仅仅写入 `.gitignore` 还不够

如果一个已经上传过的文件以后想忽略，需要先取消跟踪：

```powershell
git rm --cached 文件名
git commit -m "chore: stop tracking file"
git push
```

## 10. 如何查看远程仓库信息

查看绑定的远程仓库：

```powershell
git remote -v
```

如果你想修改远程地址：

```powershell
git remote set-url origin https://github.com/你的用户名/新的仓库名.git
```

## 11. 常见报错与处理

## 11.1 `remote origin already exists`

说明你已经绑定过远程仓库了。

先查看：

```powershell
git remote -v
```

如果地址错了，改成新的：

```powershell
git remote set-url origin https://github.com/你的用户名/你的仓库名.git
```

如果你确实想重新绑定，也可以先删除再添加：

```powershell
git remote remove origin
git remote add origin https://github.com/你的用户名/你的仓库名.git
```

## 11.2 `failed to push some refs`

通常表示远程仓库比你本地更新，先拉取再推送：

```powershell
git pull --rebase origin main
git push origin main
```

## 11.3 `rejected` 或 `non-fast-forward`

本质和上一条类似，说明远程有你本地没有的提交。

处理方式：

```powershell
git pull --rebase
git push
```

## 11.4 首次推送提示需要认证

这通常是正常的，按提示完成 GitHub 登录或输入 Token 即可。

## 11.5 GitHub 仓库不是空仓库

如果你在 GitHub 创建仓库时勾选了 `README`、`.gitignore` 或 `LICENSE`，远程就不是空仓库。

此时首次推送可能失败。

处理方式：

```powershell
git pull --rebase origin main
git push origin main
```

如果发生冲突，按 Git 提示解决冲突后，再继续提交和推送。

## 12. 最推荐你记住的完整工作流

## 12.1 第一次上传本地项目到 GitHub

```powershell
cd D:\你的项目目录
git init
git add .
git commit -m "feat: initial commit"
git branch -M main
git remote add origin https://github.com/你的用户名/你的仓库名.git
git push -u origin main
```

## 12.2 以后每次修改后的上传流程

```powershell
git status
git add .
git commit -m "你的修改说明"
git pull --rebase
git push
```

## 12.3 删除文件后的同步流程

```powershell
git add .
git commit -m "chore: 删除文件"
git push
```

或者：

```powershell
git rm 文件名
git commit -m "chore: 删除文件"
git push
```

## 13. 提交说明怎么写

建议养成好习惯，提交信息写清楚一点。

可以参考：

- `feat: 新增登录功能`
- `fix: 修复图片上传失败问题`
- `docs: 补充 Git 使用说明`
- `refactor: 重构配置加载逻辑`
- `test: 增加登录模块测试`
- `chore: 删除无用文件`

## 14. 你接下来最适合怎么练

建议你按下面顺序练习一遍：

1. 在本地新建一个测试目录，放两个文本文件。
2. 去 GitHub 创建一个空仓库。
3. 按“场景一”完整做一次首次推送。
4. 本地修改一个文件，再执行一次 `add`、`commit`、`push`。
5. 删除一个文件，再推送一次。
6. 在 GitHub 网页上改一个文件，再回本地执行 `git pull`。

如果这 6 步你能独立完成，已经具备日常使用 Git 和 GitHub 的基础能力了。

## 15. 一句话总结

日常最常用的模式其实只有这一套：

```powershell
git status
git add .
git commit -m "说明"
git pull --rebase
git push
```

第一次上传时，只是额外多了：

```powershell
git init
git remote add origin 仓库地址
git branch -M main
git push -u origin main
```

如果你愿意，下一步我可以继续帮你补一份更适合新手反复照着敲的“Git 命令速查表”。
