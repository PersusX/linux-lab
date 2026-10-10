# 《Linux系统及应用》课程实验指导书（学生用·2026版）

# 实验六 Linux 服务器与存储管理

| 项目 | 内容 |
|---|---|
| 课程名称 | Linux 系统及应用 |
| 实验定位 | 课外选做，不计入 16 个课内实验学时 |
| 建议投入 | 基础路线约 4 小时；存储挑战另计 2～4 小时 |
| 适用环境 | Ubuntu Server 24.04 LTS 虚拟机，主机与实验机使用 NAT 或主机专用网络 |
| 前置要求 | 已完成实验一，了解用户、权限、systemd、文件系统与基本网络命令 |
| 成果形式 | 配置备份、验证记录、故障排查记录与实验报告 |

> 本实验涵盖文件系统挂载、软件包管理、SSH、Samba/DHCP 以及 LVM、配额、RAID 等内容，远超一次实验可以合理完成的范围，因此采用模块化路线：基础模块必做，服务模块和存储挑战按兴趣选做，不需要一次完成全部服务。

---

## 一、学习目标

1. 理解块设备、文件系统、挂载点和 `/etc/fstab` 的关系；
2. 会用 APT/DPKG 查询、安装、更新、删除和验证 Ubuntu 软件包；
3. 会用 `systemctl`、`journalctl`、`ss` 和配置检查工具管理服务；
4. 能在隔离实验网络中配置并验证 OpenSSH；
5. 能从 Samba 或 DHCP 中选择一项完成服务配置；
6. 了解 LVM、磁盘配额和软件 RAID 的适用场景与风险；
7. 形成“备份 → 修改 → 语法检查 → 重载 → 本机验证 → 客户端验证 → 恢复”的服务管理流程。

---

## 二、安全边界与实验路线

### 2.1 强制安全要求

1. 使用专用虚拟机，开始前创建快照 `lab06-start`；
2. 服务器网卡使用 NAT 或主机专用网络，不直接暴露到公网；
3. 不在主机的真实系统磁盘上执行 `mkfs`、`pvcreate`、`mdadm --create`；
4. 存储挑战只操作新增的空白虚拟磁盘，必须先用 `lsblk -f` 和虚拟机配置双重确认设备；
5. 修改 `/etc` 下的文件前创建带日期的备份，实验报告记录恢复命令；
6. 不启用 Telnet、SMB1 或匿名可写 FTP；密码不写入报告、截图、Shell 历史或可公开配置文件。

### 2.2 推荐路线

| 模块 | 内容 | 定位 |
|---|---|---|
| A | 挂载、`fstab`、APT/DPKG | 基础必做 |
| B | OpenSSH 服务 | 基础必做 |
| C | Samba 身份验证共享 | 服务选做，与 D 二选一 |
| D | 隔离网络中的 DHCP | 服务选做，与 C 二选一 |
| E | LVM、配额、RAID | 高风险存储挑战，三选一 |

---

## 三、模块 A 文件系统挂载与软件包管理

