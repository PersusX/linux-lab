# 《Linux系统及应用》课程实验指导书（学生用·2026版）

# 实验一 Linux 系统基本使用与基础运维实践

| 项目 | 内容 |
|---|---|
| 课程名称 | Linux 系统及应用 |
| 实验名称 | 实验一 Linux 系统基本使用与基础运维实践 |
| 实验学时 | 4 学时（课内上机）＋ 课外延伸 |
| 适用专业 | 网络工程 |
| 实验类型 | 综合性实验 |
| 先修要求 | 已修完“课程0：Linux 系统及应用——概述、安装与现代实践”，已在虚拟机中完成 Ubuntu 24.04 LTS（或教师指定发行版）安装，并创建“安装完成基线”快照 |
| 考核方式 | 实验现场逐一检查 ＋ 实验报告 |
| 配套教材 | 自编实验指导书；文东戈、赵艳芹《Linux 操作系统实用教程》 |

> **写在前面**
>
> 本实验不再以“把命令敲一遍”为目标。你将扮演一名刚入职的系统运维工程师，接手一台全新的 Linux 服务器：先摸清它的环境，再完成账号、文件、权限、服务、网络等日常管理工作，最后交付一份别人可以复查的验收报告。
>
> **最终标准不是“我敲过这些命令”，而是：我能在一台陌生的 Linux 主机上确认环境、完成基本管理、定位常见问题、验证修改结果，并留下可复查的证据。**

---

## 一、实验目的

完成本实验后，你应能够：

1. 正确登录、注销、关机和重启 Linux 系统，理解 Shell、用户态与内核态的基本关系；
2. 查看当前用户、Shell、进程、系统时间、内核版本与发行版版本，并正确区分“内核版本”与“发行版版本”；
3. 完成普通用户的创建、修改、锁定、解锁和删除，理解 UID、主要组与附加组的关系；
4. 理解 Linux 文件系统树、绝对路径与相对路径，熟练完成文件和目录的创建、复制、移动、删除与查看；
5. 掌握 `cat`、`less`、`head`、`tail`、`wc`、`grep` 等日常文本处理命令，掌握 `find`、`which`、`whereis` 的区别；
6. 掌握 `tar` 归档、`dd` 块复制、硬链接与符号链接等基础文件管理技术；
7. 掌握 vi/vim 的基本工作模式与常用编辑操作，能在无图形界面的服务器上修改配置文件；
8. 掌握正则表达式基础，能用 `grep` 完成日志级别的文本检索与统计；
9. 掌握用户组管理（`groupadd`、`gpasswd`、`usermod -aG`、`groupmod`、`chgrp`）与 ACL 的初步使用；
10. 掌握 Linux 文件权限、`chmod`、`chown`、`chgrp`、`umask` 与特殊权限（setuid/setgid/sticky），并通过实验验证“目录权限决定能否进入/增删目录项、文件权限决定能否读取/修改”；
11. 能使用 `systemd/systemctl`、`NetworkManager/nmcli` 完成基本服务与网络检查，按照“现象 → 分层定位 → 命令验证 → 修改 → 再验证”的流程处理常见问题；
12. 形成运维工作的证据意识：记录环境、命令、输出、判断与改进，而不是只写“完成”。

---

## 二、实验工具及环境

### 2.1 实验环境基线

| 项目 | 推荐值 |
|---|---|
| 虚拟化平台 | VMware Workstation / VirtualBox |
| CPU | 2 vCPU |
| 内存 | 4 GB |
| 磁盘 | 40 GB |
| 网络 | NAT（1 张网卡） |
| 操作系统 | Ubuntu 24.04 LTS 或教师指定版本 |
| Shell | Bash |
| 权限 | 普通用户 ＋ `sudo` |
| 快照 | 实验开始前、完成后各保留一个 |

### 2.2 课前环境自检

开始实验前，先确认你的环境可用，并将输出留存：

```bash
cat /etc/os-release      # 发行版及版本
uname -r                 # 内核版本
hostnamectl              # 主机与系统概览
lsblk -f                 # 磁盘与文件系统
ip -br addr              # 网卡与地址
ip route                 # 默认路由
resolvectl status        # DNS
ping -c 4 223.5.5.5      # IP 连通性
curl -I https://www.example.com   # DNS+TCP+TLS+HTTP 完整验证
```

> 注意：`Ubuntu 24.04` 是**发行版版本**，`6.x.x` 是**内核版本**，实验报告中不要把两者混为一谈。

### 2.3 快照要求

- 实验开始前创建快照：`实验一-开始基线`
- 实验完成后创建快照：`实验一-完成基线`

目的不是“装成功/做成功”，而是**可回滚、可复现**。

---

## 三、实验准备：建立自己的工作区

**场景：** 运维工程师不会在 `/tmp` 或家目录根目录里随手堆放工作文件。先建立规范的工作区，再开始操作。

```bash
mkdir -p ~/linux-lab01/{basic,files,vi,grep,groups,perm,logs,backup}
cd ~/linux-lab01
pwd
```

检查目录结构（无 `tree` 命令时用 `find` 代替）：

```bash
tree ~/linux-lab01
# 或者
find ~/linux-lab01 -maxdepth 2 -print
```

**记录实验环境**——不要等出了问题才想起来“当时系统是什么版本”：

