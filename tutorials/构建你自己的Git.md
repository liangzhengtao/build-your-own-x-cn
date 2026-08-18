# 构建你自己的 Git

> 从零开始，用 Python 构建一个支持基本 Git 操作的版本控制系统。

## 📋 项目简介

Git 是现代软件开发最重要的工具之一。通过从零构建一个 Git 兼容的版本控制系统，你将深入理解有向无环图（DAG）、SHA-1 哈希、差异计算、分支合并等核心概念。

本教程将引导你构建 `MiniGit`，支持：
- `init` 初始化仓库
- `add` 暂存文件
- `commit` 提交变更
- `log` 查看提交历史
- `diff` 查看差异
- `branch` 和 `checkout` 分支管理
- `merge` 合并分支

## 🛠️ 技术栈选择

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Python 3.10+ | 主要实现语言 |
| 哈希 | hashlib (SHA-1) | 对象寻址 |
| 压缩 | zlib | 对象压缩 |
| 差异 | difflib | 文件差异计算 |
| 存储 | 文件系统 | .minigit 目录 |

## 📖 核心概念解释

### 1. Git 对象模型

Git 的核心是一个内容寻址的文件系统，所有数据以四种对象存储：

```
┌─────────────────────────────────────────────────────────┐
│                    Git 对象类型                           │
├──────────┬──────────────────────────────────────────────┤
│ blob     │ 文件内容的快照                                │
│ tree     │ 目录结构（blob 和子 tree 的列表）              │
│ commit   │ 提交信息（指向 tree 和父 commit）              │
│ tag      │ 标签（指向 commit 的命名引用）                 │
└──────────┴──────────────────────────────────────────────┘

每个对象用 SHA-1 哈希标识：
  内容 → SHA-1("类型 大小\0内容") → 40 字符的哈希值
```

### 2. 对象存储结构

```
.git/
├── objects/
│   ├── ab/
│   │   └── cdef1234567890...    ← blob 对象（zlib 压缩）
│   ├── cd/
│   │   └── 567890abcdef12...    ← tree 对象
│   └── ef/
│       └── 90abcdef123456...    ← commit 对象
├── refs/
│   ├── heads/
│   │   └── main                 ← 分支（存储 commit 哈希）
│   └── tags/
│       └── v1.0                 ← 标签
└── HEAD                         ← 当前分支引用
```

### 3. 提交历史（DAG）

```
A ← B ← C ← D  (main)
         ↖
          E ← F  (feature)
```

### 4. 暂存区（Staging Area）

```
工作目录          暂存区 (Index)       仓库 (HEAD)
────────         ────────────         ──────────
file1.txt   ──add──→  file1.txt  ──commit──→ file1.txt
file2.txt        ↗    file2.txt          ↗
 (修改)     add后加入   (已暂存)      commit后存储
```

## 🚀 分步实现指南

### 第 1 步：对象存储

