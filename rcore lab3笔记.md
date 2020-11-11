# rcore lab3笔记

## entry.asm详细分析

#### 计算boot_page_table物理页号

`lui t0, %hi(boot_page_table)` 

​	lui本身会将目标左移12位，然后取最左边32位，比如0x12345会变成0x12345000，因为内核的地址开始被设置为了0xffffffff80200000，所以boot_page_table的地址必定为0xffffffff802XXXXX，而%hi取boot_page_table的12到31位共20位，即0x802XX，使用lui加载立即数会用第31位的值填充32到63位，比如第31位为1，则32位到63位也为1；31位为0，则32位到63位也为0.

​	由此可得出lui的过程为先将0x802XX左移12位变为0x802XX000，之后根据第31位的值，也就是1，填充高32位，最终t0结果为0xffffffff802XX000.

`li t1, 0xffffffff00000000`

​	将立即数加载到临时变量t1中，便于之后计算。

`sub t0, t0, t1`

​	t0 = t0 - t1，结果为0xffffffff802XX000 - 0xffffffff00000000 = 0x802XX000。

`srli t0, t0, 12`

​	t0 = t0 >> 12，结果为0x802XX，注意这里是逻辑右移，直接补0即可。

**以上步骤为计算boot_page_table物理页号的过程，相对麻烦，好像risc-v中没有将%hi(boot_page_table)的结果直接放入寄存器中的指令。**

#### 组装satp的值并放入，最后刷新TLB

`li t1, (8 << 60)`

​	satp最右侧4位为mode，设置为8表示虚拟地址以sv39格式表示。

`or t0, t0, t1`

​	t0 = t0 | t1，最终t0的值为0x8000 0000 0008 02XX。

`csrw satp, t0`

​	将t0组装好的值通过控制状态寄存器专用指令写入到satp中，表示三级页表位置。

`sfence.vma`

​	刷新tlb，由于更改了页表基址寄存器，所以tlb之前缓存的页表已经无效，需要重新加载，刷新一下。

#### 加载栈虚拟地址

`lui sp, %hi(boot_stack_top)`

`addi sp, sp, %lo(boot_stack_top)`

​	首先加载boot_stack_top的12位到31位，由于lui指令补最高位的特性，相当于加载12位到63位，之后使用addi将0到11位加到sp上，完成boot_stack_top到sp的加载过程。

`lui t0, %hi(rust_main)`

`addi t0, t0, %lo(rust_main)`

​	同上，将rust_main地址加载到t0中。

`jr t0`

​	等同于jalr x0, 0(t0)，直接跳转到t0保存的地址，不需要保存当前的pc值，程序正式从这里开始运行。

**到这里代码部分就结束了，后面内容是自己定义栈和初始页表在内核中的位置，大小以及它们的内容，其中栈分配了16k的大小，放在.bss段。之后是初始页表，共512项，每项8B，共4K，相当于一页物理内存大小，其中填充了3项，中间一项代表外设，不讨论：**

`.8byte (0x80000 << 10) | 0xcf`

​	第一项填充，也就是页表的第2项，是低地址虚拟内存到物理内存的映射。比如说刚从opensbi跳转过来时，pc的值还是低地址，0x8020_0000，对应的虚拟页号为0x2，正好对应到页表的第三项，因为页表项的RWX不全为0，说明这个页表直接指向1G大的物理页，所以页内偏移就是12+9+9=30位，即0x20_0000，再加上物理页号0x80000左移12位的值，可得出物理地址也为0x8020_0000。

`.8byte (0x80000 << 10) | 0xcf`

​	第三项填充，也就是页表的510项，是正常虚拟内存到物理内存的映射。当你的虚拟页号为0x1ff(也就是2进制的111111111)时，正好对应到页表的第510项。由于第510项的内容和第2项的内容相同，所以最终映射到的物理地址也相同。另外，由于risc-v的规则是虚拟地址的63到39位的值要等于第38位，所以虚拟地址的表现形式为0xffffffff802XXXXX。

