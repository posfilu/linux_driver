# 第 1 课：开发环境搭建与第一个内核模块

## 学习目标

学完本课，你应该能够：

1. 说清楚驱动在 Linux 系统中处于什么位置，用户程序是怎么"调用"到驱动的
2. 理解内核模块（`.ko`）机制，熟练使用 `insmod` / `rmmod` / `lsmod` / `modinfo` / `dmesg`
3. 在你的 RK3399 板子上搭好模块编译环境（板上编译或 PC 交叉编译）
4. 独立写出一个内核模块和它的 Makefile，并理解 Kbuild 的工作方式
5. 理解 **vermagic**：为什么模块必须和正在运行的内核匹配

---

## 一、驱动在系统中的位置

### 1.1 用户空间与内核空间

```
┌───────────────────────────────────────────────┐
│  用户空间   应用程序 (cat, echo, 你的 C 程序)      │
│               │  open/read/write/ioctl (glibc)  │
├───────────────┼───────────────────────────────┤
│               ▼  系统调用 (svc 指令陷入内核)       │
│  内核空间   VFS 虚拟文件系统                       │
│               │  根据 /dev 节点的设备号找到驱动     │
│               ▼                                 │
│           设备驱动 (你要写的代码)                   │
│               │  读写寄存器 / 调用总线接口          │
├───────────────┼───────────────────────────────┤
│  硬件         ▼  GPIO / I2C / SPI / UART ...     │
└───────────────────────────────────────────────┘
```

- ARM64 上 CPU 有不同的异常级别（Exception Level）：用户程序跑在 **EL0**，内核跑在 **EL1**。
- 用户程序不能直接访问硬件，必须通过**系统调用**进入内核，再由驱动操作硬件。
- Linux 的设计哲学是"一切皆文件"：大部分设备在 `/dev` 下表现为一个文件，对它 `open/read/write/ioctl`，最终会调用到驱动里对应的函数。

### 1.2 三类设备

| 类型 | 特点 | 例子 |
|------|------|------|
| 字符设备（char） | 按字节流顺序访问，没有缓存 | 串口 `/dev/ttyS2`、`/dev/i2c-*`、`/dev/null` |
| 块设备（block） | 按块随机访问，经过页缓存 / 块层 | eMMC `/dev/mmcblk*`、SD 卡、U 盘 |
| 网络设备（net） | **没有** `/dev` 节点，通过 socket 访问 | `eth0`、`wlan0` |

在板子上执行 `ls -l /dev`，第一列开头的 `c` 表示字符设备，`b` 表示块设备。本课程的主线是**字符设备**，以及建立在它之上的各种内核子系统（GPIO、input、I2C、IIO……）。

### 1.3 驱动的两种存在形式

- **编进内核**（配置为 `=y`）：驱动是内核镜像的一部分，开机即可用。
- **编成模块**（配置为 `=m`）：驱动是一个单独的 `.ko` 文件，运行时动态加载 / 卸载。

学习阶段我们几乎都用**模块**：改完代码重新编译 `.ko`，`rmmod` 旧的、`insmod` 新的，不用重新烧写内核、不用重启。

---

## 二、认识你的 RK3399

RK3399 是瑞芯微（Rockchip）的 64 位 SoC：

- CPU：2 × Cortex-A72（大核）+ 4 × Cortex-A53（小核），ARMv8 架构，Linux 里叫 **arm64 / aarch64**
- 外设：GPIO0～GPIO4 共 5 组 GPIO、多路 I2C / SPI / UART / PWM、SARADC、USB3.0、PCIe 等
- 常见板子：Firefly-RK3399、NanoPi M4 / NanoPC-T4、ROCK Pi 4、Orange Pi 4、RockPro64、Khadas Edge 等

RK3399 板子上跑的内核大致分两类，**第 1 课先要搞清楚你的板子是哪一类**：

| 类型 | 典型版本 | 常见来源 | 特点 |
|------|---------|---------|------|
| 厂商 BSP 内核 | 4.4 / 4.19 / 5.10 | 板厂提供的 Ubuntu / Debian / Buildroot 固件 | 硬件支持全（GPU、ISP 等），但版本老，很多新 API 没有 |
| 主线内核 | 6.x | Armbian、Debian/Ubuntu 官方 arm64 内核、Manjaro 等 | API 新，文档多，和网上资料一致 |

两类都可以完成本课程。遇到 API 在不同内核版本间有差异的地方，讲义会专门注明。

