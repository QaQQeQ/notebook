
---

## 一、Shell 基础语法

### 1. Shell 简介
Shell 是 Linux 的命令解释器，是用户与内核之间的桥梁，负载接受解释用户命令，转给内核执行，然后向用户返回执行结果。  
常见的 Shell 有：`bash`、`sh`  等。  
脚本文件通常以：
```bash
#!/bin/bash
```
开头。

---

### 2. 变量与函数

#### （1）定义变量

| 类型                   | 说明                                   | 直接赋值法<br>= 赋值符号       | 预先声明法<br>declare 命令                                          | 预先声明法<br>local 命令                                        |
| -------------------- | ------------------------------------ | --------------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| **SHELL 进程中的〈环境变量〉** | ♦ 注意：可以被子进程继承                        | `export var01=xxx`    | `declare -x var01=xxx`                                       | ——                                                       |
| **SHELL 进程中的〈本地变量〉** | ♦ 注意：不可以被子进程继承                       | `var01=xxx`           | `declare -- var01=xxx`<br>或<br>`declare var01=xxx`           | ——                                                       |
| **SHELL 函数中的〈全局变量〉** | ♦ 注意：可以用于函数体之外<br>♦ 注意：SHELL 函数不是子进程 | 在函数体中：<br>`var01=xxx` | 在函数体中：<br>`declare -g var01=xxx`                             | 在函数体中：<br>`local -g var01=xxx`                           |
| **SHELL 函数中的〈局部变量〉** | ♦ 注意：仅用于函数体内<br>♦ 注意：SHELL 函数不是子进程   | ——                    | 在函数体中：<br>`declare -- var01=xxx`<br>或<br>`declare var01=xxx` | 在函数体中：<br>`local -- var01=xxx`<br>或<br>`local var01=xxx` |


#### （2）引用变量
```bash
${变量名称}
$0  # 脚本名
$1 $2 ... # 在函数体内部接受<SHELL 位置参数>的传参
$#  # 参数个数
$@  # 所有参数
$?  # 上个命令返回值
$$  # 当前进程PID
```

#### （3）函数定义示例
```bash
function hello() {
  echo "Hello, $1!"
}
hello World
```

---

### 3. 脚本执行方式
```bash
bash 1.sh     # 使用 bash 执行
./1.sh        # 需要可执行权限
source 1.sh   # 在当前 shell 执行
```

---

### 4. 引号与 echo 命令

