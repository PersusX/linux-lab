# 《Linux系统及应用》课程实验指导书（学生用·2026版）

# 实验二 Shell 编程

| 项目 | 内容 |
|---|---|
| 课程名称 | Linux 系统及应用 |
| 实验名称 | 实验二 Shell 编程 |
| 实验学时 | 4 学时（2 次课内上机，每次 2 $\times$ 45分钟）＋ 课后巩固 |
| 适用专业 | 网络工程 |
| 实验类型 | 综合性实验 |
| 先修要求 | 已完成实验一，能熟练使用 vim 编辑文件、使用 grep/正则检索文本、掌握基本文件与权限操作 |
| 考核方式 | 实验现场逐一检查 ＋ 实验报告 |
| 配套教材 | 自编实验指导书；文东戈、赵艳芹《Linux 操作系统实用教程》 |
| 配套课件 | 《Linux系统及应用：Shell编程与开发工具》第 1～6 部分 |

> **写在前面**
>
> 实验一你学会了"手工管理一台服务器"；本实验要更进一步——**把重复的工作写成脚本**。你将扮演一名运维工程师：先搞懂 Shell 的运行机制（标准流、配置文件、变量与参数），再掌握 awk、sed、bc 这些"外挂工具"，最后独立编写并调试 4 个有真实应用场景的脚本：文件清单生成、成绩分级、磁盘负荷监控、班级成绩统计。
>
> **最终标准不是"我照抄过这些脚本"，而是：给我一个运维/数据处理需求，我能自己写出脚本、调通它、验证结果，并解释每一行的作用。**

---

## 一、实验目的

完成本实验后，你应能够：

1. 准确理解 Shell 的标准流机制（stdin/stdout/stderr），熟练运用各类重定向，并能解释 `ls f1 f2 2>ef1 1>&2` 这类复合重定向的执行结果；
2. 理解 Linux 环境变量的来源与作用域，能解读 `/etc/profile`、`/etc/bashrc`、`~/.bash_profile`、`~/.bashrc` 等启动配置文件的加载顺序与分工；
3. 掌握 Shell 变量的命名、赋值、引用规则，区分三种引号与命令替换，会使用位置变量与特殊变量；
4. 掌握 `echo`/`printf` 输出与格式控制、`read` 输入与变量存储，能从命令输出中提取所需字段；
5. 掌握 `awk`、`sed`、`bc` 等常见外部工具的基本用法，能将它们嵌入脚本解决数据处理问题；
6. 掌握 Shell 程序执行的三种基本方式及其区别，掌握 `bash -x/-v`、`set -x`、`shellcheck` 等调试手段，理解不同执行方式下调试命令的书写差异；
7. 掌握 `if`、`case`、`for`、`while` 等主要程序结构，能独立编写、调试并验证四个应用脚本（C 程序清单、成绩分级、磁盘监控、成绩统计）；
8. 形成"预测 → 执行 → 验证 → 解释"的实验习惯：每个脚本都要说清"如何验证结果是对的"。

---

## 二、实验工具及环境

### 2.1 实验环境基线

| 项目 | 推荐值 |
|---|---|
| 虚拟化平台 | VMware Workstation / VirtualBox |
| 操作系统 | Ubuntu 24.04 LTS（沿用实验一的虚拟机） |
| Shell | Bash 5.x（用 `bash --version` 确认） |
| 编辑器 | vim |
| 权限 | 普通用户 ＋ `sudo` |
| 快照 | 实验开始前、完成后各保留一个 |

### 2.2 课前环境自检

```bash
echo "$SHELL"                 # 确认登录 Shell 为 /bin/bash
bash --version | head -1      # Bash 版本
command -v awk sed bc gcc     # 本实验要用的工具是否齐备
```

> Ubuntu 默认可能未安装 `bc`，如缺失请执行 `sudo apt install -y bc` 补齐；`shellcheck` 可选安装：`sudo apt install -y shellcheck`。

### 2.3 快照要求

- 实验开始前创建快照：`实验二-开始基线`
- 实验完成后创建快照：`实验二-完成基线`

### 2.4 学时分配建议

本实验内容按两次课（每次 90 分钟）组织，已计入阅读、理解与调试的学习成本：

| 课次 | 内容 | 建议用时 |
|---|---|---|
| 第 1 次课 | 实验准备 ＋ 任务A（标准流）＋ 任务B（环境变量与配置文件）＋ 任务C（变量与输入输出） | 90 分钟 |
| 第 2 次课 | 任务D（awk/sed/bc）＋ 任务E（执行方式与调试）＋ 任务F～I（四个编程实战）＋ 现场验收 | 90 分钟 |
| 课后 | 综合实战收尾、进阶选做、实验报告 | 课外完成 |

> 进度快的同学当堂推进综合实战；进度慢的同学至少保证任务A～E 与 F、G 完成，H、I 课后补做。

---

## 三、实验准备：建立自己的工作区

**场景：** 写脚本和做运维一样，文件不能随手乱堆。建立规范工作区，脚本、数据、日志、输出证据各就各位。

```bash
mkdir -p ~/linux-lab02/{scripts,data,logs,env}
cd ~/linux-lab02
pwd
```

记录实验环境（作为报告"实验环境"部分的证据）：

```bash
{
    echo "===== 实验二环境记录 ====="
    date
    hostname
    whoami
    echo "SHELL=$SHELL"
    bash --version | head -1
    cat /etc/os-release | head -2
} | tee ~/linux-lab02/logs/environment.txt
```

后续每个任务的关键输出，建议都用 `命令 | tee ~/linux-lab02/logs/xxx.txt` 留存；每个写好的脚本**保留源文件**，实验报告要附关键代码与运行输出。

---

## 四、实验内容及操作步骤

