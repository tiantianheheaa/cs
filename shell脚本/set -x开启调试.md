在 Shell 脚本中，`set -x` 是一个非常有用的调试选项，它的作用是**启用命令跟踪（debug mode）**，让脚本在执行时**打印出每一行被执行的命令及其参数**，方便开发者查看脚本的实际执行流程。  

---

### **1. `set -x` 的作用**
- **打印执行的命令**：在脚本执行时，Shell 会输出每一行命令（前面加 `+` 符号），并显示其变量替换后的结果。  
- **帮助调试**：可以快速定位脚本中的错误、逻辑问题或变量展开异常。  
- **不影响脚本逻辑**：仅用于调试输出，不会改变脚本行为。  

---

### **2. 基本用法**
#### **(1) 全局启用（整个脚本生效）**
```bash
#!/bin/bash
set -x  # 开启调试模式

echo "Hello, World!"
name="Alice"
echo "Hello, $name"
```
**输出结果：**  
```
+ echo 'Hello, World!'
Hello, World!
+ name=Alice
+ echo 'Hello, Alice'
Hello, Alice
```
（每一行命令前都有 `+` 前缀）

---

#### **(2) 局部启用（仅部分代码生效）**
```bash
#!/bin/bash
echo "Before set -x"
set -x  # 开启调试
echo "This line is debugged"
set +x  # 关闭调试
echo "After set +x"  # 不会打印执行过程
```
**输出结果：**  
```
Before set -x
+ echo 'This line is debugged'
This line is debugged
+ set +x
After set +x
```
---

### **3. 实际调试示例**
#### **示例 1：查看变量替换**
```bash
#!/bin/bash
set -x

name="Bob"
file="/tmp/test.txt"
echo "Username: $name"
cat "$file"
```
**输出结果：**  
```
+ name=Bob
+ file=/tmp/test.txt
+ echo 'Username: Bob'
Username: Bob
+ cat /tmp/test.txt
cat: /tmp/test.txt: No such file or directory
```
👉 **帮助快速发现**：  
- `file` 变量的值是否正确  
- 如果文件不存在，`cat` 会报错  

---

#### **示例 2：调试 `if` 条件判断**
```bash
#!/bin/bash
set -x

count=5
if [ "$count" -gt 3 ]; then
    echo "Count is greater than 3"
else
    echo "Count is 3 or less"
fi
```
**输出结果：**  
```
+ count=5
+ [ 5 -gt 3 ]
+ echo 'Count is greater than 3'
Count is greater than 3
```
👉 **可清晰看到**：  
- `[ "$count" -gt 3 ]` 如何展开  
- `if` 条件是否正确执行  

---

### **4. 关闭调试模式**
```bash
set +x  # 停止打印命令
```
- `set -x` 开启调试  
- `set +x` 关闭调试  

---

### **5. 其他相关调试选项**
| 选项 | 作用 |
|------|------|
| `set -e` | 遇错即停（脚本出错时立即退出） |
| `set -u` | 检查未定义变量（用未赋值变量时报错） |
| `set -o pipefail` | 管道命令中任意一个失败就返回非0 |
| `set -v` | 打印原始命令（不展开变量） |

**组合使用（推荐）**  
```bash
#!/bin/bash
set -euo pipefail  # 严格模式：错误退出+检查未定义变量+管道失败检测
set -x              # 调试模式
```

---

### **6. 适用场景**
✅ **适合使用 `set -x` 的情况**：
- 脚本运行异常，但不知道哪一步出错  
- 变量展开不符合预期（如 `$PATH`、`$USER`）  
- 循环、条件判断逻辑复杂，需要逐步观察  

❌ **避免滥用**：
- 生产环境脚本（可能暴露敏感信息）  
- 日志文件过大（影响性能）  

---

### **7. 替代调试方法**
如果不想用 `set -x`，也可以用 `echo` 打印变量：
```bash
#!/bin/bash
name="Alice"
echo "DEBUG: name=$name" >&2  # 输出到 stderr
```
但 `set -x` 更方便，尤其是复杂脚本。

---

### **总结**
| 命令 | 作用 |
|------|------|
| `set -x` | 打印脚本执行的每一行命令（调试模式） |
| `set +x` | 关闭调试模式 |
| `set -e` | 遇错即停 |
| `set -u` | 检查未定义变量 |

**调试建议**：  
1. 在开发阶段启用 `set -x`  
2. 发现问题后，用 `set +x` 关闭  
3. 结合 `set -euo pipefail` 提高脚本健壮性  

这样可以快速定位问题，提高 Shell 脚本调试效率！ 🚀
