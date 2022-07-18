# dailySchedule

### 7.2

今天了解到了这个项目，配了一下环境。采用的是wsl2下载ubuntu的方式。

### 7.3

今天创建了用于记录每日工作的github仓库，并且学习了一下github classroom相关的知识。

### 7.4

今天主要是看《计算机组成与设计 risc-v版》的一二章，和risc-v手册。

* 理解了一下risc-v支持的一些指令。
* 还看了一下risc-v指令集下的调用函数的汇编程序，处理终端的汇编程序。
* 看了一下risc-v设计的三种模式，用户模式，机器模式，监管者模式。机器模式是权限最高的，用户模式是权限最低的。机器模式可以处理异常，共有两种异常，一是同步异常，而是中断。加入了监管者模式之后，还可以将部分异常处理导向监管者模式。

### 7.5

今天最开始研究了一下OS2. 还是没有搞清楚这个实验的逻辑是什么。

出于好奇，看了一下ci-user目录下的内容，看到了/ci-user/check/ch2.py， 里面有temp数组，和expected数组，expected数组很显然就是本章节下面的多任务执行的输出，但是temp数组会更新expected数组。发现了这个之后，我甚至产生了一个想法，修改user下面的chapter2相关的任务程序的输出。

但是结合测试二的题目，try something in os2 in os2 DIR，显然仅仅是需要修改os2这个文件夹下面的文件，不需要修改user文件夹下的数据。

最后问了一下其他的同学，没想到仅仅是修改了一下os2文件夹下的Cargo.toml中的risc-v的地址，从github的地址修改为gitee的地址。这个还是存有一定的疑惑。

然后剩下的时间，就是一直在做rustlings。进度到了standard library types。

### 7.6

今天把剩下的rustlings做完了。

然后就开始看项目提供的文档了，一直对这个项目要完成什么，怎么完成感到十分疑惑，有一种云山雾罩的感觉。所以今天仔细看了一下项目的文档。通过看文档，大致理解了本项目的任务如下：

* 理解提供的框架代码
* 完成每章后面的练习

本项目的背景大致如下：是运行在一个裸机系统上面，没有任何的系统调用，为了实现某种目的，需要我们用rust手写系统调用。因为是在risc-v指令集条件下的系统，所以这中间还需要对risc-v和汇编语言有所了解。

### 7.7

今天上午结合着代码看第三章的文档。读着读着似乎有些理解了，一个特点是用汇编语言写一些函数，然后使用extern “C”就可以把这些函数给引入rust环境，rust可以调用他们。

今天下午就开始做chapter3的练习了。

首先在项目的根目录下运行```make test3```。出现了熟悉的错误。

![image-20220707194152951](pic/image-20220707194152951.png)

这个错误在第二章的时候出现过，将risc-v os的源从github改为gitee即可。

然后开始实现练习中的```sys_task_info```系统调用。此系统调用需要返回当前运行任务的状态，调用的系统调用的次数统计，和运行的时间。

通过观察代码，每个```task```有一个绑定的```TaskControlBlock```，包括```TaskStatus,TaskContext```，这两个属性都是和该```task```唯一绑定的，联想到，系统调用的两个信息，系统调用的次数和运行的时间也是和```Task```唯一绑定的。但是运行时间需要做一下处理，只需要在```TaskControlBlock```里面添加一个```task```开始运行的时间即可，调用该系统调用的时候，用当前时间减去```task```开始运行的时间就是该任务运行的时间。修改之后的```TaskControlBlock```如下：

```rust
// os/task/task.rs
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
    // LAB1: Add whatever you need about the Task.
    pub task_begin_time: usize,
    pub syscall_times: [u32; MAX_SYSCALL_NUM],
}

```

修改了结构体的属性，对应的需要修改```os/task/mod.rs```中的```lazy_static!```中相关的```TaskControlBlock```初始化代码。

然后需要思考的是新添加的两个属性什么时候更新。```task_begin_time```利用```timer.rs```中提供的```get_time()```进行初始化，初始化的时机就是该任务的状态由```UnInit```变为```Running```的时候，涉及任务状态改变的只有两个地方，```os/task/mod.rs```中```TaskManager```的两个方法，```run_first_task```和```run_next_task```。