```bash
{
    echo "===== 实验开始 ====="
    date
    hostname
    whoami
    pwd
    uname -r
    cat /etc/os-release
} | tee ~/linux-lab01/logs/environment.txt
```

后续每个任务的关键输出，建议都用 `命令 | tee ~/linux-lab01/logs/xxx.txt` 留存，作为实验报告的证据材料。

---

## 四、实验内容及操作步骤

> 全实验共 8 个任务模块（A～H）＋ 1 个综合实战 ＋ 进阶选做。
> **课内 4 学时完成 A～D 与 E、F、G 的核心部分；其余课后完成。** 各模块标注【场景】即实际运维工作中的典型情形。

### 任务A 系统登录、Shell 与系统信息（入职第一天：先摸清这台机器）

**【场景】** 你第一天入职，领到一台 Linux 服务器的账号。做任何改动之前，先回答：我是谁？我在用什么 Shell？这台机器开了多久？现在几点？

#### A-1 查看当前用户与身份

```bash
whoami
id
id -u; id -g; id -G
groups
```

观察并记录：用户名、UID、主要组、附加组。

#### A-2 确认当前 Shell 类型

```bash
echo "$SHELL"                        # 账户配置的登录 Shell
ps -p $$ -o pid,ppid,comm,args       # 当前正在运行的 Shell 进程
echo "$0"
grep "^$(whoami):" /etc/passwd       # /etc/passwd 最后一个字段
```

> 想一想：`/etc/passwd` 记录的是**登录 Shell 配置**，不一定等于**当前正在运行的 Shell**（例如登录后又手工执行了 `bash`）。两者有什么区别？

#### A-3 注销、关机与重启

```bash
exit                    # 注销当前会话
logout                  # 登录 Shell 的注销
sudo systemctl poweroff # 关机（systemd 发行版）
sudo systemctl reboot   # 重启
```

> 实验环境中不要随意关机/重启，仅在教师要求验证时执行。可用 `help exit`、`help logout` 比较两条内建命令的区别。

#### A-4 查看系统运行状态与时间

```bash
who            # 当前登录用户
w              # 更详细的登录信息
uptime         # 开机时长与负载
date           # 日期时间
timedatectl    # 时区与校时状态
```

> 想一想：为什么 `w` 显示的信息通常比 `who` 更多？

#### A-5 获取命令帮助（必须掌握的自助能力）

```bash
man ls              # 手册页，/keyword 搜索，q 退出
ls --help           # 命令自带帮助
help cd             # Bash 内建命令帮助
apropos password    # 不知道命令名、只知道功能时搜索
info coreutils      # GNU 工具 info 文档（选看）
```

**要求：** 任选 3 个命令（如 `ls`、`chmod`、`systemctl`），分别用 `man` 和 `--help` 查看，在报告中比较两种帮助信息的特点。

---

### 任务B 账号与进程管理（新员工入职、老员工离职）

**【场景】** HR 通知：新员工 labuser1 入职，需要开通账号；某员工休假需临时冻结账号；某员工离职需注销账号并清理家目录。你作为管理员逐一办理。

#### B-1 创建用户并设置密码

```bash
sudo useradd -m -s /bin/bash labuser1
sudo passwd labuser1
id labuser1
grep "^labuser1:" /etc/passwd
```

`useradd` 常用选项：

```text
-c comment    用户说明        -g group     指定主要组
-d directory  指定家目录      -G groups    指定附加组
-m            创建家目录      -s shell     指定登录 Shell
-u uid        指定 UID
```

练习创建带完整说明的用户：

```bash
sudo useradd -m -c "Linux Lab User" -s /bin/bash labuser2
sudo passwd labuser2
```

> 想一想：创建用户时不指定 `-d`，家目录在哪里？不要背答案，用 `getent passwd labuser1` 和 `eval echo ~labuser1` 在本机验证。

#### B-2 锁定与解锁账户（员工休假场景）

```bash
sudo passwd -l labuser1      # 锁定（等价：sudo usermod -L labuser1）
sudo passwd -S labuser1      # 查看账户状态
sudo passwd -u labuser1      # 解锁（等价：sudo usermod -U labuser1）
```

> 想一想：企业运维中锁定账户后为什么还要用 `passwd -S`、`getent passwd` 再确认一次？

#### B-3 修改用户信息与改名

```bash
sudo usermod -s /bin/bash labuser1                 # 改 Shell
sudo usermod -d /home/labuser1-new -m labuser1     # 改家目录并迁移内容
sudo usermod -l labstudent labuser1                # 改名：usermod -l 新名 旧名
id labstudent
```

> 想一想：改名后 UID、主要组、附加组会不会自动跟着变？用 `id labstudent`、`getent group` 验证，不要猜。
> 注：`usermod -u` 修改 UID 属于高风险操作，本实验只观察帮助信息，不对在用账户操作。

#### B-4 删除用户（员工离职场景）

删除前先检查该用户是否还有业务文件：

```bash
id labuser1
getent passwd labuser1
ls -ld /home/labuser1
```

删除账户（保留家目录）或连家目录一起删除：

```bash
sudo userdel labuser1       # 仅删账户
sudo userdel -r labuser1    # 连同家目录一起删除
```

> 注意：`-r` 并不能保证把该用户在系统其他位置的所有文件“一一找出并删除”，实际行为与文件归属、挂载点有关。工程环境中删账户前必须先确认其名下是否仍有业务文件。

