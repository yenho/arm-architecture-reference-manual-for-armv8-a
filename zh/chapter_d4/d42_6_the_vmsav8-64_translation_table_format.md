## D4.2.6 The VMSAv8-64 translation table format

本小节主要描述 VMSAv8-64 中地址转换过程中所使用的 translation table 的格式。

对于 AArch64 运行态下的一个 Exception level 中的地址转换过程：

* TCR_EL1.{SH0, ORGN0, IRGN0, SH1, ORGN1, IRGN1} 寄存器位定义了使用 TTBR0_EL1 和 TTBR1_EL1 的 translation table walk 的所访问的内存属性。
* 在 Secure 和 Non-secure EL1&0 的 stage 1 translation 中，TTBR0_EL1 和 TTBR1_EL1 都包含 ASID (Address Space Identifier) 位，而 TCR_EL1.A1 寄存器位则用于设定使用哪一个 ASID。(The architecture supports the use of an address space identifier (ASID) to allow the operating system to identify one process' address space from another's without flushing the TLB.) 

对于 translation table 的格式，在 [Overview of the VMSAv8-64 address translation stages](#) 章节中已经汇总了 lookup level 相关的信息，而在[Descriptor encodings, ARMv8 level 0, level 1, and level 2 formats](#) 章节中则介绍了 translation table entry 的相关内容。

本章节主要介绍以下几个方面的内容：

* [Translation granule size and associate block and page sizes](#).
* [Selection between TTBR0 and TTBR1](#).
* [Concatenated translation tables for the initial stage 2 lookup](#).
* [Possible translation table registers programming errors](#).


### Translation granule size and associate block and page sizes

Table D4-20 描述了在输出地址为 48 位时，不同 granule size 下的 block size 和 page size。

![](table_d4_20.png)

Translation table 中的 descriptor 的 bit[1] 用于指示该 descriptor 是否为 block descriptor，另外：

* 使用 4KB granule size 时，只有在 level 1 和 2 的 translation table 中支持 block descriptor。
* 使用 16KB 和 64KB granule size 时，只有在 level 2 的 Translation table 支持  block descriptor。

(译者注：进行地址转换时，遇到 block descriptor 就表示转换结束了，并且未解析的 IA 地址位作为 block 内的偏移，直接赋值给 OA。实际应用中可以为特定的需求分配一个 block 的内存，这个内存的地址转换相对于 page 会有更高的效率)

如果将不支持 block descriptor 的 translation table 中的 descriptor 的 bit[1] 设为 0，将会触发 Translation fault。

后续的几个表格中，描述了在 AArch64 运行态下的 translation 的各个 level 下 的 lookup 在不同 granule size 下的相关信息，包括：
* 所支持的最大的 IA 大小，以及在该 IA 大小下，该 level 的 lookup 所解析的地址位。
* 该 level 上的 translation table descriptor 所转换能的最大的 OA 范围，以及其所对应的 memory region 大小。
* 解析最大 IA 所需要的 translation table 的大小。

![](table_d4_21_1.png)
![](table_d4_21_2.png)

![](table_d4_22.png)
![](table_d4_23.png)

对于第一次 lookup 所在的 level：

* TCR.TxSZ 寄存器位所设定的 IA 范围小于上面表格中的最大值时，translation table 中保存的 descriptor 也会小于最大值，相应的 translation table 的大小也会变小。
* 对于 stage 2 translation，由于可以进行 translation table 的连接操作，其解析的 IA 大小可以超过上面表格中的值。更多的信息可以参考 [Overview of the VMSAv8-64 address translation stages](#) and [Concatenated translation tables for the initial stage 2 lookup](#) 章节。

如果给出的输入地址大于所配置的大小，那么就会触发 Translation fault。

> **NOTE:**  
对于特定大小的虚拟地址，如果使用较大的 granule size，相对于使用较小的 granule size 可以较少解析地址所需要的 lookup 次数。

关于 TCR 寄存器中对 initial lookup 的配置，可以参考 [Overview of the VMSAv8-64 address translation stages](#) 章节。


### Selection between TTBR0 and TTBR1

在 translation regime 的 stage 1 translation 中，translation table walk 第一个访问的 translation table 的地址都是从 TTBR 中得到的。在 EL1&0 translation regime 中， VA 被分割为两个 range，在这种场景下：

* 从 0x0000_0000_0000_0000 开始的 VA range 的 initial translation table 的基地址保存在 TTBR0_EL1 中。
* 以 0xFFFF_FFFF_FFFF_FFFF 结束的 VA range 的 initial translation table 的基地址保存在 TTBR1_EL1 中。

![](figure_d4_15.png)

在 translation 中，最终决定使用哪一个 TTBR 的，是需要进行转换的 VA：

* 如果 VA 的 top bits 都为 0，那么将使用 TTBR0_EL1
* 如果 VA 的 top bits 都为 1，那么将使用 TTBR1_EL1

top bits 根据配置可以是 VA[63:56] 或者 VA[55:48]，更多信息可以参考 [Address tagging in AArch64 state](#) 章节。

> **NOTE:**  
The handling of the Contiguous bit can mean that the boundary between the translation regions defined by the TCR_EL1.TnSZ values and the region for which an access generates a Translation fault is wider than shown in Figure D4-15. That is, if the descriptor for an access to the region shown as generating a fault has the Contiguous bit set to 1, the access might not generate a fault. [Possible translation table registers programming errors on page D4-1673](#) describes this possibility.

Example D4-3 描述了 VA 分割的一个例子.

**Example D4-3 Example use of the split VA range, and the TTBR0_EL1 and TTBR1_EL1 controls**

---

**TTBR0_EL1**  
用于进程相关的地址设定。
每一个进程拥有一个独立的 level 1 translation table，在进行上下文切换时：
* TTBR0_EL1 会被设置为新的进程 level 1 translation table 的基地址。
* 如果 translation table 的大小发生变化，那么 TCR_EL1 也会相应的更新。
* CONTEXTIDR_EL1 (Identifies the current Process Identifier) 会相应的更新。

**TTBR1_EL1**  
用于操作系统和 I/O 相关的地址设定，此寄存器不会在上下文切换时改变。

---

对于每个 VA subrange，输入地址的大小为 2^(64-TnSZ)，其中 TnSZ 为 TCR_EL1.T0SZ 或者 TCR_EL1.T1SZ。也就是说，有以下两个 VA subrange：

**Lower VA subrange** 0x0000_0000_0000_0000 to (2^(64-T0SZ) - 1).

**Upper VA subrange** (2^64 - 2^(64-T1SZ)) to 0xFFFF_FFFF_FFFF_FFFF.

TnSZ 的最小值为 16，此时输出地址的范围达到最大，即 48。Example D4-4 描述了在 T0SZ 和 T1SZ 都为 16 时的两个 VA subrange。

**Example D4-4 Maximum VA ranges for EL1&0 stage 1 translations**

---

当 T0SZ 和 T1SZ 都设为最小值 16 时，两个 VA subrange 达到最大范围，分别为：
**Lower VA subrange** 0x0000_0000_0000_0000 to 0x0000_FFFF_FFFF_FFFF. 

**Upper VA subrange** 0xFFFF_0000_0000_0000 to 0xFFFF_FFFF_FFFF_FFFF.

---

Figure D4-15 描述了改变 TnSZ 的大小对两个 VA range 的影响。

TnSZ 的值还决定了 initial lookup 所在的 level，更多信息可以参考 [Overview of the VMSAv8-64 address translation stages](#) 章节。

### Concatenated translation tables for the initial stage 2 lookup

[Overview of the VMSAv8-64 address translation stages](#) 章节简单介绍了 initial stage 2 translation lookup 的 concatenate translation table 功能。本小节将更详细的介绍该功能。

如果 stage 2 translation 的 top-level translation table 的 entry 小于等于 16 个，那么系统设计时，可以考虑按照下面的方式来优化：
* 在 next translation level 中使用相应数量的 concatenated translation table。
* 将 initial lookup 设定到 next translation level 上。
（译者注：top-level translation table 的 entry 小于等于 16 个也就意味着该 level 的所解析的地址位数小于 4 位，通过 concatenated translation table 机制，可以将 top-level 所解析的地址位放到 next level 上解析，这样可以减少 translation 过程中的一次 lookup 操作，达到优化的效果。另外，在 next level 上多解析 n 个地址位，就需要将 2^n 个 table 进行 concatenate，最终的 concatenated translation table 的基地址还要对齐到所有 table 加起来后的 size。）

此外，当 stage 2 translation 使用 16KB translation granule 进行 48-bit input address 转换时，必须使用 2 个 concatenated translation table， 同时第一次 lookup 必须在 level 1。

> **NOTE:**  
在这种 translation scheme 中:
* 减少了一个额外的 translation level，即较少了一次 lookup 操作。
* 需要软件进行以下的配置操作：
    - 构建 concatenated translation table 并将其基地址对齐到 table size。
    - 设定 VTTBR_EL2 为 concatenated translation table 的基地址。
    - 设定 VTCR_EL2 中的 input address range 和 initial lookup level。

在 initial level lookup 中使用 concatenated translation table 可以在该 level 上解析额外的地址位。在 level 上多解析 n 个地址位，就需要将 2^n 个 table 进行 concatenate。Example D4-5 中，描述了 level 1 lookup 中，使用 4KB translation granule 时，使用 concatenated translation table 进行额外 3 个地址位的解析。

***Example D4-5 Adding three bits of address resolution at level 1 lookup, using the 4KB granule***

---

当采用 4KB translation granule 时，在 level 1 lookup 中使用 single translation table 可以解析的地址为为 bits[38:30]。如果需要增加 3 个地址位的解析，那么就需要增加 2^3 个，即 8 个 translation table，这就意味着：
* concatenated translation table 总的大小为 8 × 4KB = 32KB。
* 用于存储 concatenated translation table 的内存块的起始地址必须对齐到 32KB。
* 此次 lookup 所解析的地址范围为 A[41:30]，其中：  
  - A[41:39] 地址位用于选择 4KB translation table。
  - A[38:30] 地址位用于索引 translation table 中的 descriptor。

---

Table D4-24 汇总了采用 4KB translation granule， concatenated translation table 的各种使用情况。

![](table_d4_24.png)

> **NOTE:**  
Concatenation 只存在与 stage 2 translation，因此上面 table 中的 input address 是 IPA。  

[Overview of the VMSAv8-64 address translation stages](#) 章节中，描述了使用 concatenattion 的所有场景。在使用 concatenated translation table 时，起始地址一定要对齐到 table 集合的 size。

### Possible translation table registers programming errors

本小节主要描述在配置 translation table 相关的寄存器时，可能出现的错误情况。

#### Misprogramming the VTCR_EL2.{T0SZ, SL0} fields

对于 stage 2 translation，VTCR_EL2 寄存器中的 T0SZ （The size offset of the memory region addressed by VTTBR_EL2） 和 SL0 （Starting level）位存在一定的关系，进行配置时，必须要满足相应的条件。这两个寄存器位之间的关系，可以参考 [Overview of the VMSAv8-64 address translation stages](./d42_5_overview_of_the_vmsav8-64_address_translation.html) 章节。

（TODO: 后面内容的翻译将在完成 The Contiguous bit 章节的翻译后再进行）
#### Misprogramming of the Contiguous bit
For more information about the Contiguous bit, and the range of translation table entries that must have the bit set to 1 to mark the entries as contiguous, see [The Contiguous bit on page D4-1715](#).

If one or more of the following errors is made in programming the translation tables, the TLB might contain overlapping entries:

* One or more of the contiguous translation table entries does not have the Contiguous bit set to 1.
* One or more of the contiguous translation table entries holds an output address that is not consistent with all of the entries pointing to the same aligned contiguous address range.
* The attributes and permissions of the contiguous entries are not all the same.

Such misprogramming of the translation tables means the output address, memory permissions, or attributes for a lookup might be corrupted, and might be equal to values that are not consistent with any of the programmed translation table values.

In some implementations, such misprogramming might also give rise to a TLB Conflict abort.

The architecture guarantees that misprogramming of the Contiguous bit cannot provide a mechanism for any of the following to occur:

* Software executing at EL1 or EL0 accessing regions of physical memory that are not accessible by programming the translation tables, from EL1, with arbitrary chosen values that do not misprogram the Contiguous bit.
* Software executing at EL1 or EL0 accessing regions of physical memory with attributes or permissions that are not possible by programming the translation tables, from EL1, with arbitrary chosen values that do not misprogram the Contiguous bit.
* Software executing in Non-secure state accessing Secure physical memory.

> **NOTE:**  
Hardware implementations must ensure that use of the Contiguous bit cannot provide a mechanism for avoiding output address range checking. This might occur if a Contiguous bit block size of 0.5GB or 1GB is used in a system with the output address size configured to 4GB. The architecture permits the implemented mechanism for preventing any avoidance of output address range checking to suppress the use of the Contiguous bit for such entries in such a system.

Where the Contiguous bit is used to mark a set of blocks as contiguous, if the address range translated by a set of blocks marked as contiguous is larger than the size of the input address supported at a stage of translation used to translate that address at that stage of translation, as defined by the TCR.TxSZ field, then this is a programming error. An implementation is permitted, but not required, to:

* Treat such a block within a contiguous set of blocks as causing a Translation fault, even though the block is valid, and the address accessed within that block is within the size of the input address supported at a stage of translation, as defined by the TCR.TxSZ field.
* Treat such a block within a contiguous set of blocks as not causing a Translation fault, even though the address accessed within that block is outside the size of the input address supported at a stage of translation, as defined by the TCR.TxSZ field, provided that both of the following apply:
    - The block is valid.
    - At least one address within the block, or contiguous set of blocks, is within the size of the input address supported at a stage of translation.