```python
# objects.py
import hashlib
import zlib
import os

class GitObject:
    """Git 对象基类"""

    def __init__(self, obj_type: str, content: bytes = b''):
        self.type = obj_type
        self.content = content

    @property
    def sha1(self) -> str:
        """计算对象的 SHA-1 哈希"""
        header = f"{self.type} {len(self.content)}\0".encode()
        return hashlib.sha1(header + self.content).hexdigest()

    def serialize(self) -> bytes:
        """序列化对象（用于存储）"""
        header = f"{self.type} {len(self.content)}\0".encode()
        return header + self.content

    @classmethod
    def deserialize(cls, data: bytes) -> 'GitObject':
        """从字节流反序列化对象"""
        null_pos = data.index(b'\0')
        header = data[:null_pos].decode()
        content = data[null_pos + 1:]

        obj_type = header.split(' ')[0]
        obj = cls(obj_type, content)
        return obj


class ObjectStore:
    """Git 对象存储"""

    def __init__(self, git_dir: str):
        self.objects_dir = os.path.join(git_dir, 'objects')

    def write(self, obj: GitObject) -> str:
        """写入对象到存储，返回 SHA-1"""
        sha1 = obj.sha1
        obj_dir = os.path.join(self.objects_dir, sha1[:2])
        obj_path = os.path.join(obj_dir, sha1[2:])

        if not os.path.exists(obj_path):
            os.makedirs(obj_dir, exist_ok=True)
            compressed = zlib.compress(obj.serialize())
            with open(obj_path, 'wb') as f:
                f.write(compressed)

        return sha1

    def read(self, sha1: str) -> GitObject:
        """从存储读取对象"""
        obj_path = os.path.join(self.objects_dir, sha1[:2], sha1[2:])

        if not os.path.exists(obj_path):
            raise FileNotFoundError(f"对象不存在: {sha1}")

        with open(obj_path, 'rb') as f:
            compressed = f.read()

        data = zlib.decompress(compressed)
        return GitObject.deserialize(data)

    def exists(self, sha1: str) -> bool:
        """检查对象是否存在"""
        obj_path = os.path.join(self.objects_dir, sha1[:2], sha1[2:])
        return os.path.exists(obj_path)


class Blob(GitObject):
    """文件内容对象"""

    def __init__(self, content: bytes = b''):
        super().__init__('blob', content)


class TreeEntry:
    """树条目"""
    def __init__(self, mode: str, name: str, sha1: str):
        self.mode = mode
        self.name = name
        self.sha1 = sha1


class Tree(GitObject):
    """目录树对象"""

    def __init__(self, entries: list = None):
        self.entries = entries or []
        content = self._serialize_entries()
        super().__init__('tree', content)

    def _serialize_entries(self) -> bytes:
        lines = []
        for entry in sorted(self.entries, key=lambda e: e.name):
            lines.append(f"{entry.mode} {entry.name}\0".encode() +
                        bytes.fromhex(entry.sha1))
        return b''.join(lines)

    @classmethod
    def from_content(cls, content: bytes) -> 'Tree':
        entries = []
        pos = 0
        while pos < len(content):
            # 解析 "mode name\0sha1(20bytes)"
            space_pos = content.index(b' ', pos)
            null_pos = content.index(b'\0', space_pos)

            mode = content[pos:space_pos].decode()
            name = content[space_pos + 1:null_pos].decode()
            sha1 = content[null_pos + 1:null_pos + 21].hex()

            entries.append(TreeEntry(mode, name, sha1))
            pos = null_pos + 21

        tree = cls(entries)
        tree.content = content
        return tree


class Commit(GitObject):
    """提交对象"""

    def __init__(self, tree_sha1: str, parent_sha1: str = None,
                 author: str = '', message: str = '', timestamp: str = ''):
        self.tree_sha1 = tree_sha1
        self.parent_sha1 = parent_sha1
        self.author = author
        self.message = message

        content = self._serialize()
        super().__init__('commit', content)

    def _serialize(self) -> bytes:
        lines = [f"tree {self.tree_sha1}"]
        if self.parent_sha1:
            lines.append(f"parent {self.parent_sha1}")
        lines.append(f"author {self.author}")
        lines.append("")
        lines.append(self.message)
        return '\n'.join(lines).encode()

    @classmethod
    def from_content(cls, content: bytes) -> 'Commit':
        text = content.decode()
        lines = text.split('\n')

        tree_sha1 = ''
        parent_sha1 = None
        author = ''

        i = 0
        while i < len(lines) and lines[i]:
            key, value = lines[i].split(' ', 1)
            if key == 'tree':
                tree_sha1 = value
            elif key == 'parent':
                parent_sha1 = value
            elif key == 'author':
                author = value
            i += 1

        message = '\n'.join(lines[i + 1:])

        commit = cls(tree_sha1, parent_sha1, author, message)
        commit.content = content
        return commit
```

### 第 2 �步：暂存区（Index）