### A-1 只读观察存储层次

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,FSVER,LABEL,UUID,MOUNTPOINTS
findmnt
findmnt /
df -hT
cat /etc/fstab
```

说明区别：

- `lsblk` 从块设备角度展示磁盘、分区、LVM 和挂载点；
- `findmnt` 从已挂载文件系统角度显示源、目标、类型和选项；
- `df` 显示已挂载文件系统的容量与使用率；
- `/etc/fstab` 描述启动或按需挂载规则，不是当前挂载状态的唯一依据。

### A-2 安全学习 `/etc/fstab`

先备份：

```bash
sudo cp -a /etc/fstab /etc/fstab.lab06.bak
```

`fstab` 常见字段：

```text
<source> <mount-point> <type> <options> <dump> <pass>
```

修改后不要立即重启，先检查：

```bash
sudo findmnt --verify --verbose
sudo mount -a
```

`mount -a` 无输出不代表所有业务验证完成，还应用 `findmnt <mount-point>` 检查实际源和选项。如果只是学习字段含义，可以不修改文件，在报告中解析现有条目。

### A-3 APT 与 DPKG

```bash
apt list --installed 2>/dev/null | head
apt-cache policy openssh-server samba
dpkg-query -W -f='${Package}\t${Version}\t${Status}\n' bash coreutils
sudo apt update
apt list --upgradable 2>/dev/null
```

不要在课堂网络环境中无条件执行整机升级。`apt update` 只更新软件包索引，`apt upgrade` 才安装可升级包。分阶段更新可能使某些包暂时保留，不应为消除提示而随意强制更新。

以 `tree` 为安全的安装测试：

```bash
apt-cache show tree | sed -n '1,20p'
sudo apt install tree
tree --version
dpkg -L tree | head
dpkg -S /usr/bin/tree
sudo apt remove tree
```

报告中比较：`apt` 负责软件源与依赖解决，`dpkg` 负责 Debian 包的底层安装数据库与单包操作。本 Ubuntu 实验不使用 CentOS 6 时代的 `rpm/yum` 命令。

---

## 四、模块 B OpenSSH 安全远程登录

### B-1 安装与识别 systemd 启动方式

```bash
sudo apt install openssh-server
systemctl status ssh.service --no-pager
systemctl status ssh.socket --no-pager
sudo ss -ltnp | grep ':22 '
```

（`ss -p` 需要 `sudo` 才能显示属于 root 的进程名。）

Ubuntu 24.04 的 OpenSSH 可使用 systemd socket activation。应检查 `ssh.socket` 和 `ssh.service` 的实际状态，不要因为某一个单元看起来未长期运行就直接判定 SSH 失败。

### B-2 配置覆盖文件

优先使用包含目录中的独立文件，便于恢复：

```bash
sudoedit /etc/ssh/sshd_config.d/00-lab06.conf
```

建议内容：

```text
PermitRootLogin no
PasswordAuthentication yes
```

实验初次连接可暂时保留密码认证，但必须禁止 root 直接登录。如已完成密钥认证且验证新会话可登录，可再将 `PasswordAuthentication` 改为 `no`。不要在当前唯一的远程会话中未经验证就禁用密码。

OpenSSH 通常采用先读到的同名选项。`Include` 片段按文件名顺序读取，因此不要以为文件名末尾的 `99` 必然覆盖前面的设置。编辑后先检查最终生效值；如果它与预期不同，查找更早的配置和 `Match` 块，不要只凭文件内容判断配置已生效。

每次修改后：

```bash
sudo sshd -t
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication'
sudo systemctl try-reload-or-restart ssh.service
```

> 为什么不直接用 `reload`：在 socket 激活模式下，如果开机后还没有任何 SSH 连接，`ssh.service` 处于 inactive 状态，`systemctl reload` 会报 `not active, cannot reload`，容易被误认为配置出错。`try-reload-or-restart` 只在服务运行时重载，服务未运行时不做任何操作，下一次连接触发启动时自然会读取新配置。

确认输出中的 `permitrootlogin no` 与当前计划的 `passwordauthentication` 值一致，再从客户端开启新会话验证。已有的 `00-lab06.conf` 直接编辑，不用空文件覆盖；如存在其他配置片段设置同一选项，可用 `sudo grep -R -nE '^(PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d` 查找来源。

### B-3 密钥登录与客户端验证

在客户端执行：

```bash
ssh-keygen -t ed25519 -a 64
ssh-copy-id student@SERVER_IP
ssh -v student@SERVER_IP
```

服务器端检查：

```bash
journalctl -u ssh.service --since '10 minutes ago' --no-pager
last -a | head
who
```

报告隐去 IP 中不必要的公网信息、用户名、公钥指纹与其他可用于识别真实系统的数据。私钥文件不得提交。

### B-4 故障排查顺序

1. `ip -br addr`：地址是否正确；
2. `ping SERVER_IP`：网络是否可达（禁用 ICMP 时不能仅依赖 ping）；
3. `ss -ltnp`：服务器是否监听预期端口；
4. `sshd -t`：配置语法是否正确；
5. `journalctl -u ssh.service`：服务日志中的直接原因；
6. `ssh -vvv`：客户端协商和认证停在哪一阶段。

---

## 五、模块 C Samba 身份验证共享（与 D 二选一）

### C-1 目标与准备

目标：创建一个只允许 `labshare` 组成员访问的 `/srv/samba/labshare`，不启用 guest 可写共享。

