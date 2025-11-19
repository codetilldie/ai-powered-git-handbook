## 第三章：Git 高级功能

当你熟练掌握了 Git 的日常操作后，就可以开始探索一些能极大提升效率、解决复杂问题的高级功能了。本章将介绍 `stash`（储藏）、交互式变基、子模块、钩子以及 Git 的“时光机” `reflog`。

### 3.1 储藏 (Stash)

`git stash` 是一个非常有用的命令，它能帮你临时保存工作区和暂存区的改动，并将你的工作区恢复到 `HEAD` 提交时的干净状态。

#### 使用场景
- **紧急任务切换**：你正在 `feature-A` 分支上开发，代码改了一半，这时线上突然出现一个紧急 Bug 需要你立即修复。但你当前的代码还不能提交，怎么办？使用 `git stash` 将当前改动存起来，然后切换到 `hotfix` 分支进行修复，修复完毕后再切回来，恢复储藏继续工作。
- **保持工作区干净**：在执行 `git pull` 或 `git rebase` 等可能与本地改动冲突的操作前，先 `stash` 一下，可以有效避免不必要的合并冲突。

#### 常用命令
```bash
# 将当前工作区和暂存区的改动储藏起来
git stash

# 为储藏添加描述信息，方便后续查找
git stash save "正在开发用户认证功能"

# 查看所有储藏
git stash list
# 输出示例:
# stash@{0}: On feature-A: 正在开发用户认证功能
# stash@{1}: WIP on master: 9c34a1b chore: update dependencies

# 恢复最近一次的储藏（并从储藏列表中移除）
git stash pop

# 恢复最近一次的储藏（但仍在储藏列表中保留）
git stash apply

# 恢复指定的储藏 (e.g., stash@{1})
git stash apply stash@{1}

# 查看某次储藏的具体改动
git stash show stash@{0}

# 移除最近一次的储藏
git stash drop

# 清空所有储藏
git stash clear
```

#### 示例：紧急任务切换

让我们通过一个完整的场景来理解 `stash` 的威力。

1.  **初始状态**：你正在 `feature-A` 分支开发，修改了 `auth.js` 并新增了 `auth.css`。
    ```bash
    # feature-A 分支
    $ git status
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git restore <file>..." to discard changes in working directory)
            modified:   auth.js

    Untracked files:
      (use "git add <file>..." to include in what will be committed)
            auth.css
    ```

2.  **储藏改动**：此时接到紧急任务，需要立即切换分支。你将当前改动（包括未暂存和新增的文件）全部储藏起来。
    ```bash
    # 使用 -u 参数可以一并储藏未追踪的文件 (Untracked files)
    $ git stash save -u "开发登录页面进行到一半"
    Saved working directory and index state On feature-A: 开发登录页面进行到一半
    
    $ git status
    On branch feature-A
    nothing to commit, working tree clean
    ```
    现在你的工作区是干净的，可以安全切换分支。

3.  **修复紧急 Bug**：切换到 `hotfix` 分支，完成修改、提交和推送。
    ```bash
    $ git switch master
    $ git switch -c hotfix-123
    # ... 进行 Bug 修复和提交 ...
    $ git push origin hotfix-123
    ```

4.  **切回原分支并恢复**：完成紧急任务后，切回原来的 `feature-A` 分支，恢复之前的工作。
    ```bash
    $ git switch feature-A
    $ git stash list
    stash@{0}: On feature-A: 开发登录页面进行到一半

    $ git stash pop
    On branch feature-A
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git restore <file>..." to discard changes in working directory)
            modified:   auth.js

    Untracked files:
      (use "git add <file>..." to include in what will be committed)
            auth.css
    Dropped stash@{0} (a1b2c3d...)
    ```
    `git stash pop` 后，你之前对 `auth.js` 和 `auth.css` 的改动被完美地恢复了，就像你从未离开过一样，可以继续安心开发。

---

### 3.2 交互式变基 (Interactive Rebase)

`git rebase -i` 是 Git 的一大神器。它允许你回到过去，以交互的方式“编辑”一系列的提交历史，使其变得更清晰、更专业。

