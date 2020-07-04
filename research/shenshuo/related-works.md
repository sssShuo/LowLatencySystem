## 相关工作

### 智能网卡 SmartNIC

#### 产生

现代的工作负载和数据中心设计给CPU核心带来了太多的网络开销。随着更快的网络速度(每个链接可高达200Gb/s)， CPU花费了太多开销对网络流量进行分类、跟踪和控制。这些昂贵的CPU内核是为通用应用程序处理而设计的，而不是消耗所有这些处理能力仅用来查看和管理数据的移动。

智能网卡（SmartNIC）应运而生，它可以将本应由CPU处理的工作转移到网卡上执行，减少CUP工作量的同时提高性能，其核心是通过FPGA(现场可编程门阵列)协助CPU处理网络负载，编程网络接口功能，具有以下特征：

* 通过FPGA本地化编程支持数据面和控制面功能定制，协助CPU处理网络负载；

* 通常包含多个端口和内部交换机，快速转发数据并基于网络数据包、应用程序套接字等智能映射到到相关应用程序；

* 检测和管理网络流量。

过去30年来，网卡已经从具有单一MAC，PHY和系统接口的简单网卡发展到具有一个，两个或更多网络接口和用于TCP/IP的硬件卸载引擎的高性能适配器、虚拟化等功能。最新的NIC基于支持10Gbit/s，25Gbit/s，50Gbit/s或100Gbit/s网络接口的输入/输出(I/O)控制器。Smart NIC将FPGA、处理器或基于处理器的智能I/O控制器与分组处理和虚拟化加速集成在一起。大多数Smart NIC可以使用标准的FPGA或处理器开发工具进行编程，越来越多的厂商也开始增加了对eBPF以及可编程语言P4的支持。

目前业界提供基于FPGA的Smart NIC的厂商包括Accolade、BittWare、Enyx、Ethernity、Napatech、Netcope、Reflex CES、Silicom和Solarflare。这些适配器集成了来自Intel或Xilinx的FPGA，其中一些采用了最新的Stratix 10和UltraScale + FPGA的FPGA。基本的I/O控制器功能是使用FPGA实现的，或者由单独的I/O控制器进行处理。Broadcom，Cavium，Intel，Kalray，Mellanox，**Netronome**，Silicom和SolidRun均可提供基于处理器的Smart NIC。Smart NIC使用带有集成处理器内核或可编程交换的处理器或智能I/O处理器，亚马逊和谷歌已经开发了自己的Smart NIC ASIC。

#### 应用场景

+ 安全隔离

出于安全性考虑，有时需要将网络与 CPU 相隔离，通常黑客攻击和恶意软件来源于网络。使用智能网卡便可以在网卡上检查网络流量、阻止攻击和进行加密传输，从而带来了性能和安全性的提升。如果主CPU受到威胁，那么智能网卡仍然可以检测恶意活动，在不立即占用CPU的情况下阻止攻击。

+ 存储虚拟化和云

智能网卡的一个较新的用例是虚拟化软件定义的存储，超融合基础架构和其他云资源。在超融合架构数据中心中，Smart NIC为虚拟化应用程序提供硬件加速与网络接口紧密结合，并可分布在大型服务器网络中，减小CPU负载，提供额外的边缘计算能力，加速特定应用和虚拟化功能，并且通过正确的语言和工具链支持，为用户提供应用加速即服务的附加价值。智能网卡甚至可以虚拟化GPU（或一些神经网络处理器），以便任何服务均可通过网络访问[^1]

![SmartNIC using cases](./related-works.assets/Capture-16.jpg)

一个好的智能网卡必须是可编程的。虽然为了获得最佳性能，大多数加速功能必须在硬件中运行，但为了获得最大的灵活性，这些功能的控制和编程需要在软件中运行。

有许多功能可以在智能网卡上编程，通常，特定的卸载（offload）方法，加密算法和传输机制不会发生太大变化，但路由规则，加密密钥和网络地址始终会发生变化。我们将前者的功能视为数据面，后者则用作控制面功能。一旦建立标准化，数据面规则和算法就可以编写到硬件上了。然而控制面规则和编程变化多样，无法硬编码到芯片中，但可以在FPGA上运行或在C可编程的Linux环境中运行。