> 💡 **调试串口**：RK3399 板子的调试串口一般是 UART2，默认波特率 **1500000**（不是常见的 115200），8N1。
> 写驱动难免把内核搞崩，崩溃时 SSH 会断开，只有串口能看到完整的 Oops 信息。强烈建议现在就把串口接好。
> 在 PC 上可以用 `picocom -b 1500000 /dev/ttyUSB0` 或 `minicom`、MobaXterm、SecureCRT 等工具。

---

## 三、搭建编译环境

### 3.1 编译模块需要什么

编译内核模块**不是**普通的 `gcc hello.c`。模块要"嵌入"到一个**特定的内核**里运行，所以编译时需要：

1. **与正在运行的内核完全一致的内核构建树**：头文件、`.config`、`Module.symvers`、Kbuild 脚本
2. **目标架构（arm64）的编译器**

内核加载模块时会检查模块里记录的 **vermagic**（版本魔数），它包含内核版本字符串和几个关键配置项，比如：

```
vermagic: 6.6.63-current-rockchip64 SMP preempt mod_unload aarch64
```

如果和运行中内核的不一致，`insmod` 会直接拒绝加载。所以**"用哪套内核源码 / 头文件编译"是本课最重要的问题**。

### 3.2 路线 A：在板子上直接编译（推荐初学者）

优点是编译器和架构天然正确，头文件和内核一定匹配，最省心。前提是能装上**与当前内核版本一致**的 headers 包。

```bash
# 1. 查看当前内核版本
uname -r

# 2. 检查构建目录是否已经存在（存在就说明 headers 已装好）
ls -l /lib/modules/$(uname -r)/build

# 3. 如果不存在，安装 headers（根据发行版选择）
#   Armbian:        sudo apt install linux-headers-current-rockchip64
#                   （或在 armbian-config 里安装；包名以 `apt search linux-headers` 结果为准）
#   Debian/Ubuntu:  sudo apt install linux-headers-$(uname -r)
#   厂商固件:        看板厂 wiki，可能需要自己从 SDK 内核源码生成

# 4. 安装编译工具
sudo apt install build-essential bc flex bison libssl-dev
```

> 常见坑：如果编译时报 `scripts/basic/fixdep: cannot execute binary file: Exec format error`，
> 说明 headers 包里的辅助工具是在 x86 上编译的。进入 `/lib/modules/$(uname -r)/build` 目录执行
> `sudo make scripts`（或 `sudo make modules_prepare`）在板子上重新编译这些工具即可。

### 3.3 路线 B：在 PC 上交叉编译

如果板子上装不了匹配的 headers（厂商固件常见），就在 PC（x86 Linux 或 WSL2）上交叉编译，再把 `.ko` 拷到板子上。

```bash
# 1. 安装交叉编译器
sudo apt install gcc-aarch64-linux-gnu build-essential bc flex bison libssl-dev

# 2. 获取与板子上运行的内核【完全一致】的源码
#    - 厂商 SDK 里的 kernel 目录，或者板厂 GitHub 上对应的分支 / tag
#    - 板子上的 .config：如果内核开启了 CONFIG_IKCONFIG_PROC，可以从 /proc/config.gz 取出

# 3. 在内核源码目录中配置并完整编译一次（得到 Module.symvers）
cp config-from-board .config
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

> 注意：
> - 版本号要**逐字符一致**，包括 `-rockchip64`、`+` 这类后缀（由 `CONFIG_LOCALVERSION` 和源码是否"dirty"决定）。
> - 只执行 `make modules_prepare` 也能编译模块，但不会生成 `Module.symvers`，开启了 `CONFIG_MODVERSIONS` 的内核可能拒绝加载，所以建议完整编译一次。

### 3.4 选哪条路线？

**先尝试路线 A**。`/lib/modules/$(uname -r)/build` 存在且能编译通过，就用路线 A；搞不定再走路线 B。
无论选哪条，都在 `board-info.md` 里写清楚你用的方式，以及遇到的问题和解决办法。

---

## 四、第一个内核模块

### 4.1 源码

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * hello.c - 最简单的内核模块
 */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/printk.h>

static int __init hello_init(void)
{
	pr_info("hello: module loaded\n");
	return 0;
}

static void __exit hello_exit(void)
{
	pr_info("hello: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name <you@example.com>");
MODULE_DESCRIPTION("RK3399 driver course - lesson 01");
```

逐行理解：

