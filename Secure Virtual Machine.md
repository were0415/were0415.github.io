# 1 Secure Virtual Machine（SVM）

## 1.1 SVM硬件总览

SVM处理器支持提供了一组硬件扩展，旨在实现经济高效的虚拟机系统。一般来说，硬件支持分为两个类别：虚拟化支持和安全支持。

### 1.1.1 虚拟化支持

AMD虚拟机架构旨在提供：

- 客户机/主机标记TLB以减少虚拟化开销

- 内存的外部(DMA)访问保护

- 辅助中断处理，虚拟中断支持和增强暂停过滤器

- •能够截取客户机的特定指令或事件

- 在VMM和客户机之间快速切换的机制

### 1.1.2 Guest 模式

通过VMRUN指令进入这种新的处理器模式，在Guest模式下，一些x86指令的行为会发生改变以促进虚拟化。

CPUID功能号4000_0000h-4000_00FFh已预留给软件使用。Hypervisors可以使用这些功能号提供一个接口，将信息从Hypervisors传递到客户机。这类似于通过使用CPUID提取有关物理CPU的信息。Hypervisors使用CPUID功能号400000[FF:00]位表示虚拟平台。

CPUID Fn0000_0001_ECX[31]已预留给Hypervisors使用，以指示Hypervisors的存在。Hypervisors将此位设置为1，物理CPU将此位设置为0。这一点可以被客户机的软件探测到，以检测它们是否在虚拟机中运行。

### 1.1.3 外部访问保护

客户机可以被授予直接访问所选I/O设备的权限。硬件支持的目的是防止一个客户机拥有的设备访问另一个客户机(或VMM)拥有的内存。

### 1.1.4 中断支持

为了中断的有效虚拟化，以下支持是在VMCB标志的控制下提供的：

1. 拦截物理中断：VMM可以请求物理中断导致正在运行的客户机退出，从而允许VMM处理中断；
2. 虚拟中断：VMM可以向客户机注入虚拟中断。
3. 共享物理APIC：SVM允许多个客户机共享一个物理APIC，并将每个客户机对APIC状态的操作与其他客户机隔离开来，这样任何客户机都无法向另一个客户机传递中断。
4. 直接传递中断：高级虚拟中断控制器(AVIC)扩展虚拟化了APIC的中断传递功能。这提供了将设备或IPI中断直接传递到一个或多个目标vCPU的功能，从而避免了让VMM确定中断路由的开销，并加快了中断传递的速度。

### 1.1.5 安全支持

SVM通过各种扩展提供了额外的系统支持：

- 内存加密：SEV和SEV-ES扩展防止恶VMM代码、内存总线跟踪或通过加密Guest内存和寄存器内容删除内存设备来检查Guest内存和Guest寄存器状态。

- 安全嵌套分页：SEV-SNP扩展为Guest内存提供了额外的保护，防止VMM代码对地址转换机制的恶意操纵。

## 1.2 SVM处理器和平台扩展

SVM硬件扩展可分为以下几类:

1. 状态转换：VMRUN, VMSAVE, VMLOAD指令，GIF（全局中断标志），以及操作GIF的指令（STGI, CLGI）。
2. 拦截：允许VMM拦截客户机中的敏感操作。
3. 中断和APIC辅助：物理中断拦截，虚拟中断支持，APIC.TPR虚拟化。
4. SMM拦截和辅助。
5. 外部DMA访问保护。
6. 嵌套分页支持两层地址转换。
7. 安全：SKINIT指令。

## 1.3 安全加密虚拟化

SEV在CPU使用AMD-V虚拟化特性的Guest模式下运行时可用。SEV支持运行加密的虚拟机，其中虚拟机的代码和数据是安全的，因此解密版本仅在虚拟机内部可用。每个虚拟机可能与一个唯一的加密密钥相关联，因此如果数据被不同的实体使用时必须使用不同的密钥访问。

### 1.3.1 如何确定是否支持SEV

对内存加密特性的支持在CPUID 8000_001F[EAX]中表示，bit 1表示支持SEV。

当使能内存加密特性时，CPUID 8000_001F[EBX]和8000_001F[ECX]提供关于内存加密使用的额外信息，例如同时支持的密钥数量、用于将页面标记为加密的页表位。此外，在某些实现中，当使能内存加密特性时，处理器的物理地址大小可能会减小，例如从48位减小到43位。在这个例子中，除非另有说明，物理地址位47:43将被视为保留。

### 1.3.2 密钥管理

每个启用SEV的客户机都与一个内存加密密钥相关联，并且SME模式与一个单独的密钥相关联。SEV特性的密钥管理不是由CPU处理，而是由单独的AMD安全处理器(AMD- SP)来处理。

CPU软件不知道这些密钥的值，但是VMM应该通过AMD-SP驱动程序协助加载虚拟机的密钥。这种协助还将确定VMM应该为特定客户机使用哪个ASID。在SEV下，ASID用作密钥索引，用于标识用于加密/解密与启用SEV的客户机关联的内存流量的加密密钥。加密密钥本身永远不会对CPU软件可见，也永远不会以透明的方式存储在芯片外。

### 1.3.3 使能SEV

在启动加密虚拟机之前，软件必须通过MSR C001_0010 (SYSCFG)使能MemEncryptionModEn（bit 23）。

![]()

![image-20230727055113274](https://github.com/were0415/were0415.github.io/blob/main/image-20230727055113274.png)

![image-20230727055447698](https://github.com/were0415/were0415.github.io/blob/main/image-20230727055447698.png)

如果VMM在VMCB的offset 090h设置了SEV使能(Bit 1)，那么在VMRUN指令期间可以在特定的虚拟机上启用SEV。

当在客户机中启用SEV时，将在VMRUN期间执行额外的一致性检查:

- 必须启用嵌套分页(VMCB offset 090h，bit 0)；
- MSR C001_0015 (HWCR) [SmmLock]必须设置；
- ASID (VMCB offset 058h)必须在SEV允许的范围内；

SEV操作允许的ASID可能是硬件支持的ASID总数的一个子集。在这个场景中，启用SEV的客户机必须使用已定义子集中的ASID，而未启用SEV的客户机可以使用剩余的ASID范围。启用SEV的客户机允许的ASID范围从1到通过CPUID 8000_001F[ECX]定义的最大值。

注意，在CPUID 8000_001F_EAX[11]设置为1的系统上，VMM必须处于64位模式，才能对启用SEV 的客户机执行VMRUN。否则，VMRUN将会失败，错误码为VMEXIT_INVALID。

当在客户机上启用SEV时，如果上述一致性检查失败，VMRUN指令将以VMEXIT_INVALID错误代码终止。如果“MemEncryptionModEn”为0，表示不能使能SEV，并且忽略SEV的VMCB控制位。

### 1.3.4 SEV加密行为

在启用SEV的情况下执行客户机时，将使用客户机页表来确定内存页的C位，从而确定该内存页的加密状态。这允许Guest确定哪些页面是私有的或共享的，但这种控制仅对数据页面可用。无论C位的软件值如何，代表指令读取和客户机页表遍历的内存访问总是被视为私有的。这种行为可确保非客户机实体(如VMM)不能将自己的代码或数据注入启用SEV的客户机。如果客户机确实希望指令页或页表中的数据能够被客户机外部的代码访问，则必须显式地将这些数据复制到共享数据页中。

需要注意的是，虽然客户机可以选择在指令页和页表地址上显式设置C位，但在硬件总是将其作为私有访问执行的情况下，这个位的值是不需要关心的。

### 1.3.5 页表支持





