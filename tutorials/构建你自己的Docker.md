# 构建你自己的 Docker

> 从零开始，用 Python 实现容器的核心原理——namespace、cgroup 和文件系统隔离。

## 📋 项目简介

Docker 容器技术彻底改变了软件部署方式。通过从零构建一个容器运行时，你将深入理解 Linux namespace、cgroup、联合文件系统（UnionFS）等核心概念。

本教程将引导你构建 `MiniContainer`，支持：
- 进程隔离（PID namespace）
- 文件系统隔离（mount namespace）
- 网络隔离（network namespace）
- 资源限制（cgroup）
- 容器镜像构建
- 容器生命周期管理

## 🛠️ 技术栈选择

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Python 3.10+ / Go | 主语言 |
| 系统调用 | Linux namespaces | 进程隔离 |
| 资源限制 | cgroups v2 | CPU/内存限制 |
| 文件系统 | overlayfs | 分层文件系统 |
| 网络 | veth pair + bridge | 网络隔离 |
| 镜像 | tar 归档 | 简化版镜像格式 |

## 📖 核心概念解释

### 1. 容器 vs 虚拟机

```
虚拟机:
┌─────────────────────────────┐
│  应用 A  │  应用 B           │
│  ┌─────┐ │  ┌─────┐         │
│  │Guest│ │  │Guest│         │
│  │ OS  │ │  │ OS  │         │
│  └─────┘ │  └─────┘         │
│      Hypervisor (VMware)    │
│      Host OS (Linux)        │
│      Hardware               │
└─────────────────────────────┘

容器:
┌─────────────────────────────┐
│  应用 A  │  应用 B           │
│  ┌─────┐ │  ┌─────┐         │
│  │Bins │ │  │Bins │         │
│  └─────┘ │  └─────┘         │
│   Container Runtime         │
│   Host OS (Linux Kernel)    │
│   Hardware                  │
└─────────────────────────────┘
```

### 2. Linux Namespace

Namespace 是 Linux 内核提供的隔离机制：

```
┌──────────────────────────────────────────┐
│            Namespace 类型                 │
├──────────┬───────────────────────────────┤
│ PID      │ 进程 ID 隔离                  │
│ NET      │ 网络栈隔离                    │
│ MNT      │ 文件系统挂载点隔离             │
│ UTS      │ 主机名隔离                    │
│ IPC      │ 进程间通信隔离                 │
│ USER     │ 用户和组 ID 隔离              │
│ CGROUP   │ Cgroup 根目录隔离             │
│ TIME     │ 系统时间隔离                   │
└──────────┴───────────────────────────────┘
```

### 3. Cgroup（控制组）

```
/sys/fs/cgroup/
├── my-container/
│   ├── cgroup.procs        # 容器中的进程列表
│   ├── cpu.max             # CPU 限制 "100000 100000" = 1 核
│   ├── memory.max          # 内存限制 (bytes)
│   ├── memory.current      # 当前内存使用
│   └── io.max              # I/O 限制
```

### 4. 联合文件系统

```
┌─────────────────────┐
│  容器可写层          │  ← 容器运行时的修改
│  (Container Layer)   │
├─────────────────────┤
│  镜像层 3 (app.py)   │  ← 应用代码
├─────────────────────┤
│  镜像层 2 (pip deps) │  ← 依赖安装
├─────────────────────┤
│  镜像层 1 (Python)   │  ← 运行环境
├─────────────────────┤
│  基础层 (Ubuntu)     │  ← 操作系统
└─────────────────────┘
         ▲
    overlayfs 联合挂载
```

## 🚀 分步实现指南

### 第 1 步：基础容器（Python + ctypes）

