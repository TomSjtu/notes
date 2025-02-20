# Python

## 虚拟环境

Python 可以创建虚拟环境以隔离不同项目的依赖关系，具体步骤如下：

1. 创建虚拟环境目录：`python -m venv myenv`
2. 激活虚拟环境：`source myenv/bin/activate`
3. 停用虚拟环境：`deactivate`

如果需要将虚拟环境中的包导出，可以在激活虚拟环境后使用`pip freeze > requirements.txt`导出依赖关系.

## uv

uv 是一个用于 Python 的虚拟环境管理工具，由 Rust 编写。

- 🚀 一个工具替代 pip, pip-tools, pipx, poetry, pyenv, twine, virtualenv 等等。
- ⚡️ 比 pip 快 10-100 倍。
- 🗂️ 提供全面的项目管理，使用通用的 lockfile。
- ❇️ 运行脚本，支持内联依赖元数据。
- 🐍 安装和管理 Python 版本。
- 🛠️ 运行和安装以 Python 包形式发布的工具。
- 🔩 包含兼容 pip 的接口，以便在使用熟悉的 CLI 的同时获得性能提升。
- 🏢 支持 Cargo 风格的工作区，适用于可扩展的项目。
- 💾 节省磁盘空间，采用用于依赖项去重的全局缓存。
- ⏬ 无需 Rust 或 Python 即可安装，可通过 curl 或 pip 安装。
- 🖥️ 支持 macOS、Linux 和 Windows。

### Python versions

uv 提供了 python 子命令用于管理 python 版本：

```SHELL
$ uv python --help
Manage Python versions and installations

Usage: uv python [OPTIONS] <COMMAND>

Commands:
  list       List the available Python installations
  install    Download and install Python versions
  find       Search for a Python installation
  pin        Pin to a specific Python version
  dir        Show the uv Python installation directory
  uninstall  Uninstall Python versions
```

### Scripts

uv 针对不同的运行环境，提供了各种命令：

- 没有依赖的 python 程序：`uv run example.py`
- 传递参数：`uv run example.py [arg]`
- 依赖某个包的 python 程序：`uv run --with [package] example.py`
- 创建某个 python 版本：`uv init --script example.py --python 3.12`
- 为 python 程序添加依赖：`uv add --script example.py 'requests<3' 'rich'`
- 锁定依赖：`uv lock --script example.py`
- 以指定版本运行 python 程序：`uv run --python 3.10 example.py`

### Projects

uv 将 python 程序的依赖信息保存在`pyproject.toml`文件中，使用`uv init`命令创建一个项目，它的结构如下：

```
.
├── .venv
│   ├── bin
│   ├── lib
│   └── pyvenv.cfg
├── .python-version
├── README.md
├── main.py
├── pyproject.toml
└── uv.lock
```

- pyproject.toml：项目的元数据。
- uv.lock：当前项目的 lockfile，包含了当前项目的依赖信息，该文件不能手动修改。

在项目中管理包可以使用以下命令：

- 添加依赖：`uv add [package]`
- 删除依赖：`uv remove [package]`
- 更新依赖：`uv lock --upgrade-package [package]`

`uv build`命令用于为项目生成源码发行版和二进制发行版。

### Tools

工具是指打包成 python 程序，并提供命令行界面的包。使用`uv tool install`命令安装工具，`uvx`命令使用工具。

### Pip interface

uv 提供了类似 pip 的接口，用来管理虚拟环境的包。

- 创建虚拟环境：`uv venv`，你可以指定名称和 python 版本。
- 安装包：`uv pip install [package]`，你可以使用`-r`选项指定 requirements.txt 文件。
- 卸载包：`uv pip uninstall [package]`.

### Utility

管理和查看 uv 的状态，例如缓存、存储目录或执行自我更新：

- 移除缓存条目：`uv cache clean`
- 清理过期缓存条目：`uv cache prune`
- 显示 uv 缓存目录路径：`uv cache dir`
- 显示 uv 工具目录路径：`uv tool dir`
- 显示 uv 安装 python 版本的路径：`uv python dir`
- 更新 uv 到最新版本：`uv self update`

### 完整使用示例

以下示例演示了如何使用 uv 完成一个 Python 项目的完整流程：

假设我们要创建一个简单的项目，该项目依赖 `requests` 和 `rich` 包，并包含一个用于展示功能的 `main.py` 文件。

1. 初始化项目  
   使用 uv 初始化项目，生成所需的项目结构：
   ```
   $ uv init --script main.py --python 3.9
   ```
   执行该命令后，项目将生成以下结构：
   ```
   .
   ├── .venv/
   ├── .python-version
   ├── README.md
   ├── main.py
   ├── pyproject.toml
   └── uv.lock
   ```

2. 添加项目依赖  
   向项目中添加 `requests` 和 `rich` 依赖：
   ```
   $ uv add --script main.py requests rich
   ```
   该命令会更新 `pyproject.toml` 文件，并生成或更新 `uv.lock` 文件，确保依赖版本被锁定。

3. 编写主脚本  
   在 `main.py` 中写入以下示例代码：
   ```python
   import requests
   from rich import print

   def main():
       response = requests.get('https://api.github.com')
       if response.status_code == 200:
           print("[bold green]成功访问 GitHub API![/bold green]")
       else:
           print("[bold red]访问失败[/bold red]")

   if __name__ == "__main__":
       main()
   ```

4. 运行脚本以验证项目  
   使用 uv 运行脚本：
   ```
   $ uv run --script main.py
   ```
   如果一切正常，终端将显示成功访问 GitHub API 的提示信息。

5. 锁定依赖版本  
   若项目依赖有更新需求，可使用以下命令更新依赖锁定信息：
   ```
   $ uv lock --script main.py
   ```

6. 打包项目  
   使用 uv 生成源码发行版和二进制发行版：
   ```
   $ uv build
   ```
   此命令会根据 `pyproject.toml` 中的配置生成可发布的项目包。