| 代码 | 含义 |
|------|------|
| `// SPDX-License-Identifier: GPL-2.0` | 内核源码统一的许可证声明，写在文件第一行 |
| `module_init(hello_init)` | 指定加载模块时调用的函数。返回 `0` 表示成功；返回负的错误码（如 `-ENOMEM`）表示失败，模块不会被加载 |
| `module_exit(hello_exit)` | 指定卸载模块时调用的函数，负责释放 init 里申请的所有资源 |
| `__init` | 把函数放进 `.init.text` 段。编进内核时，启动完成后这段内存会被释放；模块中则在加载完成后释放 |
| `__exit` | 把函数放进 `.exit.text` 段。如果驱动编进内核（永远不会卸载），这段代码会被直接丢弃 |
| `pr_info(...)` | 等价于 `printk(KERN_INFO ...)`，内核里的"printf"，输出到内核日志缓冲区 |
| `MODULE_LICENSE("GPL")` | 声明许可证。影响能否使用 `EXPORT_SYMBOL_GPL` 导出的符号，以及内核是否被标记为 tainted |
| `static` | 模块内部的函数和变量一律加 `static`，避免污染内核的全局符号表 |

### 4.2 Makefile

```makefile
# 要编译成模块的目标：hello.o -> hello.ko
obj-m += hello.o

# 内核构建目录：默认使用板子上当前内核的 headers，交叉编译时在命令行覆盖
KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(CURDIR) modules

clean:
	$(MAKE) -C $(KDIR) M=$(CURDIR) clean
```

> ⚠️ `$(MAKE)` 前面必须是 **Tab**，不能是空格，否则报 `missing separator`。

这个 Makefile 会被**读两次**：

1. 你在模块目录执行 `make`：`obj-m` 此时没什么用，执行 `all` 目标，通过 `-C $(KDIR)` 切换到**内核构建目录**，用内核自己的 Makefile 编译。
2. 内核的构建系统（Kbuild）通过 `M=$(CURDIR)` 知道外部模块在哪，**再次读取你的 Makefile**，这次只关心 `obj-m`，据此把 `hello.c` 编译成 `hello.ko`。

编译命令：

```bash
# 路线 A：板上编译
make

# 路线 B：交叉编译（在 PC 上）
make KDIR=~/rk3399/kernel ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

编译成功后目录里会多出 `hello.ko`（最终模块）、`hello.mod.c`（自动生成的模块信息，包含 vermagic）、`Module.symvers`、`modules.order` 等文件。

### 4.3 加载、查看、卸载

```bash
sudo insmod hello.ko              # 加载模块
sudo dmesg | tail                 # 查看内核日志（也可以 dmesg -w 或 journalctl -k -f 持续跟踪）
lsmod | grep hello                # 查看已加载的模块
modinfo hello.ko                  # 查看模块信息（license、vermagic 等）
ls /sys/module/hello/             # 模块在 sysfs 中的目录
sudo rmmod hello                  # 卸载模块（不写 .ko 后缀）
```

`insmod` 和 `modprobe` 的区别：`insmod` 直接加载指定文件，不处理依赖；`modprobe` 按模块名在 `/lib/modules/$(uname -r)/` 中查找，并自动加载依赖（第 2 课细讲）。

### 4.4 printk 与日志级别

| 级别 | 宏 | 数值 | 便捷写法 |
|------|----|:---:|---------|
| 紧急 | `KERN_EMERG` | 0 | `pr_emerg()` |
| 警报 | `KERN_ALERT` | 1 | `pr_alert()` |
| 严重 | `KERN_CRIT` | 2 | `pr_crit()` |
| 错误 | `KERN_ERR` | 3 | `pr_err()` |
| 警告 | `KERN_WARNING` | 4 | `pr_warn()` |
| 注意 | `KERN_NOTICE` | 5 | `pr_notice()` |
| 信息 | `KERN_INFO` | 6 | `pr_info()` |
| 调试 | `KERN_DEBUG` | 7 | `pr_debug()` |

- `cat /proc/sys/kernel/printk` 的 4 个数字中，第一个是**控制台日志级别**：只有级别数值**小于**它的消息才会打印到控制台（串口）。
- `dmesg` 读取的是内核日志缓冲区，能看到所有级别的消息（`pr_debug` 除外，见下一条）。
- `pr_debug()` 比较特殊，默认不输出：
  - 内核**没开** `CONFIG_DYNAMIC_DEBUG`：除非源码里定义了 `DEBUG` 宏，否则 `pr_debug()` 在**编译阶段**就被去掉了。
  - 内核**开了** `CONFIG_DYNAMIC_DEBUG`：代码会编译进去，但默认处于关闭状态，需要在**运行时**打开。
- 很多驱动会定义 `#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt`（要写在所有 `#include` 之前），这样所有 `pr_*()` 输出都会自动带上模块名前缀。

---

## 五、内核编程和应用编程的区别

写第一行内核代码之前，先记住这几条：