```python
# index.py
import struct
import os

class IndexEntry:
    """暂存区条目"""
    def __init__(self, path: str, sha1: str, size: int, mtime: float):
        self.path = path
        self.sha1 = sha1
        self.size = size
        self.mtime = mtime


class Index:
    """暂存区（Index/Staging Area）"""

    SIGNATURE = b'DIRC'
    VERSION = 2

    def __init__(self, git_dir: str):
        self.index_path = os.path.join(git_dir, 'index')
        self.entries: dict[str, IndexEntry] = {}
        self.load()

    def load(self):
        """加载暂存区"""
        if not os.path.exists(self.index_path):
            return

        with open(self.index_path, 'rb') as f:
            data = f.read()

        if len(data) < 12:
            return

        # 解析头部
        signature = data[:4]
        if signature != self.SIGNATURE:
            return

        version = struct.unpack('>I', data[4:8])[0]
        num_entries = struct.unpack('>I', data[8:12])[0]

        # 简化版：只读取条目数
        self.entries = {}

    def save(self):
        """保存暂存区"""
        # 简化版：序列化条目列表
        header = struct.pack('>4sII', self.SIGNATURE, self.VERSION, len(self.entries))

        entries_data = b''
        for path, entry in sorted(self.entries.items()):
            path_bytes = path.encode()
            entry_data = struct.pack('>20sI', bytes.fromhex(entry.sha1), len(path_bytes))
            entry_data += path_bytes
            # 填充到 8 字节对齐
            padding = (8 - len(entry_data) % 8) % 8
            entry_data += b'\0' * padding
            entries_data += entry_data

        with open(self.index_path, 'wb') as f:
            f.write(header + entries_data)

    def add(self, path: str, sha1: str, size: int, mtime: float):
        """添加文件到暂存区"""
        self.entries[path] = IndexEntry(path, sha1, size, mtime)

    def remove(self, path: str):
        """从暂存区移除"""
        if path in self.entries:
            del self.entries[path]

    def get_entries(self) -> list:
        """获取所有暂存的条目"""
        return list(self.entries.values())
```

### 第 3 步：核心仓库操作