#### B-5 进程查看与管理

```bash
ps                 # 当前终端的进程
ps -u "$USER"      # 当前用户的全部进程
ps aux 或 ps -ef   # 系统所有进程
pstree             # 进程树
top                # 实时进程（q 退出）
```

动手实验——启动一个后台进程再结束它：

```bash
sleep 300 &
jobs
ps -ef | grep '[s]leep'
kill %1
jobs
```

> 想一想：① `ps` 与 `top` 的区别？② `grep '[s]leep'` 为什么不会把 grep 进程自己匹配出来？

---

### 任务C 文件系统日常管理（日常文件整理、备份与归档）

**【场景】** 你需要在服务器上整理项目文件：建目录、写文件、看日志、找文件、做备份。以下操作在 `~/linux-lab01` 工作区内完成。

#### C-1 创建与查看文件

```bash
pwd
touch file1                  # 创建空文件/更新时间戳
echo "hello" > file2         # 重定向创建
cat > file3                  # 输入几行后 Ctrl+D 结束
ls -l file1; stat file1
```

查看文件内容，按场景选工具：

```bash
cat -n file1            # 短文件、带行号
less /etc/passwd        # 长文件分页：Space 翻页，/关键字 搜索，q 退出
more /etc/passwd        # 简单分页
head -n 5 file1         # 看开头
tail -n 5 file1         # 看末尾
tail -f logfile         # 实时跟踪日志（运维看日志的常用姿势）
```

显示指定行：

```bash
sed -n '10p' file            # 第 10 行
sed -n '10,20p' file         # 第 10～20 行
awk 'NR>=10 && NR<=20' file  # 等效写法
```

#### C-2 复制、移动、删除

```bash
cp file1 file2                    # 复制文件
cp file1 ~/linux-lab01/files/     # 复制到目录
cp -r dir1 dir2                   # 递归复制目录
cp -a dir1 dir2                   # 保留属性的递归复制（备份常用）
mv file1 file2                    # 重命名（Linux 中移动和重命名都是 mv）
mv dir1 ~/linux-lab01/files/      # 移动目录
rm file1                          # 删除文件
rmdir dir1                        # 删除空目录
rm -ri dir1                       # 删除非空目录（-i 逐个确认，练习时必加）
```

> **危险操作红线：执行 `rm -r`、`rm -rf` 前必须先 `pwd` 确认当前位置，看清目标路径。禁止执行 `sudo rm -rf /*` 之类的命令。**

#### C-3 创建目录树（综合练习）

```bash
mkdir -p project/{docs,src,backup,logs}
touch project/docs/{README.txt,design.txt}
touch project/src/{main.c,test.c}
touch project/logs/app.log
find project -print
```

#### C-4 绝对路径与相对路径

```text
.    当前目录      ..   父目录      ~   家目录      /   根目录
```

```bash
cd ~/linux-lab01
cd files
cd ..
cd ./files
cd ~
```

**情景题：** 不修改当前路径，如何把 `~/A/data` 目录连同内容复制到 `~/B/` 下？

```bash
cp -r ~/A ~/B/
find ~/B -maxdepth 3 -print    # 验证
```

#### C-5 which、whereis、find 的区别

```bash
which ls                         # PATH 中将执行哪个命令
whereis ls                       # 二进制/源码/手册的位置
find /etc -name "passwd"         # 在目录树中按条件查找
find . -name "f*"                # f 开头的文件
find . -name "*.c"               # C 源程序文件
find . -type f -size +1M         # 大于 1MB 的普通文件
find . -type f -mtime -1         # 最近 1 天修改过的文件
find . -type f -perm 0644        # 按权限查找
find . -type f -name "*.log" -mtime -7   # 组合条件
```

**运维场景练习：** ① 找出工作区中最近 24 小时产生的 `.log` 文件；② 找出大于 5MB 的普通文件。

> 想一想：`which passwd` 和 `find / -name passwd` 的结果为什么可能完全不同？

#### C-6 tar 归档与备份

```bash
mkdir -p ~/linux-lab01/backup/demo
echo "backup test" > ~/linux-lab01/backup/demo/a.txt
tar -cf demo.tar -C ~/linux-lab01/backup demo      # 打包
tar -tf demo.tar                                   # 查看内容
mkdir ~/linux-lab01/backup/extract
tar -xf demo.tar -C ~/linux-lab01/backup/extract   # 解包
tar -czf demo.tar.gz -C ~/linux-lab01/backup demo  # 打包+gzip 压缩
```

常用选项：`-c` 创建、`-x` 解档、`-t` 列表、`-f` 指定文件、`-z` gzip、`-j` bzip2、`-J` xz、`-v` 显示过程。

**运维习惯——修改配置前先备份，文件名带时间戳：**

```bash
tar -czf config-backup-$(date +%F-%H%M%S).tar.gz ~/linux-lab01/files
```

#### C-7 dd 命令（仅限测试文件）

```bash
dd if=/dev/zero of=~/linux-lab01/files/test.img bs=1M count=10 status=progress
ls -lh ~/linux-lab01/files/test.img
```

`if` 指定输入文件，`/dev/zero` 可持续提供零字节，常用于生成测试文件。`of` 输出文件、`bs` 块大小、`count` 块数。