#### 使用场景
- **合并多个零散提交**：在开发过程中，我们常会有 "fix typo", "update", "wip" 等零散的提交。在合并到主分支前，应该使用交互式变基将它们合并成一个有意义的完整提交。
- **修改提交信息**：发现之前的某次提交信息写错了。
- **重新排序提交**：调整提交的顺序，使逻辑更连贯。
- **删除某个提交**：移除一个无用的或错误的提交。

#### 操作流程：合并零散提交 (squash)

假设你的提交历史如下，包含了一些零散的、可以合并的提交：
```bash
$ git log --oneline
9g8h9i0 (HEAD -> feature-login) style: format code alignment
5d6e7f8 fix: correct input validation
1a2b3c4 feat: add initial login form
f12ab34 (master) initial commit
```

**目标**：将 `style` 和 `fix` 两个提交合并到 `feat` 提交中，形成一个完整的“登录功能”提交。

1.  **启动交互式变基**：我们想整理最近的 3 次提交，所以执行：
    ```bash
    git rebase -i HEAD~3
    ```
2.  **编辑变基脚本**：Git 会打开一个文本编辑器，显示这 3 次提交。
    ```
    pick 1a2b3c4 feat: add initial login form
    pick 5d6e7f8 fix: correct input validation
    pick 9g8h9i0 style: format code alignment
    ...
    ```
    按照我们的目标，将第二次和第三次提交的 `pick` 修改为 `squash`（或简写 `s`），意思是将这次提交合并到它的上一个提交中。
    ```
    pick 1a2b3c4 feat: add initial login form
    s 5d6e7f8 fix: correct input validation
    s 9g8h9i0 style: format code alignment
    ```
3.  **保存并关闭**第一个编辑器。

4.  **编辑合并后的提交信息**：Git 会自动打开第二个编辑器，其中包含了这三次提交的全部信息，让你为合并后的**新提交**撰写一个全新的、更有意义的提交信息。
    ```
    # This is a combination of 3 commits.
    # This is the 1st commit message:

    feat: add initial login form

    # This is the 2nd commit message:

    fix: correct input validation

    # This is the 3rd commit message:

    style: format code alignment
    ...
    ```
    删除所有旧信息，写入一个清晰的、总结性的信息：
    ```
    feat(auth): implement user login functionality
    ```
5.  **保存并关闭**第二个编辑器。变基完成。

6.  **查看结果**：再次查看提交历史，你会发现原来的 3 次提交已经被一个全新的、干净的提交所取代。
    ```bash
    $ git log --oneline
    c4a3b2d (HEAD -> feature-login) feat(auth): implement user login functionality
    f12ab34 (master) initial commit
    ```
至此，你的提交历史变得非常专业和易于回顾。

> **警告**: 和普通的 `rebase` 一样，**永远不要在已经推送到公共仓库的提交上使用交互式变基**，因为它同样会重写历史。

---

### 3.3 子模块 (Submodules) 与子树 (Subtree)

当你的项目需要依赖另一个独立的 Git 仓库时（例如，一个通用的库），有两种方式可以将其引入。

#### `git submodule`

你的主项目只保存了对子模块仓库的**引用**（URL 和 Commit ID）。它像一个“快捷方式”，子模块有自己完全独立的历史。

- **优点**: 保持了主项目和依赖项目的完全分离，主项目仓库体积小。
- **缺点**: 操作相对复杂。克隆主项目后，需要额外执行 `git submodule init` 和 `git submodule update` 才能拉取子模块的代码。

**示例：添加一个共享组件库**
```bash
# 1. 在你的主项目中，添加一个子模块
# 这会在你的项目中创建一个 `libs/ui-components` 目录
git submodule add https://github.com/user/ui-components.git libs/ui-components

# 2. 提交变更
# 这会生成一个 .gitmodules 文件和一个指向子模块 commit 的特殊条目
git commit -m "feat: integrate ui-components submodule"

# 3. 克隆一个包含子模块的项目
git clone https://github.com/user/main-project.git
cd main-project

# 4. 初始化并拉取子模块的代码
git submodule init
git submodule update --recursive
# 或者一步到位：
# git clone --recurse-submodules https://github.com/user/main-project.git
```