> 全实验共 9 个任务模块（A～I）＋ 1 个综合实战 ＋ 进阶选做。
> 模块 A～E 是"机制与工具"，F～I 是"编程实战"——前者是后者的地基，不要跳过。

### 任务A Shell 标准流与重定向机制（看懂数据的"十字路口"）

**【场景】** 凌晨 cron 跑的脚本把管理员邮箱塞满了报错信息；同事写的脚本"明明重定向了，错误还是刷屏"。这类问题的根源都是一个：没搞懂 0/1/2 三根"管子"。

#### A-1 三根标准流

每个进程天生带三个文件描述符：

```text
0  stdin   标准输入（默认来自键盘）
1  stdout  标准输出（默认到屏幕）——正常结果走这里
2  stderr  标准错误（默认到屏幕）——错误信息走这里
```

动手验证"正常输出和错误输出是两条不同的管道"：

```bash
cd ~/linux-lab02/env
touch f1                      # f1 存在
ls f1 f2                      # f2 不存在：一条正常输出 + 一条错误输出
ls f1 f2 1>out.txt            # 只收正常输出，错误仍刷屏
cat out.txt
ls f1 f2 2>err.txt            # 只收错误，正常输出仍到屏幕
cat err.txt
```

#### A-2 重定向基本式

```bash
date > today.txt              # >  覆盖写
date >> today.txt             # >> 追加写
ls f1 f2 &> all.txt           # &> stdout 与 stderr 全收（bash 写法）
ls f1 f2 > all2.txt 2>&1      # 等效经典写法：先把 1 指向文件，再把 2 并入 1
ls f1 f2 2>/dev/null          # 错误丢进"黑洞"
```

> **顺序敏感：** `> file 2>&1` 正确；`2>&1 > file` 错误——后者先把 2 并入"当时的 1"（即屏幕），再把 1 改道进文件，结果错误依旧刷屏。务必自己各执行一次对比。

#### A-3 必做分析题：`ls f1 f2 2>ef1 1>&2`

在当前目录有 `f1`、没有 `f2` 的前提下，**先不要执行**，在实验记录中写下你的预测，然后执行验证：

```bash
cd ~/linux-lab02/env
ls f1 f2 2>ef1 1>&2
echo "---- ef1 的内容 ----"
cat ef1
```

执行后回答三个问题（实验报告必写）：

1. 屏幕上显示了什么？`ef1` 文件里有什么？
2. 结合重定向**从左到右依次执行**的顺序，解释：`2>ef1` 先把 2 号流指向文件 `ef1`；随后 `1>&2` 把 1 号流并入"2 号流当前的去向"——此时 2 号流已经指向 `ef1`，所以 1 号流也进了 `ef1`。最终 `ls` 对 `f1` 的正常输出和对 `f2` 的报错**都写入了 ef1**，屏幕无输出。
3. 把它和 `ls f1 f2 1>ef1 2>&1`、`ls f1 f2 2>&1 1>ef1` 逐一对比执行，说明三者结果的异同及原因。

#### A-4 输入重定向与 Here-Document

```bash
wc -l < /etc/passwd                     # 0 号流改道：从文件读
cat > note.txt <<'EOF'                  # Here-doc 内嵌多行文本
第一行
第二行
EOF
cat <<EOF                               # 不加引号：允许变量替换
当前用户是 $USER，家目录是 $HOME
EOF
```

> 想一想：`<<'EOF'`（带引号）与 `<<EOF`（不带引号）对 `$USER` 的处理有何不同？这个区别在脚本里生成配置文件时非常关键。

#### A-5 管道与 tee 复习

```bash
ls f1 f2 2>&1 | cat                     # 管道只传 stdout；想连错误一起传，先并管
who | wc -l                             # 统计在线人数
df -h / | tee ~/linux-lab02/logs/df.txt # tee：屏幕照显、文件照存
```

---

### 任务B 环境变量与 Shell 启动配置文件（环境是哪里来的？）

**【场景】** 同事抱怨："我明明在 `~/.bashrc` 里加了别名，为什么用 `ssh` 执行远程命令时不生效？"——答案藏在"登录 Shell 与非登录 Shell 加载哪些文件"里。

#### B-1 查看四个关键配置文件

```bash
cat /etc/profile
cat /etc/bashrc 2>/dev/null || ls /etc/bash*       # Ubuntu 无 /etc/bashrc，对应 /etc/bash.bashrc
cat ~/.bash_profile 2>/dev/null || echo "无 ~/.bash_profile"
cat ~/.bashrc | head -30
```

> Ubuntu 上请注意对应关系：系统级登录配置是 `/etc/profile`，系统级 bashrc 是 `/etc/bash.bashrc`（相当于 RedHat 系的 `/etc/bashrc`），用户级是 `~/.profile`（相当于 `~/.bash_profile`）与 `~/.bashrc`。报告中如实记录你实验系统上的实际情况。

在 `/etc/profile` 中找到加载 `/etc/profile.d/*.sh` 的循环：

```bash
grep -n "profile.d" /etc/profile
ls /etc/profile.d/
```

#### B-2 加载顺序验证实验

**修改前先确定 Bash 实际会读取的用户级登录配置文件，对它和 `~/.bashrc` 分别备份，实验后恢复：**

```bash
cp ~/.bashrc ~/.bashrc.bak
login_file=""
for f in ~/.bash_profile ~/.bash_login ~/.profile; do
    [ -f "$f" ] && { login_file="$f"; break; }
done
[ -n "$login_file" ] || { login_file=~/.profile; touch "$login_file"; }
cp "$login_file" "$login_file.lab02.bak"

echo 'echo ">>> 正在加载 ~/.bashrc"' >> ~/.bashrc
echo "echo \">>> 正在加载 $login_file\"" >> "$login_file"
bash -l            # 启动登录 Shell，观察登录配置与 .bashrc 的加载情况
exit
bash               # 启动一个非登录 Shell，对比哪些 echo 出现了
exit
# 实验结束恢复：
mv ~/.bashrc.bak ~/.bashrc
mv "$login_file.lab02.bak" "$login_file"
```