```python
# container.py
import os
import sys
import ctypes
import ctypes.util

# ---- Linux 常量 ----
CLONE_NEWNS   = 0x00020000  # Mount namespace
CLONE_NEWUTS  = 0x04000000  # UTS namespace
CLONE_NEWIPC  = 0x08000000  # IPC namespace
CLONE_NEWUSER = 0x10000000  # User namespace
CLONE_NEWPID  = 0x20000000  # PID namespace
CLONE_NEWNET  = 0x40000000  # Network namespace

CLONE_NEWCGROUP = 0x02000000  # Cgroup namespace

libc = ctypes.CDLL(ctypes.util.find_library('c'), use_errno=True)

def unshare(flags):
    """调用 unshare 系统调用"""
    result = libc.unshare(flags)
    if result != 0:
        errno = ctypes.get_errno()
        raise OSError(errno, os.strerror(errno))


class Container:
    """MiniContainer 容器"""

    def __init__(self, name: str, image_path: str,
                 memory_limit: str = "256m",
                 cpu_limit: float = 1.0,
                 command: list = None):
        self.name = name
        self.image_path = image_path
        self.memory_limit = self._parse_memory(memory_limit)
        self.cpu_limit = cpu_limit
        self.command = command or ["/bin/sh"]
        self.rootfs = f"/tmp/container/{name}/rootfs"
        self.cgroup_path = f"/sys/fs/cgroup/{name}"

    def _parse_memory(self, mem_str: str) -> int:
        """解析内存限制字符串"""
        mem_str = mem_str.lower().strip()
        if mem_str.endswith('m'):
            return int(mem_str[:-1]) * 1024 * 1024
        elif mem_str.endswith('g'):
            return int(mem_str[:-1]) * 1024 * 1024 * 1024
        return int(mem_str)

    def run(self):
        """运行容器"""
        print(f"启动容器: {self.name}")

        # 1. 准备文件系统
        self._prepare_rootfs()

        # 2. 设置 cgroup
        self._setup_cgroup()

        # 3. 创建子进程
        pid = os.fork()

        if pid == 0:
            # 子进程：设置 namespace 隔离
            self._setup_namespaces()
            self._setup_rootfs()
            self._setup_network()

            # 执行用户命令
            os.execvp(self.command[0], self.command)
        else:
            # 父进程：等待子进程
            print(f"容器 PID: {pid}")
            _, status = os.waitpid(pid, 0)
            exit_code = os.WEXITSTATUS(status) if os.WIFEXITED(status) else 1
            print(f"容器退出，状态码: {exit_code}")

            # 清理 cgroup
            self._cleanup_cgroup()
            return exit_code

    def _prepare_rootfs(self):
        """准备根文件系统"""
        os.makedirs(self.rootfs, exist_ok=True)

        # 解压镜像到 rootfs
        import tarfile
        if os.path.isfile(self.image_path):
            with tarfile.open(self.image_path, 'r:*') as tar:
                tar.extractall(self.rootfs)
            print(f"镜像已解压到 {self.rootfs}")

    def _setup_cgroup(self):
        """设置 cgroup 资源限制"""
        os.makedirs(self.cgroup_path, exist_ok=True)

        # 设置内存限制
        with open(os.path.join(self.cgroup_path, 'memory.max'), 'w') as f:
            f.write(str(self.memory_limit))

        # 设置 CPU 限制
        # cpu.max = "$QUOTA $PERIOD"，例如 "100000 100000" = 1 核
        cpu_quota = int(self.cpu_limit * 100000)
        with open(os.path.join(self.cgroup_path, 'cpu.max'), 'w') as f:
            f.write(f"{cpu_quota} 100000")

        print(f"cgroup: 内存限制={self.memory_limit // (1024*1024)}MB, "
              f"CPU={self.cpu_limit}核")

    def _cleanup_cgroup(self):
        """清理 cgroup"""
        try:
            os.rmdir(self.cgroup_path)
        except:
            pass

    def _setup_namespaces(self):
        """设置 namespace 隔离"""
        # 创建新的 PID、Mount、UTS、IPC、NET namespace
        flags = CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWNET
        unshare(flags)

    def _setup_rootfs(self):
        """设置容器根文件系统"""
        # 将当前进程的根文件系统切换到容器的 rootfs
        os.chroot(self.rootfs)
        os.chdir('/')

    def _setup_network(self):
        """设置容器网络（简化版）"""
        # 在实际实现中，这里需要：
        # 1. 在宿主机创建 veth pair
        # 2. 将一端放入容器的 network namespace
        # 3. 配置 IP 地址和路由
        pass
```

### 第 2 步：Dockerfile 解析与镜像构建