```python
# repository.py
import os
import time
from objects import ObjectStore, Blob, Tree, TreeEntry, Commit
from index import Index

class Repository:
    """Git 仓库"""

    def __init__(self, path: str = '.'):
        self.work_dir = os.path.abspath(path)
        self.git_dir = os.path.join(self.work_dir, '.minigit')
        self.objects = ObjectStore(self.git_dir)
        self.index = Index(self.git_dir)

    @classmethod
    def init(cls, path: str = '.') -> 'Repository':
        """初始化仓库"""
        git_dir = os.path.join(path, '.minigit')
        os.makedirs(os.path.join(git_dir, 'objects'), exist_ok=True)
        os.makedirs(os.path.join(git_dir, 'refs', 'heads'), exist_ok=True)
        os.makedirs(os.path.join(git_dir, 'refs', 'tags'), exist_ok=True)

        # 创建 HEAD 文件
        with open(os.path.join(git_dir, 'HEAD'), 'w') as f:
            f.write('ref: refs/heads/main\n')

        print(f"初始化空的 MiniGit 仓库于 {git_dir}")
        return cls(path)

    def get_head(self) -> str | None:
        """获取当前 HEAD 指向的 commit"""
        head_path = os.path.join(self.git_dir, 'HEAD')
        with open(head_path, 'r') as f:
            ref = f.read().strip()

        if ref.startswith('ref: '):
            ref_path = os.path.join(self.git_dir, ref[5:])
            if os.path.exists(ref_path):
                with open(ref_path, 'r') as f:
                    return f.read().strip()
            return None

        return ref

    def set_head(self, commit_sha1: str, ref: str = None):
        """更新 HEAD"""
        if ref:
            ref_path = os.path.join(self.git_dir, ref)
            os.makedirs(os.path.dirname(ref_path), exist_ok=True)
            with open(ref_path, 'w') as f:
                f.write(commit_sha1 + '\n')
        else:
            head_path = os.path.join(self.git_dir, 'HEAD')
            with open(head_path, 'r') as f:
                current_ref = f.read().strip()
            if current_ref.startswith('ref: '):
                ref_path = os.path.join(self.git_dir, current_ref[5:])
                with open(ref_path, 'w') as f:
                    f.write(commit_sha1 + '\n')

    def add(self, paths: list[str]):
        """将文件添加到暂存区"""
        for path in paths:
            full_path = os.path.join(self.work_dir, path)

            if not os.path.exists(full_path):
                raise FileNotFoundError(f"文件不存在: {path}")

            with open(full_path, 'rb') as f:
                content = f.read()

            # 创建 blob 对象
            blob = Blob(content)
            sha1 = self.objects.write(blob)

            # 更新暂存区
            stat = os.stat(full_path)
            self.index.add(path, sha1, stat.st_size, stat.st_mtime)
            print(f"添加: {path}")

        self.index.save()

    def commit(self, message: str, author: str = "User <user@example.com>"):
        """创建提交"""
        # 从暂存区构建 tree 对象
        tree_sha1 = self._build_tree()

        # 获取父提交
        parent_sha1 = self.get_head()

        # 创建提交对象
        timestamp = f"{int(time.time())} +0800"
        commit = Commit(tree_sha1, parent_sha1,
                       f"{author} {timestamp}", message)
        commit_sha1 = self.objects.write(commit)

        # 更新分支引用
        self.set_head(commit_sha1)

        print(f"[{'main' if not parent_sha1 else 'main'} "
              f"{commit_sha1[:7]}] {message}")

        return commit_sha1

    def _build_tree(self) -> str:
        """从暂存区构建 tree 对象"""
        entries = []
        for entry in self.index.get_entries():
            entries.append(TreeEntry(
                mode='100644',
                name=entry.path,
                sha1=entry.sha1
            ))

        tree = Tree(entries)
        return self.objects.write(tree)

    def log(self, max_count: int = 10):
        """查看提交历史"""
        sha1 = self.get_head()
        commits = []

        while sha1 and len(commits) < max_count:
            obj = self.objects.read(sha1)
            commit = Commit.from_content(obj.content)
            commits.append((sha1, commit))
            sha1 = commit.parent_sha1

        for sha1, commit in commits:
            print(f"commit {sha1}")
            print(f"Author: {commit.author}")
            print(f"\n    {commit.message}\n")

    def diff(self, path: str = None):
        """查看工作目录与暂存区的差异"""
        import difflib

        if path:
            paths = [path]
        else:
            paths = [e.path for e in self.index.get_entries()]

        for file_path in paths:
            full_path = os.path.join(self.work_dir, file_path)

            # 暂存区中的内容
            if file_path in self.index.entries:
                entry = self.index.entries[file_path]
                blob = self.objects.read(entry.sha1)
                staged_lines = blob.content.decode('utf-8', errors='replace').splitlines(keepends=True)
            else:
                staged_lines = []

            # 工作目录中的内容
            if os.path.exists(full_path):
                with open(full_path, 'r') as f:
                    working_lines = f.readlines()
            else:
                working_lines = []

            # 计算差异
            diff = difflib.unified_diff(
                staged_lines, working_lines,
                fromfile=f"a/{file_path}",
                tofile=f"b/{file_path}",
            )

            diff_output = ''.join(diff)
            if diff_output:
                print(diff_output)

    def branch(self, name: str = None):
        """创建或列出分支"""
        heads_dir = os.path.join(self.git_dir, 'refs', 'heads')

        if name:
            # 创建新分支
            current_sha1 = self.get_head()
            if not current_sha1:
                print("错误: 当前没有提交，无法创建分支")
                return

            branch_path = os.path.join(heads_dir, name)
            if os.path.exists(branch_path):
                print(f"错误: 分支 '{name}' 已存在")
                return

            with open(branch_path, 'w') as f:
                f.write(current_sha1 + '\n')
            print(f"创建分支: {name}")
        else:
            # 列出所有分支
            current_sha1 = self.get_head()
            for branch_name in sorted(os.listdir(heads_dir)):
                branch_path = os.path.join(heads_dir, branch_name)
                with open(branch_path, 'r') as f:
                    sha1 = f.read().strip()

                marker = '* ' if sha1 == current_sha1 else '  '
                print(f"{marker}{branch_name}")

    def checkout(self, branch_name: str):
        """切换分支"""
        branch_path = os.path.join(self.git_dir, 'refs', 'heads', branch_name)

        if not os.path.exists(branch_path):
            print(f"错误: 分支 '{branch_name}' 不存在")
            return

        with open(branch_path, 'r') as f:
            commit_sha1 = f.read().strip()

        # 更新 HEAD 指向新分支
        head_path = os.path.join(self.git_dir, 'HEAD')
        with open(head_path, 'w') as f:
            f.write(f'ref: refs/heads/{branch_name}\n')

        # 恢复文件（简化版）
        self._checkout_tree(commit_sha1)
        print(f"切换到分支 '{branch_name}'")

    def _checkout_tree(self, commit_sha1: str):
        """从提交恢复工作目录"""
        obj = self.objects.read(commit_sha1)
        commit = Commit.from_content(obj.content)

        tree_obj = self.objects.read(commit.tree_sha1)
        tree = Tree.from_content(tree_obj.content)

        for entry in tree.entries:
            if entry.mode.startswith('100'):  # 普通文件
                blob = self.objects.read(entry.sha1)
                file_path = os.path.join(self.work_dir, entry.name)
                os.makedirs(os.path.dirname(file_path), exist_ok=True)
                with open(file_path, 'wb') as f:
                    f.write(blob.content)

    def status(self):
        """查看仓库状态"""
        print("在分支 main\n")

        staged = list(self.index.entries.keys())
        if staged:
            print("暂存的变更:")
            for path in staged:
                print(f"  新文件: {path}")
        else:
            print("没有暂存的变更")

        # 检查未跟踪的文件
        tracked = set(self.index.entries.keys())
        untracked = []
        for root, dirs, files in os.walk(self.work_dir):
            # 跳过 .minigit 目录
            dirs[:] = [d for d in dirs if d != '.minigit']
            for f in files:
                full = os.path.relpath(os.path.join(root, f), self.work_dir)
                if full not in tracked:
                    untracked.append(full)

        if untracked:
            print("\n未跟踪的文件:")
            for path in untracked[:10]:
                print(f"  {path}")
```

