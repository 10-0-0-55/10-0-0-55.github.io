# stack hijack 

## hijack ret

最常见的就是，栈溢出 hijack ret 

## hijack ebp

有的时候，无法栈溢出到 ret ，无法直接 hijack 上 ret addr ，但是可以 hijack ebp ，由于 x86 架构的原因，在函数返回的时候，会通过机制，将 ebp+0x4(或 rbp+0x8 ) 认为成 ret addr ，因此就可以通过溢出到 ebp ，来控制流程

例题 pwnable.kr simple login

## stack hijack

栈上的 hijack 的目的就是能够 hijack eip ，而控制 eip 可以通过覆盖返回地址，也可以通过覆盖 ebp ，控制 ebp 来控制 eip

