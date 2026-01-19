[仙人指路](https://docs.python.org/zh-cn/3.13/library/venv.html)

当在虚拟环境中使用时，常见安装工具如 [pip](https://pypi.org/project/pip/) 将把 Python 软件包安装到虚拟环境而无需显式地指明这一点。

虚拟环境是（主要的特性）：

- 用来包含支持一个项目（库或应用程序）所需的特定 Python 解释器、软件库和二进制文件。 它们在默认情况下与其他虚拟环境中的软件以及操作系统中安装的 Python 解释器和库保持隔离。
    
- 包含在一个目录中，根据惯例被命名为项目目录下的 `.venv` 或 `venv`，或是有许多虚拟环境的容器目录下，如 `~/.virtualenvs`。
    
- 不可签入 Git 等源代码控制系统。
    
- 被认为是可丢弃的 -- 它应当能被简单地删除并从头开始重建。 你不应在此环境中放置任何项目代码。
    
- 不被视为是可移动或可复制的 —— 你只能在目标位置重建相同的环境。

### 创建虚拟环境
[虚拟环境](https://docs.python.org/zh-cn/3.13/library/venv.html#venv-def) 是通过执行 `venv` 模块来创建的：
`python -m venv /path/to/new/virtual/environment`

加载虚拟环境：
`source _<venv>_/bin/activate`

查看当前的Python虚拟环境   
`echo $PYTHONPATH`  
`which python`

取消激活一个虚拟环境
`deactivate`