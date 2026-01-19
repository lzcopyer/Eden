## windows 搭建Assembly环境
下载安装[vs code](https://code.visualstudio.com/)
### 1. **安装Assembly插件**
- MASM：用于进行代码语法提示；（作者：blindtiger）
- MASM/TASM：提供了DOSBox、Debug等工具；（作者：clcxsrolau）

### 2. **配置环境变量**
MSAM插件路径
`C:\Users\jasonkay\.vscode\extensions\xsro.masm-tasm-0.X.X`
**同时为了将来在DOSBox中方便映射，可以将masm目录拷贝一份到其他盘的根目录；**

### 3. **在DOSBox中使用Debug**
首先打开DOSBox；
在刚进入DOSBox时，我们是**没有挂载任何文件目录的（此时会提示我们在Z盘）**，这时是找不到debug的；
并且输入`c:`尝试跳转到C盘会出现提示：
``` cmd
Z:\> c: 
Drive C does not exist! 
You must mount it first. Type intro or intro mount for more information.
```
我们可以挂载MASM的目录到C盘，如下：
``` cmd
Z:\>mount c c:\Users\jasonkay\.vscode\extensions\xsro.masm-tasm-0.8.4\tools\masm Drive C is mounted as local directory 
c:\Users\jasonkay\.vscode\extensions\xsro.masm-tasm-0.8.4\tools\masm 
# 切换至C盘 
Z:\>c: 
# 使用Debug 
C:\>debug -r AX=0000 BX=0000 CX=0000 .... # 退出Debug -q
```

## Linux 搭建Assembly环境

### 1. **安装汇编器和工具链**
``` bash
apt install -y nasm binutils gcc gdb make
```

### 2. **编译与链接**
``` bash
#32位程序（兼容64位系统）
nasm -f elf32 hello.asm -o hello.o  # 生成目标文件
ld -m elf_i386 hello.o -o hello     # 链接为可执行文件

#64位程序
nasm -f elf64 hello.asm -o hello.o
ld hello.o -o hello

#运行程序
./hello
```

### 3. **调试程序**

#### 使用GDB调试
``` bash
nasm -f elf32 -g hello.asm -o hello.o  # 添加调试信息
ld -m elf_i386 hello.o -o hello
gdb ./hello
```

#### 常用GDB命令
``` bash
(gdb) break _start   # 在入口设置断点
(gdb) run            # 运行程序
(gdb) stepi          # 单步执行
(gdb) info registers # 查看寄存器
(gdb) x/s &msg       # 查看字符串内容
```

### 4. **高级配置**
使用Makefile自动化：

创建 `Makefile`：
``` makefile
ASM = nasm
LD = ld
ASM_FLAGS = -f elf32 -g
LD_FLAGS = -m elf_i386

all: hello

hello: hello.o
	$(LD) $(LD_FLAGS) $< -o $@

hello.o: hello.asm
	$(ASM) $(ASM_FLAGS) $< -o $@

clean:
	rm -f hello hello.o
```

运行：
``` bash
make    # 编译
make clean  # 清理
```