结论（写入报告）：登录 Bash 先读取 `/etc/profile`（Ubuntu 中由它加载 `/etc/profile.d/*.sh`），再按顺序查找 `~/.bash_profile`、`~/.bash_login`、`~/.profile`，**只读取找到的第一个**。Ubuntu 默认的 `~/.profile` 通常会在运行 Bash 时再加载 `~/.bashrc`，但这取决于文件中的实际内容。非登录交互 Bash 读取 `/etc/bash.bashrc` 与 `~/.bashrc`。后加载的设置可以覆盖先加载的同名设置。

#### B-3 常见环境变量

```bash
env | sort                    # 全部环境变量
printenv PATH                 # 单个查看
echo $PATH; echo $HOME; echo $LANG
set | head -20                # set 显示所有变量（含未导出的本地变量）
which -a python3 2>/dev/null  # PATH 搜索顺序：从左到右，谁在前用谁
```

#### B-4 本地变量 vs 环境变量（export 跨边界实验）

```bash
myvar="hello"
bash -c 'echo "子Shell看到: [$myvar]"'     # 空！本地变量不传给子进程
export myvar="hello"
bash -c 'echo "子Shell看到: [$myvar]"'     # 这次有了
```

> 继承方向是**父 → 子单向**：子 Shell 里改 `$myvar`，父 Shell 不受影响。自己验证一次。

#### B-5 定制自己的环境（选做）

往 `~/.bashrc` 末尾追加（先备份）：

```bash
echo 'alias ll="ls -lh"' >> ~/.bashrc
echo 'export EDITOR=vim' >> ~/.bashrc
source ~/.bashrc              # 改完立即生效，不必重开终端
ll
```

> 想一想：为什么改完 `~/.bashrc` 要用 `source`？直接 `./.bashrc` 行不行？（提示：与任务E 的三种执行方式有关。）

---

### 任务C 变量、引号、位置参数与输入输出（脚本的基本功）

**【场景】** 写备份脚本要拼带日期的文件名；写部署脚本要接收命令行参数；写交互脚本要问用户"确认删除吗"。这些是每一个 Shell 脚本的地基。

#### C-1 命名、赋值与引用

```bash
name="zhangsan"        # 对：等号两边不能有空格
echo $name
echo "我的名字是${name}同学"   # ${} 圈定变量名边界，防止粘连歧义
```

> 铁律：`name = "zhangsan"`（带空格）会被解析成"运行命令 name，参数是 = 和 zhangsan"，直接报错。变量命名规则：字母、数字、下划线组成，**数字不开头**；环境变量惯例全大写，脚本内变量惯例小写。

#### C-2 三种引号与命令替换

```bash
course=linux
echo '课程名是$course'         # 单引号：原样输出，$ 不解析
echo "课程名是$course"         # 双引号：解析 $变量 与 $( )
echo "今天是$(date +%F)"       # $( ) 命令替换：先执行命令再嵌入结果
count=$(grep -c 'bash$' /etc/passwd)
echo "使用bash的用户有${count}个"
```