> **红线：`dd` 的 `of=` 禁止指向 `/dev/sda` 等真实磁盘设备。** `dd` 按块复制、可操作设备与镜像，这正是它区别于 `cp`（文件级复制）、`mv`（移动/重命名）的地方，也是它危险的原因。

#### C-8 多行命令与命令块

```bash
mkdir -p ~/linux-lab01/test \
    && echo "directory created" \
    && date && pwd

{
    echo "===== report ====="
    date; hostname; whoami; uname -r
} > ~/linux-lab01/logs/report.txt
```

#### C-9 硬链接与符号链接

```bash
cd ~/linux-lab01/files
echo "hello link" > original.txt
ln original.txt hardlink.txt     # 硬链接
ln -s original.txt symlink.txt   # 符号链接
ls -li                           # 重点观察 inode 号
rm original.txt                  # 删除原文件
cat hardlink.txt                 # 仍可读取
cat symlink.txt                  # 悬空链接，报错
readlink symlink.txt
```

> 结论：硬链接与原文件共享 inode；符号链接只保存目标路径。删除原文件后，硬链接仍可访问数据，符号链接失效。

---

### 任务D vi/vim 编辑器（纯终端环境下修改配置）

**【场景】** 生产服务器没有图形界面，唯一可靠的编辑器是 vi/vim。网络组同事需要你修改一份配置文件并保留修改痕迹。

#### D-1 三种工作模式

1. **普通（命令）模式**：打开文件后的默认模式，用于移动、删除、复制；
2. **插入模式**：按 `i`/`a`/`o` 等进入，用于输入文本，`Esc` 返回普通模式；
3. **末行（命令行）模式**：按 `:` 进入，用于保存、退出、替换等。

```bash
vim ~/linux-lab01/vi/test.txt
```

输入以下内容后练习保存与退出：

```text
Linux system administration
File permission
User management
Network troubleshooting
```

```text
i 进入插入 → 输入内容 → Esc 返回普通模式
:w 保存   :wq 保存退出   :q! 不保存强制退出
```

#### D-2 常用操作速查

| 类别 | 按键 |
|---|---|
| 光标移动 | `h j k l`（或方向键）；滚屏 `Ctrl+F/B/D/U` |
| 跳转 | `nG` 跳到第 n 行（如 `10G`） |
| 删除 | `x` 删字符；`dw` 删词；`dd` 删整行；`D` 删到行尾；前加数字表示重复，如 `3dd` |
| 插入/追加 | `i` 光标前；`I` 行首；`a` 光标后；`A` 行尾；`o` 下一行新开；`O` 上一行新开 |
| 复制粘贴 | `yy` 复制当前行；`3yy` 复制 3 行；`p` 粘贴到之后；`P` 粘贴到之前 |
| 读入文件 | `:r /etc/hostname` |
| 查找替换 | `/关键字` 查找、`n` 下一个；`:s/old/new/` 替换当前行；`:%s/old/new/g` 全文替换 |

#### D-3 综合练习

创建 `~/linux-lab01/vi/config.txt`，完成：

1. 输入 20 行文本；
2. 删除第 5 行；
3. 复制第 10～12 行；
4. 将第 1 行移动到文件尾部；
5. 查找某个单词并全文替换；
6. 用 `:r` 读入 `/etc/hostname` 的内容；
7. 保存退出后重新打开验证。

> 想一想：`dd` 与 `D`、`p` 与 `P` 分别有什么不同？vi 的三种模式与 gedit 这类图形编辑器的工作方式有何本质区别？

---

### 任务E grep 与正则表达式（日志排查）

**【场景】** 业务方反馈“系统昨天下午报错”。面对上万行日志，肉眼翻页不现实——grep ＋ 正则表达式是运维的第一件武器。

#### E-1 准备测试数据

```bash
cat > ~/linux-lab01/grep/data.txt <<'EOF'
Linux is powerful.
Linux system administration.
user01 192.168.1.10
user02 192.168.1.20
server01 10.0.0.10
server02 10.0.0.20
ERROR disk full
WARNING high memory
INFO service started
file01.txt
file02.txt
main.c
test.c
README.md
EOF
```

#### E-2 grep 基本用法

```bash
grep "Linux" data.txt        # 查找
grep -i "linux" data.txt     # 忽略大小写
grep -n "Linux" data.txt     # 显示行号
grep -v "Linux" data.txt     # 反向匹配（显示不含的行）
grep -c "Linux" data.txt     # 统计匹配行数
grep -R "ERROR" ~/linux-lab01  # 递归搜索目录
```

#### E-3 正则表达式基础

```bash
grep "user.." data.txt                               # . 匹配任意单个字符
grep '^server' data.txt                              # ^ 行首
grep '\.c$' data.txt                                 # $ 行尾，\. 转义点号
grep '^user[0-9]' data.txt                           # [0-9] 字符集合
grep -E 'user[0-9]+' data.txt                        # + 一次或多次（扩展正则 -E）
grep -E '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' data.txt    # 匹配 IPv4 的简单形式
```

重复符：`*` 零次或多次、`+` 一次或多次、`?` 零次或一次、`{n}`、`{n,m}`（后四种需 `-E`）。

#### E-4 grep 与 pgrep 的对比

```bash
ps -ef | grep ssh    # 在文本/进程列表中过滤
pgrep -a ssh         # 直接按进程名查 PID
```

