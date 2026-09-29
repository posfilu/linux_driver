# RK3399 Linux 驱动开发课程

在 RK3399 开发板上，从零开始系统学习 Linux 设备驱动开发。

- **老师**：Claude，负责讲课、出题、布置作业、验收
- **学生**：你，负责完成理论题和动手作业，并在板子上实际运行
- **规则**：每一课验收通过后，才发布下一课

---

## 学习流程

```
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │ 1. 发布讲义 │ → │ 2. 学习+作业 │ → │ 3. 提交验收 │ → │ 4. 老师验收 │
 └──────────┘    └──────────┘    └──────────┘    └────┬─────┘
       ↑                                              │
       └──────────── 通过：发布下一课 ◄─────────────────┤
                     不通过：按 review.md 修改后重新提交 ◄─┘
```

1. **发布讲义**：老师在 `lessons/lessonXX/README.md` 发布本课讲义、理论题、实操作业和验收标准。
2. **学习 + 做作业**：你阅读讲义，在板子上动手，把作业放进 `lessons/lessonXX/homework/`。
3. **提交验收**：`git push` 之后，在对话里说一句：**"第 X 课作业已提交，请验收"**。
4. **老师验收**：老师审阅代码、答案和运行日志，写出 `lessons/lessonXX/review.md`：
   - ✅ **通过**：更新 [PROGRESS.md](PROGRESS.md)，发布下一课。
   - ❌ **未通过**：review.md 里列出必须修改的问题，你改完后再次提交。
   - 验收时老师可能追问 1～2 个问题（相当于"答辩"），答上来才算通过。

## 作业提交约定

每课的作业目录结构（具体文件以每课讲义为准）：

```
lessons/lessonXX/
├── README.md          # 讲义（老师写）
├── review.md          # 验收报告（老师写）
└── homework/          # 你的作业（你写）
    ├── answers.md     # 理论题答案
    ├── report.md      # 实验过程记录：执行的命令 + 终端输出 + 你的分析
    └── <代码目录>/     # 驱动源码、Makefile、用户态测试程序、设备树片段等
```

提交要求：

