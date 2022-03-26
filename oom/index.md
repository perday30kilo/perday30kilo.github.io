# Oom


# oomkiller 简介

## 按需分配物理页面
　　很多情况下，一个进程会申请一块很大的内存，但只是用到其中的一小部分。为了避免内存的浪费，在分配页面时，Linux 采用的是按需分配物理页面的方式。譬如说，某个进程调用malloc()申请了一块小内存，这时内核会分配一个虚拟页面，但这个页面不会映射到实际的物理页面。
 ![alt 内存分配机制](oom1.png)
　　从图中可以看到，当程序首次访问这个虚拟页面时，会触发一个缺页异常 (page fault)。这时内核会分配一个物理页面，让虚拟页面映射到这个物理页面，同时更新进程的页表 (page table)。

## Memory Overcommit
Memory Overcommit
　　这种按需分配物理页面的方式，可以大大节省物理内存的使用，但有时会导致 Memory Overcommit。所谓 Memory Overcommit，也就是说，所有进程使用的虚拟内存超过了系统的物理内存和交换空间的总和。默认情况下，Linux 是允许 Memory Overcommit 的。并且在大多数情况下，Memory Overcommit 也是安全的，因为很多进程只是申请了很多内存，但实际使用到的内存并不多。
　　但万一很多进程都使用了申请来的大部分内存，就可能导致物理内存和交换空间不够用了，这时内核的 OOM Killer 就会出马，它会选择杀掉一个或多个进程，这样就能腾出一些内存给其它进程使用。
　　在 Linux 中，可以通过内核参数vm.overcommit_memory去控制是否允许 overcommit：

默认值是 0，在这种情况下，只允许轻微的 overcommit，而比较明显的 overcommit 将不被允许。
如果设置为 1，表示总是允许 overcommit。
如果设置为 2，则表示总是禁止 overcommit。也就是说，如果某个申请内存的操作将导致 overcommit，那么这个操作将不会得逞。
　　那么对内核来说，怎样才算 overcommit 呢？Linux 设定了一个阈值，叫做 CommitLimit，如果所有进程申请的总内存超过了 CommitLimit，那就算是 overcommit 了。在/proc/meminfo中可以看到 CommitLimit 的大小：
```
1
2
$ cat /proc/meminfo | grep CommitLimit
CommitLimit:     3829768 kB
```
　CommitLimit 的值是这样计算的：
```
CommitLimit = [swap size] + [RAM size] * vm.overcommit_ratio / 100

```
　其中的vm.overcommit_ratio也是内核参数，它的默认值是 50。

## OOM Killer
　当物理内存和交换空间不够用时，OOM Killer 就会选择杀死进程，那么它是怎样知道要先杀死哪个进程呢？其实 Linux 的每个进程都有一个 oom_score (位于/proc/<pid>/oom_score)，这个值越大，就越有可能被 OOM Killer 选中。oom_score 的值是由很多因素共同决定的，这里列举几个因素：

如果进程消耗的内存越大，它的 oom_score 通常也会越大。
如果进程运行了很长时间，并且消耗很多 CPU 时间，那么通常它的 oom_score 会偏小。
如果进程以 superuser 的身份运行，那么它的 oom_score 也会偏小。
　　如何才能尽量防止某个重要的进程被杀死呢？Linux 每个进程都有一个 oom_adj (位于/proc/<pid>/oom_adj)，这个值的范围是 [-17, +15]，进程的 oom_adj 会影响 oom_score 的计算，也就是说，我们可以通过调小进程的 oom_adj 从而降低进程的 oom_score。对于一些比较重要的进程，例如 MySQL，我们想尽量避免它被 OOM Killer 杀死，这时候就可以调低它的 oom_adj 的值，例如：
```
$ sudo echo -10 > /proc/$(pidof mysqld)/oom_adj

```
交换空间
　　通常来说操作系统都会开启交换空间，那么交换空间有什么作用呢？

允许系统将一些长期没有用到的物理页面换出到交换空间，这样就能节省物理内存的使用。
当物理内存不够使用时，系统可以利用交换空间作为缓冲，防止一些进程因为内存不够而被 OOM Killer 杀死。
　　vm.swppiness可以用来配置交换空间，取值范围是 [0, 100]，在 Linux 3.5 之后，它有这些作用：

设置为 0 表示禁止交换空间的使用，只有当系统 OOM 时才允许使用交换空间。
设置为 1 不会禁止交换空间的使用，但系统会尽量不去使用交换空间。
设置为 100 表示系统会很喜欢使用交换空间。
　　交换空间是位于磁盘之上的，对操作系统来说，访问磁盘的速度远远慢于访问物理内存。所以我们希望，当物理内存足够使用时，系统能尽量不去使用交换空间，这样能降低页面换入换出的频率，因为频繁的页面换入换出操作会严重影响系统的性能。为了达到这种效果，我们可以把vm.swappiness设置为 1：


```
	
sudo echo 1 >  /proc/sys/vm/swappiness

```

## 排查命令
dmesg
如果发现自己的java进程悄无声息的消失了，几乎没有留下任何线索，那么dmesg一发，很有可能有你想要的。
```
sudo dmesg|grep -i kill|less
```