> 想一想：什么时候用 `grep`，什么时候用 `pgrep`？

#### E-5 日志分析实战

```bash
cat > ~/linux-lab01/logs/app.log <<'EOF'
2026-09-04 10:01:01 INFO service started
2026-09-04 10:02:10 INFO user login
2026-09-04 10:03:15 WARNING memory high
2026-09-04 10:04:01 ERROR database timeout
2026-09-04 10:05:20 INFO retry
2026-09-04 10:06:30 ERROR connection refused
EOF
```

完成以下分析，**全部用命令得出，不允许手工数**：

```bash
grep -c 'ERROR' app.log                            # ERROR 出现几次？
grep -n 'ERROR' app.log                            # 第一次 ERROR 在第几行？
grep -E 'WARNING|ERROR' app.log                    # 只看告警和错误
grep -oE 'INFO|WARNING|ERROR' app.log | sort | uniq -c   # 各级别统计
grep 'ERROR' app.log | tail -n 1                   # 最后一条 ERROR 的时间
```

把巡检结果存成报告（“人工自动化”的第一步）：

```bash
{
    echo "===== LOG CHECK ====="; date
    echo "ERROR COUNT:"; grep -c 'ERROR' app.log
    echo "LEVEL SUMMARY:"; grep -oE 'INFO|WARNING|ERROR' app.log | sort | uniq -c
    echo "LAST ERROR:"; grep 'ERROR' app.log | tail -n 1
} | tee ~/linux-lab01/logs/log-report.txt
```

---

### 任务F 用户组管理（组建项目协作组）

**【场景】** 公司成立 devops 项目组，需要创建组、把成员拉进组、指定一名组管理员负责日常成员管理。

#### F-1 组的查看与创建

```bash
groups                     # 当前用户所属组
id labuser1                # 指定用户的组信息
getent group               # 系统组数据库
sudo groupadd devops       # 创建组
getent group devops
```

#### F-2 组成员管理

```bash
sudo usermod -aG devops labuser1     # 追加附加组（推荐）
id labuser1                          # 验证
sudo gpasswd -a labuser1 devops      # 另一种添加方式
sudo gpasswd -d labuser1 devops      # 从组中移除
sudo gpasswd -A labadmin devops      # 指定组管理员
```

> **极其重要：`usermod -G devops labuser1`（不带 -a）会重设该用户的全部附加组列表，可能把人从其他组里“挤出去”；`-aG` 才是“追加”。**
>
> 想一想：`gpasswd -a` 与 `usermod -aG` 都能加人，管理入口与使用场景有何不同？`-g`（主要组）与 `-G`（附加组）有何区别？

#### F-3 组的修改与删除

```bash
sudo groupmod -n newgroup oldgroup   # 改组名
sudo groupdel devops                 # 删除组
```

> 删除组前先用 `getent group devops`、`id <用户名>` 确认没有用户把它当作**主要组**——主要组不能直接删除。

#### F-4 /etc/group 与 /etc/gshadow

```bash
cat /etc/group          # 组名:x:GID:成员列表
sudo cat /etc/gshadow   # 组密码、组管理员、组成员
```

> 想一想：能否直接编辑 `/etc/group` 创建新组？技术上文件是文本可以编辑，但工程实践中应使用 `groupadd`/`gpasswd` 等命令——它们保证格式正确、逻辑一致、便于审计。

#### F-5 chgrp 与 ACL（选做）

```bash
sudo chgrp devops file.txt           # 修改文件所属组
sudo chgrp -R devops project/        # 递归
getfacl file.txt                     # 查看 ACL
setfacl -m u:labuser1:r file.txt     # 给指定用户单独加读权限
setfacl -x u:labuser1 file.txt       # 删除该条 ACL
```

> ACL 的适用场景：文件属于 alice、组是 devops，但 bob 需要额外读取——不必改所有者、不必把 bob 拉进 devops，一条 ACL 即可。这就是“owner/group/other 三类权限表达不了”时的补充手段。

---

### 任务G 文件权限与特殊权限（权限故障分析）

**【场景】** 同事抱怨“这个文件我明明删不掉/进不去这个目录”。理解权限模型，才能解释这些“明明”。

#### G-1 看懂权限

```bash
ls -l
```

```text
-rw-r-----x 1 alice devops 123 Sep 4 10:00 file.txt
│└┬┘└┬┘└┬┘
│ │  │  └── 其他用户(other)
│ │  └───── 所属组(group)
│ └──────── 所有者(owner)
└────────── 文件类型：- 普通文件 d 目录 l 符号链接 c 字符设备 b 块设备 p 管道 s socket
```

数字表示法：`r=4，w=2，x=1`，故 `rwx=7`、`rw-=6`、`r-x=5`、`---=0`。

```bash
chmod 640 filea                    # 数字方式：owner=rw- group=r-- other=---
chmod u+x script.sh                # 符号方式
chmod g+w file.txt
chmod u=rwx,g=rx,o= file           # 组合
```

#### G-2 chown 与权限边界

```bash
sudo chown labuser1 file1                  # 改所有者
sudo chown labuser1:devops file1           # 同时改所有者和组
touch own.txt
chown labuser1 own.txt                     # 普通用户执行，观察报错
```

> 结论：普通用户**不能**把自己拥有的文件随意“送”给别的用户，`chown` 通常需要管理员权限。