#### `git subtree`

将另一个仓库的文件和历史**完全复制**到你的主项目中。它不像一个引用，而像一个“复印件”。

- **优点**: 操作简单，克隆主项目后，所有代码都在，无需额外步骤。
- **缺点**: 会使主项目的历史变得复杂，仓库体积增大。

**示例：添加一个辅助脚本库**
```bash
# 1. 添加一个远程，指向你要引入的仓库（方便后续更新）
git remote add helper-scripts https://github.com/user/helper-scripts.git

# 2. 使用 subtree add 将对方仓库的 master 分支内容拉取到 `tools/scripts` 目录下
# --prefix 指定了子目录，--squash 将对方的历史合并成一次提交
git subtree add --prefix=tools/scripts helper-scripts master --squash

# 3. 更新子树内容
git subtree pull --prefix=tools/scripts helper-scripts master --squash
```

**选择建议**: 如果你只是想使用某个第三方库，而不关心其后续更新，或者更新频率很低，`subtree` 更简单。如果你需要频繁地在主项目和依赖项目之间切换开发，或者希望严格保持项目分离，`submodule` 是更好的选择。

---

### 3.4 Git Hooks

Git Hooks (钩子) 是在 Git 仓库中特定事件（如 `commit`, `push`）发生时自动执行的脚本。它们是实现工作流自动化的强大工具。

- **位置**: 钩子脚本都存放在 `.git/hooks/` 目录下。当你初始化一个仓库时，Git 会在这里放入一堆以 `.sample` 结尾的示例脚本。
- **激活**: 要想激活一个钩子，只需去掉 `.sample` 后缀，并赋予其可执行权限 (`chmod +x .git/hooks/pre-commit`)。

#### 常用钩子

- **`pre-commit`**: 在你执行 `git commit` 后，Git 创建提交对象**之前**运行。常用于执行代码风格检查 (linting) 或单元测试。如果此脚本以非零状态退出，Git 将会中止本次提交。
- **`commit-msg`**: 在 `pre-commit` 钩子通过后运行，用于检查提交信息是否符合团队规范（例如，是否遵循 Conventional Commits 格式）。
- **`pre-push`**: 在你执行 `git push` 后，任何数据传输到远程仓库**之前**运行。常用于确保要推送的代码通过了所有测试。

#### 示例：一个简单的 `pre-commit` 钩子

假设我们想在每次提交前，检查代码中是否意外包含了 “TODO” 或 “FIXME” 等关键词。

1.  **创建钩子文件**：在你的项目根目录下，创建（或重命名） `.git/hooks/pre-commit` 文件，并写入以下 Shell 脚本：

    ```sh
    #!/bin/sh

    # 检查暂存区的文件中是否包含 'TODO' 或 'FIXME'
    # --cached 参数让 grep 只检查暂存区
    if git grep --cached -n -E 'TODO|FIXME'; then
        echo "错误：检测到代码中存在 'TODO' 或 'FIXME' 关键词。"
        echo "请处理这些标记后再提交。"
        exit 1 # 以非零状态退出，中止提交
    fi

    echo "代码检查通过，继续提交..."
    exit 0 # 以零状态退出，允许提交
    ```

2.  **赋予执行权限**：
    ```bash
    chmod +x .git/hooks/pre-commit
    ```

3.  **测试钩子**：现在，如果你修改一个文件，在其中加入 `// TODO: apen-dev-gemini`，然后尝试提交，你会看到：
    ```bash
    $ git add .
    $ git commit -m "test hook"
    src/main.js:10: // TODO: apen-dev-gemini
    错误：检测到代码中存在 'TODO' 或 'FIXME' 关键词。
    请处理这些标记后再提交。
    ```
    提交被成功阻止了。只有当你移除这些关键词后，`git commit` 才能成功执行。这极大地保证了提交到仓库中的代码质量。

