>目前，没有一个开箱即用的、被广泛称为“AI Agent”的工具能**直接**在标准的 `git commit` **之后**自动完成这个复杂的闭环（发送、优化、返回结果）。本方案是使用 `post-commit` hook 和一个 Python 脚本实现这个需求。


# 原理概述

这是最直接的实现方式，利用 Git 内置的自动化机制：

- **Git Hooks：** 可以使用 `post-commit` 或 `post-receive` 这样的 Git Hook。
    
    - **`post-commit` (本地)：** 在本地提交完成后触发。它会执行一个脚本，可以让这个脚本：
        
        1. 获取本次提交的 **diff** (`git diff HEAD^ HEAD`)。
            
        2. 将这个 diff **发送**给一个 AI 服务的 API（例如 OpenAI API、Claude API、GitHub Copilot Chat API 等）。
            
        3. 接收 AI 的**响应**。
            
        4. 将优化建议**打印**到终端或**保存**为新文件。
            
- **AI Service API：** 你需要编写脚本来调用这些 AI 模型提供的 API 接口，并清晰地指示模型进行“代码优化”、“代码审查”或“性能改进”。

# 准备工作

- **Python 环境：** 确保系统安装了 Python 3。
    
- **Git 仓库：** 确保在 Git 仓库的根目录下操作。
    
- **AI API Key：** 需要一个 AI 服务（如 OpenAI）的 API Key。为了安全，请使用环境变量存储它。

# 部署本地 Git Hook

## 1.在`git`仓库创建 `Hook` 文件

``` Bash
touch .git/hooks/post-commit
chmod +x .git/hooks/post-commit
```

## 2.配置`base_url`和`api key`

``` Bash
# 根据使用的AI模型进行修改
export OPENAI_BASE_URL="https://dashscope.aliyuncs.com/compatible-mode/v1"
export OPENAI_API_KEY="sk-6142a3b523ed419e95a6f89525d4ea72"
```

## 3.编写`Hook`文件

``` Bash
#!/bin/bash

# 定义 Python 脚本的路径 (你可以把它放在仓库的任何地方，这里假设在根目录)
PYTHON_SCRIPT="./ai_optimiser.py"

# 检查 Python 脚本是否存在
if [ ! -f "$PYTHON_SCRIPT" ]; then
    echo "Error: Python script '$PYTHON_SCRIPT' not found."
    exit 1
fi

echo "--- 🤖 正在将提交的代码发送给 AI 进行审查和优化..."

# 执行 Python 脚本
# 注意: Git Hooks 默认在仓库根目录执行，所以路径是相对的
python3 "$PYTHON_SCRIPT"

echo "--- 🤖 AI 审查和优化流程完成。"
```

## 4.编写 `Python` 脚本 (`ai_optimiser.py`)