1. **没有 C 标准库**：不能用 `printf`、`malloc`、`strlen`（libc 版）等，内核有自己的实现：`printk`、`kmalloc`、`strlen`（`<linux/string.h>`）等。
2. **不要随意使用浮点运算**：内核切换时默认不保存浮点 / SIMD 寄存器，随意使用会破坏用户进程的状态。
3. **栈很小**：arm64 内核栈通常只有 16KB，不能定义大数组局部变量，也不要写深递归。
4. **没有内存保护**：用户程序访问野指针只会导致自己段错误；内核访问野指针会导致 **Oops** 甚至整机 panic。
5. **处处有并发**：多核、中断、抢占都可能让你的代码被同时执行（第 6 课细讲）。
6. **出错必须清理**：init 函数里申请的资源，在失败路径和 exit 函数里都要释放干净，否则内存泄漏只能靠重启解决。

---

## 六、常见问题速查

| 现象 | 原因 | 解决 |
|------|------|------|
| `make: *** /lib/modules/xxx/build: No such file or directory` | 没装 headers | 按 3.2 节安装，或改走路线 B |
| `insmod: ERROR: could not insert module hello.ko: Invalid module format` | vermagic 不匹配，或架构不对 | 用 `dmesg` 看具体原因，用 `modinfo hello.ko` 对比 `uname -r` |
| `insmod: ERROR: ... Operation not permitted` | 没用 root 权限，或内核强制要求模块签名 | 加 `sudo`；检查 `CONFIG_MODULE_SIG_FORCE` |
| `insmod: ERROR: ... Unknown symbol in module` | 模块用到了内核没有导出的符号 | `dmesg` 查看是哪个符号 |
| dmesg 出现 `loading out-of-tree module taints kernel.` | 加载了内核源码树之外的模块 | 正常现象，学习阶段可以忽略 |
| `Makefile:x: *** missing separator.  Stop.` | 命令行前面用了空格 | 改成 Tab |

---

## 七、理论题

在 `homework/answers.md` 中作答。用自己的话回答，可以查资料，但要能讲清楚。

1. 用户在 shell 中执行 `echo 1 > /dev/xxx`，从 shell 到驱动，大致经过了哪些环节？写出关键步骤即可。
2. 在你的板子上分别找出**字符设备、块设备、网络设备**各一个实例（写出你用的命令和结果），并说明网络设备为什么没有 `/dev` 节点。
3. 驱动编进内核（`=y`）和编成模块（`=m`）各有什么优缺点？什么情况下驱动**必须**编进内核？
4. `module_init` 指定的函数返回负值会发生什么？此时 `module_exit` 指定的函数会被调用吗？为什么？
5. 什么是 vermagic？
   - (a) 为什么在 x86 PC 上用 PC 自己的内核头文件编译出的 `.ko` 不能在 RK3399 上加载？
   - (b) 即使都是 arm64，只要内核版本字符串不同就不能加载，内核为什么要这么"严格"？
6. 去掉 `MODULE_LICENSE("GPL")` 会发生什么？（先动手做实验 3-b，再根据实验结果回答）
7. `/proc/sys/kernel/printk` 中 4 个数字分别是什么含义？为什么 `pr_debug()` 的内容默认看不到？
8. 内核代码为什么不能调用 `printf` / `malloc`？为什么不能随意使用浮点运算？
9. 为什么强烈建议接调试串口？RK3399 调试串口默认波特率是多少？如果串口终端显示一堆乱码，最可能是什么原因？

---

## 八、实操作业

### 作业 1：板子信息采集 → `homework/board-info.md`

在板子上执行以下命令，把**命令和完整输出**整理到 `board-info.md`。这些信息决定了后续课程的内容和硬件实验安排。

```bash
cat /proc/device-tree/model; echo
cat /proc/device-tree/compatible | tr '\0' '\n'
cat /etc/os-release
uname -a
cat /proc/version
nproc; cat /proc/cpuinfo | grep -E 'CPU part' | sort | uniq -c
free -h; df -h /
ls -l /lib/modules/$(uname -r)/build
gcc --version | head -1
# 内核配置：优先用 /proc/config.gz，没有的话用 /boot/config-$(uname -r)
PAT='CONFIG_MODULES=|CONFIG_MODULE_UNLOAD|CONFIG_MODVERSIONS|CONFIG_MODULE_SIG|CONFIG_DYNAMIC_DEBUG=|CONFIG_IKCONFIG|CONFIG_LOCALVERSION'
zcat /proc/config.gz 2>/dev/null | grep -E "$PAT" || grep -E "$PAT" /boot/config-$(uname -r)
cat /proc/sys/kernel/printk
ls /sys/class/leds/
ls /dev/i2c-* /dev/spidev* 2>/dev/null
sudo cat /sys/kernel/debug/gpio | head -50
```