- **日志直接粘贴文本**，放在 ```` ``` ```` 代码块里，不要截图（老师要能逐行看）。
- **不要提交编译产物**（`.ko`、`.o`、`.mod.c` 等），仓库根目录的 `.gitignore` 已经帮你排除了。
- **答案用自己的话写**。可以查资料，但要能讲清楚，验收时会追问。
- 遇到解决不了的问题，把现象、命令和完整报错贴出来问老师，不要卡太久。

## 课程大纲

共 4 个阶段、19 课。前两个阶段基本不需要外设，后两个阶段开始接真实硬件。

### 第一阶段：内核模块入门

| 课次 | 主题 | 核心内容 |
|:---:|------|---------|
| 01 | 开发环境搭建与第一个内核模块 | 驱动在系统中的位置、板上编译 / 交叉编译、Kbuild、insmod/rmmod、printk、vermagic |
| 02 | 模块参数、符号导出与内核日志 | `module_param`、`EXPORT_SYMBOL(_GPL)`、模块依赖与 modprobe、多文件模块、dynamic debug |
| 03 | 内核编程基础 | kmalloc/kzalloc/vmalloc、GFP 标志、内核链表 `list_head`、错误码与 `ERR_PTR`、goto 错误回滚 |

### 第二阶段：字符设备驱动

| 课次 | 主题 | 核心内容 |
|:---:|------|---------|
| 04 | 字符设备驱动框架 | 设备号 `dev_t`、`cdev`、`file_operations`、class/device 自动创建 `/dev` 节点 |
| 05 | 用户空间数据交互 | read/write/llseek、`copy_to_user`/`copy_from_user`、ioctl 命令设计、misc 设备 |
| 06 | 并发与竞态 | 原子操作、自旋锁、互斥锁、读写锁；如何复现和消除竞态 |
| 07 | 阻塞 IO 与 poll | 等待队列、阻塞/非阻塞读写、poll/select/epoll、异步通知 fasync |
| 08 | 时间、延迟与延后执行 | jiffies、内核定时器、hrtimer、msleep/udelay、工作队列 workqueue |
| 09 | sysfs / procfs / debugfs | 设备属性 `DEVICE_ATTR`、seq_file、debugfs 调试接口 |

### 第三阶段：设备树与 RK3399 硬件驱动

| 课次 | 主题 | 核心内容 | 需要的硬件 |
|:---:|------|---------|-----------|
| 10 | 设备树与 platform 驱动 | dts/dtsi/dtb、节点与属性、`compatible` 匹配、platform_driver、devm_ 资源管理、设备树 overlay | 无 |
| 11 | pinctrl 与 GPIO 子系统 | RK3399 引脚复用、gpiod API、LED 驱动 | LED + 限流电阻（或板载 LED） |
| 12 | 中断处理 | request_irq / threaded irq、上半部/下半部、按键消抖 | 按键 |
| 13 | input 子系统 | input_dev、按键事件上报、evtest 测试 | 按键 |
| 14 | I2C 驱动 | i2c_driver、i2c_transfer / SMBus、regmap、传感器驱动 | I2C 传感器（如 AHT20 / SHT30 / BMP280 / MPU6050） |
| 15 | SPI 驱动 | spi_driver、spi_sync、spidev 对比 | SPI 设备（可选，如 SPI OLED） |
| 16 | IIO 子系统 | IIO 框架、RK3399 SARADC、把 I2C 传感器改写成 IIO 驱动 | 同第 14 课 |

### 第四阶段：进阶与综合

| 课次 | 主题 | 核心内容 |
|:---:|------|---------|
| 17 | 内核调试技术 | 分析 Oops、addr2line、ftrace、dynamic debug、调试串口 |
| 18 | mmap 与 DMA 基础 | 内存映射、`remap_pfn_range`、DMA API 基础 |
| 19 | 综合项目 | 一个完整的外设驱动：设备树 + 总线驱动 + 子系统接口 + 用户态程序 + 文档 |

> 大纲会根据你的板子型号、内核版本和手头的外设调整。第 1 课作业会采集这些信息。

## 硬件准备清单

| 物品 | 必要性 | 说明 |
|------|:----:|------|
| RK3399 开发板 + 电源 | 必需 | 能正常启动 Linux |
| USB 转 TTL 串口线 | **强烈推荐** | 看内核启动日志和崩溃信息。RK3399 调试串口默认波特率 **1500000**，要选支持该速率的芯片（CP2102 / CH340 / FT232 等），电平 **3.3V** |
| LED + 330Ω～1kΩ 电阻 | 第 11 课 | 很多板子有板载 LED 可以直接用 |
| 轻触按键 | 第 12 课 | |
| I2C 传感器模块 | 第 14 课 | 任选一个你有的即可 |
| SPI 模块 | 可选 | |
| 杜邦线、面包板 | 推荐 | |
| 逻辑分析仪 | 可选 | 调试 I2C/SPI 时序非常有用，几十元的就够 |

> ⚠️ **电平安全**：RK3399 的 IO 是 3.3V 或 1.8V（取决于引脚所在的电源域），**绝对不能直接接 5V 信号**。接线前先查你板子的引脚图或原理图，确认该引脚的电平。

## 参考资料

- 《Linux Device Drivers, 3rd Edition》（LDD3）：经典，API 有些过时，但原理仍然适用
- 内核源码自带文档：`Documentation/driver-api/`，在线版 https://docs.kernel.org/
- 在线查内核源码：https://elixir.bootlin.com/ （可切换内核版本，查 API 非常方便）
- Bootlin 培训资料（免费）：https://bootlin.com/training/kernel/
- RK3399 数据手册 / TRM（Technical Reference Manual）：查寄存器、引脚复用时使用

## 当前进度

见 [PROGRESS.md](PROGRESS.md)。