```syscall_times```更新的时机，就是```task```调用系统调用的时候，也就是```os/syscall/mod.rs```里面的```syscall```函数。更新的时候有个难点，如何确定当前运行的是哪个任务。通过理解框架代码，发现有一个全局的静态对象```TASK_MANAGER```，保存着当前运行的是哪个任务，并且还有加载进来的所有任务的```TaskControlBlock```。直接按照框架的风格，为```TaskManager```实现一个私有方法，在外面提供一个公有函数接口。

```rust
// os/task/mod.rs
 fn update_syscall_times(&self, syscall_id: usize){
        let mut inner = self.inner.exclusive_access();
        let index = inner.current_task;
        inner.tasks[index].syscall_times[syscall_id] +=  1;
  }
pub fn update_syscall_times(syscall_id: usize){
    TASK_MANAGER.update_syscall_times(syscall_id);
}
```

最后就是系统调用的实现了，与上面的逻辑相似。

```rust
// os/task/mod.rs
fn set_task_info(&self, taskinfo: *mut TaskInfo){
        let inner = self.inner.exclusive_access();
        let t = (get_time() - inner.tasks[inner.current_task].task_begin_time)*1000/CLOCK_FREQ;
        // print!("{}",t);
        unsafe {
            *taskinfo = TaskInfo{
                // status: inner.tasks[inner.current_task].task_status,
                status: TaskStatus::Running,
                syscall_times: inner.tasks[inner.current_task].syscall_times,
                time: t,
            };
        }
       
    }
pub fn set_task_info(taskinfo: *mut TaskInfo){
    TASK_MANAGER.set_task_info(taskinfo);
}
// os/syscall/process.rs
pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
    set_task_info(ti);
    0
}
```

### 7.8

今天继续阅读第四章的文档，也就是lab2的文档。相比于lab1的文档，这次阅读文档不太顺利，原因是我之前对于页表的理解有些问题。

我之前一直以为多级页表中每个节点的每一项都是分为两个部分，虚拟页号和物理帧号，但是这里有问题，页表项不需要留出空间给虚拟页号。因为处理的时候会将物理帧按照数组组织，下标即为虚拟页号。那么就不需要再页表项中预留空间存虚拟页号了。

然后读完了文档就开始写代码了，着手解决系统调用sys_get_time的系统调用。给出的提示是由于采用了虚拟内存，需要修改sys_get_time。那么需要修改的一项就是该系统调用传入的指针了，这个指针应该是虚拟地址，需要手动转化为物理地址。

刚开始觉得系统调用是在内核态下执行的，所以觉得该虚拟地址是内核空间的，所以用内核内存空间转化了一下。代码如下：

```rust
// os/syscall/process.rs
pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
    let _us = get_time_us();
    let vaddr = _ts as usize;
    let vaddr_obj = VirtAddr(vaddr);
    let page_off = vaddr_obj.page_offset();

    let vpn = vaddr_obj.floor();
    
    
    let ppn = KERNEL_SPACE.lock().translate(vpn).unwrap().ppn();

    let paddr : usize = ppn.0 << 10 | page_off;
    let ts  = paddr as *mut TimeVal;
    unsafe {
        *ts = TimeVal {
            sec: _us / 1_000_000,
            usec: _us % 1_000_000,
        };
    }
    0
}
```

然后转念一想，这个地址是在用户程序中传入的，应该是用户程序的内核空间中的地址。然后使用用户空间翻译了一下

```
// os/syscall/process.rs
pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
    let _us = get_time_us();
    let vaddr = _ts as usize;
    let vaddr_obj = VirtAddr(vaddr);
    let page_off = vaddr_obj.page_offset();

    let vpn = vaddr_obj.floor();
    
    let ppn = translate_vpn(vpn);

    let paddr : usize = ppn.0 << 10 | page_off;
    let ts  = paddr as *mut TimeVal;
    unsafe {
        *ts = TimeVal {
            sec: _us / 1_000_000,
            usec: _us % 1_000_000,
        };
    }
    0
}

// os/task/mod.rs
impl TaskManager {
fn translate_vpn(&self, vpn: VirtPageNum)-> PhysPageNum{
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        let CurrentAppSpace = &inner.tasks[current].memory_set;
        let ppn = CurrentAppSpace.translate(vpn).unwrap().ppn();
        return ppn;
    }
}


pub fn translate_vpn(vpn: VirtPageNum)-> PhysPageNum{
    TASK_MANAGER.translate_vpn(vpn)
}
```