> 现代脚本一律用 `$( )`，不用反引号 `` ` ``——后者嵌套时要层层转义。想在双引号里打印字面 `$`，用 `\$`。

#### C-3 位置参数与特殊变量

编写 `~/linux-lab02/scripts/args.sh`：

```bash
#!/bin/bash
# args.sh —— 演示位置参数与特殊变量
echo "脚本名: $0"
echo "第1个参数: $1"
echo "第2个参数: $2"
echo "参数个数: $#"
echo "全部参数: $@"
echo "本脚本PID: $$"
```

```bash
bash args.sh hello world
bash args.sh            # 不传参：$1 为空——所以正式脚本要做参数检查
```

参数检查的标配写法（后续每个脚本都要用）：

```bash
[ $# -lt 2 ] && { echo "用法: $0 参数1 参数2"; exit 1; }
```

常用特殊变量速查：`$?` 上一条命令退出码（0 成功）；`$#` 参数个数；`$@`/`$*` 全部参数；`$$` 当前 Shell 的 PID；`$!` 最近一个后台进程的 PID。

#### C-4 输出语句与格式控制：echo 与 printf

```bash
echo -e "姓名\t成绩\n张三\t90"            # -e 解析转义：\t 制表、\n 换行
echo -e "\033[31m【告急】磁盘不足\033[0m"  # 31红/32绿/33黄，\033[0m 复位
printf "%-10s %5d\n" "张三" 90 "李四" 85   # printf：左对齐10字符 + 占5位整数
printf "平均: %.2f\n" 87.666              # 保留两位小数
```

> 复杂排版优先 `printf`——它不像 `echo` 那样因选项/转义在不同 Shell 间有差异。

#### C-5 输入语句与变量存储：read

```bash
read -p "请输入你的名字: " uname
echo "你好, $uname"
read -sp "请输入密码: " upass; echo      # -s 不回显（输密码），手动补换行
read -t 5 -p "5秒内确认(y/n): " ans      # -t 限时，超时返回非0
echo "你的选择: $ans"
read a b c <<< "one two three"           # 一次读多个变量
echo "$a / $b / $c"
```

#### C-6 从命令输出中提取字段（运维核心技能）

给定一条"胖输出"，只取你要的那一列：

```bash
df -h /                                 # 原始输出
df -h / | tail -1                       # 先取数据行
df -h / | tail -1 | awk '{print $5}'    # 再取第5列（Use%）
df -h / | tail -1 | awk '{print $5}' | tr -d '%'   # 去掉百分号，得到纯数字
usage=$(df -h / | tail -1 | awk '{print $5}' | tr -d '%')
echo "根分区使用率: ${usage}%"           # 提取结果存进变量
```

再练两个不同分隔符的例子：

```bash
grep "^$USER:" /etc/passwd | cut -d: -f1,7    # cut 按 : 取第1、7列
who | awk '{print $1}' | sort -u              # 提取登录用户名并去重
```

> 思路模板（任务H、I 都要用）：**先取全量 → 按行过滤（grep/tail）→ 按列提取（awk/cut）→ 清洗（tr -d）→ 存变量。**

#### C-7 变量替换与模式截断（选学，实战会用到）

```bash
port=${PORT:-8080}; echo "端口: $port"     # 变量为空则用默认值
f="main.c"
echo "${f%.c}"        # 去尾 .c  → main（取主名）
echo "${f##*.}"       # 贪婪去头 → c（取扩展名）
```

---

### 任务D 外部工具：awk、sed、bc（脚本的"外挂武器库"）

**【场景】** Shell 本身不擅长算术和列处理——整数运算还能凑合，浮点和"按列统计"必须请外援。这三个工具是运维数据处理的三剑客。

#### D-1 awk：按列处理与统计

准备数据：

```bash
cat > ~/linux-lab02/data/score1.txt <<'EOF'
1班 张三 85
1班 李四 92
2班 王五 78
2班 赵六 88
3班 钱七 95
3班 孙八 60
EOF
```

```bash
awk '{print $1, $2}' ~/linux-lab02/data/score1.txt     # 取第1、2列
awk '$3 >= 90' ~/linux-lab02/data/score1.txt           # 条件过滤：90分以上
awk '{sum+=$3} END{print "总分:", sum, "平均:", sum/NR}' ~/linux-lab02/data/score1.txt
awk -F: '{print $1}' /etc/passwd | head -3             # -F 指定分隔符
```

awk 要点：`$0` 整行、`$1..$n` 各列、`NR` 行号（处理完即总行数）、`NF` 列数；结构是 `条件{动作}`，`END{}` 在所有行处理完后执行。

#### D-2 sed：流式编辑

```bash
sed 's/1班/一班/' ~/linux-lab02/data/score1.txt        # 替换（只输出不改原文件）
sed -n '2,4p' ~/linux-lab02/data/score1.txt            # 只打印第2～4行
sed '/张三/d' ~/linux-lab02/data/score1.txt            # 删除含"张三"的行
sed -i.bak 's/60/61/' ~/linux-lab02/data/score1.txt    # -i 原地改，.bak 自动备份
```

> `-i` 直接改文件是高危操作，养成 `-i.bak` 留备份的习惯。

#### D-3 bc：浮点运算

```bash
echo $((7/2))                  # Shell 整数运算：3（小数被砍）
echo "7/2" | bc                # bc 默认也是整数：3
echo "scale=2; 7/2" | bc       # scale 控制小数位：3.50
echo "scale=4; 10/3" | bc      # 3.3333
avg=$(echo "scale=2; (85+92+78)/3" | bc); echo "平均: $avg"
```

> 记忆锚点：Shell 算术用 `$(( ))`（只算整数）；要小数就找 `bc`；批量按列统计找 `awk`。

#### D-4 组合小实战

```bash
# 统计 /etc/passwd 中各登录 Shell 的使用人数
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn

# 计算本机内存使用百分比（小数两位）
free | awk '/Mem/{printf "%.2f%%\n", $3/$2*100}'
```

---

### 任务E 脚本的三种执行方式与调试（会跑更要会调）

**【场景】** 同事说"脚本我 source 了一下，结果把我终端的环境变量全改了"；你的脚本"结果不对又不报错"。本任务解决两件事：脚本怎么跑、错了怎么查。

#### E-1 第一个脚本

编写 `~/linux-lab02/scripts/sysinfo.sh`：

```bash
#!/bin/bash
# sysinfo.sh —— 系统健康速览
echo "===== 主机: $(hostname) ====="
echo "时间: $(date '+%F %T')"
echo "在线用户: $(who | wc -l) 人"
df -h / | tail -1
free -h | head -2
```

> 第一行 `#!/bin/bash` 叫 shebang，告诉内核用哪个解释器执行本文件。脚本开头写注释（作者、功能、日期）是职业习惯。

#### E-2 三种执行方式对比实验（重点）

先准备一个"变量探针"脚本 `~/linux-lab02/scripts/var.sh`：

```bash
#!/bin/bash
MYNAME="linux"
echo "脚本内 MYNAME=$MYNAME"
```

依次用三种方式执行，**每次执行后都检查当前 Shell 里变量的留存情况**：

```bash
cd ~/linux-lab02/scripts
unset MYNAME                          # 清空，保证实验干净

# 方式一：路径执行（需要执行权限，开子Shell）
chmod +x var.sh
./var.sh
echo "方式一后 MYNAME=[$MYNAME]"      # 空：子Shell的变量留不住

# 方式二：解释器执行（不需要执行权限，同样开子Shell）
bash var.sh
echo "方式二后 MYNAME=[$MYNAME]"      # 空

# 方式三：source / .（在当前Shell内执行）
unset MYNAME
source var.sh
echo "方式三后 MYNAME=[$MYNAME]"      # linux：变量留在当前Shell了
```

结论（写入报告）：

| 执行方式 | 是否需 +x 权限 | 在哪执行 | 变量/别名/cd 是否影响当前 Shell |
|---|---|---|---|
| `./script.sh` | 需要 | 子 Shell | 否 |
| `bash script.sh` | 不需要 | 子 Shell | 否 |
| `source script.sh`（或 `. script.sh`） | 不需要 | **当前 Shell** | **是** |

这解释了 B-5 的问题：改完 `~/.bashrc` 必须用 `source`（在当前 Shell 执行）才能不重登生效。

#### E-3 调试：bash -x / -v 与 set -x

编写一个故意留有隐患的脚本 `~/linux-lab02/scripts/hi.sh`：

```bash
#!/bin/bash
name=$1
echo "hello $name"
if [ "$name" = "root" ]; then
    echo "欢迎管理员"
fi
```

```bash
bash -x hi.sh root      # -x：逐行打印"变量替换后"的执行轨迹（+ 开头）
bash -x hi.sh           # 不传参，看轨迹里 name 为空时发生了什么
bash -v hi.sh root      # -v：先打印脚本原样行，再执行
```

局部追踪（脚本很长时避免刷屏）：在可疑代码段前后加 `set -x` / `set +x`：

```bash
#!/bin/bash
echo "正常段"
set -x                  # 打开追踪
name=$1
[ "$name" = "root" ]
set +x                  # 关闭追踪
echo "结束"
```

> **思考（报告必答）：三种执行方式下，调试命令的书写有什么不同？**
> `./script.sh` 要在 shebang 里临时改成 `#!/bin/bash -x`，或改用 `bash -x ./script.sh`；`bash script.sh` 直接 `bash -x script.sh`；而 `source` 是在**当前 Shell** 执行，不能加 `-x` 选项，只能在脚本内（或当前 Shell 先执行 `set -x`）用 `set -x`/`set +x` 控制，调试完记得 `set +x` 关闭，否则你的终端会一直刷屏。

#### E-4 shellcheck 静态检查

```bash
shellcheck ~/linux-lab02/scripts/hi.sh     # 未安装则 sudo apt install -y shellcheck
```

把报告的警告逐条看懂（如 SC2086"变量应加双引号"）并修复。之后的每个脚本**提交前都必须过一遍 shellcheck**。

---

### 任务F 编程实战 1：for 循环生成 C 程序文件清单

**【场景】** 学期末收作业：你的注册目录（家目录）下有 `a_sub`、`b_sub` 两个子目录，里面散落着 C 源程序和编译出的目标文件。写一个脚本，自动列出两个子目录下所有 C 程序文件（`.c`）及其对应的目标文件（`.o`）清单。

#### F-0 搭建实验环境

```bash
mkdir -p ~/a_sub ~/b_sub
cd ~/a_sub
cat > hello.c <<'EOF'
#include <stdio.h>
int main(void){ printf("hello from a_sub\n"); return 0; }
EOF
cat > add.c <<'EOF'
int add(int a, int b){ return a + b; }
EOF
gcc -c hello.c add.c            # 生成 hello.o add.o
cd ~/b_sub
cat > main.c <<'EOF'
#include <stdio.h>
int main(void){ printf("hello from b_sub\n"); return 0; }
EOF
gcc -c main.c                   # 生成 main.o
echo "test" > note.txt          # 干扰文件：不应出现在清单里
```

#### F-1 先用单行 for 循环热身

```bash
for f in ~/a_sub/*.c; do echo "发现源文件: $f"; done
for d in ~/a_sub ~/b_sub; do echo "== $d =="; ls "$d"; done
```

#### F-2 编写脚本 `~/linux-lab02/scripts/clist.sh`

要求：用 for 循环完成；输出带表头、分目录展示；每个 `.c` 文件旁边标注其对应 `.o` 是否存在（用 `${src%.c}.o` 拼接目标文件名，用 `[ -f ... ]` 判断）。参考骨架：

```bash
#!/bin/bash
# clist.sh —— 列出 ~/a_sub 与 ~/b_sub 下的 C 源文件及其目标文件
for dir in ~/a_sub ~/b_sub; do
    echo "===== 目录: $dir ====="
    for src in "$dir"/*.c; do
        [ -e "$src" ] || { echo "(无 C 源文件)"; break; }   # 防止通配符未匹配时原样输出
        obj="${src%.c}.o"
        if [ -f "$obj" ]; then
            printf "%-20s 目标文件: %-20s [存在]\n" "$(basename "$src")" "$(basename "$obj")"
        else
            printf "%-20s 目标文件: [缺失]\n" "$(basename "$src")"
        fi
    done
done
```

#### F-3 验证实验结果（报告必写）

"如何证明清单是对的？"至少用两种独立方法交叉验证：

```bash
find ~/a_sub ~/b_sub \( -name "*.c" -o -name "*.o" \) # 方法1：find 全量列出
ls -l ~/a_sub ~/b_sub                                # 方法2：人工 ls 对照
bash clist.sh | tee ~/linux-lab02/logs/clist.out    # 脚本输出与前两者逐一比对
```

> 思考题：`"$dir"/*.c` 在目录里没有 `.c` 文件时会怎么样？骨架里的 `[ -e "$src" ]` 一行解决的是什么问题？

---

### 任务G 编程实战 2：学生成绩分级程序（if/case 与输入校验）

**【场景】** 教务老师要一个"查分小工具"：键盘输入一个百分制成绩（0～100），屏幕显示对应等级：90 及以上"优秀"、80～89"良"、70～79"中"、60～69"及格"、60 以下"不及格"。

#### G-1 if-elif 版本 `~/linux-lab02/scripts/grade_if.sh`

```bash
#!/bin/bash
# grade_if.sh —— 输入成绩，输出等级
read -p "请输入成绩(0-100): " score

if [ "$score" -ge 90 ]; then
    echo "优秀"
elif [ "$score" -ge 80 ]; then
    echo "良"
elif [ "$score" -ge 70 ]; then
    echo "中"
elif [ "$score" -ge 60 ]; then
    echo "及格"
else
    echo "不及格"
fi
```

#### G-2 非法输入处理（本题的重点，报告必写）

**先制造故障再修：** 依次输入 `abc`、``（直接回车）、`120`、`-5`，观察脚本的报错或错误输出。然后回答：如果输入的数据不合法，应该怎么处理？

加固方案——在 read 之后、判断之前插入校验：

```bash
# 1. 空输入拦截
[ -z "$score" ] && { echo "错误: 输入不能为空"; exit 1; }

# 2. 非数字拦截（正则：必须全部由数字组成）
[[ "$score" =~ ^[0-9]+$ ]] || { echo "错误: 请输入非负整数"; exit 2; }

# 3. 越界拦截
[ "$score" -gt 100 ] && { echo "错误: 成绩不能超过100"; exit 3; }
```

加固后用 `abc`、空、`120`、`88` 各测一遍，全部符合预期才算完成。进阶：把"输入→校验→输出"包进 `while true` 循环，输入 `q` 退出，非法输入提示后重新输入而不是退出。

#### G-3 case 版本 `~/linux-lab02/scripts/grade_case.sh`

用"成绩的十分位"做 case 模式改写同一功能，体会两种结构的适用场景：

```bash
#!/bin/bash
# grade_case.sh —— case 版成绩分级
read -p "请输入成绩(0-100): " score
[ -z "$score" ] && { echo "错误: 输入不能为空"; exit 1; }
[[ "$score" =~ ^[0-9]+$ ]] || { echo "错误: 请输入非负整数"; exit 2; }
[ "$score" -le 100 ] || { echo "错误: 成绩不能超过100"; exit 3; }

case $((score / 10)) in
    10|9) echo "优秀" ;;
    8)    echo "良" ;;
    7)    echo "中" ;;
    6)    echo "及格" ;;
    *)    echo "不及格" ;;
esac
```

> 想一想：case 分支里的 `10|9` 是什么含义？`*)` 在这里兜住了哪些情况？if-elif 和 case 哪个更适合这种"区间分级"问题，为什么？

---

### 任务H 编程实战 3：/home 磁盘占用监控脚本

**【场景】** 为便于系统管理员对磁盘分配进行管理：当文件系统 `/home` 的占用量变化时给出提示——小于 50% 提示"用户文件系统磁盘使用负荷量小"；大于等于 50%、小于 90% 提示"用户文件系统磁盘使用负荷量正常"；大于等于 90% 提示"用户文件系统磁盘使用负荷量偏大"。

#### H-1 先搞清楚：df 和 du 回答的是两个不同问题

```bash
df -h /home                # 文件系统层面：/home 所在分区总容量、已用、可用、Use%
du -sh /home               # 目录层面：/home 目录树下实际占用的空间合计
du -sh /home/* 2>/dev/null # 各用户家目录分别占多少
```

> 关键辨析（报告必写）：`df` 看的是**文件系统/分区**的占用（含已删除但被占用、保留块等），`du` 看的是**目录树**的文件合计。本题要判断"`/home` 在系统磁盘中的占用百分比"，属于**文件系统使用率**，应取 `df` 的 `Use%` 字段；若题目改成"哪些用户把 /home 挤满了"，才轮到 `du`。

#### H-2 提取使用率数字

```bash
df -h /home | tail -1                          # 找到数据行
usage=$(df -h /home | tail -1 | awk '{print $5}' | tr -d '%')
echo "当前/home使用率: ${usage}%"               # 纯数字，可用于整数比较
```

#### H-3 编写 `~/linux-lab02/scripts/diskwatch.sh`

```bash
#!/bin/bash
# diskwatch.sh —— 监测 /home 文件系统使用负荷
# 不传参数时读取真实使用率；传入数字时可测试各个分支
usage=${1:-$(df /home | tail -1 | awk '{print $5}' | tr -d '%')}
[[ "$usage" =~ ^[0-9]+$ ]] || { echo "错误: 使用率必须是整数"; exit 1; }

echo "当前 /home 使用率: ${usage}%"
if [ "$usage" -ge 90 ]; then
    echo -e "\033[31m用户文件系统磁盘使用负荷量偏大\033[0m"
elif [ "$usage" -ge 50 ]; then
    echo "用户文件系统磁盘使用负荷量正常"
else
    echo "用户文件系统磁盘使用负荷量小"
fi
```

#### H-4 验证方法

1. 不带参数直接运行脚本，与 `df -h /home` 的实际百分比对照，确认落入的档位正确；
2. **构造输入验证所有分支**：分别运行 `bash diskwatch.sh 30`、`bash diskwatch.sh 60`、`bash diskwatch.sh 95`，确认三次输出依次为“负荷量小”、“负荷量正常”和“负荷量偏大”。这种方法不依赖磁盘实际容量，也不需要创建大文件；
3. 扩展（选做）：用 `while true; do ... sleep 30; done` 把脚本改成"每 30 秒巡检一次、使用率跨越档位时才提示"的常驻监控，提示信息追加写入 `~/linux-lab02/logs/disk.log`。

---

### 任务I 编程实战 4：班级成绩统计（读取文件与 awk 统计）

**【场景】** `score.txt` 保存了三个班级学生某门课程的考试成绩，格式为"班级 姓名 成绩"。统计每个班级的学生人数与平均分。

#### I-1 准备数据

```bash
cat > ~/linux-lab02/data/score.txt <<'EOF'
1班 张三 85
1班 李四 92
1班 王五 78
2班 赵六 88
2班 钱七 90
2班 孙八 55
2班 周九 73
3班 吴十 95
3班 郑一 60
3班 王二 82
EOF
```

#### I-2 方法一：while read 逐行读取（学习读文件的标准姿势）

编写 `~/linux-lab02/scripts/classstat.sh`：

```bash
#!/bin/bash
# classstat.sh —— 统计各班人数与平均分（while read 版）
data=${1:-~/linux-lab02/data/score.txt}
[ -f "$data" ] || { echo "数据文件不存在: $data"; exit 1; }
c1=0; c2=0; c3=0
s1=0; s2=0; s3=0

while read -r cls name score; do
    case "$cls" in
        1班) c1=$((c1+1)); s1=$((s1+score)) ;;
        2班) c2=$((c2+1)); s2=$((s2+score)) ;;
        3班) c3=$((c3+1)); s3=$((s3+score)) ;;
    esac
done < "$data"

for i in 1 2 3; do
    eval "c=\$c$i; s=\$s$i"        # 按名字间接取变量
    [ "$c" -gt 0 ] && echo "${i}班: ${c}人, 平均分: $(echo "scale=1; $s/$c" | bc)"
done
```

> 要点：`while read -r a b c; do ...; done < 文件` 是逐行读文件的标准写法；`-r` 禁止反斜杠转义，读啥是啥；重定向 `<` 把整个文件喂给循环。

#### I-3 方法二：awk 一行流（体会工具选型的差异）

```bash
awk '{cnt[$1]++; sum[$1]+=$3}
     END{ for (c in cnt) printf "%s: %d人, 平均分: %.1f\n", c, cnt[c], sum[c]/cnt[c] }' \
    ~/linux-lab02/data/score.txt | sort -V
```

数组 `cnt[班级]`、`sum[班级]` 随读随累计，`END` 里统一输出——**不需要预知有几个班**，这是 awk 版相比 case 版的本质优势。awk 关联数组的遍历顺序不固定，因此最后使用 `sort -V` 让报表按班级稳定排序。

#### I-4 验证与扩展（报告必写）

1. 用 `awk '{print $1}' score.txt | sort | uniq -c` 核对各班人数，手工抽一个班验算平均分；
2. **思考题（数据升级）：如果数据文件变成 3 个班级、5 门不同课程的成绩**（例如格式 `班级 姓名 语文 数学 英语 物理 化学`），程序应该怎么调整？提示：awk 版只需把 `sum[$1]+=$3` 改为对第 3～7 列分别建数组累计，或先 `for (i=3;i<=NF;i++)` 循环累加各科；while read 版则需要 7 个读入变量。动手改一版并验证。

---

### 综合实战：运维工具箱 toolbox.sh（必做，现场考核内容）

**【场景】** 把本实验的四个实战脚本整合成一个带菜单的"运维工具箱"，交付给值班同学使用。

编写 `~/linux-lab02/scripts/toolbox.sh`，要求：

| # | 验收项 | 要求 |
|---|---|---|
| 1 | 菜单框架 | 用 `select` 或 `case`+循环实现菜单：1)系统速览 2)C文件清单 3)成绩分级 4)磁盘监测 5)成绩统计 0)退出 |
| 2 | 复用整合 | 各功能调用或内嵌任务 F～I 的成果，不允许贴死路径（用变量或 `$HOME`） |
| 3 | 防御式设计 | 脚本开头检查依赖命令（`command -v awk bc`）；所有外部输入先校验再使用；数据文件路径支持参数覆盖、缺省用默认值（`${1:-...}`） |
| 4 | 输出规范 | 关键结果用 `printf` 对齐；告警信息用红色；每次操作带时间戳追加写入 `~/linux-lab02/logs/toolbox.log` |
| 5 | 调试证据 | 提供一次 `bash -x` 排错过程的记录（哪怕是故意制造的 bug） |
| 6 | 质检 | `shellcheck toolbox.sh` 无未修复警告 |
| 7 | 快照 | 创建 `实验二-完成基线` 快照 |

