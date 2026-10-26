# 不拆机、不进 BIOS、不接显示器：远程开启 HP 商用机 VT-x 的完整路径

## 起因

一台 HP ProDesk 600 G2 SFF，装了 Proxmox VE 当家用服务器。机器放在角落，没有显示器、没有键盘，全靠 SSH 和 Web 管理。

问题来了：虚拟机慢得离谱。

同机对比测试（`dd if=/dev/zero of=/dev/null bs=1M count=200`，纯 CPU 零拷贝）：

```
LXC 容器          20.1 GB/s    101%   ← 正常
PVE 宿主机         19.8 GB/s    100%   ← 基准
虚拟机（无 VT-x）    3.2 GB/s     16%   ← 慢 6.2 倍
```

开机时间也差得远：虚拟机约 60 秒，容器约 10 秒。

原因很明确：**BIOS 里 VT-x 是关的**。

```
$ grep -c vmx /proc/cpuinfo
0

$ systemctl status 早报
kvm: disabled by bios
```

麻烦的是，这台机器的 BIOS 界面根本看不见。显示器接在 DisplayPort 上，通过一根 DP→HDMI 转接线连到显示器。Linux 起来之后画面正常，但 **BIOS 自检和设置界面全程无信号**。

按 F10 能听到主板"嘟"一声——说明按键被收到了，确实进了设置界面——但屏幕一片黑。

盲操作试过几轮：F9 恢复默认、F10 保存退出、方向键挪动确认框按钮，都失败了。重启后 `grep -c vmx` 依然是 0。

**盲操这条路走不通：没有反馈闭环，每一轮都要重启验证，纯赌。**

---

## 为什么软件层面绕不过去

第一反应是：能不能在操作系统里直接把 VT-x 打开？

先要知道 BIOS 是怎么关掉它的。Intel 的 CPU 里有一个 MSR 寄存器，地址 `0x3A`，叫 `IA32_FEATURE_CONTROL`，两个关键位：

- **bit 0 = LOCK**
- **bit 2 = VMX outside SMX enable**

读出来是这样：

```bash
$ rdmsr 0x3a
1

$ rdmsr -f 0:0 -p 0 0x3a
1          # bit0 LOCK = 1，已锁定

$ rdmsr -f 2:2 -p 0 0x3a
0          # bit2 VMX 使能 = 0，未开启
```

**LOCK 位已经是 1。**

Intel 的规则写得很死：一旦 LOCK 置 1，这个寄存器在本次上电周期内就被永久锁死，任何软件（包括内核、包括 root）都不能再改。想改只能断电重启，让 BIOS 自己重新设置。

再看 CPU 本身的能力：

```bash
$ cpuid -1 -r | head -2
0x00000001 0x00: eax=0x000506e3 ebx=0x02100800 ecx=0x7ffafbff edx=0xbfebfbff
```

`ecx = 0x7ffafbff`，bit 5（VMX）是 **1**。

也就是说：**CPU 硬件完全支持虚拟化，纯粹是 BIOS 把它关了并上了锁。**

内核看到 MSR 被锁、使能位为 0，就会主动把 `vmx` 这个 flag 从 `/proc/cpuinfo` 里剔除（所以 `lscpu` 的 flags 里只剩 `smx` 没有 `vmx`）——这是防止上层误判"支持虚拟化"然后调 KVM 失败。

### 「那更新 BIOS 呢？」

也不行，三个理由：

1. **BIOS 升级只换固件代码，不动设置项。** 现在是关的，刷完还是关的。
2. 这台机器手上只有 Windows 版的 `.exe` 刷新包，PVE 上跑不了。
3. HP 的"从 U 盘刷 BIOS"功能本身也要先能进 BIOS 界面才能用。

**结论：必须改 BIOS 设置，躲不开。**

---

## 转机：HP 官方的 Linux 工具链

大多数人不知道，HP 其实一直在维护一套 Linux 版的 BIOS 管理工具：

**HP Flash and Replicated Setup Utilities for Linux**

下载地址（这个路径是活的，2026 年 3 月还更新过）：

```
https://ftp.hp.com/pub/softpaq/sp171001-171500/sp171387.tgz
```