**以上就是页表内容的分析，在设置satp，刷新tlb之后，内核正式转换到虚拟地址，但是此时pc的值还未更改，仍然是低地址，所以需要页表中加入低地址转换，当jr t0后，跳转到rust_main的位置之后，pc的值也就变成了正常的高位虚拟地址，开始使用第510项的页表项来转换。不过这里的页表内容非常少，转换也很粗糙，每个项会映射1G的大小，非常不实用，所以需要在内核启动后再精细化页表和内核的内存布局。**

## 内存管理

### 创建的类

#### MemorySet

包含Mapping和Segment动态数组

```rust
pub struct MemorySet {
    /// 维护页表和映射关系
    pub mapping: Mapping,
    /// 每个字段
    pub segments: Vec<Segment>,
}
```

##### Mapping

```rust
pub struct Mapping {
    /// 保存所有使用到的页表
    page_tables: Vec<PageTableTracker>,
    /// 根页表的物理页号
    root_ppn: PhysicalPageNumber,
    /// 所有分配的物理页面映射信息
    mapped_pairs: VecDeque<(VirtualPageNumber, FrameTracker)>,
}
```

###### PageTableTracker

```rust
/// 类似于 [`FrameTracker`]，用于记录某一个内存中页表
pub struct PageTableTracker(pub FrameTracker);
```

###### PageTable

```rust
pub struct PageTable {
    pub entries: [PageTableEntry; PAGE_SIZE / 8],
}
```

###### PageTableEntry

```rust
pub struct PageTableEntry(usize);
```

###### FramerTracker

```rust
pub struct FrameTracker(pub(super) PhysicalPageNumber);
```

###### PhysicalPageNumber

```rust
pub struct PhysicalPageNumber(pub usize);
```

###### VirtualPageNumber

```rust
pub struct VirtualPageNumber(pub usize);
```

##### Segment

```rust
pub struct Segment {
    /// 映射类型
    pub map_type: MapType,
    /// 所映射的虚拟地址
    pub range: Range<VirtualAddress>,
    /// 权限标志
    pub flags: Flags,
}
```

###### MapType

```rust
pub enum MapType {
    /// 线性映射，操作系统使用
    Linear,
    /// 按帧分配映射
    Framed,
}
```

###### Range

```rust
/// 表示一段连续的页面
pub struct Range<T: From<usize> + Into<usize> + Copy> {
    pub start: T,
    pub end: T,
}
```

###### VirtualAddress

```rust
pub struct VirtualAddress(pub usize);
```

###### Flags

```rust
/// 页表项中的 8 个标志位
pub struct Flags: u8 {
        /// 有效位
        const VALID =       1 << 0;
        /// 可读位
        const READABLE =    1 << 1;
        /// 可写位
        const WRITABLE =    1 << 2;
        /// 可执行位
        const EXECUTABLE =  1 << 3;
        /// 用户位
        const USER =        1 << 4;
        /// 全局位，我们不会使用
        const GLOBAL =      1 << 5;
        /// 已使用位，用于替换算法
        const ACCESSED =    1 << 6;
        /// 已修改位，用于替换算法
        const DIRTY =       1 << 7;
}
```

### 总结

​	以上的类从大到小，逐渐抽象，分为两大类，一类是内存的抽象，一类是页表的抽象。

​	以内存抽象来看，物理页，虚拟页是最小单位，之上是Segment，包含了一类权限相同的页，再之上是MemorySet，也就是最大的抽象，代表了一个程序的内存空间，分为许多个Segment，每个Segment包含了不同的内容，权限也不尽相同。

​	页表抽象来看，页表项是最小单位，之上是最多能保存512个页表项的页表，一个PageTable和一个PageTableTracker相对应，通过页表项和PageTableTracker也可以找到PageTable，再之上就是Mapping，保存了一个进程所有的页表信息，三级页表的地址，和进程已分配物理页面的虚拟页号和物理页的对应关系。