---

### 进阶选做（学有余力者完成，可获加分）

1. **批量重命名**：把某目录下所有 `.jpg` 改名为 `.png`（提示：`for` + `${f%.jpg}.png`）；再加码：给所有文件加日期前缀 `20260919-原名`。
2. **服务存活自检**：写一个 `check_app.sh`，检测指定进程（如 `sleep` 模拟的服务）是否存在，不存在则自动拉起并写日志（提示：`pgrep -x`）；思考配合 crontab 每分钟执行要注意什么（cron 环境变量极简，脚本内写绝对路径）。
3. **主机存活扫描**：`host_scan.sh` 扫描 `192.168.1.1~20`，在线主机写入 `alive.txt`、离线写入 `dead.txt`，最后统计各多少台（提示：`ping -c1 -W1`、for 循环、`&&/||`）。
4. **多课程成绩完整版**：完成 I-4 的扩展，数据含 5 门课程，输出"各班每门课的平均分 + 全班总平均分"，用 awk 实现并与手工抽验对照。
5. **登录环境调查**：在你本机上完整梳理一次登录 Shell 的加载链：`/etc/profile` → `profile.d/*.sh` → `~/.profile` → `~/.bashrc`，画出加载顺序图，标注每个文件里实际设置了什么。

---

## 五、实验注意事项（安全纪律）