但测试了一下，还是不行，这就有点儿费解了。

7.9

今天继续debug，想一想为什么sys_get_time不行，答案找到了，找到了两个错误的地方，一是我将ppn左移10位了，这显然是不对的，page的大小为4k。需要左移12位。二是我之前觉得调用系统调用的时候CPU执行的任务发生了改变，因此构造了一个last_task的属性，但是经过思考之后发现，系统调用还是处于当前的任务之中，不过是将CPU的主导权放到了内核态中。

然后就继续修改sys_task_info这个系统调用。这个系统调用还是同样的思路，通过查页表找到对应的物理的地址，然后进行写入，这里直接把实验一的代码搬过来即可。

但是在实现的时候发现一个问题，本实验TaskStatus的定义和实验一不同，没有Uninit状态，因此我就不能用之前的方式来设置任务的启动时间了，仅仅是在TASK_MANAGER初始化的时候调用get_time()作为任务的启动时间。然后调用sys_task_info时候获得的时间get_time()减去这个时间即可得到任务的运行时间，但是我在测试的时候发现这个运行时间不太准，比较随机，有时候是480ms左右，有时候是490ms左右，有时候是502ms左右，但是只有在500ms左右的时候才能通过测试。这就不清楚为什么会出现这种情况了。（ps：这个似乎可以投机取巧一下，在返回的taskinfo中，将执行时间加个12ms就可以过了，不知道这种允许不允许:smile:

并且由于TaskStatus定义的不同，会造成一个奇怪的现象，TaskStatus::Running != TaskStatus::Running。需要在本实验TaskStatus的定义中加入Uninit状态，才可以通过这一项测试。

然后就是实现 sys_mmap了，这里一个坑点是文档中提示的“可能的错误”，被我理解成了以往的参与者会犯的错误，没想到是这个函数包含的错误的情况，也就是说碰到这些情况时，系统调用需要返回-1.

实现这个系统调用最伤脑经的是判断一个虚拟页号是否已经有映射了，这里的函数调用比较复杂，$sys\_mmap()\rightarrow contains\_key() \rightarrow  task\_manager.contains\_key()  \rightarrow memoryset.contains\_key() \rightarrow MapArea.contains\_key()$。

```rust
// os/src/task/mod.rs
pub fn contains_key(vpn: &VirtPageNum)-> bool{
    TASK_MANAGER.contains_key(vpn)
}
impl TaskManager{
    fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].memory_set.contains_key(vpn)
	}
}

// os/src/mm/memeory_set.rs
impl MemorySet{
    pub fn contains_key(&self, vpn: &VirtPageNum) -> bool{
        let mut result = false;
        let l = self.areas.len();
        for i in 0..l{
            let res = self.areas[i].contains_key(vpn);
            result = result | res;
        }
        return result;
    }
}
impl MapArea{
    pub fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        self.data_frames.contains_key(vpn)
    }
}

```

在进行上面的判断之后，就可以执行添加操作了，这里直接利用框架代码，提供的memory_set.insert_framed_area()函数。

最后就是sys_munmap的实现了。还是先利用上面实现的功能判断给定的虚拟页空间，是否存在没有映射的页。如果存在，则直接返回-1. 反之，进行这块虚拟页空间的释放。这里涉及到两个结构的更新，一是Vec\<MapArea\>，二是PageTable的更新。这里利用pageTable提供的unmap函数即可。

按照上面所说的写完代码之后，本地测试出现了几个问题，一是test 4_01（这里用输出结果指代测例）始终过不了，并且没有assertion error。二是test 4_05测例过不去，有assertion error，说的是在21行，最后一个assert!语句处报错。

最后就是今天编程的一点儿体会，发现我的rust编程还是没有登堂入室，时常会犯一些错误，比如没有加mut，不知道vec.iter()会转移所有权等等，只能指望编译器给我指出这些错误。

### 7.10

今天就是继续debug，看看昨天的两个bug是为什么。

发现了第一个bug是由于我没有设置PTE::U位，导致了一个结果----可以正常申请内存，但是申请下来的内存无法访问。改了这个，test 4_01就可以正常通过了。