```bash
sudo apt install samba smbclient
sudo groupadd -f labshare
sudo usermod -aG labshare "$USER"
sudo mkdir -p /srv/samba/labshare
sudo chgrp labshare /srv/samba/labshare
sudo chmod 2770 /srv/samba/labshare
sudo smbpasswd -a "$USER"
```

`2770` 中的 setgid 使新建文件继承目录组，但 Samba 配置权限不能超越 Linux 文件系统权限。新增辅助组后，需重新登录或用 `newgrp labshare` 使当前会话获得新组资格。

### C-2 配置共享

备份：

```bash
sudo cp -a /etc/samba/smb.conf /etc/samba/smb.conf.lab06.bak
sudoedit /etc/samba/smb.conf
```

在文件末尾添加：

```ini
[labshare]
    path = /srv/samba/labshare
    browsable = yes
    read only = no
    guest ok = no
    valid users = @labshare
    force group = labshare
    create mask = 0660
    directory mask = 2770
```

语法检查通过后再重启服务：

```bash
testparm -s
sudo systemctl restart smbd.service
systemctl status smbd.service --no-pager
smbclient -L localhost -U "$USER"
smbclient //localhost/labshare -U "$USER"
```

在运行 `smbclient` 的客户端先准备测试文件：

```bash
printf 'lab06 upload test\n' > /tmp/lab06-upload.txt
```

进入共享后，在 `smb: \>` 提示符下输入 `ls`、`mkdir testdir`、`lcd /tmp`、`put lab06-upload.txt` 和 `ls`，确认上传文件可见。`put` 必须指定本地文件名。然后回到 Linux 中检查：

```bash
find /srv/samba/labshare -maxdepth 2 -printf '%M %u:%g %p\n'
sudo smbstatus
```

### C-3 安全要点

- 不启用 SMB1；
- 不把口令直接写入命令行或报告；
- 不使用 `chmod 777` 作为权限故障的默认解决方法；
- 首先检查 Linux 目录权限、Samba `valid users`、用户映射和日志。

---

## 六、模块 D DHCP 服务（与 C 二选一）

### D-1 实验网络强制要求

DHCP 服务器只能向独立的内部虚拟网络提供租约，该网络不得存在另一台 DHCP 服务器。若使用虚拟化软件的“主机专用网络”，必须先关闭该网络自带的 DHCP 服务。服务器可另用一块 NAT 网卡安装软件包，但 Kea 只能监听实验网卡。禁止将实验 DHCP 连入校园网、家庭路由器 LAN 或公共网络。

### D-2 先设计，再配置

报告先填写：

| 项目 | 示例（按实际网络修改） |
|---|---|
| 子网 | `192.168.56.0/24` |
| 服务器静态地址 | `192.168.56.2` |
| 地址池 | `192.168.56.100` ～ `192.168.56.200` |
| 排除范围 | 网关、服务器、保留地址 |
| 默认网关 | 本例无路由器，不下发；若实验网确有网关，再填写实际地址 |
| DNS 服务器 | 本例不下发；若需要，再填写实验网内可达的 DNS 地址 |
| 租约时间 | 本例 600 秒，可根据实验时长调整 |

本实验采用 Ubuntu 24.04 的 Kea DHCPv4。下列示例假设服务器的隔离实验网卡名为 `enp0s8`，客户端的隔离实验网卡也由 `systemd-networkd` 管理；操作前都要用 `ip -br link` 查出实际名称并替换。

在服务器上，确认 `enp0s8` 是隔离实验网卡，且没有其他地址或现存 DHCP 服务后，为本次会话临时设置静态地址：

```bash
ip -br link
sudo ip link set enp0s8 up
sudo ip address add 192.168.56.2/24 dev enp0s8
ip -br addr show enp0s8
sudo apt install kea-dhcp4-server
sudo systemctl stop kea-dhcp4-server.service
sudo cp -a /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.lab06.bak
sudoedit /etc/kea/kea-dhcp4.conf
```

安装包后 Kea 服务可能自动启动；先停止它，避免默认配置在实验网卡上运行。只有在隔离网络、监听网卡和配置语法都确认无误后，才按下文重新启动。

