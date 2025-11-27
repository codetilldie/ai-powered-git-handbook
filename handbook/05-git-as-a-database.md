## 第五章：从数据库视角理解 Git

为了真正揭开 Git 的神秘面纱，我们可以换一个全新的视角：**把 Git想象成一个特殊设计、高度优化的数据库系统**。这个数据库的核心是一个简单的“键值对”存储，但它之上构建了一套强大的版本控制逻辑。

从这个角度看，Git 的所有操作，如 `add`, `commit`, `branch` 等，都可以被理解为是对这个数据库中几张核心“表”进行的数据操作（CURD）。

下面，我们将 Git 的核心数据抽象为四张表：`objects` (对象表)，`refs` (引用表)，`index` (索引/暂存区表) 和 `HEAD` (头指针表)。

---

### 5.1 `objects` (对象表) - Git 的数据核心

这张表是 Git 数据库的基石，对应于 `.git/objects` 目录。它存储了项目的所有实际数据，包括文件内容、目录结构和提交历史。这是一个**内容寻址**的系统，意味着“键”就是“内容”本身的哈希值。

**表设计 (`objects`)**

| 列名 (Column) | 类型 (Type)  | 描述 (Description)                                                                              |
| :------------ | :----------- | :---------------------------------------------------------------------------------------------- |
| `hash`        | `string` (PK) | **主键**。由对象内容计算出的 SHA-1 哈希值。Git 通过它来索引和检索对象。                           |
| `type`        | `string`     | 对象类型，只能是 `blob`, `tree`, `commit`, `annotated_tag` 四种之一。                             |
| `size`        | `integer`    | 对象内容的字节大小。                                                                            |
| `content`     | `binary`     | 对象的原始数据，通常经过 zlib 压缩。这是 Git 存储的核心信息。                                   |

#### 表内容详解 (`type`):

- **`blob` (二进制大对象)**:
  - **用途**: 存储文件的**内容**。
  - **说明**: Git 不关心文件名，只关心文件内容。当你 `git add` một file时，Git 会计算文件内容的哈希值，并将文件内容作为一个 `blob` 对象存入此表。如果两个文件的内容完全相同（即使文件名不同），它们也会指向同一个 `blob` 对象。这是 Git 高效存储的基础。

- **`tree` (树对象)**:
  - **用途**: 存储**目录结构**。
  - **说明**: `tree` 对象的内容是一张列表，每一行包含一个文件或子目录的信息，包括：文件模式、对象类型 (`blob` 或 `tree`)、对应的 `hash` 值和文件名。它就像一个目录的快照，通过 `hash` 将文件名和文件内容 (`blob`) 关联起来。

- **`commit` (提交对象)**:
  - **用途**: 将一个**版本快照**串联起来，形成历史。
  - **说明**: `commit` 对象的内容包含了：
    - 指向项目顶层 `tree` 对象的 `hash` (即本次提交的版本快照)。
    - 指向父提交的 `hash` (parent commit)。普通提交有一个父提交，合并提交有两个或多个。这构成了 Git 的历史链。
    - 作者 (author) 和提交者 (committer) 的信息。
    - 提交信息 (commit message)。

- **`annotated_tag` (附注标签对象)**:
  - **用途**: 存储一个附注标签的元数据。
  - **说明**: 与 `commit` 类似，它包含标签创建者、日期、附注信息，并指向一个 `commit` 对象的 `hash`。

---

### 5.2 `refs` (引用表) - 人类可读的指针

`objects` 表中的 `hash` 是机器友好的，但人类无法记忆。`refs` 表的作用就是给这些 `hash` 起了个方便人类读写的名字，比如 `master` 分支或 `v1.0` 标签。它对应于 `.git/refs` 目录。

**表设计 (`refs`)**

| 列名 (Column) | 类型 (Type)  | 描述 (Description)                                                     |
| :------------ | :----------- | :--------------------------------------------------------------------- |
| `path`        | `string` (PK) | **主键**。引用的完整路径，如 `refs/heads/master`, `refs/tags/v1.0.0`。 |
| `hash`        | `string`     | 该引用指向的 `objects` 表中的 `hash` 值。对于分支，它指向一个 `commit` 对象；对于附注标签，它指向一个 `annotated_tag` 对象。 |

#### 表内容详解:

- **分支 (Branch)**: `git branch new-feature` 命令，相当于在此表中 `INSERT` 一条新记录：`path` 为 `refs/heads/new-feature`，`hash` 为当前 `commit` 的 `hash`。
- **标签 (Tag)**: `git tag v1.0` 命令，相当于在此表中 `INSERT` 一条新记录：`path` 为 `refs/tags/v1.0`，`hash` 为指定 `commit` 的 `hash`。

---

### 5.3 `HEAD` (头指针表) - “你在这里”的指示牌

这张“表”非常特殊，因为它永远只有一条记录。它指明了你当前的工作环境，即**当前分支**。它对应于 `.git/HEAD` 文件。

**表设计 (`HEAD`)**

| 列名 (Column) | 类型 (Type) | 描述 (Description)                                                                          |
| :------------ | :---------- | :------------------------------------------------------------------------------------------ |
| `mode`        | `string`    | 模式，`symbolic` (符号引用) 或 `detached` (分离头指针)。                                        |
| `points_to`   | `string`    | 如果 `mode` 是 `symbolic`，值是 `refs` 表中的一条 `path` (如 `refs/heads/master`)；如果是 `detached`，值是 `objects` 表中的一个 `commit` `hash`。 |

#### 表内容详解:

- 当你 `git switch master` 时，`HEAD` 表被 `UPDATE`：`mode` 设为 `symbolic`，`points_to` 设为 `refs/heads/master`。
- 当你 `git checkout <commit_hash>` 时，`HEAD` 表被 `UPDATE`：`mode` 设为 `detached`，`points_to` 设为该 `commit` 的 `hash`。

---

### 5.4 `index` (索引/暂存区表) - 下一次提交的“购物篮”

`index` 表是理解工作区、暂存区和仓库之间关系的关键。它是一个连接你的工作目录和 Git 仓库的中间层，负责构建下一次提交的内容。它对应于 `.git/index` 文件。

**表设计 (`index`)**

| 列名 (Column) | 类型 (Type)  | 描述 (Description)                                                        |
| :------------ | :----------- | :------------------------------------------------------------------------ |
| `path`        | `string` (PK) | **主键**。被追踪的文件的路径。                                            |
| `blob_hash`   | `string`     | 文件内容在 `objects` 表中对应的 `blob` `hash`。                           |
| `metadata`    | `json`       | 文件的元数据，如权限、创建和修改时间等，用于快速判断文件是否被修改。      |

#### 工作流示例:

1.  **`git add file.txt`**:
    1.  Git 读取 `file.txt` 的内容，计算其哈希值（假设为 `blob_hash_1`），并将其作为 `blob` 对象存入 `objects` 表。
    2.  Git 在 `index` 表中 `INSERT` 或 `UPDATE` 一条记录：`path` 为 `file.txt`，`blob_hash` 为 `blob_hash_1`。

2.  **`git commit -m "..."`**:
    1.  Git 首先锁定 `index` 表，根据表中的所有记录创建一个 `tree` 对象，并将其存入 `objects` 表。
    2.  然后，Git 创建一个 `commit` 对象，内容包括：
        - 指向上一步创建的 `tree` 对象的 `hash`。
        - 指向当前 `HEAD` 指向的 `commit` `hash` (作为父提交)。
        - 作者、提交者信息和提交信息。
    3.  这个新的 `commit` 对象被存入 `objects` 表。
    4.  最后，Git 读取 `HEAD` 表，找到当前分支的 `path` (如 `refs/heads/master`)，然后 `UPDATE` `refs` 表中对应记录的 `hash`，使其指向这个新的 `commit` 对象。

通过这个数据库模型，Git 的核心操作都变得清晰明了，它们不再是神秘的魔法，而是一系列严谨的、可预测的数据库事务。

---

### 5.5 复杂操作的数据库视角

理解了基础的表结构后，我们可以进一步用它来剖析一些更复杂的操作。

#### 远程仓库交互：`fetch`, `push`

远程仓库可以被看作是另一个独立的 “Git 数据库”。`fetch` 和 `push` 本质上是两个数据库之间的同步操作。

- **`git fetch origin`**:
  1. **连接远程数据库**: Git 通过 `origin` 的 URL 连接到远程服务器。
  2. **差异化 `SELECT`**: 比较本地 `objects` 表和远程 `objects` 表，找出所有在远程存在但在本地不存在的 `object`（包括 `commit`, `tree`, `blob` 等）。
  3. **`INSERT` 新对象**: 将所有差异 `object` 下载到本地，并 `INSERT` 进本地的 `objects` 表。
  4. **`UPDATE` 远程引用**: Git 下载远程的 `refs` 列表。对于远程的 `master` 分支（`refs/heads/master`），Git 会在本地的 `refs` 表中创建或更新一条记录，其 `path` 为 `refs/remotes/origin/master`，并将其 `hash` 指向 `fetch` 下来的最新 `commit`。
  - **核心**: `fetch` 是一个对本地数据库“只增不改”的安全操作。它只增加新对象和远程跟踪分支，绝不修改你本地的工作分支（`refs/heads/*`）或 `HEAD` 表。