当前版本 **v3.26，SP171387**，约 730 KB。

它包含两个东西：

| 工具 | 作用 |
|---|---|
| `hp-flash` | 刷 BIOS 固件 |
| `hp-repsetup` | **读写 BIOS 的全部设置项**（Replicated Setup） |

`hp-repsetup` 正是我们要的。

但它依赖一个内核模块 `hpuefi`，用来提供 UEFI 的访问通道。这个模块不在 Debian/Ubuntu 的仓库里，HP 也没给现成的 deb 包，需要自己编译。

### 包里的隐藏坑

解开 `sp171387.tgz` 后，会看到一个 `install.sh`。

**别急着跑，它一定会失败。**

```bash
$ tar xzf sp171387.tgz
$ cd non-rpms/hp-flash-3.26_x86_64
$ ls bin/
# 空的
$ sudo ./install.sh
install: cannot stat 'bin/hp-flash': No such file or directory
```

`bin/` 目录是空的。真正的二进制在 `builds/` 下面，按发行版分好了：

```
builds/
  hp-flash.rh70  hp-flash.rh80  hp-flash.rh90  hp-flash.rh100  hp-flash.rh610
  hp-flash.sled12  hp-flash.sled15
  hp-flash.u1604  hp-flash.u1804  hp-flash.u2004  hp-flash.u2204
  hp-flash.u2404  hp-flash.u2510
  hp-repsetup.（同样的后缀）
  test-distro.sh
```

Debian 13 用 `u2404`（glibc 版本最接近 Ubuntu 24.04）。

手工复制即可：

```bash
sudo mkdir -p /opt/hp/hp-flash/bin
sudo cp builds/hp-repsetup.u2404 /opt/hp/hp-flash/bin/hp-repsetup
sudo cp builds/hp-flash.u2404    /opt/hp/hp-flash/bin/hp-flash
sudo chmod 0744 /opt/hp/hp-flash/bin/*
```

### 编译 hpuefi 内核模块

源码在 `non-rpms/hpuefi-mod-3.07.tgz`。

**先说结论：HP 官方 Readme 声称支持到内核 6.18，但我在 7.0.2 上编译通过了，不需要打任何补丁。**

环境准备：

```bash
apt-get update
apt-get install -y --no-install-recommends gcc make proxmox-headers-$(uname -r)
```

（Debian 用 `linux-headers-$(uname -r)`，PVE 用 `proxmox-headers-$(uname -r)`。）

编译：

```bash
tar xzf hpuefi-mod-3.07.tgz
cd hpuefi-mod-3.07
make
make install
depmod -a
```

输出里有两条警告，都可以忽略：

```
warning: pahole version differs from the one used to build the kernel
Skipping BTF generation for hpuefi.ko due to unavailability of vmlinux
```

加载：

```bash
modprobe hpuefi
mkdir -p /dev
/lib/modules/$(uname -r)/kernel/drivers/hpuefi/mkdevhpuefi
ls -l /dev/hpuefi
# crw-r--r-- 1 root root 235, 0 /dev/hpuefi
```

会看到 `module verification failed: signature and/or required key missing - tainting kernel`——模块未签名，正常现象。

**前提：Secure Boot 必须是关闭的**，否则未签名模块加载不了。

检查：

```bash
# Secure Boot 状态
bootctl status | grep -i secure
```

---

## 读取全部 BIOS 设置

```bash
cd /opt/hp/hp-flash/bin
./hp-repsetup -g
```

会生成 `HPSETUP.TXT`，**17476 字节，UTF-16LE 编码，带 CRLF**。

格式长这样：

```
Serial Number
    6CR6348XGQ

System BIOS Version
    N02 Ver. 02.38

Virtualization Technology (VTx)
     Disable
     Enable

Virtualization Technology for Directed I/O (VTd)
     Disable
     Enable
```

**注意两个细节：**

1. `*` 标记当前选中项。上面这份是没选中任何项的原始导出（第一项是默认值），实际导出的文件里会有 `*` 前缀。
2. **每一项前面有一个 TAB 字符**，不是空格。

### 坑：`-a` 参数

Readme 里写着 `-a` 是 ASCII 模式。