### 第 4 步：命令行接口

```python
# minigit.py
"""MiniGit - 从零构建的版本控制系统"""
import sys
from repository import Repository

def main():
    if len(sys.argv) < 2:
        print("用法: minigit <command> [args...]")
        print("命令: init, add, commit, log, status, diff, branch, checkout")
        return

    command = sys.argv[1]

    if command == 'init':
        repo = Repository.init('.')

    elif command == 'add':
        repo = Repository('.')
        repo.add(sys.argv[2:])

    elif command == 'commit':
        repo = Repository('.')
        if '-m' in sys.argv:
            msg_idx = sys.argv.index('-m') + 1
            message = sys.argv[msg_idx]
        else:
            message = input("提交信息: ")
        repo.commit(message)

    elif command == 'log':
        repo = Repository('.')
        repo.log()

    elif command == 'status':
        repo = Repository('.')
        repo.status()

    elif command == 'diff':
        repo = Repository('.')
        path = sys.argv[2] if len(sys.argv) > 2 else None
        repo.diff(path)

    elif command == 'branch':
        repo = Repository('.')
        name = sys.argv[2] if len(sys.argv) > 2 else None
        repo.branch(name)

    elif command == 'checkout':
        repo = Repository('.')
        repo.checkout(sys.argv[2])

    else:
        print(f"未知命令: {command}")

if __name__ == '__main__':
    main()
```

## 🎯 进阶挑战

### 初级挑战
- [ ] 实现 `rm` 命令（从暂存区移除）
- [ ] 支持 `.gitignore` 忽略规则
- [ ] 添加彩色差异输出

### 中级挑战
- [ ] 实现三路合并算法
- [ ] 添加标签（tag）支持
- [ ] 实现 `stash` 暂存功能
- [ ] 支持远程仓库操作（clone/push/pull）

### 高级挑战
- [ ] 实现 packfile 压缩
- [ ] 添加 rebase 功能
- [ ] 实现 cherry-pick
- [ ] 支持 Git 协议兼容

## 📚 参考资源

- [Git 内部原理](https://git-scm.com/book/zh/v2/Git-内部原理-Git-对象) — Pro Git 书籍
- [Build Your Own Git](https://wyag.thb.lt/) — 用 Python 构建 Git
- [libgit2](https://libgit2.org/) — Git 的 C 实现
- [Git 源码](https://github.com/git/git) — Git 官方源码

---

**难度**: ⭐⭐⭐⭐ | **预计时长**: 8+ 小时 | **语言**: Python

> 💡 **提示**：先理解 Git 的对象模型（blob/tree/commit），再实现命令。用 `git cat-file -p <sha>` 来查看真实 Git 对象的格式。
