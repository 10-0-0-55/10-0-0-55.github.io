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
	
	...
	
	for(;;)
	{
		int iters = 0;
		while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks(av))     // unsorted_chunk (av) 为 main arena + 88 即 main arena 中存储 unsorted bin fd bk 的地方
		{
			bck = victim->bk;               // victim 为将被从 unsorted bin 链上取下来的那一块
			
			...
			
			unsorted_chunks (av)->bk = bck; 
			bck->fd = unsorted_chunks (av);
			
			...
		}
	}
	
}
```

在笔者看来 victim 的 bk 可以称得上是 unsorted bin malloc 的发起点，相反 victim 的 fd 甚至基本毫无提及。在检测很少的 unsorted bin 的 malloc 的，如果能修改 free 掉的 unsorted bin 的 bk，那么就能在 fake_bk+0x10 的地址上写上 (main arena+88) 的值，比如将 global_max_fast 的值改大，则之后的 chunk 的 malloc 和 free 皆变成了 fastbin 的 malloc 和 free，但是 unsorted bin 的机制基本可以说是废掉了，再使用便会 crash。<br/>

主流的两种 unsorted bin attack ，一种是更改 global_max_fast 另一种是 house of orange 中更改 _IO_list_all (这个我还没搞懂，留坑=.=) <br/>

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