## 

Linux 内核根据应用程序的要求分配内存，通常来说应用程序分配了内存但是并没有实际全部使用，为了提高性能，这部分没用的内存可以留作它用，这部分内存是属于每个进程的，内核直接回收利用的话比较麻烦，所以内核采用一种过度分配内存（over-commit memory）的办法来间接利用这部分 “空闲” 的内存，提高整体内存的使用效率。一般来说这样做没有问题，但当大多数应用程序都消耗完自己的内存的时候麻烦就来了，因为这些应用程序的内存需求加起来超出了物理内存（包括 swap）的容量，内核（OOM killer）必须杀掉一些进程才能腾出空间保障系统正常运行。用银行的例子来讲可能更容易懂一些，部分人取钱的时候银行不怕，银行有足够的存款应付，当全国人民（或者绝大多数）都取钱而且每个人都想把自己钱取完的时候银行的麻烦就来了，银行实际上是没有这么多钱给大家取的。

内核检测到系统内存不足、挑选并杀掉某个进程的过程可以参考内核源代码 linux/mm/oom_kill.c ，当系统内存不足的时候，out_of_memory() 被触发，然后调用 select_bad_process() 选择一个 “bad” 进程杀掉，如何判断和选择一个 “bad” 进程呢，总不能随机选吧？挑选的过程由 oom_badness() 决定，挑选的算法和想法都很简单很朴实：最 bad 的那个进程就是那个最占用内存的进程。
```
/**
 * oom_badness - heuristic function to determine which candidate task to kill
 * @p: task struct of which task we should calculate
 * @totalpages: total present RAM allowed for page allocation
 *
 * The heuristic for determining which task to kill is made to be as simple and
 * predictable as possible.  The goal is to return the highest value for the
 * task consuming the most memory to avoid subsequent oom failures.
 */
unsigned long oom_badness(struct task_struct *p, struct mem_cgroup *memcg,
			  const nodemask_t *nodemask, unsigned long totalpages)
{
	long points;
	long adj;

	if (oom_unkillable_task(p, memcg, nodemask))
		return 0;

	p = find_lock_task_mm(p);
	if (!p)
		return 0;

	adj = (long)p-&gt;signal-&gt;oom_score_adj;
	if (adj == OOM_SCORE_ADJ_MIN) {
		task_unlock(p);
		return 0;
	}

	/*
	 * The baseline for the badness score is the proportion of RAM that each
	 * task's rss, pagetable and swap space use.
	 */
	points = get_mm_rss(p-&gt;mm) + p-&gt;mm-&gt;nr_ptes +
		 get_mm_counter(p-&gt;mm, MM_SWAPENTS);
	task_unlock(p);

	/*
	 * Root processes get 3% bonus, just like the __vm_enough_memory()
	 * implementation used by LSMs.
	 */
	if (has_capability_noaudit(p, CAP_SYS_ADMIN))
		adj -= 30;

	/* Normalize to oom_score_adj units */
	adj *= totalpages / 1000;
	points += adj;

	/*
	 * Never return 0 for an eligible task regardless of the root bonus and
	 * oom_score_adj (oom_score_adj can't be OOM_SCORE_ADJ_MIN here).
	 */
	return points &gt; 0 ? points : 1;
}
```

从上面的 oom_kill.c 代码里可以看到 oom_badness() 给每个进程打分，根据 points 的高低来决定杀哪个进程，这个 points 可以根据 adj 调节，root 权限的进程通常被认为很重要，不应该被轻易杀掉，所以打分的时候可以得到 3% 的优惠（adj -= 30; 分数越低越不容易被杀掉）。我们可以在用户空间通过操作每个进程的 oom_adj 内核参数来决定哪些进程不这么容易被 OOM killer 选中杀掉。比如，如果不想 MySQL 进程被轻易杀掉的话可以找到 MySQL 运行的进程号后，调整 oom_score_adj 为 -15（注意 points 越小越不容易被杀）：

```
# ps aux | grep mysqld
mysql    2196  1.6  2.1 623800 44876 ?        Ssl  09:42   0:00 /usr/sbin/mysqld

# cat /proc/2196/oom_score_adj
0
# echo -15 &gt; /proc/2196/oom_score_adj
```

 ## 找出最有可能被 OOM Killer 杀掉的进程
```
# vi oomscore.sh
#!/bin/bash
for proc in $(find /proc -maxdepth 1 -regex '/proc/[0-9]+'); do
    printf "%2d %5d %s\n" \
        "$(cat $proc/oom_score)" \
        "$(basename $proc)" \
        "$(cat $proc/cmdline | tr '\0' ' ' | head -c 50)"
done 2&gt;/dev/null | sort -nr | head -n 10

# chmod +x oomscore.sh
# ./oomscore.sh
18   981 /usr/sbin/mysqld
 4 31359 -bash
 4 31056 -bash
 1 31358 sshd: root@pts/6
 1 31244 sshd: vpsee [priv]
 1 31159 -bash
 1 31158 sudo -i
 1 31055 sshd: root@pts/3
 1 30912 sshd: vpsee [priv]
 1 29547 /usr/sbin/sshd -D
```

# linux里哪些命令会出发oom killer