```bash
./hp-repsetup -g -a /tmp/test.txt
```

**它会把内容打到屏幕上，但不生成那个文件。**

别问我怎么知道的。老老实实用默认的 UTF-16 格式，然后用 `iconv` 转换着看：

```bash
iconv -f UTF-16LE -t UTF-8 HPSETUP.TXT > HPSETUP_readable.txt
```

---

## 修改设置：星号的位置很关键

找到了：

```
421: Virtualization Technology (VTx)
422-     Disable
423-     Enable
```

把 `*` 从 422 行挪到 423 行。

**但这里有一个真实的坑。**

如果你直接做字符串替换，很容易得到：

```
*\tEnable      ← 错的
```

而不是：

```
\t*Enable      ← 对的
```

星号必须在 TAB **之后**。HP 的解析器认 `\t*Enable`，不认 `*\tEnable`。写错了不报错，只是静默不生效。

另外：**不要用 `sed`。** UTF-16 编码下 `sed` 会破坏字节序，而且 CRLF 会让 `$` 匹配失效。

用 Python 处理，注意保留 BOM：

```python
import sys

path = 'HPSETUP.TXT'
with open(path, 'rb') as f:
    raw = f.read()

has_bom = raw.startswith(b'\xff\xfe')
text = raw.decode('utf-16-le')
if has_bom:
    text = text[1:]          # 去掉 BOM 字符本身

lines = text.split('\r\n')

def set_mark(line, on):
    """把 * 挪到第一个非 TAB 字符之前"""
    s = line.replace('*', '')
    idx = len(s) - len(s.lstrip('\t'))
    return s[:idx] + ('*' if on else '') + s[idx:]

for i, line in enumerate(lines):
    if line.startswith('Virtualization Technology (VTx)'):
        lines[i+1] = set_mark(lines[i+1], False)
        lines[i+2] = set_mark(lines[i+2], True)
    if line.startswith('Virtualization Technology for Directed I/O (VTd)'):
        lines[i+1] = set_mark(lines[i+1], False)
        lines[i+2] = set_mark(lines[i+2], True)

out = '\r\n'.join(lines).encode('utf-16-le')
if has_bom:
    out = b'\xff\xfe' + out

with open(path, 'wb') as f:
    f.write(out)
```

改完用 `cat -A` 检查：

```bash
$ cat -A HPSETUP.TXT | sed -n '421,426p'
^IVirtualization Technology (VTx)^M$
^IDisable^M$
^I*Enable^M$          ← 对了
^IVirtualization Technology for Directed I/O (VTd)^M$
^IDisable^M$
^I*Enable^M$
```

`^I` 是 TAB，`*` 紧跟在它后面，`^M$` 是 CRLF。正确。

文件大小应该不变，还是 17476 字节。

---

## 写回 BIOS

**先备份！**

```bash
cp HPSETUP.TXT HPSETUP.orig.TXT
```

写入：

```bash
./hp-repsetup -s HPSETUP.TXT
echo "exit=$?"
```

会看到一大片这样的输出：

```
Serial Number                       (Access Denied)
System BIOS Version                 (Access Denied)
Memory Size                         (Access Denied)
Import Secure Boot Keys             (Unsupported)
Save/Restore MBR of System Hard Drive  (Unsupported)
Processor 1 DIMM 1 Size             (Access Denied)
```

**这些都是正常的，不是失败。**

- `(Access Denied)`——只读的硬件信息（序列号、内存、BIOS 版本），本来就不能写。
- `(Unsupported)`——这台机型不支持的功能。

只要退出码是 0，关键项就写进去了。

### 验证写入

再读一次：

```bash
./hp-repsetup -g
iconv -f UTF-16LE -t UTF-8 HPSETUP.TXT | sed -n '421,426p'
```

应该看到：

```
Virtualization Technology (VTx)
     Disable
    *Enable

Virtualization Technology for Directed I/O (VTd)
     Disable
    *Enable
```

设置已经进 Flash 了。

---

## 重启并验证

```bash
reboot
```

回来后逐项检查：