将实验配置文件内容设为以下最小示例；网卡名、子网和地址池必须与自己的设计表一致：

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["enp0s8"] },
    "lease-database": { "type": "memfile", "lfc-interval": 3600 },
    "valid-lifetime": 600,
    "subnet4": [{
      "id": 1,
      "subnet": "192.168.56.0/24",
      "pools": [{ "pool": "192.168.56.100 - 192.168.56.200" }]
    }]
  }
}
```

本例只分配地址、掩码和租期，没有默认网关，也没有下发 DNS。隔离网内若确有网关或 DNS，可参照 Ubuntu 的 Kea 文档在 `option-data` 中加入 `routers` 和 `domain-name-servers`，并同步修改设计表与验收记录。

先检查配置，再启动服务：

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl restart kea-dhcp4-server.service
systemctl status kea-dhcp4-server.service --no-pager
sudo ss -lunp | grep ':67 '
sudo journalctl -u kea-dhcp4-server.service -n 30 --no-pager
```

### D-3 通用验证流程

1. 确认客户端只有实验网卡处于该隔离网，且这块网卡启用了 DHCPv4；记录其网卡名。
2. 在客户端运行 `sudo networkctl renew <客户端实验网卡名>`，再用 `ip -4 addr show <客户端实验网卡名>` 和 `networkctl status <客户端实验网卡名>` 检查地址与租约来源。若网卡不由 `systemd-networkd` 管理，应使用该客户端实际网络管理器的续租方式。
3. 在服务器上运行 `sudo journalctl -u kea-dhcp4-server.service --since '5 minutes ago' --no-pager`，核对分配日志中的客户端与地址；客户端地址应落在设计的地址池内。
4. 核对地址和掩码；本例应**没有**由实验网卡提供的默认网关与 DNS。如在设计表中显式配置了它们，再检查其实际值。
5. 结束后运行 `sudo systemctl stop kea-dhcp4-server.service`，确认服务不再监听 UDP 67，并按记录恢复配置或虚拟机快照。

**验收不只看“客户获得了地址”：**必须证明地址来自自己的服务器，且各选项与设计表一致。

---

## 七、模块 E 存储挑战（LVM、配额、RAID 三选一）

### E-1 通用准入检查

- [ ] 已创建快照，并确认可恢复；
- [ ] 已添加专用空白虚拟磁盘，其中没有需要保留的数据；
- [ ] 已记录系统原磁盘与新磁盘的序列号、大小和设备名；
- [ ] 所有破坏性命令中的设备路径由教师或同组成员复核；
- [ ] 已用 `lsblk -f` 和 `findmnt /` 确认系统盘的实际设备名（不一定是 `/dev/sda`，也可能是 `/dev/vda`、`/dev/nvme0n1` 等），练习对象不是它，也不是它的任何分区。

### E-2 LVM 路线

学习概念：PV（物理卷）→ VG（卷组/存储池）→ LV（逻辑卷）→文件系统→挂载点。

建议任务：

1. 在空白虚拟盘上创建 PV；
2. 创建实验专用 VG，名称如 `vg_lab06`；
3. 创建小型 LV `lv_data`，格式化后挂载到 `/mnt/lab06-lvm`；
4. 写入测试文件并记录哈希；
5. 扩展 LV 和文件系统，验证数据不变；
6. 卸载后按“文件系统 → LV → VG → PV”的逆序清理。

不将缩小文件系统或 LV 作为入门必做；缩小比扩展更容易造成数据丢失，且不是所有文件系统都支持缩小。

### E-3 磁盘配额路线

在独立实验文件系统上完成：

1. 确认内核和工具支持的配额类型；
2. 为挂载配置启用用户配额；
3. 检查并启用配额；
4. 为专用测试用户设置软限制和硬限制；
5. 分别测试低于软限制、超过软限制和超过硬限制的行为；
6. 记录容量限制和 inode 数量限制的区别。

配额基于文件系统，不是基于普通目录路径的通用限制。如 `/home` 不是独立文件系统，不应为完成实验而盲目修改系统盘分区。

### E-4 软件 RAID 路线

只使用专用空白虚拟磁盘：RAID 5 需要 3 块成员盘＋1 块替换盘（共 4 块），RAID 1 需要 2 块成员盘＋1 块替换盘（共 3 块）。磁盘大小相同即可，每块 1～2 GiB 就够用。建议任务：