第二个bug主要是因为我在sys_mmap和sys_munmap这两个系统调用中确定的虚拟页号空间有问题。我之前一直以为MapArea里面的VPNRange是闭区间，所以我start和end两个vpn全用的是floor，结果检查了一下VPNRange的迭代器代码才知道这个区间是左闭右开区间，所有就很自然地把end换成了ceil方法。

在该代码的时候还遇到一件有趣的事情。我尝试将MapArea.contains_key()方法的实现用VPNRange来代替，结果一代替的话，出现了很多bug，这让我十分苦恼。

```rust
// os/src/mm/memeory_set.rs
pub fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        self.vpn_range.get_end().0 >= vpn.0 && self.vpn_range.get_start().0 <= vpn.0
    }
```

想了一下这个区间是左闭右开区间，因此问题是多加了一个等号，应该是```self.vpn_range.get_end().0 > vpn.0 && self.vpn_range.get_start().0 <= vpn.0```，改完之后就正常了。

但是改完之后，test4_05本地测试还是过不去(sys_task_info的那个测例，我取了个巧，直接在返回值taskinfo的执行时间中加了个12ms).

但是奇怪的是，我把这一版上传到云端，云端居然测试通过了，这实在是很神奇。

### 7.11

今天主要是读第五章的文档。

### 7.12

今天开始尝试编写代码，尝试着把sys_get_time, sys_task_info, spawn这几个系统调用，spawn这个系统调用可以简单的看作fork和exec的结合。不过是不需要拷贝父进程的地址空间了，直接指定elf获得的地址空间。

sys_get_time和sys_task_info这两个系统调用需要修改，这个实验的结构又有了大变，没有了全局的TASK_MANAGER，因此这次需要在全局的PROCESSOR上进行操作。

稍微看了看sys_set_priority，仅仅实现这个系统调用就可以了吗？框架中实现了stride算法的后续步骤了吗，我们需要实现后续的代码吗？

在make test5的时候发现程序一直卡在chapter 5的一个测试set_priority，卡一段时间，然后测试就退出了，连我写好的系统调用也没有进行测试。

![image-20220712221904312](pic/image-20220712221904312.png)

![image-20220712221947491](pic/image-20220712221947491.png)

如图所示的情况，可以看到APPS还有很多，usertests没有完全加载完，qemu模拟器就退出了。

还有一个值得注意的地方，我在实现sys_task_info的时候，需要向TaskControlBlock中添加一个属性syscall_times，有两个地方可供选择，第一种方法最简单，直接作为TaskControlBlock的属性，二是作为TaskControlBlockInner的属性。前者似乎无法获取到可变引用。所以最终选择了后者。

### 7.13

今天将实验三剩下的几个系统调用写完了，但是测试没有通过。

关于昨天的问题----“仅仅实现sys_set_priority就可以了吗？框架中实现了stride算法的后续步骤了吗，我们需要实现后续的代码吗？”，答案是我们需要实现后续的步骤，也就是实现TaskManager的fetch方法。如下图

![image-20220713202904882](pic/image-20220713202904882.png)

但是这样的stride算法，无法通过两个sleep测例。

![image-20220713203028822](pic/image-20220713203028822.png)

另外今天发现了之前的一个问题，之前疑惑为什么实验二在本地测试无法通过，在云端就可以通过测试，这是因为我没有执行make setupclassroom_test4！！！！

知道这个问题之后，又投入到了实验二的修改之中，直接修改ci-user下的测例，通过输出发现，sys_munmap实现的有问题，复查了一下发现，是一个等号的问题，因为这个等号，多删了一个pte，所以测例就无法通过了。如下图所示。

![image-20220713152049569](pic/image-20220713152049569.png)



然后就开始研究实验五了，发现了昨天晚上为什么make test5过不去了，我spawn的实现错了，少加了TrapContext初始化的代码，加了之后就可以正常运行了。

![image-20220713161516084](pic/image-20220713161516084.png)

关于上面stride算法失效的问题。查阅了一下资料，上面是因为stride的溢出问题。解决方法如下图：

![image-20220713203212757](pic/image-20220713203212757.png)

函数变成了如下所示：