#### G-3 umask 与默认权限

```bash
umask                    # 查看当前值（如 0022）
mkdir dirx && ls -ld dirx
umask 027
mkdir diry && ls -ld diry   # 对比 dirx 与 diry 的权限差异
```

文件默认基础权限按 666、目录按 777 计算，`umask` 屏蔽对应位。想一想：为什么文件和目录的基础权限不同？

#### G-4 目录权限的真实含义（重点实验）

对目录而言：

```text
r   可以列出目录项（ls）
w   可以在目录中增、删、改名
x   可以进入/遍历该目录、访问已知名称的对象
```

搭建验证环境：

```bash
mkdir -p ~/linux-lab01/perm/testdir
echo "secret" > ~/linux-lab01/perm/testdir/secret.txt
chmod 400 ~/linux-lab01/perm/testdir/secret.txt
chmod 700 ~/linux-lab01/perm/testdir
sudo useradd -m -s /bin/bash permuser && sudo passwd permuser
su - permuser        # 或 sudo -iu permuser，切换到测试用户验证
```

设计并验证（报告中不能只写“可以/不可以”，必须解释原因）：

1. 用户对文件无 `r`，但对目录有 `x`：能否 `cat` 该文件？
2. 用户对文件无 `w`，但对所在目录有 `wx`：能否修改该文件？能否**删除**该文件？能否新建文件？
3. 设计一个目录：使指定用户可以向其中写入已知文件名的文件，但不能列出目录内容、不能删除文件。（提示：`r` 与 `w` 分开控制）

> 核心结论：删除文件修改的是**目录项**，所以删除能力取决于**目录的 w 权限**，而不是文件自身的 w 权限。

#### G-5 特殊权限

```bash
# sticky bit：共享目录防互删（/tmp 即 drwxrwxrwt）
mkdir ~/linux-lab01/perm/share
chmod 1777 ~/linux-lab01/perm/share && ls -ld ~/linux-lab01/perm/share

# setgid 目录：新文件继承目录的组（团队共享目录常用）
mkdir ~/linux-lab01/perm/team
chmod 2770 ~/linux-lab01/perm/team
sudo chgrp devops ~/linux-lab01/perm/team && ls -ld ~/linux-lab01/perm/team

# setuid：观察系统现有 setuid 程序，仅在自己的测试文件上练习设置/取消
find /usr/bin -perm -4000 -type f 2>/dev/null
chmod u+s testfile && ls -l testfile
chmod u-s testfile
```

> 想一想：为什么 setuid 是高风险权限？为什么共享目录要用 sticky bit？

---

### 任务H systemd 与网络检查（服务巡检与网络排障）

**【场景】** 例行巡检：确认系统服务正常、网络可用。业务方报障“机器昨天还能上网，今天不通了”。

#### H-1 systemd 服务管理

```bash
systemctl                                   # 概览
systemctl status ssh                        # 查看服务（服务名因发行版而异）
systemctl list-unit-files --type=service    # 列出所有服务单元
systemctl --type=service --state=running    # 正在运行的服务
sudo systemctl start ssh                    # 启动
sudo systemctl restart ssh                  # 重启
sudo systemctl enable ssh                   # 开机自启
systemctl is-enabled ssh                    # 是否自启
systemctl is-active ssh                     # 是否运行中
systemctl --failed                          # 失败的服务
```

> 想一想：`enable` 和 `start` 是一回事吗？（`enabled + inactive` 的状态是可能存在的。）
> 注意：不要为练习随意停止 NetworkManager、图形界面等关键服务。

#### H-2 主机名与网络信息

```bash
hostnamectl                                 # 查看主机名
sudo hostnamectl set-hostname linux-lab01   # 修改主机名
nmcli device status                         # 网卡状态
nmcli connection show --active              # 活动连接（现代发行版用 NetworkManager）
ip -br addr; ip route; resolvectl status    # IP / 路由 / DNS
```

> 命名规范思考：`server` 与 `lab-web-01` 哪个主机名更适合批量管理和排障？为什么？

#### H-3 NAT 无法上网：分层排障演练

不要一看“不能上网”就改 IP。按四层定位：

```text
第1层 宿主机：宿主机自己能上网吗？
第2层 VMware：虚拟网络编辑器中 VMnet8 的 NAT/DHCP 是否启用、网段是否一致？
第3层 虚拟机：ip -br addr 有无 IP？ip route 有无 default 默认路由？resolvectl status 的 DNS 是否正常？
第4层 服务验证：ping -c 4 223.5.5.5（IP 连通性）→ curl -I https://www.example.com（DNS+TCP+TLS+HTTP）
```

> 为什么不能只用 `ping` 验证网络？`ping` 只验证 IP/ICMP 路径；`curl https://域名` 同时覆盖 DNS、TCP、TLS、HTTP，才是完整验证。

#### H-4 信息采集（交接演练，禁止直接改配置）

业务方只说“昨天还能访问网络，今天不通了，你先看看”。第一阶段只做信息采集：

```bash
{
    hostnamectl; cat /etc/os-release; uname -r
    uptime; who; id
    ip -br addr; ip route; resolvectl status
    systemctl --failed
} | tee ~/linux-lab01/logs/incident-baseline.txt
ping -c 4 223.5.5.5
curl -I https://www.example.com
```