并用文字补充说明：

- 你的**编译方式**（路线 A 还是 B），搭建过程中遇到了什么问题、怎么解决的
- 是否已接好**调试串口**（没有的话写计划什么时候接）
- 你手头有哪些可用的**外设**：LED、按键、传感器（写具体型号）、SPI 模块、逻辑分析仪等

### 作业 2：hello 模块 → `homework/hello/`

在 4.1 节示例的基础上，编写 `hello.c` 和 `Makefile`，要求：

1. **加载时**打印一行，包含：
   - 当前运行的内核版本字符串（和 `uname -r` 的结果相同）
   - 执行加载操作的进程名和 PID（想一想：这个进程是谁？）
2. **卸载时**打印模块从加载到卸载一共"存活"了多少毫秒。
3. 所有打印自动带 `hello: ` 前缀（使用 `pr_fmt`，不要在每条打印里手写前缀）。
4. Makefile 同时支持板上编译和交叉编译（`KDIR` 可在命令行覆盖）。
5. 代码符合内核编码风格：Tab 缩进、SPDX 头、函数和全局变量都是 `static`。

> 提示：去 [elixir.bootlin.com](https://elixir.bootlin.com/) 查这些东西（记得切换到你的内核版本）：
> `<linux/utsname.h>` 中的 `utsname()`；`<linux/sched.h>` 中的 `current`；`<linux/jiffies.h>` 中的 `jiffies` 和 `jiffies_to_msecs()`。

在 `homework/report.md` 中记录：编译命令和输出、`modinfo hello.ko`、`insmod`、`lsmod | grep hello`、`ls /sys/module/hello/`、`rmmod`，以及对应的 `dmesg` 输出。

### 作业 3：故意犯错实验 → 记录在 `homework/report.md`

每个实验都要记录**操作、现象（完整输出）、你的解释**。做完一个实验后，把代码恢复正常再做下一个。

- **(a) init 失败**：让 `hello_init` 返回 `-ENODEV`，重新编译并 `insmod`。记录 `insmod` 的报错和 `dmesg` 输出。此时 `lsmod` 能看到这个模块吗？报错信息里的文字和 `-ENODEV` 有什么关系？
- **(b) 去掉许可证**：删掉 `MODULE_LICENSE` 这一行再编译。记录现象：编译报错，还是能编译但加载时有提示？（不同内核版本的表现不同，如实记录你的即可）
- **(c) tainted 标志**：在一个**新启动**的系统上，记录加载 hello 模块前后 `cat /proc/sys/kernel/tainted` 的值。查阅内核文档 [Tainted kernels](https://docs.kernel.org/admin-guide/tainted-kernels.html)，解释这个数值中每个被置位的 bit 的含义。
- **(d) 看见 pr_debug**：在模块里加一条 `pr_debug()`，确认默认在 `dmesg` 中看不到它；再用**至少一种方法**让它显示出来，写清楚你用的方法和原理。

### 选做（加分项）

- **多文件模块**：把 hello 拆成 `hello_main.c` 和 `hello_util.c` 两个源文件（比如把"计算存活时间"的函数放进 `hello_util.c`），最终仍然编译出**一个** `hello.ko`。
  思考：为什么这时不能再有一个叫 `hello.c` 的源文件？

---

## 九、提交清单

```
lessons/lesson01/homework/
├── answers.md          # 理论题 1～9
├── board-info.md       # 作业 1
├── report.md           # 作业 2、3 的操作记录和分析
└── hello/
    ├── hello.c         # 作业 2（选做时换成 hello_main.c + hello_util.c）
    └── Makefile
```

提交后在对话中说：**"第 1 课作业已提交，请验收"**。

## 十、验收标准

必须**全部满足**才算通过：

- [ ] `answers.md`：9 道理论题全部作答，核心概念正确
- [ ] `board-info.md`：命令输出完整，编译方式、串口情况、外设清单都有说明
- [ ] `hello.c`：满足作业 2 的全部 5 条要求，能编译、能加载、能卸载
- [ ] `Makefile`：支持通过 `KDIR` 覆盖构建目录
- [ ] `report.md`：作业 2 的完整操作日志；作业 3 的 (a)(b)(c)(d) 四个实验都有现象记录和自己的解释
- [ ] 代码通过内核风格检查 `checkpatch.pl`，没有 ERROR（验收时老师会跑）
- [ ] 答辩：验收时老师追问的问题能答上来

选做完成的，会在验收报告中注明加分。