1. **只在虚拟机中练习。** 本实验涉及修改 Shell 配置文件、创建/删除文件，禁止在宿主机或他人机器上操作。
2. **改配置文件先备份。** 修改 `~/.bashrc`、`~/.profile` 前 `cp` 留底，实验后恢复；`/etc/profile`、`/etc/bash.bashrc` 是系统级文件，本实验**只读不写**——改错可能导致无法正常登录。
3. **验证用的测试文件用后清理。** 任务H 造的大文件必须 `rm` 删除；执行任何 `rm` 前 `pwd` 确认位置。
4. **脚本先在小数据上验证。** 每个脚本先用任务中给的样例数据跑通、用独立方法交叉验证，再谈扩展。
5. **红队习惯：** 对自己的脚本做"恶意输入测试"——空输入、非数字、超范围值、不存在的路径，一个健壮的脚本应该全部拦得住。
6. 命令与脚本应亲自执行、逐行理解；引用外部资料时注明来源，以本机 `man` 手册为准。

---

## 六、思考题

### 基础思考题（实验报告中选答不少于 8 题；第 1、6、11 题为必答）

1. **（必答）** 当前目录下有 `f1`、没有 `f2`，解释 `ls f1 f2 2>ef1 1>&2` 的运行结果，并对比 `1>ef1 2>&1`、`2>&1 1>ef1` 的异同。
2. 标准输入、输出、错误的文件描述符分别是几？为什么要把正常输出和错误输出分成两条流？
3. 登录 Shell 与非登录 Shell 分别加载哪些配置文件？顺序如何？为什么改完 `~/.bashrc` 执行 `source ~/.bashrc` 就能生效？
4. Shell 变量赋值时等号两边为什么不能有空格？单引号、双引号、`$( )` 对 `$` 的处理有何区别？
5. 本地变量与环境变量有什么区别？`export` 的继承方向是什么？子 Shell 修改了 export 变量，父 Shell 能看到吗？
6. **（必答）** Shell 程序执行的三种基本方式是什么？三者在执行权限、执行位置（当前 Shell/子 Shell）、变量影响上各有何差异？对应的调试命令书写方式有什么不同？
7. 位置变量 `$0`、`$1`、`$#`、`$@`、`$?` 各是什么含义？脚本开头为什么要做参数个数检查？
8. `[ ]` 中变量为什么必须加双引号？`[ 10 > 2 ]` 的结果是什么、为什么？整数比较应该用什么？
9. `$(( ))`、`expr`、`bc` 分别适合什么计算场景？为什么 `echo $((7/2))` 的结果是 3？
10. awk 中 `$0`、`$1`、`NR`、`NF`、`END` 各是什么含义？`while read ... done < file` 中 `-r` 和 `<` 各起什么作用？
11. **（必答）** Linux/UNIX 中的 Shell 程序与 DOS 批处理文件（.bat）比较，有何异同？（提示：从解释器与内核的关系、变量与流程控制能力、管道/重定向机制、外部工具生态、可移植性等角度比较。）

