# 虚拟化容器

容器是云计算、微服务等诸多软件业界核心技术的共同基石，容器的首要目标是让软件分发部署过程从传统的发布安装包、靠人工部署转变为直接发布已经部署好的、包含整套运行环境的虚拟化镜像。在容器技术成熟之前，主流的软件部署过程是由系统管理员编译或下载好二进制安装包，根据软件的部署说明文档准备好正确的操作系统、第三方库、配置文件、资源权限等各种前置依赖以后，才能将程序正确地运行起来。[Chad Fowler](http://chadfowler.com/)在提出“不可变基础设施”这个概念的文章《[Trash Your Servers and Burn Your Code](http://chadfowler.com/2013/06/23/immutable-deployments.html)》里，开篇就直接吐槽：要把一个不知道打过多少个升级补丁，不知道经历了多少任管理员的系统迁移到其他机器上，毫无疑问会是一场灾难。

让软件能够在任何环境、任何物理机器上达到“一次编译，到处运行”曾是 Java 早年的宣传口号，这并不是一个简单的目标，不设前提的“到处运行”，仅靠 Java 语言和 Java 虚拟机是不可能达成的，因为一个计算机软件要能够正确运行，需要有以下三方面的兼容性来共同保障（这里仅讨论软件兼容性，不去涉及“如果没有摄像头就无法运行照相程序”这类问题）：

- **ISA 兼容**：目标机器指令集兼容性，譬如 ARM 架构的计算机无法直接运行面向 x86 架构编译的程序。
- **ABI 兼容**：目标系统或者依赖库的二进制兼容性，譬如 Windows 系统环境中无法直接运行 Linux 的程序，又譬如 DirectX 12 的游戏无法运行在 DirectX 9 之上。
- **环境兼容**：目标环境的兼容性，譬如没有正确设置的配置文件、环境变量、注册中心、数据库地址、文件系统的权限等等，任何一个环境因素出现错误，都会让你的程序无法正常运行。

:::quote 额外知识：ISA 与 ABI

[指令集架构](https://en.wikipedia.org/wiki/Instruction_set_architecture)（Instruction Set Architecture，ISA）是计算机体系结构中与程序设计有关的部分，包含了基本数据类型，指令集，寄存器，寻址模式，存储体系，中断，异常处理以及外部 I/O。指令集架构包含一系列的 Opcode 操作码（即通常所说的机器语言），以及由特定处理器执行的基本命令。

[应用二进制接口](https://en.wikipedia.org/wiki/Application_binary_interface)（Application Binary Interface，ABI）是应用程序与操作系统之间或其他依赖库之间的低级接口。ABI 涵盖了各种底层细节，如数据类型的宽度大小、对象的布局、接口调用约定等等。ABI 不同于[应用程序接口](https://en.wikipedia.org/wiki/API)（Application Programming Interface，API），API 定义的是源代码和库之间的接口，因此同样的代码可以在支持这个 API 的任何系统中编译，而 ABI 允许编译好的目标代码在使用兼容 ABI 的系统中无需改动就能直接运行。

:::

笔者把使用仿真（Emulation）以及虚拟化（Virtualization）技术来解决以上三项兼容性问题的方法都统称为虚拟化技术。根据抽象目标与兼容性高低的不同，虚拟化技术又分为下列五类：

- **指令集虚拟化**（ISA Level Virtualization）。通过软件来模拟不同 ISA 架构的处理器工作过程，将虚拟机发出的指令转换为符合本机 ISA 的指令，**代表为[QEMU](https://www.qemu.org/)和[Bochs](http://bochs.sourceforge.net/)。**指令集虚拟化就是仿真，能提供了几乎完全不受局限的兼容性，甚至能做到直接在 Web 浏览器上运行完整操作系统这种令人惊讶的效果，但由于每条指令都要由软件来转换和模拟，它也是性能损失最大的虚拟化技术。
- **硬件抽象层虚拟化**（Hardware Abstraction Level Virtualization）。以软件或者直接通过硬件来模拟处理器、芯片组、内存、磁盘控制器、显卡等设备的工作过程。既可以使用纯软件的二进制翻译来模拟虚拟设备，也可以由硬件的[Intel VT-d](https://en.wikipedia.org/wiki/X86_virtualization#Intel-VT-d)、[AMD-Vi](<https://en.wikipedia.org/wiki/X86_virtualization#AMD_virtualization_(AMD-V)>)这类虚拟化技术，将某个物理设备直通（Passthrough）到虚拟机中使用，代表为[VMware ESXi](https://www.vmware.com/)和[Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/about/)。如果没有预设语境，一般人们所说的“虚拟机”就是指这一类虚拟化技术。
- **操作系统层虚拟化**（OS Level Virtualization）。无论是指令集虚拟化还是硬件抽象层虚拟化，都会运行一套完全真实的操作系统来解决 ABI 兼容性和环境兼容性问题，虽然 ISA 兼容性是虚拟出来的，但 ABI 兼容性和环境兼容性却是真实存在的。而操作系统层虚拟化则不会提供真实的操作系统，而是采用隔离手段，使得不同进程拥有独立的系统资源和资源配额，看起来仿佛是独享了整个操作系统一般，其实系统的内核仍然是被不同进程所共享的。<br/>操作系统层虚拟化的另一个名字就是本章的主角“容器化”（Containerization），由此可见，容器化仅仅是虚拟化的一个子集，只能提供操作系统内核以上的部分 ABI 兼容性与完整的环境兼容性。这意味着如果没有其他虚拟化手段的辅助，在 Windows 系统上是不可能运行 Linux 的 Docker 镜像的（现在可以，是因为有其他虚拟机或者 WSL2 的支持），反之亦然。也同样决定了**如果 Docker 宿主机的内核版本是 Linux Kernel 5.6，那无论上面运行的镜像是 Ubuntu、RHEL、Fedora、Mint 或者任何发行版的镜像，看到的内核一定都是相同的 Linux Kernel 5.6。容器化牺牲了一定的隔离性与兼容性，换来的是比前两种虚拟化更高的启动速度、运行性能和更低的执行负担。**
- **运行库虚拟化**（Library Level Virtualization）。与操作系统虚拟化采用隔离手段来模拟系统不同，运行库虚拟化选择使用软件翻译的方法来模拟系统，它以一个独立进程来代替操作系统内核来提供目标软件运行所需的全部能力，这种虚拟化方法获得的 ABI 兼容性高低，取决于软件是否能足够准确和全面地完成翻译工作，其代表为[WINE](https://www.winehq.org/)（Wine Is Not an Emulator 的缩写，一款在 Linux 下运行 Windows 程序的软件）和[WSL](https://docs.microsoft.com/en-us/windows/wsl/about)（特指 Windows Subsystem for Linux Version 1）。
- **语言层虚拟化**（Programming Language Level Virtualization）。由虚拟机将高级语言生成的中间代码转换为目标机器可以直接执行的指令，代表为 Java 的 JVM 和.NET 的 CLR。虽然厂商肯定会提供不同系统下都有相同接口的标准库，但本质上这种虚拟化并不直接解决任何 ABI 兼容性和环境兼容性问题。