```bash
# 1. CPU 有没有报告虚拟化能力
$ grep -c vmx /proc/cpuinfo
8

# 2. MSR 锁的状态
$ rdmsr 0x3a
5
# 二进制 101：bit0 LOCK=1，bit2 VMX enable=1 ← 开了

$ rdmsr -f 2:2 -p 0 0x3a
1

# 3. KVM 模块
$ lsmod | grep kvm
kvm_intel             507904  0
kvm                  1441792  1 kvm_intel

# 4. 设备节点
$ ls -l /dev/kvm
crw-rw---- 1 root kvm 10, 232 /dev/kvm

# 5. CPU flags
$ lscpu | grep -o vmx
vmx
```

全绿。

---

## 实测性能

同一台机器，同一个测试（`dd if=/dev/zero of=/dev/null bs=1M count=200`）：

| | 吞吐 | 相对宿主 |
|---|---|---|
| PVE 宿主机 | 19.8 GB/s | 100% |
| LXC 容器 | 20.1 GB/s | 101% |
| **虚拟机（开 VT-x 前）** | **3.2 GB/s** | **16%** |
| **虚拟机（开 VT-x 后）** | **16.7 GB/s** | **84%** |

**提升 5.2 倍。**

虚拟机的 qemu 进程名也变了：之前是 `qemu-system-x86_64`（纯软件模拟），现在是 `kvm`（硬件加速）。

开机时间：约 60 秒 → 约 10 秒。

---

## 适用范围与前提

**已实测：**
- HP ProDesk 600 G2 SFF（BIOS `N02 Ver. 02.38`，2019 年固件）
- Proxmox VE 9.2.2（内核 `7.0.2-6-pve`）

**理论适用（未实测，需自行验证）：**
- HP 官方 Readme 声称支持「2015 年及以后的 HP 桌面工作站」与「2018 年及以后的 HP 商用笔记本和台式机」
- 我那台是 2015 年的桌面机（非工作站），官方支持范围之外，但实际能用
- 不同机型的 `HPSETUP.TXT` 行号和设置项名称会不同，用 `grep` 找，别记行号

**必须满足：**
- Secure Boot 关闭（模块未签名）
- root 权限
- 能装内核头文件和编译器

**风险：**
- 写 BIOS 设置有理论风险。操作前先备份原始 `HPSETUP.TXT`，写坏了可以 `-s` 把原文件刷回去。
- 刷 BIOS 固件（`hp-flash`）风险更高，断电可能砖。**本文只读写设置，没刷固件。**
- 虚拟机、断电保护这些和 VT-x 无关，别混在一起做。

---

## 附带收获

同一套工具还能干别的：

- `hp-repsetup` 能改**所有** BIOS 设置，包括启动顺序、Secure Boot、TPM、风扇策略、USB 端口禁用
- 批量部署同型号机器时，可以在一台上调好配置，导出 `HPSETUP.TXT`，其余机器直接 `-s` 导入——这就是 Replicated Setup 的本意
- `hp-flash` 可以在 Linux 下刷 BIOS 固件（需自担风险）

一个顺带的小提示：如果只是想进 BIOS 看画面，还有个更简单的办法——

```bash
systemctl reboot --firmware-setup
```

这条命令会设置 UEFI 的 `OsIndications` 变量（`EFI_OS_INDICATIONS_BOOT_TO_FW_UI`），主板下次启动直接停在设置界面，**不用按任何键**。

检查主板支不支持：

```bash
od -An -tx1 /sys/firmware/efi/efivars/OsIndicationsSupported-*
# 06 00 00 00 01 00 00 00 00 00 00 00
#             ^^ bit0 = 1，支持
```

但注意：**它只解决"怎么进"，不解决"看不见"**。如果显示器在 BIOS 阶段就是没信号，进去了照样黑屏。我这次就是卡在这一步，才转去走工具链的。

---

## 小结

真正难的不是「怎么改 BIOS 设置」，而是三件事叠在一起：

1. BIOS 给 VT-x 上了硬锁，软件层面无解
2. 机器无头，显示器在 BIOS 阶段不亮
3. HP 的工具能用，但包里的路径是坏的、参数有陷阱、编码要小心

把这三条一个个拆开，路就通了。

希望这篇能省掉别人几个小时的试错。