| 类型                  | 用法    | 示例                   |
| ------------------- | ----- | -------------------- |
| 单引号 `' '`           | 不解析变量 | `'My name is $USER'` |
| 双引号 `" "`           | 可解析变量 | `"My name is $USER"` |
| 反引号 `` ` ``, 或 $( ) | 命令替换  | `echo $(date)`       |

```bash
echo "当前时间: $(date)"
```

---

### 5. Expansion命令行展开

#### （1）波浪号展开
```bash
cd ~
```

#### （2）花括号展开
```bash
echo file{1..3}.txt
```

#### （3）参数和变量展开
```bash
echo "$USER"
```

#### （4）算术展开
```bash
echo $((1+2*3))
```

#### （5）命令替换
```bash
echo "今天是$(date)"
```

#### （6）分词
```bash
var="a b";for i in $var   ## 循环：两遍
```

#### （7）路径名展开
```bash
ls /usr/bin/*.sh
```

#### （8）进程替换
```bash
## 注意：bash 支持 <(commands)进程替换；sh不支持
##      bash 和 sh 均支持 >(commands)进程替换
diff <(ls dir1) <(ls dir2)  ##比较：两个目录的文件列表
```

---

## 二、流程控制语法

### 1. if 条件判断
```bash
if [ $a -gt $b ]; then
  echo "a > b"
elif [ $a -eq $b ]; then
  echo "a = b"
else
  echo "a < b"
fi
```

### 2. case 语句
```bash
read -p "输入一个字母: " char
case $char in
  [a-z]) 
	  echo "小写字母"
  ;;
  [A-Z]) 
	  echo "大写字母"
  ;;
  *) 
	  echo "其他字符"
  ;;
esac
```

### 3. for 循环
```bash
## 算数条件判断循环
for ((i=1;i<10;i++)); do
  echo "第 $i 次循环"
done

## 遍历循环
for i in {1..5}; do
  echo "第 $i 次循环"
done
```

### 4. while 循环
```bash
count=1
while [ $count -le 5 ]; do
  echo "循环 $count"
  ((count++))
done
```

### 5. while read 循环
**注意：本质上是while循环语句+read命令**
```bash
export x="123"
echo "ni hao" | while read a; do
	x=$((x+1000))
	echo "子进程中的x=$x"
done
echo "父进程中的x=$x"
```

### 6. until 循环
```bash
count=1
until [ $count -gt 5 ]; do
  echo "until 循环 $count"
  ((count++))
done
```

---

## 三、表达式与运算

### 1. 算术计算
```bash
a=5; b=3
echo $((a + b))
let sum=a+b
##安装bc外部命令之后
echo "(1+2)*(3+4)" | bc  ## 计算结果 = 21
echo "scale=2;10/3" | bc  ## 计算结果 = 3.33
```

### 2. 条件表达式
- 操作符
  `用于：test/[ expression ]、[[ expression ]] 字符条件表达式`
  `-eq   等于`
  `-ne   不等于`
  `-gt   大于`
  `-ge   大于等于`
  `-lt   小于`
  `-le   小于等于`
  `用于：(())、let、bc、expr、expr1?expr2:expr3 算术条件表达式`
  `<, <=, >, >=   小于，小于等于，大于，大于等于`
  `==,  !=   等于，不等于`
- 组合符
  &&   逻辑与
  ||   逻辑或
---

## 四、常用命令总结

### 1. 文本处理命令
| 命令     | 功能        | 示例                          |
| ------ | --------- | --------------------------- |
| `cut`  | 按列提取文本    | `cut -d':' -f1 /etc/passwd` |
| `tr`   | 字符替换/删除   | `tr a-z A-Z`                |
| `sort` | 排序        | `sort -r file.txt`          |
| `uniq` | 分组        | `uniq -c file.txt`          |
| `wc`   | 统计行/字/字符数 | `wc -l file.txt`            |

---

### 2. 文件与路径命令
| 命令         | 功能     | 示例                     |
| ---------- | ------ | ---------------------- |
| `dirname`  | 取路径目录名 | `dirname /etc/passwd`  |
| `basename` | 取文件名   | `basename /etc/passwd` |

---

### 3. 输入输出命令
| 命令     | 功能    | 示例             |
| ------ | ----- | -------------- |
| `read` | 从键盘输入 | `read name`    |
| `echo` | 输出信息  | `echo "Hello"` |

---

### 4. 系统监控与控制命令
| 命令 | 功能 | 示例 |
|------|------|------|
| `watch` | 周期执行命令 | `watch -n 1 df -h` |
| `timeout` | 限时执行命令 | `timeout 5 ping 8.8.8.8` |
| `sleep` | 暂停指定时间 | `sleep 2` |
| `wait` | 等待子进程结束 | `wait $pid` |

---

### 5. 字符与字符串操作命令
| 命令     | 功能         | 示例                        |
| ------ | ---------- | ------------------------- |
| `expr` | 计算与字符串操作   | `expr length "abc"`       |
| `grep` | 文本搜索(功能之一) | `grep "root" /etc/passwd` |

---

### 6. 其他实用命令
| 命令       | 功能       | 示例       |
| -------- | -------- | -------- |
| `expect` | 自动交互执行脚本 | 适用于自动化脚本 |

---
## 五、总结
Shell 编程是 Linux 运维与自动化的核心技能。  
掌握变量、流程控制、命令替换与常用命令后，可以快速编写批处理脚本、系统监控、自动部署等工具。  
熟悉命令组合与调试技巧，可以显著提升脚本编写效率。