#### Agilio SmartNIC

我们计划使用 **Agilio CX SmartNIC 2x10GbE**[^2] 这款智能网卡来实现 eBPF offload 的<font color=red>**相关功能**</font>。

> Censyu： 补充下相关功能具体是什么。。。​<font style="font-style: normal;">:sweat_smile:</font>

其核心部件为**NFP (Network Flow Processor)**，它是一个多线程多核的网络流处理器，相关参数如下[^3]：

![NPF-4000](related-works.assets/NPF-4000.jpg)

它拥有60个流处理核可以支持我们使用 BPF/eBPF offload，具体流程如下图所示：

![compile process](related-works.assets/compile.png)



### BPF 衍生应用

BPF（Berkeley Packet Filter）这一概念提出于1992年，最初是一种过滤数据包并避免从内核到用户空间的无用数据包副本的方法。后来在2013年 Alexei Starovoitov 彻底重塑了它，开始添加新的功能（如映射、尾调用等）并改善 BPF 的性能。这个新版本被称为 **eBPF**（extended BPF），而前者则变为cBPF（classic BPF）。同时 JIT（Just-In-Time） compiling machine 也被重写，使得eBPF甚至比cBPF更贴近本地机器语言。eBPF后来被应用在网络编程中。除了套接字过滤器，eBPF程序还可以附加到TC（Linux Traffic Control Tool）的出入接口，并以有效的方式执行各种数据包处理任务。[^4]

#### eBPF

eBPF(extended Berkeley Packet Filter)，自2014年引入Linux内核以来，已经迅速被许多系统所采用，包括网络监控、网络流量操作和系统监控等，所有这些都可以通过Agilio CX SmartNIC编程和卸载方法来加速。

eBPF以多种方式扩展了经典的BPF虚拟机，引入了以下概念：

+ 与x86-64类似的架构：BPF使用64位寄存器，并将可用寄存器的数量从2（累加器和 X 寄存器）增加到10。 eBPF还扩展了操作码的数量。
+ 与网络子系统分离： BPF受限于基于数据包的数据模型。由于它用于包过滤，因此其代码存在于网络子系统中。但是，eBPF VM不再受限于数据模型，它可以用于任何目的。现在可以将eBPF程序附加到跟踪点或 kprobe 中。这为eBPF打开了插桩（Instrumentation），性能分析以及其他内核子系统中的更多用途的大门。
+ 全局数据存储（Map）：Map是一种通用数据结构，以键值对的形式存储不同类型的数据。它们允许在eBPF内核程序之间以及内核和用户空间应用程序之间共享数据。
+ 辅助函数（Helper functions）：如数据包重写，校验和计算或数据包克隆。与用户空间编程不同，这些函数在内核中执行。此外，还可以从eBPF程序执行系统调用。
+ 尾调用（Tail-calls）：eBPF程序限制为4096字节。尾调功能允许eBPF程序通过控制新的eBPF程序，克服此限制。[^5]

#### Hardware Offload

see more in [P4, eBPF and Linux TC offload](https://open-nfp.org/the-classroom/p4-ebpf-and-linux-tc-offload/)



### References

[^1]:	[Why You Need a SmartNIC and Smart NIC Use Cases](http://www.mellanox.com/blog/2018/09/why-you-need-smart-nic-use-cases/)   

[^2]:	[Agilio CX SmartNIC 2x10GbE](https://www.netronome.com/products/agilio-cx/)
[^3]:[Netronome NFP-4000 Flow Processor](https://www.netronome.com/media/documents/PB_NFP-4000.pdf)  

[^4]:	[Dive into BPF: a list of reading material](https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf)
[^5]:	[A brief introduction to XDP and eBPF](https://blogs.igalia.com/dpino/2019/01/07/introduction-to-xdp-and-ebpf/)  