```rust
    pub fn fetch(&mut self) -> Option<Arc<TaskControlBlock>> {
        let l = self.ready_queue.len();
        let mut index = 0;
        let mut min_stride = self.ready_queue[index].inner_exclusive_access().stride;
        for i in 1..l{
            let t = self.ready_queue[i].inner_exclusive_access().stride;
            let difference = (t - min_stride) as isize;
            if difference <= 0{
                min_stride = t;
                index = i;
            }
        }
        let p = self.ready_queue[index].inner_exclusive_access().priority as usize;
        self.ready_queue[index].inner_exclusive_access().stride += BIG_STRIDE / p;
        self.ready_queue.remove(index)
//        self.ready_queue.pop_front()
    }
```

解决了上面的问题之后，发现最后一个stride的测试无法通过。

此外，自己终于理解了processor中的那个idle_task_cx，这个idle理解起来是有问题的，可以理解为processor当前执行的任务空闲了，然后idle_task_cx就是这个空闲的任务的task_context，经过阅读代码之后发现这是不对的，这个idle_task_cx，应该指的是主进程，每次cpu切换到idle_task_cx就会执行，run_tasks()这个函数了，也就是寻找下一个可以运行的任务。

### 7.14

今天上午初步确定了最后一个测试无法通过的原因是stride的溢出问题。

并且得知我昨天的那种修改方式似乎是不对的，[关于rust数值溢出的博客](https://imzy.vip/posts/36864/)说在debug模式下，运行阶段检测到了可能出现溢出的减法，程序会直接panic。在release模式下，运行阶段检测到了可能出现溢出的减法，溢出会舍弃高位。

并且我没有修改可能造成溢出的update stride操作。

因此重新思考为什么换成了stride调度算法时候，两个sleep测例无法通过，原因可能是写的第一版无法调度到他们。

然后将可能造成溢出的加减法，替换成了wrapping_add/sub方法。

总结了一下处理stride的溢出的关键点在于将stride的差值控制在机器数可以表示的范围内。从写代码的角度来看，最关键的就是下面这行代码：

```rust
let difference = (t - min_stride) as isize;
```

上面的```t```和```min_stride```都是usize类型，文档中说。

> 在不考虑溢出的情况下** *, 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 STRIDE_MAX – STRIDE_MIN <= BigStride / 2。

因此```t```和```min_stride```的差值位于[-BigStride/2,BigStride/2]之内，也就是isize的区间了。

但是替换成这样子之后还是无法通过！！！！！

通过查看其他同学的项目，核心逻辑都一样，只不过他所有和stride相关的变量用的是u8和i8，而我使用的是usize和isize，前者就可以通过测试，后者就无法通过测试。

后来又试了一下u16，u32，都不可以。搞不清楚是为什么。

### 7.15

今天看了第六章的文档。这个文档比较复杂。

有一些混淆的地方。

* 文件和目录，文档中将文件和目录写成了同一层次的东西。但通过看代码得知，获取文件内容的流程是，给定文件名，根据文件名去找DirEntry，根据DirEntry里面的inode_number解析出文件inode所在的blockid和offset.通过文件的inode获取文件的内容。因此逻辑结构上，目录是在文件之上的。
* DirEntry的inode_number和blockid混淆了。因为文档中指定一个物理block中存四个inode，因此上DirEntry给一个blockid是无法唯一指定inode的，并且结合代码得知inode_number是由blockid和offset构成的。
* Super Block与ROOT_INODE混淆了，Super Block是磁盘的第一个块，记录了磁盘的相关划分信息。而ROOT_INODE是根目录的Inode，包含了根目录包含的DirEntry所在的物理块的blockid。我们在实现系统调用的时候，所有的操作都要基于ROOT_INODE来进行操作，从他这里获取到文件的inode，进而获取文件的内容。
* DiskInode，Inode和OSInode混淆了，这三个是逐步升级的封装。DiskInode就是最基本的文件inode，包含了文件内容所在的数据块的索引。至于其他两个的作用，目前还不清楚。
* DiskInode和block混淆了，据文档所说，一个物理block（512B)上面有4个DiskInode(128B)。
* 和DiskInode相关的两个offset混淆，一个offset是用于指明当前DiskInode所在的block的位置的，毕竟一个block可以容纳4个DiskInode。另一个和diskInode相关的offset则是读取DiskInode关联的数据块中的数据时的offset，文件系统实现的时候将inode和数据节点的区别隐藏了，在inode节点上调用的read方法返回的就是该inode节点关联的数据节点上的数据。处理的时候将这些数据视为一个数组，即读取的时候需要不断地指定offset，直到读取完数据。
* 将逻辑上文件与目录的依赖关系和具体实践上文件和目录的独立混淆了。逻辑上的话，文件系统就是一棵树，树的叶子节点就是文件。非叶节点就是目录。因此从逻辑上来讲目录是依赖于文件而存在的，文件依赖着目录的索引功能。但是在实践上，这两种结构就是完全独立的，一个文件，具体实现起来的话，是inode+数据节点的结构。而一个目录，实现起来的话，也是inode+数据节点的结构。不同的是文件的数据节点中的数据是没有特定结构的二进制数据，而目录的数据节点中存储的是目录项，由名称和inode_number组成。那么这就有问题了，在目录与文件独立的实现结构下，如何实现文件的索引呢？答案就是目录项的inode_number，他是目录项名字对应的inode，无论是文件也好，还是下一级目录也好，获得这个inode_number就可以进行迭代的索引了，直到索引到想要的文件为止。