> **注意**: `.git/hooks` 目录本身不会被 Git 版本控制，因此团队成员需要手动设置自己的钩子。为了在团队中共享钩子，通常会使用 [Husky](https://typicode.github.io/husky/) 等第三方工具。

---

### 3.5 cherry-pick, reflog 等高级命令

#### `git cherry-pick <commit-hash>` (拣樱桃)

- **作用**: 将**单个**指定的提交“复制”一份，应用到当前所在的 `HEAD` 分支上。
- **场景**: `develop` 分支上一个修复 Bug 的提交（`fix-auth-bug`）需要被紧急应用到 `master` 分支，但 `develop` 分支本身还不能发布。

**示例：**

1.  **初始状态**：`master` 分支落后于 `develop` 分支，且 `develop` 分支有一个我们需要的提交 `b2b2b2b`。
    ```bash
    # develop 分支的历史
    $ git log --oneline develop
    c3c3c3c (HEAD -> develop) feat: add new dashboard
    b2b2b2b fix: correct authentication bug
    a1a1a1a feat: initial commit for user profile

    # master 分支的历史
    $ git log --oneline master
    a1a1a1a (master) feat: initial commit for user profile
    ```

2.  **执行 Cherry-pick**：切换到 `master` 分支，然后拣选我们需要的提交。
    ```bash
    $ git switch master
    $ git cherry-pick b2b2b2b
    [master 3d3d3d3] fix: correct authentication bug
     Date: Wed Nov 19 11:00:00 2025 +0800
     1 file changed, 2 insertions(+)
    ```

3.  **最终结果**：`b2b2b2b` 提交的**内容**被复制成了一个**新的提交** `3d3d3d3` 并应用在了 `master` 分支上。
    ```bash
    $ git log --oneline master
    3d3d3d3 (HEAD -> master) fix: correct authentication bug
    a1a1a1a feat: initial commit for user profile
    ```

#### `git reflog` (引用日志)

- **作用**: `reflog` 是 Git 的终极“后悔药”。它记录了你的 `HEAD` 和所有分支指针**在过去一段时间内的所有移动**。`commit`, `rebase`, `reset`, `checkout`... 你的每一次“危险”操作都被它看在眼里。
- **场景**: 你不小心用 `git branch -D` 强制删除了一个还未合并的特性分支。

**示例：恢复被误删的分支**

1.  **误删分支**：你有一个名为 `feature-super-secret` 的分支，并做了一些提交。然后不小心强制删除了它。
    ```bash
    $ git log --oneline feature-super-secret
    d4d4d4d (HEAD -> feature-super-secret) feat: implement the secret feature
    c3c3c3c initial commit

    $ git switch master
    $ git branch -D feature-super-secret
    Deleted branch feature-super-secret (was d4d4d4d).

    # 此刻，git log 无法再看到这个分支和它的提交了
    ```

2.  **使用 reflog 查找线索**：`reflog` 记录了你所有的操作，包括最后一次在那个被删分支上的提交。
    ```bash
    $ git reflog
    a1a1a1a HEAD@{0}: switch: from feature-super-secret to master
    d4d4d4d HEAD@{1}: commit: feat: implement the secret feature
    c3c3c3c HEAD@{2}: switch: created and switched to branch 'feature-super-secret'
    ...
    ```
    我们从日志中看到，在切换到 `master` 之前，`HEAD` 指向 `d4d4d4d`，这正是我们丢失的提交！

3.  **从 reflog 恢复分支**：直接从这个丢失的提交哈希值上创建一个新分支，即可完美恢复。
    ```bash
    $ git branch feature-super-secret-recovered d4d4d4d

    # 检查一下新恢复的分支
    $ git log --oneline feature-super-secret-recovered
    d4d4d4d (HEAD -> feature-super-secret-recovered) feat: implement the secret feature
    c3c3c3c initial commit
    ```
    所有工作都找回来了！`reflog` 是你本地操作的终极安全网（但请注意它不会被推送到远程）。