``` Python
import os
import subprocess
import json
from openai import OpenAI
from pathlib import Path

# --- 配置 ---
try:
    client = OpenAI()
except Exception as e:
    print(f"Error: 无法初始化 OpenAI 客户端。请确保设置了 OPENAI_API_KEY 环境变量。详细错误: {e}")
    exit(1)

MODEL_NAME = "gpt-4-turbo" # 使用强大的模型以获得高质量的优化
OUTPUT_SUFFIX = ".ai_optimized" # 副本文件的后缀

# --- 核心 Prompt 模板 ---
OPTIMIZATION_PROMPT_TEMPLATE = """
你是一名专业的软件工程师和代码优化 AI。
我刚刚完成了一次 Git 提交。以下是这次提交所修改的代码差异 (diff)。

你的任务是：
1. **审查**这段代码差异。
2. **识别**潜在的优化点（性能、安全、可读性）。
3. **重要：** 你必须以 **JSON 格式**返回你的优化结果。
4. **JSON 结构要求：**
   - 必须包含一个顶级键 "optimized_files"，其值是一个列表。
   - 列表中的每个对象代表一个被修改的文件，包含两个键：
     - "filename": 文件路径和名称（必须是本次提交中修改的文件）。
     - "optimized_code": **该文件**优化后的**完整代码内容**（不要只返回修改部分，必须是完整文件）。
   - 必须包含一个顶级键 "review_summary"，其值是本次审查的简短文字总结。

如果代码质量很高，没有明显的优化，请在 "review_summary" 中说明，并让 "optimized_files" 为空列表 []。

--- 提交的 Diff ---
{diff_content}
"""

# --- 辅助函数 ---

def get_last_commit_diff():
    """获取上一次提交的差异内容 (HEAD^ 到 HEAD)"""
    try:
        # 使用 git diff HEAD^ HEAD 获取本次 commit 的所有文件修改
        result = subprocess.run(
            ['git', 'diff', 'HEAD^', 'HEAD'],
            capture_output=True,
            text=True,
            check=True,
            encoding='utf-8'
        )
        return result.stdout
    except subprocess.CalledProcessError as e:
        # 第一次提交时 HEAD^ 可能不存在，尝试使用 git show
        try:
            result = subprocess.run(
                ['git', 'show', 'HEAD'],
                capture_output=True,
                text=True,
                check=True,
                encoding='utf-8'
            )
            return result.stdout
        except:
             return None

def send_to_ai_for_review(diff_content):
    """调用 OpenAI API 发送 Diff 并获取优化代码"""
    print(f"正在发送 {len(diff_content)} 个字符的 Diff 到 {MODEL_NAME}...")

    full_prompt = OPTIMIZATION_PROMPT_TEMPLATE.format(diff_content=diff_content)

    try:
        response = client.chat.completions.create(
            model=MODEL_NAME,
            messages=[
                {"role": "system", "content": "你是一个专业的代码优化 AI 机器人，必须以标准的 JSON 格式返回结果。"},
                {"role": "user", "content": full_prompt}
            ],
            temperature=0.1,
            # 启用 JSON 模式以确保返回标准的 JSON
            response_format={"type": "json_object"}
        )
        return response.choices[0].message.content
    except Exception as e:
        print(f"\n--- AI API 调用失败 ---\n错误信息: {e}")
        return None

def save_optimized_files(optimized_data):
    """解析 AI 响应并保存优化后的代码到副本文件"""
    if not optimized_data:
        print("AI 没有返回有效的优化数据。")
        return

    try:
        # 尝试解析 AI 返回的 JSON 字符串
        data = json.loads(optimized_data)
        
        summary = data.get('review_summary', '无总结。')
        optimized_files = data.get('optimized_files', [])
        
        print("\n" + "="*50)
        print("✨ AI 代码优化建议和审查报告 ✨")
        print(f"总结: {summary}")
        print("="*50)

        if not optimized_files:
            print("AI 审查后认为代码质量很高，没有需要优化的文件。")
            return

        for file_info in optimized_files:
            filename = file_info.get('filename')
            optimized_code = file_info.get('optimized_code')

            if filename and optimized_code is not None:
                # 检查文件路径是否在当前工作目录内
                file_path = Path(filename)
                if not file_path.is_file():
                    print(f"Warning: 文件 '{filename}' 不存在或路径错误，跳过保存。")
                    continue
                    
                # 创建副本文件名
                output_filename = filename + OUTPUT_SUFFIX
                
                try:
                    with open(output_filename, 'w', encoding='utf-8') as f:
                        f.write(optimized_code)
                    
                    print(f"✅ 文件已优化并保存为副本: {output_filename}")
                    
                except Exception as e:
                    print(f"❌ 写入文件副本失败: {output_filename}。错误: {e}")

    except json.JSONDecodeError:
        print("\n--- AI 返回数据解析失败 ---")
        print("AI 返回的原始数据不是有效的 JSON 格式。请检查 Prompt。")
        print(f"原始响应（前500字）:\n{optimized_data[:500]}...")
    except Exception as e:
        print(f"\n--- 脚本执行错误 ---\n错误: {e}")


def main():
    """主执行函数"""
    diff = get_last_commit_diff()

    if not diff or "diff --git" not in diff:
        print("没有可分析的 Diff 内容，跳过 AI 审查。")
        return

    # 1. 发送 Diff 到 AI
    ai_response = send_to_ai_for_review(diff)

    # 2. 解析并保存优化后的文件
    if ai_response:
        save_optimized_files(ai_response)
        
    print("\n--- 🤖 AI 审查和优化流程完成。---")


if __name__ == "__main__":
    # 在 Git Hook 环境中执行时，确保工作目录是仓库根目录
    main()
```

## 5.使用&测试

在 `git commit` 命令执行成功后，`post-commit` hook 将自动运行。并将`AI`修改过后的代码保存在副本中。

---
诚然这是一个低级且抽象的需求，它违背了第一性原理。从出发点（提升`coding`效率和质量）来说这并不是一个很好的解决方式，在我看来更好的解决方式是使用`copliot`和`claude code`等专业工具，这些工具是交互式的帮助程序员分析需求并提出建议，还可以构建高质量的代码结构。但是这个需求却无意间为我打开了一扇通向`AIagent`的大门。