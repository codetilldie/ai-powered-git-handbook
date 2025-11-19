## 第五章：常见问题与解决方案 (FAQ)

本章汇集了开发者在使用 Git 过程中最常遇到的问题，并提供简单直接的解决方案。

### 5.1 如何撤销一次错误的提交？

这是一个非常常见的问题，根据你的具体需求，有不同的“后悔药”。

#### 场景一：安全地撤销一次**已推送到远程**的提交

如果你想撤销的提交已经被 `push` 到了公共仓库，**绝对不要**使用会修改历史的命令（如 `git reset`）。正确的做法是使用 `git revert`。

- **`git revert <commit-hash>`**: `revert` 会创建一个**新的提交**，这个新提交的内容刚好与你想要撤销的那次提交内容相反。它“反做”了那次提交的修改。
  - **优点**: 这是一种非破坏性的操作，它保留了完整的历史记录，对于团队协作来说是绝对安全的。别人 `pull` 代码时，只会看到一次新的“撤销提交”，不会造成历史混乱。

```bash
# 假设我们的提交历史如下，我们想撤销 "feat: add user profile" 这个提交
$ git log --oneline
1a2b3c4d feat: add user profile
c89efd7a docs: update README
f12ab345 initial commit

# 执行 revert
$ git revert 1a2b3c4d
# Git 会打开编辑器让你确认新的提交信息，默认为 "Revert 'feat: add user profile'"
# 保存退出后，revert 完成

# 再次查看历史，会发现多了一个新的“撤销”提交
$ git log --oneline
e8b4d5a1 Revert "feat: add user profile"
1a2b3c4d feat: add user profile
c89efd7a docs: update README
f12ab345 initial commit

# 现在可以将这个变更安全地推送到远程仓库
$ git push
```

#### 场景二：修改**还未推送到远程**的最后一次提交

如果你刚刚提交，但发现有些小错误（比如一个拼写错误，或者漏掉一个文件），并且**还没有 push**，最简单的办法是使用 `--amend`。

- **`git commit --amend`**: 这个命令会将暂存区的改动合并到**上一次**的提交中，而不是创建一个全新的提交。它也允许你重新编辑上一次的提交信息。

```bash
# 假设你刚刚提交完，又发现一个错误
git commit -m "feat: add user profile"

# 修复那个错误后，将修改的文件再次添加到暂存区
git add fixed-file.js

# 使用 --amend 来更新上一次提交
git commit --amend --no-edit
# --no-edit 参数表示你不想修改上次的提交信息，只想把新的改动加进去
```

#### 场景三：彻底丢弃**还未推送到远程**的几次提交

如果你在本地做了一系列错误的提交，并且确定不再需要它们，也**没有 push**，你可以使用 `git reset` 将你的分支“重置”到过去某个时间点。

- **`git reset <commit-hash>`**: `reset` 会将当前分支的指针移动到你指定的那个提交。它有三种模式：
  - **`--soft`**: **最温和**。只移动分支指针，你的工作区和暂存区都**不会**有任何变化。
  - **`--mixed` (默认)**: 移动分支指针，同时**重置暂存区**。你的工作区代码**不变**，但所有改动都处于“未暂存”状态。
  - **`--hard`**: **最危险**！移动分支指针，同时**重置暂存区和工作区**。所有未提交的本地修改将**永久丢失**！

```bash
# 假设你想彻底丢弃最近两次的提交
# 先用 git log 找到你想回到的那个提交的哈希值
git log

# 回到指定的提交，并抛弃之后的所有提交和改动
git reset --hard 1a2b3c4d
```
> **警告**: 再次强调，**永远不要**对已经推送到公共仓库的提交使用 `git reset`。

---

### 5.2 `.gitignore` 文件为什么不起作用？

最常见的原因是：**你想要忽略的文件已经被 Git 追踪了**。

`.gitignore` 文件只能忽略那些**从未被 Git 追踪过**的文件（untracked files）。如果一个文件已经被 `git add` 和 `git commit` 过，那么它就已经在 Git 的“管理名单”里了，`.gitignore` 对它自然就无效了。

**解决方案**:
你需要先从 Git 仓库中“移除”该文件的追踪，然后再提交这次移除。

```bash
# 1. 从 Git 的追踪列表（暂存区）中移除文件或文件夹，但保留本地物理文件
# 如果是单个文件
git rm --cached path/to/your/file.log
# 如果是整个文件夹（比如 node_modules）
git rm -r --cached node_modules/

# 2. 确认你的 .gitignore 文件里已经包含了要忽略的规则
# 例如，在 .gitignore 文件里写入:
# /node_modules
# *.log

# 3. 提交这次“移除追踪”的操作
git commit -m "chore: Stop tracking log files and node_modules"
```
完成这三步后，Git 就不会再“关心”这些文件的任何变化了。

---

### 5.3 如何修改历史提交信息？

#### 修改最后一次提交的信息

同场景二，如果提交还**未被 push**，使用 `git commit --amend` 是最简单的。
```bash
git commit --amend -m "这是一个新的、正确的提交信息"
```

#### 修改更早的、多次的提交信息

如果你需要修改的不是最后一次提交，而是历史中更早的某次或某几次提交，你需要使用**交互式变基 (`git rebase -i`)**。

**操作步骤**:
1.  找到你要修改的提交的**前一个**提交的哈希值。假设你要修改最近 3 次提交中的某一个，可以执行：
    ```bash
    git rebase -i HEAD~3
    ```
2.  在打开的编辑器中，找到你想要修改信息的那一行，将行首的 `pick` 改为 `reword` (或 `r`)。
    ```
    pick 1a2b3c4 A commit I don't want to change
    reword 5d6e7f8 A commit with a typo in the message
    pick 9g8h9i0 Another commit
    ```
3.  保存并关闭文件。
4.  Git 会逐一处理，当遇到 `reword` 的提交时，它会暂停并打开一个新的编辑器，让你输入新的提交信息。
5.  修改完成后，保存并退出。`rebase` 会继续进行，直到完成。

> **警告**: 这同样是修改历史的操作，**请勿**在已经推送到公共仓库的提交上使用。