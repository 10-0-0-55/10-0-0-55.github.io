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
			if (__glibc_unlikely (bck->fd != victim)
				|| __glibc_unlikely (victim->fd != unsorted_chunks (av)))
			  malloc_printerr ("mallloc(): unsorted double linked list corrupted");
			if (__libc_unlikely (prev_inuse (next)))
			  malloc_printerr ("malloc(): invalid next->prev_inuse (unsorted)");
			
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

在笔者看来，unsorted bin 中取下 victim 那一块时 ，victim 的 bk 可以称得上是 unsorted bin malloc 的发起点，相反 victim 的 fd 甚至基本毫无提及。在检测很少的 unsorted bin 的 malloc 的(基本上检测就只有 `check(victim->bk -> fd == victim)` 但是不知道为什么不相等，这条检测也过了)，如果能修改 free 掉的 unsorted bin 的 bk，将 unsorted bin 的 bk 改成 目标地址-0x10，那么就能在 fake_bk+0x10 的地址上写上 (main arena+88) 的值，比如将 global_max_fast 的值改大，则之后的 chunk 的 malloc 和 free 皆变成了 fastbin 的 malloc 和 free，但是 unsorted bin 的机制基本可以说是废掉了，再使用便会 crash。<br/>

主流的两种 unsorted bin attack ，一种是更改 global_max_fast 另一种是 house of orange 中更改 _IO_list_all (这个我还没搞懂，留坑=.=) <br/>

最近一直在看 NCTF2018 的 houseofhomura 给出的漏洞是可以修改 unsorted bin 的 fd bk ，exp 是利用 unsorted bin 的机制，成功将破坏的 unsorted bin 的机制恢复，并且使得 bin 进 small bin（看的我一愣一愣的，例子的坑又留下了 =.=）<br/>

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