根据证据判断故障属于哪一类，报告中必须给出“现象 → 证据 → 判断 → 下一步验证”的完整链条：

```text
A. 网卡没有 IP        B. 有 IP 但没有默认路由    C. IP 能通但域名不能解析
D. DNS 正常但 HTTPS 失败    E. 网络正常，服务自身故障
```

---

### 综合实战：新服务器交付验收（必做，现场考核内容）

**【场景】** 你负责交付一台新的 Linux 虚拟机给项目组，交付单要求以下各项均可复查。完成全部项目并生成证据文件。

| # | 验收项 | 要求 |
|---|---|---|
| 1 | 系统信息 | `cat /etc/os-release`、`uname -r`、`hostnamectl` 输出正确，内核/发行版版本表述无误 |
| 2 | 存储 | `lsblk -f`、`df -h` 输出留存 |
| 3 | 用户 | 创建 `opsuser` 并设置密码 |
| 4 | 用户组 | 创建 `ops` 组并将 `opsuser` 加入（附加组方式） |
| 5 | 共享目录 | 创建 `~/ops-share`，使 ops 组成员可协作（综合运用组权限，可参考进阶实验一） |
| 6 | 网络 | `ip -br addr`、`ip route`、`resolvectl status` 正常；`ping -c 4 223.5.5.5` 与 `curl -I https://www.example.com` 均成功 |
| 7 | 服务 | `systemctl --failed` 无异常；任选一个安全服务完成 `status/start/restart/is-active/is-enabled` 全流程 |
| 8 | 交付报告 | 生成 `~/linux-lab01/logs/delivery-report.txt`，含时间、主机名、系统版本、内核、用户、磁盘、IP、路由、DNS、服务状态、网络测试结果 |
| 9 | 快照 | 创建 `实验一-交付验收基线` 快照 |

证据文件参考生成方式：

```bash
{
    echo "===== LINUX LAB 01 FINAL EVIDENCE ====="
    date; hostnamectl; whoami; id
    cat /etc/os-release; uname -r
    free -h; lsblk -f; df -h
    ip -br addr; ip route; resolvectl status
    systemctl --failed
} | tee ~/linux-lab01/logs/final-evidence.txt
```

---

### 进阶选做（学有余力者完成，可获加分）

1. **最小权限共享目录**：建 devops 组（alice、bob 在内，charlie 在外），创建 `/opt/devops`，要求组内可协作读写、charlie 不可写、新文件继承 devops 组、成员不能互删文件。（组合运用 `groupadd`、`usermod -aG`、`chgrp`、`chmod 2770`+sticky、`umask`）
2. **磁盘空间告警分析**：`df -h` 与 `du -sh` 对比；用 `dd` 造一个 100MB 文件、`tail -f` 占用后 `rm` 删除，观察 `df -h` 并用 `lsof +L1` 解释“文件删了空间没释放”。
3. **异常权限文件扫描**：`find ~/linux-lab01 -perm /6000 -ls`、`-perm -0002` 查找特殊权限与 world-writable 文件/目录。
4. **配置备份与回滚演练**：建立 `app.conf` → tar 带时间戳备份 → 故意改错 → 从备份恢复 → 用 `grep` 验证。说明为什么改生产配置前必须备份、备份名为什么带时间、回滚为什么必须验证。
5. **安装介质可信性调查**：记录所用 ISO 的发行版、版本、架构、文件名、SHA256、来源、获取时间；用 `sha256sum` 校验，说明“能安装成功”为什么不能证明镜像可信。
6. **EOL 与生命周期调查**：任选两个发行版/版本，查官方资料填写版本、架构、包管理器、生命周期、EOL、安全公告地址、适用场景，注明来源与访问日期。
7. **批量交付方案设计**：若需安装 100 台 Linux，从镜像来源与可信性、统一分区、主机名/网络/用户/软件的自动化配置、结果验证、失败回滚九个方面设计方案（了解 Kickstart/Preseed/autoinstall、cloud-init、Ansible、GitOps 的定位）。

---

## 五、实验注意事项（安全纪律）

1. **只在虚拟机中练习。** 本实验涉及用户删除、权限修改、文件删除、服务操作，禁止在宿主机或他人机器上操作。
2. **危险命令先确认对象。** 执行 `rm -r`、`userdel -r`、`chmod`、`chown`、`dd` 前必须先 `pwd`、看清路径。严禁 `sudo rm -rf /*`。
3. **`dd` 的 `of=` 只允许指向测试文件**，禁止指向 `/dev/sda` 等真实磁盘设备。
4. **先快照，后实验**；关键阶段（工作区建立、综合实战前）各留一个快照。
5. **不要随意停止关键服务**（NetworkManager、sshd、图形桌面等），不要修改自己登录账户的 UID 与 Shell。
6. 命令应亲自执行；引用外部资料（版本号、EOL 日期等）时注明来源、日期与适用范围，以本机 `man` 和官方文档为准。

---

## 六、思考题

### 基础思考题（实验报告中选答不少于 8 题）

