# syscall

## syscall

### 系统调用号对应的表

[syscall](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)

### 系统调用时更改的寄存器

```
IF (CS.L ≠ 1 ) or (IA32_EFER.LMA ≠ 1) or (IA32_EFER.SCE ≠ 1)
(* Not in 64-Bit Mode or SYSCALL/SYSRET not enabled in IA32_EFER *)
    THEN #UD;
FI;
RCX ← RIP;
                            (* Will contain address of next instruction *)
RIP ← IA32_LSTAR;
R11 ← RFLAGS;
RFLAGS ← RFLAGS AND NOT(IA32_FMASK);
CS.Selector ← IA32_STAR[47:32] AND FFFCH (* Operating system provides CS; RPL forced to 0 *)
(* Set rest of CS to a fixed value *)
CS.Base ← 0;
                            (* Flat segment *)
CS.Limit ← FFFFFH;
                            (* With 4-KByte granularity, implies a 4-GByte limit *)
CS.Type ← 11;
                            (* Execute/read code, accessed *)
CS.S ← 1;
CS.DPL ← 0;
CS.P ← 1;
CS.L ← 1;
                            (* Entry is to 64-bit mode *)
CS.D ← 0;
                            (* Required if CS.L = 1 *)
CS.G ← 1;
                            (* 4-KByte granularity *)
CPL ← 0;
SS.Selector ← IA32_STAR[47:32] + 8;
                            (* SS just above CS *)
(* Set rest of SS to a fixed value *)
SS.Base ← 0;
                            (* Flat segment *)
SS.Limit ← FFFFFH;
                            (* With 4-KByte granularity, implies a 4-GByte limit *)
SS.Type ← 3;
                            (* Read/write data, accessed *)
SS.S ← 1;
SS.DPL ← 0;
SS.P ← 1;
SS.B ← 1;
                            (* 32-bit stack segment *)
SS.G ← 1;
                            (* 4-KByte granularity *)
```

值得注意的是 syscall 会用 rcx 记录下一条指令的值，如果程序允许你的输入在一个 rwxp 段，但是每次输入十分短的时候，可以通过 syscall 找到当前写入的地址进行续写


```
start :
	xor rax, rax 
	syscall 
	dec edx
	mov rsi, rcx
	jmp start
```
最开始的时候由于输入限制只能 10 byte，因此控制不了 rsi，可以通过 syscall 找到之前写入的地址，运用 edx 只取 rdx 的低 32 位，jmp (-127 ~ +128) 的值进行调用 SYS_read

[Syscall Operation](https://github.com/HJLebbink/asm-dude/wiki/SYSCALL)
