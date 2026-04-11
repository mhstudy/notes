# Git

> 🔗 **官方网站**：https://git-scm.com/
> 📖 **官方文档**：https://git-scm.com/doc
> 📌 **学习版本**：Git 2.x

---

## 第1章 Git 概述 ⭐

Git 是目前世界上最先进的**分布式版本控制系统**。

### 工作区域

```
工作区（Working Directory） → 暂存区（Stage） → 本地仓库（Repository） → 远程仓库（Remote）
         git add                 git commit              git push
```

---

## 第2章 常用命令速查 🔥

### 2.1 基础操作

```bash
git init                    # 初始化仓库
git clone <url>             # 克隆远程仓库
git status                  # 查看状态
git add .                   # 添加所有文件到暂存区
git commit -m "message"     # 提交
git push origin main        # 推送到远程
git pull origin main        # 拉取远程更新
```

### 2.2 分支操作

```bash
git branch                  # 查看分支
git branch dev              # 创建分支
git checkout dev            # 切换分支
git checkout -b dev         # 创建并切换
git merge dev               # 合并分支
git branch -d dev           # 删除分支
```

### 2.3 版本回退

```bash
git log --oneline           # 查看提交历史
git reflog                  # 查看所有操作记录
git reset --hard HEAD^      # 回退一个版本
git reset --hard <commit>   # 回退到指定版本
git stash                   # 暂存当前修改
git stash pop               # 恢复暂存
```

### 2.4 远程操作

```bash
git remote -v               # 查看远程仓库
git remote add origin <url> # 添加远程仓库
git fetch                   # 获取远程更新（不合并）
git pull                    # 获取并合并
```

---

## 第3章 Git 工作流 🔥

### 3.1 常见工作流

| 工作流 | 说明 | 适用团队 |
|:---|:---|:---|
| **Git Flow** | master + develop + feature/release/hotfix 分支 | 大型项目 |
| **GitHub Flow** 🔥 | main + feature 分支 + PR | 中小团队 |
| **GitLab Flow** | 环境分支 production/staging + feature | CI/CD 场景 |

### 3.2 Git Flow 分支模型

```
master  ─────●─────────────────●──────────▶ (生产发布)
              \               /
release        ──●──●──●────   (发布准备)
                /
develop ──●──●──●──●──●──●──●──●──●──────▶ (开发主线)
           \      /  \      /
feature     ──●──    ──●──    (功能开发)
```

---

## 第4章 冲突解决 ⭐

### 4.1 合并冲突

```bash
# 合并时出现冲突
git merge dev
# CONFLICT (content): Merge conflict in xxx.java

# 手动编辑冲突文件
<<<<<<< HEAD
当前分支的内容
=======
合并分支的内容
>>>>>>> dev

# 解决后提交
git add .
git commit -m "resolve merge conflict"
```

### 4.2 变基冲突

```bash
# rebase 时出现冲突
git rebase main
# 解决冲突后
git add .
git rebase --continue

# 放弃 rebase
git rebase --abort
```

---

## 第5章 实用配置 ⭐

### 5.1 .gitignore

```gitignore
# IDE
.idea/
*.iml
.vscode/

# 编译输出
target/
*.class
*.jar

# 日志
*.log
logs/

# 系统文件
.DS_Store
Thumbs.db

# 环境配置
.env
*.local
```

### 5.2 常用别名配置

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"
```

### 5.3 SSH 配置（免密推送）

```bash
# 生成 SSH 密钥
ssh-keygen -t rsa -C "your_email@example.com"

# 查看公钥（添加到 GitHub/GitLab）
cat ~/.ssh/id_rsa.pub

# 测试连接
ssh -T git@github.com
```

---

## 第6章 面试要点 🔥🔥

### Q1：git merge 和 git rebase 的区别？
> merge 会产生新的合并 commit，保留完整历史。rebase 会将提交移到目标分支最新位置，历史线性化。
> **团队协作建议**：公共分支用 merge，个人分支用 rebase。

### Q2：git reset 和 git revert 的区别？
> reset 直接回退版本（修改历史），revert 创建新的 commit 来撤销（不修改历史）。
> **公共分支只能用 revert**（不能改历史）。

### Q3：git fetch 和 git pull 的区别？
> `git pull` = `git fetch` + `git merge`。fetch 只下载远程更新但不合并，pull 自动合并。

### Q4：如何回退已 push 的 commit？
> 1. `git revert <commit>` —— 安全方式，新建撤销 commit
> 2. `git reset --hard <commit>` + `git push -f` —— 强制覆盖（**危险！慎用**）

### Q5：什么是 Git 的三棵树？
> **工作区**（Working Directory）：实际文件
> **暂存区**（Staging Area / Index）：`git add` 后的快照
> **版本库**（Repository）：`git commit` 后的历史