1. 用 `mdadm` 创建实验阵列；
2. 观察 `/proc/mdstat` 同步进度；
3. 建立文件系统、挂载、写入并验证数据；
4. 在快照保护下将一个成员标记为故障，观察降级状态；
5. 移除故障成员、加入替换盘并观察重建，重建完成后再次验证数据；如果没有准备替换盘，也可以对移除的那块盘执行 `mdadm --zero-superblock`，清除超级块后把它当作新盘重新加入；
6. 说明 RAID 不等于备份：误删、加密勒索和应用层数据损坏会被同步到所有成员。

---

## 八、不再作为必做的旧内容

### 8.1 Telnet

Telnet 不提供现代意义上的加密与安全认证，不用于远程管理实验。如需观察明文协议，应在隔离环境中使用抓包样例或专用教学服务，而不在主机上启用 Telnet 服务器。

### 8.2 匿名可写 FTP

文件传输优先使用 SFTP/SCP，它们复用 SSH 的加密与认证。FTP 协议原理可作为网络课程的协议分析内容，但本指导书不再要求配置匿名可写 FTP 服务。

### 8.3 CentOS 6 时代的镜像与 YUM 示例

不使用已停止维护的 CentOS 6 镜像、过时网易源 URL 或 `--force --nodeps` 作为 Ubuntu 24.04 实验的软件包管理方案。如需对比 RPM 与 DEB 生态，应使用受支持的 Rocky Linux/AlmaLinux 虚拟机单独开设对比实验。

---

## 九、通用服务验证模板

每个服务都按以下顺序记录：

| 阶段 | 需回答的问题 | 典型工具 |
|---|---|---|
| 设计 | 服务面向谁，监听哪个地址/端口，认证和授权规则是什么 | 网络拓扑、参数表 |
| 备份 | 如何恢复修改前配置 | `cp -a`、虚拟机快照 |
| 语法 | 配置文件能否被服务解析 | `sshd -t`、`testparm` 或服务专用检查器 |
| 运行 | 单元是否正常，是否需要启用 | `systemctl status/is-enabled` |
| 监听 | 端口、协议、监听地址是否正确 | `ss -lntup` |
| 日志 | 服务端认为失败原因是什么 | `journalctl -u` |
| 本机验证 | 不经网络中间设备时是否正常 | `localhost`、服务客户端 |
| 远程验证 | 真实客户端能否连接、认证和完成业务 | SSH/Samba/DHCP 客户端 |
| 清理 | 如何停止服务、删除测试数据并恢复配置 | 恢复备份、卸载、停用单元 |

---

## 十、实验报告与验收

### 10.1 报告必须包含

1. 虚拟机网络拓扑、IP 地址与安全边界；
2. 所选模块、选择理由和预期成果；
3. 修改前备份、修改内容、语法检查和恢复方法；
4. 服务状态、监听端口、日志与客户端的四类验证证据；
5. 至少一个真实故障的现象、假设、排查命令、原因和修复；
6. 实验结束后的清理与复原结果。

### 10.2 验收标准

- [ ] 完成模块 A 和 B；
- [ ] 完成 Samba 或 DHCP 中的一项；
- [ ] 所有配置修改均有备份和恢复方法；
- [ ] 服务配置通过专用语法检查；
- [ ] 能从 `systemctl`、`ss`、`journalctl` 和客户端四个角度解释服务；
- [ ] 不存在公网暴露、默认密码、root SSH 登录、SMB1 或匿名可写共享；
- [ ] 报告不包含密码、私钥或真实凭据；
- [ ] 存储挑战（如选做）全程只操作专用空白虚拟磁盘。

### 10.3 参考资料

- Ubuntu Server 文档：`https://documentation.ubuntu.com/server/`
- Ubuntu Kea DHCPv4 配置指南：`https://ubuntu.com/server/docs/how-to/networking/install-isc-kea/`
- OpenSSH 手册：`man ssh`、`man sshd_config`
- Samba 配置手册：`man smb.conf`、`man testparm`
- systemd 手册：`man systemctl`、`man journalctl`、`man systemd.unit`
- LVM 手册：`man lvm`、`man pvcreate`、`man vgcreate`、`man lvcreate`
- 软件 RAID 手册：`man mdadm`