### 进阶思考题（选答）

1. `df -h /home` 与 `du -sh /home` 的数字为什么经常对不上？什么场景用哪个？
2. 脚本在终端手动执行正常，放进 crontab 就失败，最可能的原因是什么？如何根治？
3. 任务F 中 `"$dir"/*.c` 在目录无匹配文件时的行为是什么？除了骨架里的 `[ -e "$src" ]` 判断，还有什么机制可以解决（提示：`nullglob`）？
4. 任务I 的 awk 版为什么不需要预知班级数量？如果要求"按平均分从高到低输出班级排名"，管道应该怎么接？

---

## 七、实验报告要求

实验报告应包含：**实验目的、实验环境、实验内容与步骤、实验结果（输出证据）、结果分析与思考** 五个部分。

1. **实验目的**：不得照抄指导书，用自己的话写"通过本实验掌握/验证/理解了什么"。
2. **实验环境**：记录发行版及版本、内核、Bash 版本（附 `bash --version`、`cat /etc/os-release`、`uname -r` 输出），说明本机 `/etc/bashrc` 与 `~/.bash_profile` 的实际存在情况。
3. **实验内容与步骤**：按任务模块写清操作顺序；任务 A-3、F、G、H、I 必须附上**脚本源代码**与**运行输出**。
4. **实验结果（输出证据）**：每个脚本提供"输入 → 输出 → 结果判断"三段式证据，**必须说明你是如何验证结果正确的**（交叉验证命令、构造的测试数据、边界用例）。推荐用 `tee` 保存的文本，而不是只有截图。
5. **分析与思考**：完成所选思考题（含 3 道必答题）；成绩分级题必须写出"非法输入如何处理"的完整方案与测试记录；磁盘监控题必须写出 df 与 du 的辨析。
6. 报告末尾附：toolbox.sh 源代码、shellcheck 检查输出、实验验收清单自查结果。