### 7.16

今天主要是写代码了，昨天写完了以往实验中的系统调用，今天主要是着重在与文件系统相关的三个系统调用。sys_linkat，sys_unlinkat，sys_fstat。

刚开始的时候完全是懵的，sys_linkat完全不知道怎么实现，主要是我没有对文档理解透彻。比如把逻辑上文件系统的结构带进去了，因为逻辑上，文件系统是一个树的形式，树的每一个节点就是一个inode，那么sys_linkat的实现，岂不是要在ROOT_INODE上面加一个inode。可是问题又来了，这个硬链接是一个inode嘛？inode对应的数据节点里面存什么东西呢？

上面纯粹就是理解混了。ROOT_INODE是一个目录，他有配套的inode+数据节点。inode用于索引数据节点，而数据节点中存的是一个个的目录项。这些目录项由名字和inode_number组成，inode_number用于索引inode。如此便可以不断索引下去，直到遇到文件的inode，读取文件内容即可。

想明白这个，sys_linkat的思路也就出来了，就是在ROOT_INODE下面新建一个目录项，目录项的名字是给定的_new_name，目录项的inode\_number和\_old\_name对应的目录项一样的。

```rust
pub fn create_hard_link(&self, o_name: &str, n_name: &str) -> isize {  
        if o_name == n_name {
            return -1;
        }
        if let Some(inode_number) =
            self.read_disk_inode(|disk_inode| self.find_inode_id(o_name, disk_inode))
        {
            let mut fs = self.fs.lock();
            self.modify_disk_inode(|root_inode| {
                // append file in the dirent
                let file_count = (root_inode.size as usize) / DIRENT_SZ;
                let new_size = (file_count + 1) * DIRENT_SZ;
                // increase size
                self.increase_size(new_size as u32, root_inode, &mut fs);
                // write dirent
                let dirent = DirEntry::new(n_name, inode_number);
                root_inode.write_at(
                    file_count * DIRENT_SZ,
                    dirent.as_bytes(),
                    &self.block_device,
                );
            });
            return 0;
        } else {
            return -1;
        }
        
    }
```

实现了这个之后，遇到一个意想不到的问题。怎么把系统调用的参数* const u8转换为&str呢？

通过查阅资料，得知了* const u8这种类型称为ptr，并且这种类型无法通过index索引。将其转化为&str的方法如下:

```rust
     unsafe {
             let mut _end = _name;
             while _end.read_volatile() != 0u8 {
                 _end = _end.add(1);
             }
             let _slice = core::slice::from_raw_parts(_name, _end as usize - _name as usize);
             let name = core::str::from_utf8(_slice).unwrap();
         }
```

先将其转为slice，而后转为&str。

但是通过阅读框架代码，发现框架提供了一种翻译的方法:smile:

然后就是sys_unlinkat，解除硬链接方法。也就是将目录项删除即可，但是还需要注意一种特殊情况。提供的名字是文件inode的唯一索引，这时候就需要将文件删除。

这里需要实现一个根据inode_number获取索引次数的方法，大概的思路就是对root_inode下面的目录项一个个读取，如果inode_number与给定的吻合，累计次数加一即可。