```python
# builder/dockerfile.py
import os
import shutil
import subprocess

class DockerfileParser:
    """Dockerfile 解析器"""

    def __init__(self, dockerfile_path: str):
        self.dockerfile_path = dockerfile_path
        self.instructions = []

    def parse(self) -> list:
        """解析 Dockerfile"""
        with open(self.dockerfile_path, 'r') as f:
            lines = f.readlines()

        current_instruction = None
        for line in lines:
            line = line.strip()

            # 跳过注释和空行
            if not line or line.startswith('#'):
                continue

            # 处理行续接符
            if line.endswith('\\'):
                if current_instruction is None:
                    current_instruction = line[:-1]
                else:
                    current_instruction += line[:-1]
                continue

            if current_instruction:
                line = current_instruction + ' ' + line
                current_instruction = None

            # 解析指令
            parts = line.split(' ', 1)
            instruction = parts[0].upper()
            args = parts[1] if len(parts) > 1 else ''
            self.instructions.append((instruction, args))

        return self.instructions


class ImageBuilder:
    """镜像构建器"""

    def __init__(self, tag: str):
        self.tag = tag
        self.layers = []  # 存储各层的路径
        self.build_dir = f"/tmp/build/{tag}"
        self.env = {}
        self.workdir = '/'
        self.cmd = []

    def build(self, dockerfile_path: str) -> str:
        """构建镜像"""
        parser = DockerfileParser(dockerfile_path)
        instructions = parser.parse()

        os.makedirs(self.build_dir, exist_ok=True)
        layer_dir = os.path.join(self.build_dir, 'layers')
        current_root = os.path.join(layer_dir, '0-base')
        os.makedirs(current_root, exist_ok=True)

        for i, (instruction, args) in enumerate(instructions):
            print(f"步骤 {i+1}/{len(instructions)}: {instruction} {args[:50]}...")

            layer_path = os.path.join(layer_dir, f'{i+1}-{instruction.lower()}')
            os.makedirs(layer_path, exist_ok=True)

            if instruction == 'FROM':
                self._handle_from(args, current_root)
            elif instruction == 'RUN':
                self._handle_run(args, current_root, layer_path)
            elif instruction == 'COPY':
                self._handle_copy(args, current_root, layer_path)
            elif instruction == 'ENV':
                self._handle_env(args)
            elif instruction == 'WORKDIR':
                self._handle_workdir(args)
            elif instruction == 'CMD':
                self._handle_cmd(args)
            elif instruction == 'EXPOSE':
                pass  # 记录但不执行

            self.layers.append(layer_path)

        # 打包为镜像 tar
        image_path = f"/tmp/images/{self.tag}.tar"
        os.makedirs(os.path.dirname(image_path), exist_ok=True)

        import tarfile
        with tarfile.open(image_path, 'w') as tar:
            tar.add(layer_dir, arcname='layers')

        print(f"\n镜像构建完成: {image_path}")
        print(f"镜像大小: {os.path.getsize(image_path) / 1024:.1f} KB")

        return image_path

    def _handle_from(self, image: str, root_dir: str):
        """处理 FROM 指令"""
        # 简化版：从本地 tar 文件加载基础镜像
        base_image = f"/tmp/images/{image}.tar"
        if os.path.exists(base_image):
            import tarfile
            with tarfile.open(base_image, 'r:*') as tar:
                tar.extractall(root_dir)
        else:
            # 创建最小文件系统
            for d in ['bin', 'etc', 'lib', 'lib64', 'usr', 'tmp', 'proc', 'sys', 'dev']:
                os.makedirs(os.path.join(root_dir, d), exist_ok=True)

    def _handle_run(self, command: str, root_dir: str, layer_dir: str):
        """处理 RUN 指令"""
        # 在容器环境中执行命令
        result = subprocess.run(
            ['chroot', root_dir, '/bin/sh', '-c', command],
            capture_output=True, text=True
        )
        if result.returncode != 0:
            print(f"警告: 命令执行失败: {result.stderr}")

    def _handle_copy(self, args: str, root_dir: str, layer_dir: str):
        """处理 COPY 指令"""
        parts = args.split()
        src, dst = parts[0], parts[1]
        dst_path = os.path.join(root_dir, dst.lstrip('/'))
        os.makedirs(os.path.dirname(dst_path), exist_ok=True)

        if os.path.isdir(src):
            shutil.copytree(src, dst_path, dirs_exist_ok=True)
        else:
            shutil.copy2(src, dst_path)

    def _handle_env(self, args: str):
        """处理 ENV 指令"""
        key, value = args.split(' ', 1)
        self.env[key] = value

    def _handle_workdir(self, args: str):
        """处理 WORKDIR 指令"""
        self.workdir = args

    def _handle_cmd(self, args: str):
        """处理 CMD 指令"""
        import json
        try:
            self.cmd = json.loads(args)
        except:
            self.cmd = ['/bin/sh', '-c', args]
```

### 第 3 步：容器管理器