​	Mapping和Segment加起来组成了一个进程基本的内存信息，通过Mapping，可以更新stap，刷新tlb，实现虚拟地址到物理地址的转换，同时能处理进程结束后的内存释放。通过Segment可以控制进程内存内每个区域的访问权限，它的映射类型，主要区分为线性映射和分配映射，前者用于操作系统的内核初始化，后者用于进程的内存分配。

## 内核页表初始化过程分析

#### 1. main函数中调用MemorySet中的new_kernel()

​	首先获取linker.ld中的四个段的地址，之后将段精细化，将每个段抽象为自定义的结构体段segment，之后设置每个段的映射类型，地址范围以及flag标记，代表这个段的权限，共5段，分别代表.text、.rodata、.data、.bss以及内存的剩余空间，从内核结束地址开始到内存结束地址结束。

```rust
let mut mapping = Mapping::new()?;

// 每个字段在页表中进行映射
for segment in segments.iter() {
    mapping.map(segment, None)?;
}
Ok(MemorySet {
    mapping,
    segments,
})
```

​	之后，将刚才分配的空间真正的映射到内存中，调用mapping结构体中的map函数来执行映射步骤，成功则返回一个初始化好的MemorySet。

#### 2. MemorySet中的Mapping new()函数

```rust
pub fn new() -> MemoryResult<Mapping> {
    let root_table = PageTableTracker::new(FRAME_ALLOCATOR.lock().alloc()?);
    let root_ppn = root_table.page_number();
    Ok(Mapping {
        page_tables: vec![root_table],
        root_ppn,
        mapped_pairs: VecDeque::new(),
    })
}
```

​	首先创建一个一级页表，也就是根页表，然后调用帧分配器为根页表分配一个物理页面。之后定义一个保存根页表物理页号的变量，其中mapped_pairs代表为进程分配的物理页面（不保存页表本身分配的页面），因为目前还未分配，所以没有值。最后返回新创建的Mapping。

#### 3. MemorySet中调用Mapping map()函数

​	首先在new_kernel()中创建了一个新的Mapping，之后迭代上面创建的5个segment，分别调用map()函数进行内存映射，因为内核segment的特殊性，map函数只执行下面的3行代码：

```rust
for vpn in segment.page_range().iter() {
    self.map_one(vpn, Some(vpn.into()), segment.flags | Flags::VALID)?;
}
```

​	map()函数内部，首先会判断segment的映射类型，因为内核的segment映射类型均为Linear，而且初始数据为None，所以直接循环segment的地址范围，为每个范围代表的页表分别映射物理地址，并传入虚拟页号、物理页号、对应的权限和是否可用flag，最后返回一个OK。

#### 4. Mapping中调用map_one()函数

```rust
/// 为给定的虚拟 / 物理页号建立映射关系
fn map_one(
    &mut self,
    vpn: VirtualPageNumber,
    ppn: Option<PhysicalPageNumber>,
    flags: Flags,
) -> MemoryResult<()> {
    // 定位到页表项
    let entry = self.find_entry(vpn)?;
    assert!(entry.is_empty(), "virtual address is already mapped");
    // 页表项为空，则写入内容
    *entry = PageTableEntry::new(ppn, flags);
    Ok(())
}
```

其中MemoryResult是一个Result类型的定制类型:

```rust
pub type MemoryResult<T> = Result<T, &'static str>;
```

调用Mapping中的find_entry()函数根据vpn找到对应的三级页表项，如果该页表项已经存在，说明已经映射完毕，为空的话就调用pageTableEntry的new()函数写入物理页号和权限，因为是线性映射，所以物理页号可以根据偏移直接推出，也不需要调用分配器分配物理页面以及在mapped_pairs中保存对应信息。最后返回一个成功标记OK。

#### 5. main函数调用MemorySet中的activate()函数

##### activate()分析：

​		memoryset中调用的实际上市mapping里面的activate函数，这个函数很简单，就是讲刚才映射的页表基址地址放入satp中，并刷新tlb，替代了一开始写的那个粗糙的页表，这样页表的精细化就真正完成了。

**至此，rcore的内存管理模块已经完成，为接下来的进程管理打下基础。**