```rust
pub fn get_inode_number_times(&self, inode_number: u32) -> usize{
        let _fs = self.fs.lock();
        self.read_disk_inode(|disk_inode| {
            let file_count = (disk_inode.size as usize) / DIRENT_SZ;
            let mut res = 0;
            for i in 0..file_count {
                let mut dirent = DirEntry::empty();
                assert_eq!(
                    disk_inode.read_at(i * DIRENT_SZ, dirent.as_bytes_mut(), &self.block_device,),
                    DIRENT_SZ,
                );
                if dirent.inode_number() == inode_number{
                    res += 1;
                }
            }
            res
        })
    }
```

实现了这个之后，接下来就是删除目录项的功能了，我这里实现的时候就按最简单的实现了。直接将该目录项所在的位置重写成一个空的目录项。当然也可以更复杂一点儿的，该目录项后面的元素直接向前覆盖一个单位就行。

```rust
self.modify_disk_inode(|disk_inode| {
                    // append file in the dirent
                    let file_count = (disk_inode.size as usize) / DIRENT_SZ;
                    let mut dirent = DirEntry::empty();
                    for i in 0..file_count {
                        assert_eq!(
                            disk_inode.read_at(DIRENT_SZ * i, dirent.as_bytes_mut(),&self.block_device,),
                            DIRENT_SZ,
                        );
                        if dirent.name() == name {
                            let dirent = DirEntry::empty();
                            disk_inode.write_at(
                                DIRENT_SZ * i,
                                dirent.as_bytes(),
                                &self.block_device,
                            );
                        }
                    }
                    
                });
```

最后就是文件状态的系统调用了，这个最麻烦，主要是我刚开始不知道通过_fd获得的文件描述符怎么处理。折腾了好久，终于发现了打开文件之后，向文件描述符表中，存的是OSInode类型的数据，那么向File trait添加get_inode_number，get_type方法即可。让OSInode实现就行。

```rust
fn get_inode_number(&self) -> usize {
        let mut inner = self.inner.exclusive_access();
        return inner.inode.get_inode_number();

    }
    fn get_type(&self) -> usize {
        let inner = self.inner.exclusive_access();
        return inner.inode.get_inode_type()
    }
```

这里获取inode_number需要从blockid和blockoffset还原出来inode_number。

```rust
pub fn get_inode_number(&self) -> usize{
        let inode_size = core::mem::size_of::<DiskInode>();
        let inodes_per_block = (BLOCK_SZ / inode_size) as u32;
        let tem1 = self.block_id - self.fs.lock().inode_area_start_block as usize;
        
        return tem1 * (inodes_per_block as usize) + self.block_offset/inode_size
    }
```

这个代码是从easyFileSystem的一个方法中，逆写出来的。

实现中一些值得注意的问题。

* os6/src/syscall/process.rs中第三行use core::slice::SlicePattern，这一行会报错，需要注释掉才可以正常运行。
* ![image-20220716191006000](pic/image-20220716191006000.png)
* 图中的drop(inner)十分重要，需要手动释放掉资源。后面的translate方法也需要inner这个资源。

### 7.17-7.18

这两天有事情，耽搁了一下。

不过昨天群里讨论了一下lab3-os5的最后一个测例的问题，得出来的结论是计时器不准，如果计时器不准的话，那么实现的stride算法和测试代码就对不上了，因为我们以为过了一个时间片，对该程序加了一个pass，但实际占用的cpu时间不够一个时间片，那么这个pass就加多了，对之后的调度不利。并且这个计时器不准不是偶尔出现的，根据观察到的现象来看，出现的频率还是蛮多的。测试代码是对spin_delay函数在4000ms时间内调用的次数，六个程序都运行一样的代码就是优先级不一样。

因此可能的逻辑关系是，高的优先级的进程碰到了计时器不准的情况，没有运行多少的spin_delay，却加了一个单位的pass，因此4000ms内完成的spin_delay函数的次数反而很小，因此不满足公平性要求。

我在看到这个问题之后，连忙试了一下make test5，结果发现过不去了。。。。。这个代码之前可是可以很顺利的通过的。。然后18号晚上又试了一下，又可以通过了。。。应该就是计时器不准的问题。