- **`git push origin master`**:
  1. **连接远程数据库** 并锁定远程的 `master` 分支。
  2. **计算差异**: Git 从本地 `refs` 表中找到 `refs/heads/master` 指向的 `commit`。然后，它顺着提交历史向上追溯，找出所有远程数据库 `objects` 表中缺失的对象。
  3. **`INSERT` 缺失对象**: 将所有这些缺失的 `object` 上传并 `INSERT` 到远程数据库的 `objects` 表中。
  4. **`UPDATE` 远程引用**: 请求远程数据库 `UPDATE` 其 `refs` 表中 `path` 为 `refs/heads/master` 的记录，将 `hash` 更新为本地 `master` 分支最新的 `commit` `hash`。
  - **安全检查**: 如果远程分支的 `hash` 并非本地分支 `hash` 的直系祖先（即非 Fast-Forward），远程数据库会默认拒绝这次 `UPDATE` 操作，以防覆盖他人的提交。

#### `git rebase master` (当前在 `feature` 分支)

`rebase` 是一个“批量重写”操作，它创建全新的 `commit` 记录来形成线性历史。

1.  **`SELECT` 差异**: Git 找到 `feature` 分支和 `master` 分支的共同祖先 `commit`。然后 `SELECT` 出所有 `feature` 分支上在该祖先之后发生的 `commit` 对象列表。
2.  **暂存变更**: Git “回放”这个 `commit` 列表，将每个 `commit` 引入的变更（diff）暂存起来。
3.  **移动 `HEAD`**: 将 `HEAD` 指针临时移动到 `master` 分支的头部。
4.  **批量 `INSERT` 新 `commit`**: 遍历暂存的变更：
    a. 应用第一个变更，然后创建一个**新的 `commit` 对象**。这个新 `commit` 的父提交是 `master` 分支的头部。
    b. 应用第二个变更，然后创建另一个**新的 `commit` 对象**，其父提交是上一步创建的新 `commit`。
    c. ...依此类推，直到所有变更都作为新 `commit` 被重新应用。
5.  **`UPDATE` 分支引用**: 最后，Git `UPDATE` `refs` 表中 `refs/heads/feature` 的 `hash`，使其指向最后一个被创建的新 `commit`。
- **结果**: 原来的 `feature` 分支上的 `commit` 对象被“遗弃”（没有任何 `ref` 指向它们），最终会被 Git 的垃圾回收机制清理。`rebase` 的本质是用一批新的、拥有相同变更但不同 `hash` 的 `commit` 替换掉旧的 `commit`。

#### `git cherry-pick <commit_hash>`

`cherry-pick` 可以看作是单次、精准的 `rebase`。

1.  **`SELECT` 目标**: Git 从 `objects` 表中 `SELECT` 出 `<commit_hash>` 对应的 `commit` 对象及其父 `commit` 对象。
2.  **计算变更**: 通过比较这两个 `commit` 所指向的 `tree` 对象，Git 计算出该 `commit` 引入的精确变更（diff）。
3.  **应用变更**: Git 将这个变更应用到当前 `HEAD` 指向的工作目录和 `index` 表中。
4.  **`INSERT` 新 `commit`**: 执行一次标准的 `commit` 操作，在 `objects` 表中 `INSERT` 一个**新的 `commit` 对象**。这个新 `commit` 的内容（diff）与被 `cherry-pick` 的 `commit` 相同，但它的父提交是当前的 `HEAD`，因此它拥有一个全新的 `hash`。
5.  **`UPDATE` `HEAD`**: Git 更新当前分支的 `ref`（以及 `HEAD`），使其指向这个新创建的 `commit`。

通过这套数据库模型，即使是 `rebase` 和 `cherry-pick` 这种“修改历史”的命令，其底层原理也变得清晰：它们从不真正“修改” `objects` 表中的任何已有对象，而总是通过创建**新对象**，然后移动 `refs` 表中的指针来实现所谓的“修改”。这正是 Git 数据完整性的核心保障。