```python
# manager.py
import os
import json
from container import Container

class ContainerManager:
    """容器管理器"""

    def __init__(self, state_dir: str = '/tmp/minicontainer'):
        self.state_dir = state_dir
        os.makedirs(state_dir, exist_ok=True)

    def run(self, image: str, name: str = None, command: list = None,
            memory: str = '256m', cpu: float = 1.0) -> int:
        """运行容器"""
        if not name:
            name = f"container-{os.urandom(4).hex()}"

        image_path = f"/tmp/images/{image}.tar"

        container = Container(
            name=name,
            image_path=image_path,
            memory_limit=memory,
            cpu_limit=cpu,
            command=command or ['/bin/sh'],
        )

        # 保存容器状态
        self._save_state(name, {
            'image': image,
            'status': 'running',
            'command': command,
        })

        exit_code = container.run()

        self._save_state(name, {
            'image': image,
            'status': 'exited',
            'exit_code': exit_code,
        })

        return exit_code

    def list_containers(self, all: bool = False):
        """列出容器"""
        containers = []
        for name in os.listdir(self.state_dir):
            state = self._load_state(name)
            if all or state.get('status') == 'running':
                containers.append((name, state))

        return containers

    def _save_state(self, name: str, state: dict):
        path = os.path.join(self.state_dir, name, 'state.json')
        os.makedirs(os.path.dirname(path), exist_ok=True)
        with open(path, 'w') as f:
            json.dump(state, f)

    def _load_state(self, name: str) -> dict:
        path = os.path.join(self.state_dir, name, 'state.json')
        if os.path.exists(path):
            with open(path, 'r') as f:
                return json.load(f)
        return {}
```

### 第 4 步：命令行接口

```python
# minicontainer.py
"""MiniContainer - 从零构建的容器运行时"""
import argparse
import sys
from manager import ContainerManager

def main():
    parser = argparse.ArgumentParser(description='MiniContainer - 轻量级容器运行时')
    subparsers = parser.add_subparsers(dest='command')

    # run 命令
    run_parser = subparsers.add_parser('run', help='运行容器')
    run_parser.add_argument('image', help='镜像名称')
    run_parser.add_argument('command', nargs='*', default=['/bin/sh'])
    run_parser.add_argument('--name', help='容器名称')
    run_parser.add_argument('-m', '--memory', default='256m', help='内存限制')
    run_parser.add_argument('--cpu', type=float, default=1.0, help='CPU 限制')

    # ps 命令
    ps_parser = subparsers.add_parser('ps', help='列出容器')
    ps_parser.add_argument('-a', '--all', action='store_true')

    # build 命令
    build_parser = subparsers.add_parser('build', help='构建镜像')
    build_parser.add_argument('-t', '--tag', required=True, help='镜像标签')
    build_parser.add_argument('-f', '--file', default='Dockerfile')

    args = parser.parse_args()
    manager = ContainerManager()

    if args.command == 'run':
        exit_code = manager.run(
            image=args.image,
            name=args.name,
            command=args.command,
            memory=args.memory,
            cpu=args.cpu,
        )
        sys.exit(exit_code)

    elif args.command == 'ps':
        containers = manager.list_containers(all=args.all)
        print(f"{'CONTAINER ID':<15} {'IMAGE':<15} {'STATUS':<10}")
        for name, state in containers:
            print(f"{name[:12]:<15} {state.get('image','?'):<15} {state.get('status','?'):<10}")

    elif args.command == 'build':
        from builder.dockerfile import ImageBuilder
        builder = ImageBuilder(args.tag)
        builder.build(args.file)

    else:
        parser.print_help()

if __name__ == '__main__':
    main()
```

## 🎯 进阶挑战

### 初级挑战
- [ ] 实现 `docker exec` 在运行的容器中执行命令
- [ ] 添加容器日志收集
- [ ] 实现 `docker stop` 和 `docker rm`

### 中级挑战
- [ ] 实现 overlayfs 多层镜像
- [ ] 添加 veth 网络配置（容器间通信）
- [ ] 实现端口映射（-p 参数）
- [ ] 支持 Docker Registry 镜像拉取

### 高级挑战
- [ ] 实现 seccomp 系统调用过滤
- [ ] 添加 AppArmor/SELinux 安全策略
- [ ] 实现容器编排（简单版 Docker Compose）
- [ ] 支持 rootless 容器

## 📚 参考资源

- [Docker 官方文档](https://docs.docker.com/) — Docker 使用指南
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec) — 容器运行时标准
- [runc 源码](https://github.com/opencontainers/runc) — OCI 运行时参考实现
- [Linux man pages: namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Linux man pages: cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html)
- 《自己动手写 Docker》— 中文书籍

---

**难度**: ⭐⭐⭐⭐⭐ | **预计时长**: 15+ 小时 | **语言**: Python / Go

> 💡 **提示**：容器的核心就是 namespace + cgroup + rootfs。先从最简单的 "chroot + fork" 开始，逐步添加隔离功能。需要 Linux 环境和 root 权限。
