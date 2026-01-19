---

## 如何在 Linux 上使用 Conda

Conda 是一个开源的包管理器和环境管理器，它能在任何操作系统上快速安装、运行和更新包及其依赖。在 Linux 上使用 Conda，能让你轻松地管理不同项目所需的 Python 版本和库，避免版本冲突。

### 1. 安装 Conda

首先，你需要从 Miniconda 或 Anaconda 的官方网站下载安装脚本。Miniconda 只包含 Conda 及其依赖，而 Anaconda 则预装了许多常用的科学计算库。对于大多数用户来说，**Miniconda** 更轻量级，是更好的选择。

1. 打开你的终端。
    
2. 使用 `wget` 命令下载最新的安装脚本。你可以去 [Miniconda 官网](https://docs.conda.io/en/latest/miniconda.html) 找到最新的下载链接。
    
    ``` Bash
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
    ```
    
3. 运行下载的脚本。按照提示操作，按 `Enter` 键阅读许可协议，然后输入 `yes` 同意并继续。
    
    ``` Bash
     Miniconda3-latest-Linux-x86_64.sh
    ```
    
4. 安装程序会询问你是否要初始化 Conda。**推荐输入 `yes`**，这样 Conda 会自动配置到你的 shell 配置文件中（如 `.rc` 或 `.zshrc`）。
    
5. 安装完成后，需要重启你的终端或运行以下命令来使配置生效：
    
    ``` ****
    source ~/.rc  # 如果你使用的是 
    # 或者
    source ~/.zshrc   # 如果你使用的是 zsh
    ```
    
6. 验证安装是否成功，运行：
    
    ``` Bash
    conda --version
    ```
    

### 2. Conda 常用命令

安装完成后，你就可以开始使用 Conda 了。以下是一些最常用的命令，能帮助你高效管理环境和包。

#### 环境管理

- **创建新环境**：创建一个名为 `my_env` 的新环境，并指定 Python 版本。
    
    ``` Bash
    conda create --name my_env python=3.10
    ```
    
- **激活环境**：激活你刚刚创建的环境。激活后，你的终端提示符会显示当前环境的名称。
    
    ``` Bash
    conda activate my_env
    ```
    
- **退出环境**：退出当前环境，返回到基础环境 (`base`)。
    
    ``` Bash
    conda deactivate
    ```
    
- **查看所有环境**：列出你创建的所有 Conda 环境。
    
    ``` Bash
    conda env list
    ```
    
- **克隆环境**：克隆一个已有的环境，创建一个新环境。
    
    ``` Bash
    conda create --name my_new_env --clone my_env
    ```
    
- **删除环境**：删除一个不再需要的环境。
    
    ``` Bash
    conda env remove --name my_env
    ```
    

#### 包管理

在激活了某个环境之后，你就可以在该环境中安装、更新或删除包。

- **安装包**：在当前激活的环境中安装 `numpy` 和 `pandas` 包。
    
    ``` Bash
    conda install numpy pandas
    ```
    
- **更新包**：更新当前环境中的某个包到最新版本。
    
    ``` Bash
    conda update numpy
    ```
    
- **删除包**：卸载当前环境中的某个包。
    
    ``` Bash
    conda remove numpy
    ```
    
- **查找包**：搜索 Conda 源中是否存在某个包。
    
    ``` Bash
    conda search numpy
    ```
    
- **查看已安装的包**：列出当前环境已安装的所有包。
    
    ``` Bash
    conda list
    ```
    

### 3. 使用 `environment.yml` 管理环境

为了方便团队协作和项目部署，推荐使用 `environment.yml` 文件来定义环境。这个文件记录了环境中所有的依赖包，方便你一键创建或复现环境。

- **导出环境**：将当前激活的环境及其所有依赖导出到 `environment.yml` 文件中。
    
    ``` Bash
    conda env export > environment.yml
    ```
    
- **从文件创建环境**：使用 `environment.yml` 文件创建一个新的环境。
    
    ``` Bash
    conda env create -f environment.yml
    ```
    