1. 如何得到当前 Shell 类型？`/etc/passwd` 中最后一个字段是什么？当前运行的 Shell 与登录 Shell 有什么区别？
2. `who`、`w`、`id`、`whoami` 有什么区别？
3. `ps` 与 `top` 有什么区别？
4. 创建用户时不指定 `-d`，家目录在哪里？（要求给出验证命令）
5. `usermod -l` 改名后，用户组会自动变化吗？为什么？
6. `cp`、`mv`、`dd` 分别适合什么场景？`dd if=/dev/zero of=file` 中 `if` 的作用是什么？
7. `which`、`whereis`、`find` 有什么区别？
8. 硬链接与符号链接有什么区别？删除原文件后各有何表现？
9. `chmod 640 file` 表示什么？`umask 027` 会影响什么？
10. `usermod -g`、`usermod -G`、`usermod -aG` 有什么区别？`gpasswd -a` 与 `usermod -aG` 有什么区别？
11. 目录的 `x` 权限是什么含义？文件没有写权限，为什么仍可能被删除？
12. sticky bit 的作用是什么？setgid 目录为什么适合团队共享？
13. ACL 解决了传统权限模型中的什么问题？
14. `systemctl enable` 和 `systemctl start` 有什么区别？
15. 为什么现代系统应优先学习 NetworkManager/nmcli，而不能只会旧的 network-scripts？
16. 为什么网络排障不能一上来就改 IP？

### 进阶思考题（选答）

1. **权限设计**：某目录要求 alice 读写、devops 组读写、其他用户无权限、新文件继承 devops 组、成员不能互删文件。给出 owner/group/chmod/setgid/sticky/umask 的完整设计与验证步骤。
2. **网络故障**：某虚拟机 `ip addr` 显示有 IP，`ping 223.5.5.5` 成功，但 `curl -I https://www.example.com` 失败。列出至少三个可能原因并设计验证顺序。
3. **服务故障**：某服务 `is-enabled` 返回 enabled 但 `is-active` 返回 inactive。这意味着什么？应执行哪些命令继续排查？
4. **磁盘空间**：`df -h` 显示根分区 95%，但 `du -sh /*` 合计没有那么大。列出可能原因与下一步排查命令。
5. **批量交付**：比较手工安装、镜像克隆、cloud-init、Ansible、GitOps 在一致性、重复性、审计、回滚、维护成本五个维度上的差异。

---

## 七、实验报告要求

实验报告应包含：**实验目的、实验环境、实验内容与步骤、实验结果（输出证据）、结果分析与思考** 五个部分。

1. **实验目的**：不得照抄指导书，用自己的话写“通过本实验掌握/验证/理解了什么”。
2. **实验环境**：记录宿主机、虚拟化软件、发行版及版本、内核、架构、CPU、内存、磁盘、网络模式（附 `hostnamectl`、`cat /etc/os-release`、`uname -r`、`lsblk -f`、`ip -br addr` 等输出）。
3. **实验内容与步骤**：按任务模块写清操作顺序（如：创建用户 → 设置密码 → 查看 UID/GID → 加入附加组 → 验证成员关系），关键命令必须给出。
4. **实验结果（输出证据）**：每个重要实验提供“命令 → 输出 → 结果判断”。**推荐用 `命令 | tee 文件` 保存的文本作为证据，而不是只有截图。**
5. **分析与思考**：完成所选思考题；对权限实验、排障实验必须给出解释（为什么），不能只写“可以/不可以”。
6. 报告末尾附：最终证据文件（`final-evidence.txt`）内容、实验验收清单自查结果。

> 如实记录实验过程，包括失败的尝试和纠正过程——这正是“判断与改进”部分的评分依据。

---

## 八、实验验收清单（现场检查用）

### 基础部分

- [ ] 能解释当前 Shell 类型（两种以上方法）
- [ ] 能使用 `pwd`、`whoami`、`id`、`who/w`、`date`
- [ ] 能使用 `man`、`--help`、`apropos` 获取帮助
- [ ] 能创建/修改/锁定/解锁/删除用户，并解释主要组与附加组
- [ ] 能查看并管理进程（`ps`、`top`、`kill`）
- [ ] 能创建/删除目录，复制/移动/删除文件
- [ ] 能分页查看文件、显示指定行
- [ ] 能使用 `find` 按名称/类型/时间/大小查找，能说明与 `which`/`whereis` 的区别
- [ ] 能使用 `tar` 归档/解档，理解 `dd` 的用途与风险
- [ ] 能说明硬链接与符号链接的区别
- [ ] 能用 vim 完成基本编辑（三种模式、删除/复制/粘贴/查找替换）
- [ ] 能用 `grep`＋正则完成日志检索与统计
- [ ] 能创建/删除组、管理组成员
- [ ] 能使用 `chmod`/`chown`/`chgrp`，能解释 `umask`
- [ ] 能解释目录 r/w/x 的含义并完成权限验证实验
- [ ] 能解释 setuid/setgid/sticky，了解 ACL

### 现代运维部分

- [ ] 能区分内核版本与发行版版本
- [ ] 能使用 `hostnamectl`、`systemctl`（含 enable/start 区别）
- [ ] 能使用 `nmcli`、`ip addr`、`ip route`、`resolvectl`
- [ ] 能按四层流程完成 NAT 网络排障
- [ ] 能记录系统基线、保留命令输出证据
- [ ] 完成综合实战：新服务器交付验收（9 项）
- [ ] 已创建“实验一-完成基线”快照

---

> 本指导书与第一周课程配套使用。遇到发行版/版本差异时，以本机 `man` 手册、发行版官方文档为准。
