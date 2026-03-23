# Linux 内核启动流程深度研究报告

> **摘要**：本报告对 Linux 内核（以 Linux 7.0-rc5 为参考版本）的完整启动流程进行深度研究。报告从整体框架出发，
> 分别针对 **x86/x86_64**、**ARM64**、**ARM32**、**RISC-V** 四大主流架构展开详述，
> 随后逐一解析各关键子系统的初始化过程，最后以一张包含所有细节的完整启动流程图作为总结。
> 所有流程节点均标有编号，并附有清晰的文字说明。

---

## 目录

1. [第一部分：整体启动框架（大框架）](#第一部分整体启动框架大框架)
2. [第二部分：架构相关启动流程](#第二部分架构相关启动流程)
   - 2.1 [x86 / x86_64 启动流程](#21-x86--x86_64-启动流程)
   - 2.2 [ARM64（AArch64）启动流程](#22-arm64aarch64-启动流程)
   - 2.3 [ARM32 启动流程](#23-arm32-启动流程)
   - 2.4 [RISC-V 启动流程](#24-risc-v-启动流程)
3. [第三部分：子系统初始化详解](#第三部分子系统初始化详解)
   - 3.1 [内存管理子系统](#31-内存管理子系统)
   - 3.2 [调度器子系统](#32-调度器子系统)
   - 3.3 [中断与定时子系统](#33-中断与定时子系统)
   - 3.4 [虚拟文件系统（VFS）](#34-虚拟文件系统vfs)
   - 3.5 [网络子系统](#35-网络子系统)
   - 3.6 [设备驱动模型（Driver Core）](#36-设备驱动模型driver-core)
4. [第四部分：完整详细启动流程总览](#第四部分完整详细启动流程总览)
5. [参考源码位置](#参考源码位置)

---

## 第一部分：整体启动框架（大框架）

Linux 内核的启动从加电（Power-On）到进入用户空间（Userspace），可以划分为以下五个大阶段：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Linux 内核启动整体流程                              │
│                                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │  阶段 1  │   │  阶段 2  │   │    阶段 3    │   │    阶段 4/5     │  │
│  │  固件层  │──▶│ 引导加载 │──▶│   内核早期   │──▶│ 内核通用初始化 │  │
│  │(Firmware)│   │(Bootloader)│  │  架构初始化  │   │ + 用户空间启动 │  │
│  └──────────┘   └──────────┘   └──────────────┘   └─────────────────┘  │
│                                                                         │
│  时序：上电 → UEFI/BIOS → GRUB/U-Boot → head.S → start_kernel() → /sbin/init
└─────────────────────────────────────────────────────────────────────────┘
```

### 整体启动时序图

```
时间轴 ──────────────────────────────────────────────────────────────────▶

阶段①        阶段②          阶段③              阶段④              阶段⑤
硬件固件      引导加载器      内核早期初始化      内核通用初始化      用户空间

┌──────┐    ┌──────────┐   ┌──────────────┐   ┌────────────────┐  ┌──────────┐
│UEFI/ │    │GRUB2/    │   │arch/*/boot/  │   │init/main.c     │  │/sbin/init│
│BIOS  │    │U-Boot/   │   │head.S        │   │start_kernel()  │  │systemd/  │
│POST  │    │Das U-Boot│   │(汇编，MMU关) │   │(C语言，通用)   │  │SysVinit  │
└──┬───┘    └────┬─────┘   └──────┬───────┘   └───────┬────────┘  └──────────┘
   │              │               │                    │
   │① 自检POST   │               │                    │
   │② 初始化硬件  │               │                    │
   │③ 定位Boot设备│              │                    │
   │──────────────▶              │                    │
   │              │④ 加载内核镜像 │                    │
   │              │⑤ 解压 vmlinuz│                    │
   │              │⑥ 传递cmdline │                    │
   │              │⑦ 跳转到内核入│口                  │
   │              │──────────────▶                    │
   │              │              │⑧ CPU基础初始化     │
   │              │              │⑨ 建立临时页表      │
   │              │              │⑩ 开启MMU           │
   │              │              │⑪ 建立内核栈        │
   │              │              │⑫ 跳入C语言入口     │
   │              │              │────────────────────▶│
   │              │              │                    │⑬ setup_arch()
   │              │              │                    │⑭ 内存初始化
   │              │              │                    │⑮ 调度器初始化
   │              │              │                    │⑯ 中断/定时初始化
   │              │              │                    │⑰ VFS/网络初始化
   │              │              │                    │⑱ rest_init()
   │              │              │                    │─────────────▶
   │              │              │                    │             ⑲ PID 1
   │              │              │                    │             ⑳ /sbin/init
```

### 五大阶段说明

| 编号 | 阶段名称 | 运行位置 | 主要任务 |
|------|---------|---------|---------|
| ① | 固件（Firmware）| ROM/Flash | POST自检、硬件枚举、寻找启动设备 |
| ② | 引导加载器（Bootloader）| RAM | 加载并解压内核镜像，传递参数 |
| ③ | 内核早期初始化 | arch汇编代码 | CPU/MMU初始化，建立最初页表，跳入C代码 |
| ④ | 内核通用初始化 | `init/main.c` | 内存/调度/驱动/VFS全面初始化 |
| ⑤ | 用户空间启动 | 用户态 | 执行 `/sbin/init`，启动系统服务 |

---

## 第二部分：架构相关启动流程

### 2.1 x86 / x86_64 启动流程

x86 架构启动过程历史悠久，包含实模式（Real Mode）→ 保护模式（Protected Mode）→ 长模式（Long Mode，64位）的切换，是所有架构中最复杂的。

#### x86_64 启动时序图

```
      BIOS/UEFI                  GRUB2                  内核汇编层               内核C层
         │                         │                        │                       │
  ①      │ 加电，执行0xFFFF_FFF0   │                        │                       │
         │ (Reset Vector)          │                        │                       │
  ②      │ POST：内存/CPU检测      │                        │                       │
         │                         │                        │                       │
  ③      │ 搜索可引导设备(MBR/GPT) │                        │                       │
         │─────────────────────────▶                        │                       │
  ④      │                         │ 加载 stage1/stage2     │                       │
         │                         │ 读取 grub.cfg          │                       │
  ⑤      │                         │ 加载 vmlinuz + initrd  │                       │
         │                         │ 解压内核到内存          │                       │
  ⑥      │                         │ 设置 boot_params 结构体│                       │
  ⑦      │                         │ 跳转至 header.S 入口   │                       │
         │                         │─────────────────────────▶                      │
  ⑧      │                         │            [实模式 16-bit]                     │
         │                         │            arch/x86/boot/header.S              │
         │                         │            执行 bootsect 代码                  │
  ⑨      │                         │            arch/x86/boot/main.c               │
         │                         │            检测内存 (e820), BIOS询问           │
  ⑩      │                         │            arch/x86/boot/compressed/head_64.S  │
         │                         │            切换至保护模式（32-bit）             │
  ⑪      │                         │            解压 vmlinux（内核自解压）           │
  ⑫      │                         │            [长模式 64-bit]                     │
         │                         │            arch/x86/kernel/head_64.S           │
         │                         │            startup_64:                          │
         │                         │              - 建立初始页表(4级/5级分页)        │
         │                         │              - 加载 GDT/IDT                    │
         │                         │              - 启用 CR0/CR4/EFER 寄存器        │
         │                         │              - 设置 %gs 指向 per-CPU 区域      │
         │                         │              - 跳入 x86_64_start_kernel        │
         │                         │─────────────────────────────────────────────────▶
  ⑬      │                         │                                        arch/x86/kernel/head64.c
         │                         │                                        x86_64_start_kernel():
  ⑭      │                         │                                          clear_bss()
  ⑮      │                         │                                          copy_bootdata()
  ⑯      │                         │                                          load_ucode_bsp()
  ⑰      │                         │                                          init_top_pgt()
  ⑱      │                         │                                          x86_64_start_reservations()
  ⑲      │                         │                                          start_kernel()  ──▶ [通用阶段]
```

#### x86_64 关键寄存器状态变化

```
阶段        CR0[PG]  CR4[PAE]  EFER[LME]  地址模式
────────────────────────────────────────────────────
复位后         0        0         0        实模式16位
开启保护模式    1        0         0        保护模式32位
开启PAE        1        1         0        PAE保护模式
开启长模式      1        1         1        IA-32e 64位
```

#### UEFI 启动（EFI Stub）分支

当内核以 EFI Stub 方式启动时（无需 GRUB），流程有所不同：

```
  UEFI固件
     │
  ① UEFI 加载 vmlinuz（识别为 PE 格式 EFI 应用）
     │
  ② 执行 arch/x86/boot/header.S 中的 EFI 头部
     │  (IMAGE_DOS_SIGNATURE = "MZ", 指向 PE header)
     │
  ③ efi_stub_entry → efi_main()  [drivers/firmware/efi/libstub]
     │  - 分配内存
     │  - 解析 FDT / ACPI 表
     │  - 退出 UEFI Boot Services
     │
  ④ 跳转至 startup_64 → x86_64_start_kernel → start_kernel
```

---

### 2.2 ARM64（AArch64）启动流程

ARM64 启动相对简洁，通常由 U-Boot 或 UEFI 加载内核，内核通过 FDT（设备树）获取硬件描述。

#### ARM64 启动时序图

```
  固件/U-Boot                  内核汇编层                          内核C层
      │                           │                                   │
  ①  │ 加电，SoC ROM 执行        │                                   │
     │  BL1（固化在芯片中）        │                                   │
  ②  │ 加载 BL2（NAND/eMMC）     │                                   │
  ③  │ BL2 加载 BL31（EL3固件）  │                                   │
  ④  │ BL2 加载 BL32（可选OP-TEE）│                                   │
  ⑤  │ BL2 加载 BL33（U-Boot/UEFI）                                  │
     │ 跳转至 BL33 (EL2/EL1)     │                                   │
  ⑥  │ U-Boot: 初始化DDR,外设    │                                   │
  ⑦  │ U-Boot: 从存储加载 Image  │                                   │
  ⑧  │ U-Boot: 准备 bootargs     │                                   │
  ⑨  │ U-Boot: x0=FDT地址, 跳转  │                                   │
     │────────────────────────────▶                                   │
  ⑩  │               arch/arm64/kernel/head.S                        │
     │               SYM_CODE_START(primary_entry)                    │
     │               要求：MMU=OFF, D-cache=OFF                       │
     │               x0 = FDT 物理地址                                │
  ⑪  │               preserve_boot_args()  保存x0-x3                 │
  ⑫  │               record_mmu_state()    记录MMU状态               │
  ⑬  │               el2_setup()           配置 EL2 (Hypervisor层)   │
  ⑭  │               __cpu_setup()         CPU特性检测(CPUID)         │
  ⑮  │               __create_page_tables() 建立初始页表             │
     │               (idmap + kernel mapping)                         │
  ⑯  │               __enable_mmu()        开启MMU                   │
  ⑰  │               __primary_switched()  MMU已开启                 │
     │                 - 清零BSS段                                    │
     │                 - 设置 init_task 的栈                          │
     │                 - 调用 start_kernel                            │
     │────────────────────────────────────────────────────────────────▶
  ⑱  │                                                     start_kernel()
     │                                                     [通用阶段，见第三部分]
```

#### ARM64 异常级别（Exception Level）说明

```
EL3 (最高特权)  ── TrustZone Secure Monitor（BL31，ATF）
EL2             ── Hypervisor / U-Boot / UEFI
EL1             ── Linux 内核（通常运行在 EL1）
EL0             ── 用户空间应用程序
```

内核在 `el2_setup()` 中协商运行级别：若硬件支持虚拟化且引导时处于 EL2，内核可配置为在 EL2 运行（KVM host）或从 EL2 降级到 EL1。

#### ARM64 次级 CPU（Secondary CPU）启动

```
  主 CPU（Bootstrap CPU）           次级 CPU（Secondary CPUs）
         │                                    │
  ①      │ smp_init()                         │（在 CPU_OFF 电源状态）
  ②      │ 通过 PSCI（Power State           │
         │   Coordination Interface）         │
         │ 发送 CPU_ON 命令                   │
         │───────────────────────────────────▶│
  ③      │                              secondary_entry:
         │                              secondary_startup()
  ④      │                              __cpu_setup()  CPU特性
  ⑤      │                              __enable_mmu() 开启MMU
  ⑥      │                              secondary_start_kernel()
  ⑦      │                              notify_cpu_starting()
  ⑧      │                              cpu_startup_entry() → idle loop
```

---

### 2.3 ARM32 启动流程

ARM32（传统 ARM，如 Cortex-A9/A15）架构已逐渐被 ARM64 取代，但在嵌入式领域仍广泛使用。

#### ARM32 启动时序图

```
  U-Boot / 固件                 压缩内核（zImage）           解压后的内核              内核C层
       │                              │                           │                      │
  ①   │ 准备:                        │                           │                      │
     │  r0 = 0                        │                           │                      │
     │  r1 = machine type ID          │                           │                      │
     │  r2 = ATAGs/DTB 地址           │                           │                      │
  ②  │ 跳转至 zImage 入口             │                           │                      │
     │──────────────────────────────▶ │                           │                      │
  ③  │              arch/arm/boot/compressed/head.S               │                      │
     │              确定自身位置（PIC代码）                        │                      │
  ④  │              建立解压缓冲区                                 │                      │
  ⑤  │              调用 decompress_kernel()                      │                      │
  ⑥  │              将vmlinux解压到RAM（通常 0x8000 偏移）        │                      │
  ⑦  │              跳转至解压后内核入口                          │                      │
     │              ─────────────────────────────────────────────▶│                      │
  ⑧  │                                              arch/arm/kernel/head.S               │
     │                                              ENTRY(stext)                         │
  ⑨  │                                              safe_svcmode_maskall  切换SVC模式   │
  ⑩  │                                              __lookup_processor_type  CPU类型查找 │
  ⑪  │                                              __vet_atags  校验 ATAGs/DTB           │
  ⑫  │                                              __create_page_tables  建立页表       │
  ⑬  │                                              __enable_mmu  开启MMU                │
  ⑭  │                                              __mmap_switched  MMU后处理           │
     │                                                - 复制数据段(.data)                 │
     │                                                - 清零 BSS                          │
     │                                                - 跳入 start_kernel                │
     │                                              ─────────────────────────────────────▶
  ⑮  │                                                                           start_kernel()
```

---

### 2.4 RISC-V 启动流程

RISC-V 是开放指令集架构，Linux 支持 RV32 和 RV64，启动流程通过 OpenSBI 固件进行抽象。

#### RISC-V 启动时序图

```
  硬件 ROM              OpenSBI / M-Mode固件         Linux内核（S-Mode）           内核C层
      │                         │                          │                          │
  ①  │ 上电，执行 MROM          │                          │                          │
     │ (Machine ROM)            │                          │                          │
  ②  │ 加载 OpenSBI             │                          │                          │
     │ ─────────────────────────▶                          │                          │
  ③  │             OpenSBI 初始化 M-Mode                   │                          │
     │             - 初始化UART(早期调试输出)               │                          │
     │             - 设置 PMP（物理内存保护）               │                          │
     │             - 初始化 SBI（Supervisor Binary Interface）                        │
  ④  │             从存储加载 Linux Image                   │                          │
  ⑤  │             a0 = hartid（硬件线程ID）               │                          │
     │             a1 = DTB物理地址                         │                          │
     │             切换至 S-Mode，跳转内核                  │                          │
     │             ─────────────────────────────────────────▶                         │
  ⑥  │                                         arch/riscv/kernel/head.S              │
     │                                         SYM_CODE_START(_start)                │
  ⑦  │                                         j _start_kernel                       │
     │                                         SYM_CODE_START(_start_kernel)         │
  ⑧  │                                         - 禁用中断（csrw sie, zero）          │
  ⑨  │                                         - 设置全局指针寄存器 gp               │
  ⑩  │                                         - 检测 hart（CPU核）是否为主核        │
  ⑪  │                                         - setup_vm()  建立初始虚拟内存        │
  ⑫  │                                         - relocate()  跳至虚拟地址执行        │
  ⑬  │                                         - setup_trap_vector() 设置陷阱向量    │
  ⑭  │                                         - 设置初始内核栈                      │
  ⑮  │                                         - tail start_kernel                   │
     │                                         ─────────────────────────────────────▶
  ⑯  │                                                                       start_kernel()
```

#### RISC-V 特权级别说明

```
M-Mode（Machine Mode）      ── 最高特权，OpenSBI 固件运行于此
S-Mode（Supervisor Mode）   ── Linux 内核运行于此（通过 SBI ecall 访问M-Mode服务）
U-Mode（User Mode）         ── 用户空间应用程序
```

RISC-V 通过 **SBI（Supervisor Binary Interface）** 提供统一的固件服务接口，类似于 x86 的 BIOS 中断或 ARM 的 PSCI。

---

## 第三部分：子系统初始化详解

所有架构汇聚于 `start_kernel()`（`init/main.c:1008`）后，进入架构无关的通用初始化阶段。以下按调用顺序详述各子系统。

### start_kernel() 完整调用序列（含行号）

```
init/main.c: start_kernel()
│
├── ① set_task_stack_end_magic(&init_task)    # 设置init进程栈魔数（栈溢出检测）
├── ② smp_setup_processor_id()                # 设置启动CPU的逻辑ID
├── ③ debug_objects_early_init()              # 调试对象早期初始化
├── ④ init_vmlinux_build_id()                 # 记录内核构建ID
├── ⑤ cgroup_init_early()                     # cgroup子系统早期初始化
├── ⑥ local_irq_disable()                     # 关中断（整个初始化过程中断关闭）
├── ⑦ boot_cpu_init()                         # 标记启动CPU为在线/活跃/存在
├── ⑧ page_address_init()                     # 初始化高端内存页地址哈希表
├── ⑨ setup_arch(&command_line)               # 【架构特定初始化，最重要的函数之一】
├── ⑩ mm_core_init_early()                    # 内存核心早期初始化
├── ⑪ jump_label_init()                       # 静态键（static keys）初始化
├── ⑫ static_call_init()                      # 静态调用初始化
├── ⑬ early_security_init()                   # LSM 早期安全初始化
├── ⑭ setup_boot_config()                     # 解析 bootconfig
├── ⑮ setup_command_line(command_line)        # 保存命令行参数
├── ⑯ setup_nr_cpu_ids()                      # 确定CPU数量
├── ⑰ setup_per_cpu_areas()                   # 分配 per-CPU 变量区域
├── ⑱ smp_prepare_boot_cpu()                  # 架构相关的启动CPU准备
├── ⑲ early_numa_node_init()                  # NUMA节点早期初始化
├── ⑳ boot_cpu_hotplug_init()                 # CPU热插拔初始化
├── ㉑ parse_early_param()                     # 解析 early_param 参数
├── ㉒ random_init_early()                     # 随机数早期初始化（KASLR支持）
├── ㉓ setup_log_buf(0)                        # 初始化 printk 日志缓冲区
├── ㉔ vfs_caches_init_early()                 # VFS 早期缓存（dcache, inode cache）
├── ㉕ sort_main_extable()                     # 排序异常表（用于fixup）
├── ㉖ trap_init()                             # 初始化 CPU 陷阱/异常向量
├── ㉗ mm_core_init()                          # 内存核心完整初始化（buddy, slab）
├── ㉘ maple_tree_init()                       # Maple Tree（VMA管理数据结构）初始化
├── ㉙ poking_init()                           # 内核代码修改（kprobes等）初始化
├── ㉚ ftrace_init()                           # 函数跟踪框架初始化
├── ㉛ early_trace_init()                      # 早期 tracing 初始化
├── ㉜ sched_init()                            # 【调度器初始化】
├── ㉝ radix_tree_init()                       # Radix 树初始化
├── ㉞ housekeeping_init()                     # CPU隔离/housekeeping 初始化
├── ㉟ workqueue_init_early()                  # 工作队列早期初始化
├── ㊱ rcu_init()                              # RCU（Read-Copy-Update）初始化
├── ㊲ kvfree_rcu_init()                       # kfree RCU 初始化
├── ㊳ trace_init()                            # Trace 事件初始化
├── ㊴ context_tracking_init()                 # 上下文跟踪初始化
├── ㊵ early_irq_init()                        # 早期中断控制器初始化
├── ㊶ init_IRQ()                              # 【中断系统完整初始化】
├── ㊷ tick_init()                             # 时钟滴答初始化
├── ㊸ rcu_init_nohz()                         # RCU nohz 初始化
├── ㊹ timers_init()                           # 低精度定时器初始化
├── ㊺ srcu_init()                             # SRCU 初始化
├── ㊻ hrtimers_init()                         # 高精度定时器初始化
├── ㊼ softirq_init()                          # 软中断初始化
├── ㊽ timekeeping_init()                      # 时间维护系统初始化
├── ㊾ time_init()                             # 架构相关时钟初始化
├── ㊿ random_init()                           # 随机数完整初始化（/dev/random）
├── ㊿+1 kfence_init()                         # KFENCE 内存安全检测初始化
├── ㊿+2 boot_init_stack_canary()              # 栈保护金丝雀值初始化
├── ㊿+3 perf_event_init()                     # 性能事件初始化
├── ㊿+4 profile_init()                        # 内核 profiling 初始化
├── ㊿+5 call_function_init()                  # SMP 函数调用初始化
├── ㊿+6 local_irq_enable()                    # 【开启中断！】
├── ㊿+7 kmem_cache_init_late()                # slab 分配器后期初始化
├── ㊿+8 console_init()                        # 【控制台初始化，开始输出日志】
├── ㊿+9 lockdep_init()                        # 死锁检测初始化
├── ㊿+10 setup_per_cpu_pageset()              # per-CPU 页集合初始化
├── ㊿+11 numa_policy_init()                   # NUMA 内存策略初始化
├── ㊿+12 acpi_early_init()                    # ACPI 早期初始化
├── ㊿+13 calibrate_delay()                    # 校准 BogoMIPS（延迟循环）
├── ㊿+14 pid_idr_init()                       # PID 分配器初始化
├── ㊿+15 anon_vma_init()                      # 匿名VMA初始化
├── ㊿+16 cred_init()                          # 进程凭证初始化
├── ㊿+17 fork_init()                          # fork 子系统初始化
├── ㊿+18 proc_caches_init()                   # 进程相关 slab 缓存
├── ㊿+19 key_init()                           # 内核密钥环初始化
├── ㊿+20 security_init()                      # LSM 安全框架完整初始化
├── ㊿+21 vfs_caches_init()                    # 【VFS 完整初始化】
├── ㊿+22 pagecache_init()                     # 页缓存初始化
├── ㊿+23 signals_init()                       # 信号机制初始化
├── ㊿+24 proc_root_init()                     # /proc 文件系统根初始化
├── ㊿+25 nsfs_init()                          # 命名空间文件系统初始化
├── ㊿+26 cpuset_init()                        # cpuset 初始化
├── ㊿+27 cgroup_init()                        # cgroup 完整初始化
├── ㊿+28 acpi_subsystem_init()                # ACPI 子系统完整初始化
└── ㊿+29 rest_init()                          # 【创建init进程，进入idle】
```

---

### 3.1 内存管理子系统

内存管理经历三个演进阶段：

```
启动早期                           内存管理演进                           运行时
   │                                                                        │
   │  阶段A: memblock 分配器                                                │
   │  ─────────────────────────────────────────────────────────────         │
   │  ① setup_arch() 调用 e820/DTB 解析物理内存布局                       │
   │  ② memblock_add() 注册可用内存区域                                   │
   │  ③ memblock_reserve() 保留内核代码/数据/固件区域                     │
   │  ④ memblock_alloc() 用于早期内存分配（页表、per-CPU等）              │
   │                                                                        │
   │  阶段B: 页分配器（Buddy System）初始化                                │
   │  ─────────────────────────────────────────────────────────────         │
   │  ⑤ mm_core_init() → free_area_init() 初始化 zone（DMA/Normal/High）  │
   │  ⑥ mem_init() 将 memblock 管理的页移交给 buddy 系统                 │
   │  ⑦ 每个 zone 初始化 free_area[MAX_ORDER] 链表                       │
   │  ⑧ page_alloc_init_late() 后期页分配器初始化                        │
   │                                                                        │
   │  阶段C: Slab/Slub 分配器初始化                                       │
   │  ─────────────────────────────────────────────────────────────         │
   │  ⑨ kmem_cache_init() 初始化 kmalloc 缓存（8B, 16B, ... 8MB）        │
   │  ⑩ kmem_cache_init_late() slab 分配器完成初始化                     │
   │  ⑪ proc_caches_init() 创建 task_struct/mm_struct 等内核对象缓存     │
   └────────────────────────────────────────────────────────────────────────

内存区域布局（以 x86_64 为例）：
┌────────────────────────────────────────────────────────┐
│  虚拟地址空间（x86_64，48-bit）                         │
│                                                        │
│  0xFFFF_FFFF_FFFF_FFFF ┐                              │
│                         │ 内核空间（128 TB）           │
│  - vmalloc 区域         │                              │
│  - 内核直接映射区        │                              │
│  - 内核代码/数据        │                              │
│  0xFFFF_8000_0000_0000 ┘                              │
│         ...（不规范地址，canonical hole）               │
│  0x0000_7FFF_FFFF_FFFF ┐                              │
│                         │ 用户空间（128 TB）           │
│  0x0000_0000_0000_0000 ┘                              │
└────────────────────────────────────────────────────────┘
```

---

### 3.2 调度器子系统

```
sched_init() 初始化流程：

① 初始化 init_task（进程0，idle进程）的调度实体
② 初始化每个 CPU 的 runqueue（运行队列）
   - cfs_rq：完全公平调度队列（红黑树）
   - rt_rq：实时调度队列（优先级位图）
   - dl_rq：Deadline 调度队列
③ 初始化调度类（sched_class）
   - stop_sched_class    > 最高优先级（停止CPU的特殊进程）
   - dl_sched_class      > SCHED_DEADLINE（EDF算法）
   - rt_sched_class      > SCHED_FIFO / SCHED_RR
   - fair_sched_class    > SCHED_NORMAL / SCHED_BATCH（CFS）
   - idle_sched_class    > SCHED_IDLE
④ 初始化调度时钟（sched_clock_init）
⑤ kernel_init_freeable() 中调用 sched_init_smp()
   - 构建 CPU topology（物理核、超线程、NUMA节点）
   - 建立调度域（sched_domain）层次结构
   - 启用负载均衡

CFS 调度原理（简化）：
┌─────────────────────────────────────────┐
│          CFS 运行队列（红黑树）          │
│                                         │
│    小 vruntime ──▶ 大 vruntime          │
│    ┌───┐  ┌───┐  ┌───┐  ┌───┐         │
│    │ A │  │ B │  │ C │  │ D │         │
│    │vr=│  │vr=│  │vr=│  │vr=│         │
│    │100│  │200│  │350│  │500│         │
│    └───┘  └───┘  └───┘  └───┘         │
│      ↑                                  │
│    最左节点 = 下一个调度的进程           │
└─────────────────────────────────────────┘
```

---

### 3.3 中断与定时子系统

```
中断初始化序列：

① early_irq_init()
   - 分配 irq_desc 数组（或 radix tree）
   - 初始化稀疏 IRQ 描述符表

② init_IRQ()  [架构相关]
   - x86: 初始化 APIC（本地APIC + IO-APIC）
   - ARM64: 初始化 GIC（Generic Interrupt Controller）
   - RISC-V: 初始化 PLIC（Platform Level Interrupt Controller）
   - 注册 irq_chip 操作函数集

③ softirq_init()
   - 注册 TASKLET_SOFTIRQ、HI_SOFTIRQ 等
   - 初始化 per-CPU 软中断向量

④ timers_init() / hrtimers_init()
   - 初始化低精度定时器（时间轮算法）
   - 初始化高精度定时器（红黑树）

⑤ timekeeping_init()
   - 初始化系统时钟（xtime, wall_clock）
   - 初始化 clocksource（TSC/HPET/ARM arch timer等）

⑥ time_init() [架构相关]
   - x86: 初始化 PIT/HPET/TSC
   - ARM64: 初始化 arch_timer（ARM通用定时器）
   - 注册 clock_event_device

中断处理路径：
硬件中断 → IDT/IVT → do_IRQ() → irq_desc→handle_irq()
         → action→handler()     → softirq 处理
         → ret_from_intr        → 可能触发调度
```

---

### 3.4 虚拟文件系统（VFS）

```
VFS 初始化序列：

① vfs_caches_init_early()  [在内存初始化前]
   - 初始化 dcache（目录缓存）哈希表
   - 初始化 inode_hashtable

② vfs_caches_init()  [完整初始化]
   - names_cachep: 文件名 slab 缓存
   - filp_cachep: file 结构体 slab 缓存
   - dentry_cache: 目录项 slab 缓存
   - inode_cachep: inode slab 缓存
   - 调用 files_init() 初始化文件描述符表
   - 调用 mnt_init()  初始化 VFS 挂载框架

③ proc_root_init()
   - 注册 proc_fs_type
   - 创建 /proc 根目录
   - 挂载 procfs

④ kernel_init_freeable() → wait_for_initramfs()
   - 等待 initrd/initramfs 就绪

⑤ kernel_init_freeable() → prepare_namespace()
   - 挂载真正的根文件系统（从 cmdline 的 root= 参数）
   - 执行 /linuxrc 或 /init（如果是 initramfs）

VFS 层次结构：
┌──────────────────────────────────────────────┐
│           用户空间系统调用                    │
│    open() read() write() mkdir() ...         │
└─────────────────┬────────────────────────────┘
                  │ sys_call_table
┌─────────────────▼────────────────────────────┐
│              VFS 层（通用接口）               │
│  - file_operations                           │
│  - inode_operations                          │
│  - super_operations                          │
│  - dentry_operations                         │
└──┬──────────┬──────────┬───────────┬─────────┘
   │          │          │           │
┌──▼──┐  ┌───▼──┐  ┌────▼──┐  ┌────▼──┐
│ext4 │  │ xfs  │  │ tmpfs │  │ proc  │
│btrfs│  │ nfs  │  │ sysfs │  │ devfs │
└──┬──┘  └───┬──┘  └────┬──┘  └───────┘
   │          │          │
┌──▼──────────▼──────────▼──────────────────────┐
│           块设备层 / 网络层 / 内存层            │
└────────────────────────────────────────────────┘
```

---

### 3.5 网络子系统

```
网络子系统初始化（通过 do_initcalls() 机制）：

网络子系统不在 start_kernel() 中直接调用，而是通过 __initcall 宏注册，
在 do_basic_setup() → do_initcalls() 阶段按级别顺序执行。

initcall 级别（level 0-7）：
  core_initcall    (level 1): sock_init()  套接字核心
  postcore_initcall(level 2): net_ns_init() 网络命名空间
  arch_initcall    (level 3): 架构相关网络
  subsys_initcall  (level 4): net_dev_init() 网络设备框架
                              inet_init()    IPv4协议栈
  fs_initcall      (level 5): netfs 相关
  device_initcall  (level 6): 具体网卡驱动

① sock_init()
   - 初始化 net_families[] 协议族数组
   - 创建 sock slab 缓存

② net_dev_init()
   - 初始化 softnet_data（per-CPU 网络数据）
   - 注册 NET_TX_SOFTIRQ 和 NET_RX_SOFTIRQ
   - 初始化网络设备哈希表

③ inet_init()（IPv4 协议栈）
   - 注册 AF_INET 协议族
   - 注册 TCP, UDP, ICMP, RAW 协议
   - 初始化路由表（fib_hash_init）
   - 初始化 ARP 模块
   - 初始化 IP 分片重组

④ 网卡驱动（device_initcall）
   - PCI 总线扫描 → 匹配驱动 → probe()
   - 注册 net_device → 分配 TX/RX 队列
   - NAPI 初始化
```

---

### 3.6 设备驱动模型（Driver Core）

```
设备驱动框架初始化序列：

① driver_init()  [do_basic_setup() 中的第一个重要调用]
   ├── devtmpfs_init()    创建 devtmpfs（/dev 的基础）
   ├── devices_init()     初始化 /sys/devices
   ├── buses_init()       初始化 /sys/bus
   ├── classes_init()     初始化 /sys/class
   └── firmware_init()    初始化 /sys/firmware

② sysfs_init()           初始化 sysfs 文件系统

③ kobject_uevent_init()  初始化 uevent（热插拔通知机制）

④ 总线扫描（按 initcall 顺序）
   - platform_bus_init()  平台总线
   - pci_driver_init()    PCI 总线
   - usb_init()           USB 总线
   - i2c_init()           I2C 总线
   - spi_init()           SPI 总线

⑤ 设备探测（probe）
   - bus_for_each_drv() 遍历已注册驱动
   - bus_for_each_dev() 遍历已注册设备
   - driver_match_device() 匹配
   - driver_probe_device() 调用 drv->probe()

驱动绑定模型：
┌──────────────────────────────────────────────────────┐
│                    总线（bus）                        │
│                                                      │
│  ┌─────────────────┐       ┌─────────────────┐       │
│  │  设备（device）  │◀─────▶│  驱动（driver）  │       │
│  │ dev->bus = &bus  │ match │ drv->bus = &bus  │       │
│  │ 来自枚举/DTS/ACPI│       │ 来自 insmod/built-in│    │
│  └─────────────────┘       └─────────────────┘       │
│                    匹配成功 → probe()                 │
└──────────────────────────────────────────────────────┘
```

---

### rest_init() 与用户空间过渡

```
rest_init() 流程：

① rcu_scheduler_starting()           # 启动RCU调度
② user_mode_thread(kernel_init, ...)  # 创建 PID 1（init进程）
③ kernel_thread(kthreadd, ...)        # 创建 PID 2（kthreadd，内核线程守护进程）
④ system_state = SYSTEM_SCHEDULING    # 切换到调度状态
⑤ complete(&kthreadd_done)            # 通知init进程kthreadd就绪
⑥ schedule_preempt_disabled()         # 运行一次调度
⑦ cpu_startup_entry(CPUHP_ONLINE)     # 进入 idle 循环（CPU 0 永远循环于此）

kernel_init()（PID 1）：
① wait_for_completion(&kthreadd_done) # 等待kthreadd就绪
② kernel_init_freeable()              # SMP启动，驱动初始化，挂载根文件系统
③ async_synchronize_full()            # 等待所有异步init完成
④ free_initmem()                      # 释放 __init 代码段内存
⑤ mark_readonly()                     # 标记内核只读区域（STRICT_KERNEL_RWX）
⑥ system_state = SYSTEM_RUNNING       # 系统正式运行
⑦ 尝试执行（按顺序）：
   - /init（如果是 initramfs）
   - cmdline 指定的 init=xxx
   - /sbin/init
   - /etc/init
   - /bin/init
   - /bin/sh
   ↓（任何一个成功则内核init任务完成）

进程树初始状态：
PID 0: swapper/0  (idle, CPU 0)    ← cpu_startup_entry()
PID 1: init       (kernel_init)    ← 最终 exec /sbin/init
PID 2: kthreadd                    ← 所有内核线程的父进程
PID 3: kworker/...                 ← 工作队列线程
PID 4: kdevtmpfs                   ← devtmpfs 线程
...
```

---

## 第四部分：完整详细启动流程总览

以下是涵盖所有细节的完整启动序列，以 x86_64 + UEFI + systemd 为典型案例，所有步骤均有编号。

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              Linux 内核完整启动流程（x86_64 + UEFI 典型路径）               ║
╚══════════════════════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 1: 硬件固件（UEFI）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [1]  上电 / 复位，CPU 从 Reset Vector（0xFFFF_FFF0）读取第一条指令
 [2]  UEFI POST：CPU 频率/cache/内存控制器自测
 [3]  UEFI 初始化 RAM（SPD 读取，DDR 训练）
 [4]  UEFI 初始化 PCIe/USB/存储控制器
 [5]  UEFI 构建 ACPI 表（RSDP/XSDT/MADT/DSDT）
 [6]  UEFI 构建 E820/EFI Memory Map（物理内存布局表）
 [7]  UEFI 扫描 ESP 分区（EFI System Partition），加载 bootloader

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 2: 引导加载器（GRUB2 / EFI Stub）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [8]  GRUB2 从 ESP 加载 grub.efi，读取 grub.cfg
 [9]  GRUB2 定位 vmlinuz（压缩内核）和 initrd（初始 RAM 磁盘）
 [10] GRUB2 将 vmlinuz 和 initrd 加载到 RAM
 [11] GRUB2 构造 boot_params 结构（包含 cmdline、e820、initrd 地址等）
 [12] GRUB2 调用 UEFI ExitBootServices()，UEFI 释放控制权
 [13] GRUB2 跳转至内核入口（对于 EFI Stub 直接由 UEFI 调用内核 PE 入口）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 3: x86_64 内核汇编早期初始化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [14] arch/x86/boot/header.S：BIOS 引导时的 16-bit 实模式代码（EFI下跳过）
 [15] arch/x86/boot/compressed/head_64.S：切换至 64-bit 长模式
 [16] 在长模式下调用 decompress_kernel() 解压 vmlinux
 [17] arch/x86/kernel/head_64.S：startup_64 标签
       - 设置 CR3（物理地址 → 4级页表）
       - 加载临时 GDT（全局描述符表）
       - 清零 BSS 段
       - 设置 %rsp（内核栈指针，指向 init_thread_union）
       - 通过 initial_code 跳入 x86_64_start_kernel

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 4: x86_64 C 语言早期初始化（arch/x86/kernel/head64.c）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [18] x86_64_start_kernel()
       [18a] cr4_init_shadow()              保存 CR4 寄存器
       [18b] reset_early_page_tables()      清除临时页表
       [18c] clear_bss()                   再次确认 BSS 清零
       [18d] copy_bootdata()               复制 boot_params 到内核变量
       [18e] load_ucode_bsp()              加载 CPU 微码（microcode update）
       [18f] init_top_pgt()               初始化顶层页表
       [18g] x86_64_start_reservations()  保留早期内存区域（cmdline, e820等）
       [18h] start_kernel()               ──▶ 进入通用初始化

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 5: start_kernel() — 通用内核初始化（init/main.c）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [19] set_task_stack_end_magic()           init_task 栈溢出检测魔数
 [20] smp_setup_processor_id()            设置 CPU 逻辑 ID
 [21] cgroup_init_early()                 cgroup v1/v2 早期框架
 [22] local_irq_disable()                 【关中断保护初始化过程】
 [23] boot_cpu_init()                     标记 BSP（Bootstrap Processor）
 [24] setup_arch(&command_line)           ★ 最重要的架构初始化函数
       [24a] parse_early_param()          解析 console=, mem=, root= 等早期参数
       [24b] e820__memory_setup()         x86: 建立物理内存布局
       [24c] ioremap_init()              I/O 内存映射初始化
       [24d] init_mem_mapping()          建立内核线性映射区
       [24e] initmem_init()              初始化 NUMA 拓扑
       [24f] dma_contiguous_reserve()    DMA 连续内存预留
       [24g] acpi_table_init()           解析 ACPI 表
       [24h] early_trap_init()           设置早期异常处理
 [25] mm_core_init()                      ★ 内存核心初始化
       [25a] mem_init()                  将页移交给 buddy 系统
       [25b] kmem_cache_init()           初始化 SLUB 分配器
       [25c] vmalloc_init()              初始化 vmalloc 区域
 [26] trap_init()                         IDT 完整设置（x86: 256个中断向量）
 [27] sched_init()                        ★ 调度器初始化（CFS + RT + DL）
 [28] workqueue_init_early()              工作队列框架（用于延迟工作）
 [29] rcu_init()                          RCU 无锁同步机制初始化
 [30] early_irq_init() + init_IRQ()       ★ 中断控制器初始化（APIC等）
 [31] tick_init() + timers_init()         时钟滴答和定时器初始化
 [32] hrtimers_init()                     高精度定时器（用于 nanosleep 等）
 [33] softirq_init()                      软中断（网络/块设备的下半部）
 [34] timekeeping_init()                  系统时钟（CLOCK_REALTIME等）
 [35] time_init()                         架构时钟源（TSC/HPET）
 [36] random_init()                       /dev/random 熵池初始化
 [37] local_irq_enable()                  【开中断！从此可以接收中断】
 [38] console_init()                      ★ 控制台初始化（printk 日志开始显示）
 [39] fork_init()                         进程创建子系统（task_struct 缓存）
 [40] security_init()                     LSM 安全框架（SELinux/AppArmor等）
 [41] vfs_caches_init()                   ★ VFS 完整初始化
 [42] pagecache_init()                    页缓存（文件读写缓存）
 [43] signals_init()                      UNIX 信号机制
 [44] proc_root_init()                    /proc 文件系统根节点
 [45] cgroup_init()                       控制组（资源限制）完整初始化
 [46] acpi_subsystem_init()              ACPI 子系统完整初始化
 [47] rest_init()                         ★ 创建 init 进程，进入 idle

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 6: rest_init() — 进程创建（init/main.c:714）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [48] rcu_scheduler_starting()            启动 RCU 调度器
 [49] PID 1 创建：user_mode_thread(kernel_init)   init 进程
 [50] PID 2 创建：kernel_thread(kthreadd)          内核线程守护进程
 [51] CPU 0 进入 cpu_startup_entry()              成为 idle 进程（swapper/0）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 7: kernel_init()（PID 1）— SMP启动与驱动初始化
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [52] kernel_init_freeable()
       [52a] smp_prepare_cpus()           准备所有 CPU
       [52b] workqueue_init()             启动工作队列内核线程
       [52c] do_pre_smp_initcalls()       SMP 前的 initcall（level 0-3）
       [52d] lockup_detector_init()       看门狗/锁死检测初始化
       [52e] smp_init()                  ★ 启动所有次级 CPU（AP）
             - 对每个 CPU 发送 INIT-SIPI-SIPI
             - 次级 CPU 执行 secondary_startup_64
             - 次级 CPU 各自初始化本地 APIC, GDT, IDT
             - 次级 CPU 进入 cpu_startup_entry() idle
       [52f] sched_init_smp()            构建调度域（NUMA感知调度）
       [52g] do_basic_setup()            ★★ 核心驱动和子系统初始化
             - driver_init()             设备模型 (sysfs, kobject, uevent)
             - do_initcalls()            按 level 0-7 执行所有 __initcall
               Level 0: early_initcall  （如 rcu, irq 相关）
               Level 1: core_initcall   （如 sock_init, bus_init）
               Level 2: postcore_initcall（如 net 命名空间）
               Level 3: arch_initcall   （架构相关）
               Level 4: subsys_initcall （如 pci_init, inet_init, usb_init）
               Level 5: fs_initcall     （如 ext4_init, nfs_init）
               Level 6: device_initcall （如各种设备驱动 probe）
               Level 7: late_initcall   （延后初始化）
       [52h] wait_for_initramfs()        等待 initramfs 解压完成
       [52i] console_on_rootfs()         打开 /dev/console (stdin/stdout/stderr)
       [52j] prepare_namespace()         挂载真正的根文件系统
             - 等待块设备就绪（async 驱动探测）
             - 解析 root= 参数（如 /dev/sda1, UUID=..., PARTUUID=...）
             - mount_root() → 挂载根文件系统到 /
             - sys_chroot("/") → 切换根目录
 [53] async_synchronize_full()           等待所有异步 __init 完成
 [54] free_initmem()                     释放 __init 段（节省内存，通常 ~1-3MB）
 [55] mark_readonly()                    标记内核代码只读（W^X 防护）
 [56] system_state = SYSTEM_RUNNING      ★ 系统进入运行状态

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 阶段 8: 用户空间启动
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 [57] 尝试执行初始进程（按以下顺序）：
       (a) /init          （如果 initramfs 提供）
       (b) cmdline 的 init= 参数（如 init=/bin/bash 调试用）
       (c) CONFIG_DEFAULT_INIT（编译时配置的默认 init）
       (d) /sbin/init     （传统 SysVinit / systemd 链接到这里）
       (e) /etc/init
       (f) /bin/init
       (g) /bin/sh        （紧急 shell）
 [58] kernel_init 进程通过 execve() 替换自己为 /sbin/init
 [59] systemd（现代发行版）作为 PID 1 运行
       [59a] 解析 /etc/systemd/system/*.target
       [59b] 并行启动各种 service units
       [59c] 挂载剩余文件系统（/sys, /dev, /run 等）
       [59d] 激活 getty（登录终端）
       [59e] 启动网络服务、显示管理器等
 [60] 用户登录，系统完全就绪

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 系统稳态进程结构
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PID 0: [swapper/N]    - 每个 CPU 的 idle 进程（不在进程表中）
  PID 1: systemd        - 用户空间始祖进程
  PID 2: [kthreadd]     - 内核线程始祖（所有[]进程的父）
  PID 3: [rcu_gp]       - RCU 宽限期线程
  PID 4: [rcu_par_gp]   - RCU 并行宽限期
  PID 5: [kworker/0:0H] - 工作队列（high priority）
  ...
  PID N: [migration/N]  - 每个CPU的迁移线程（实时优先级）
  PID M: [ksoftirqd/N]  - 软中断守护进程
```

---

## 参考源码位置

| 文件路径 | 描述 |
|---------|------|
| `init/main.c` | `start_kernel()`, `rest_init()`, `kernel_init()` |
| `arch/x86/boot/header.S` | x86 引导头，实模式入口 |
| `arch/x86/boot/compressed/head_64.S` | x86_64 解压头，长模式切换 |
| `arch/x86/kernel/head_64.S` | x86_64 内核入口 `startup_64` |
| `arch/x86/kernel/head64.c` | `x86_64_start_kernel()` |
| `arch/x86/kernel/setup.c` | x86 `setup_arch()` 实现 |
| `arch/arm64/kernel/head.S` | ARM64 `primary_entry`, `__primary_switched` |
| `arch/arm64/kernel/setup.c` | ARM64 `setup_arch()` 实现 |
| `arch/arm/boot/compressed/head.S` | ARM32 解压头 |
| `arch/arm/kernel/head.S` | ARM32 内核入口 `stext` |
| `arch/riscv/kernel/head.S` | RISC-V `_start`, `_start_kernel` |
| `mm/memblock.c` | 早期内存分配器 memblock |
| `mm/page_alloc.c` | Buddy 系统页分配器 |
| `mm/slub.c` | SLUB 对象分配器 |
| `kernel/sched/core.c` | 调度器核心（`sched_init`, CFS） |
| `kernel/irq/irqdesc.c` | 中断描述符管理 |
| `fs/dcache.c` | VFS 目录缓存 |
| `fs/inode.c` | VFS inode 管理 |
| `net/socket.c` | 套接字层 (`sock_init`) |
| `net/ipv4/af_inet.c` | IPv4 协议栈 (`inet_init`) |
| `drivers/base/init.c` | 设备驱动模型 (`driver_init`) |
| `include/linux/init.h` | `__init`, `__initcall`, `core_initcall` 宏定义 |

---

*本报告基于 Linux 7.0-rc5 内核源码分析，参考架构文档及源代码注释撰写。*
*报告路径：`Documentation/linux-kernel-boot-process.md`*