> 如实记录实验过程，包括失败的尝试和纠正过程——`bash -x` 的排错记录正是评分依据之一。

---

## 八、实验验收清单（现场检查用）

### 机制与工具部分

- [ ] 能解释 0/1/2 三根标准流，能现场解释 `ls f1 f2 2>ef1 1>&2` 的结果
- [ ] 能说出登录/非登录 Shell 的配置文件加载顺序，本机四个文件的实际对应关系
- [ ] 能正确使用变量（赋值无空格、三种引号、`$( )` 命令替换、位置参数）
- [ ] 能用 `echo -e`/`printf` 控制输出格式，用 `read` 接收输入并存储
- [ ] 能从 `df`/`who`/`/etc/passwd` 等输出中提取指定字段并存入变量
- [ ] 能用 awk 取列/过滤/求平均，用 sed 替换/删行，用 bc 做浮点运算
- [ ] 能说出脚本三种执行方式的区别，并现场演示变量留存实验
- [ ] 能用 `bash -x`、`set -x/+x` 调试脚本，能用 shellcheck 检查脚本

### 编程实战部分

- [ ] clist.sh：正确列出 a_sub、b_sub 下的 `.c` 与 `.o`，能说出两种验证方法
- [ ] grade_if.sh / grade_case.sh：五档分级正确，空输入/非数字/越界全部拦截
- [ ] diskwatch.sh：三档提示正确，能解释为什么用 df 而不是 du
- [ ] classstat.sh：各班人数与平均分正确，能说出"5 门课程"时的调整方案
- [ ] 综合实战 toolbox.sh 通过 7 项验收（菜单、复用、防御、日志、调试证据、shellcheck、快照）
- [ ] 已创建 `实验二-完成基线` 快照

---

> 本指导书与《Linux系统及应用：Shell编程与开发工具》课件第 1～6 部分配套使用（gcc/gdb 内容属实验三）。遇到发行版/版本差异时，以本机 `man` 手册、发行版官方文档为准。
