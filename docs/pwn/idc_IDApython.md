# IDA 强大的插件 idc 和 IDApython 

idc 和 IDApython 帮助分析者利用与 IDA 进行交互，得到自动化分析程序的结果

## IDApython
IDApython 的安装可以很容易在网上查到，但是对于 IDApython 教程的资料除了看到的一个官方文档以外没看到特别多特别好的资料，于是想稍微记录一下自己搞 IDApython 遇到的一些坑和 IDApython 常用的函数 (希望学弟和学妹明年做的时候不会没资料 =.=

###　安装 IDApython
首先确认自己的 IDApython 是否装好了，有两种办法。

一种从 IDA 上面的 File->Script Command 打开之后看 Scripting language 是否是存在 IDC 和 Python， 然后可以测试一下库是否全，基本上可以利用一个小脚本检测 import 库是否会报错

```
import idaapi
import idc
import idautils

if __name__ == "__main__":
	fp = open("Test.txt", "w")
	fp.write("Success\n")
	fp.close()
```

当然运行脚本的时候需要用 IDA 打开一个随便的文件，然后运行完脚本可以在那个文件的那一层里面多出来一个 Test.txt 文件，里面写着 `Success` 就说明没问题，否则库就是没装全

另外一种办法就是在 IDA 最下面有一个 Output window 下面有一个可选的，查看是否存在 IDC 和 Python，在里面选 Python 然后尝试 import 三个库就可以了，没报错就没问题。`import` 一个不存在的库试一下 `import requests` 就会发现报错

(如果没安装好，可以直接查询吾爱破解 IDA 7.0 可以看到一个已经绿色版+配好 Python 的 IDA

### 一些脚本
下面是我看到的一些脚本，我拿到本地测试过能跑的

#### example 01 
打印全部函数的名字

```
from idaapi import *
from idautils import *
import idc

idc.Wait()              # 单线程的感觉，分析完一个之后才会运行后续脚本
ea = BeginEA()          # 找到程序 _start (针对 ELF 而言)　返回是 <type 'long'>

fp = open("Z:\\home\\vangelis\\Desktop\\rop_fun_output.txt", "w")
fp.write("check\n")

idx = 0

# 在.text段中 SegStart() 找到程序地址最小的汇编指令的地址，SegEnd() 找到程序地址最大的汇编指令的地址

for funcea in Functions(SegStart(ea), SegEnd(ea)):  # Functions() 返回为 <type 'generator'>
    functionName = GetFunctionName(funcea)          # funcea 类型为　<type 'long'>  为地址 GetFunctionName() 将地址转换为函数名
    fp.write("%d " % idx)
    fp.write(functionName + "\n")
    idx += 1

fp.close()
idc.Exit(0)             # 在脚本运行结束之后能够自动关闭 IDA
```

##### idc Wait()
```
Wait():
$ grep -H "Wait" ./*.py
./idc_bc695.py:def Wait(): return auto_wait()

idc.py 
 412   │ def auto_wait():
 413   │     """
 414   │     Process all entries in the autoanalysis queue
 415   │     Wait for the end of autoanalysis
 416   │ 
 417   │     @note:    This function will suspend execution of the calling script
 418   │             till the autoanalysis queue is empty.
 419   │     """
 420   │     return ida_auto.auto_wait()
```

在 IDA 文件里面用 powershell 打开，运行 `.\ida.exe -c -S"test.py" input_file` 即可， -c -S 是两个参数，

然后我神经刀，跑到 linux 里面来试行不行，我的 IDA 放在 ~/ 目录下，然后运行脚本的时候开始报错，后来觉得是因为用的 wine 调用的 ida.exe 所以在 -S 里面还是要用 windows 下反斜杠而不是斜杠 
```
# vangelis @ vangelis-PC in ~/IDA [22:47:42] C:2
$ ./ida.exe -c -S".\Vangelis\test.py" ~/Desktop/Analysis_File/rop/rop
```
同时在脚本里面也是要用映射过去的地址，所以用的 `Z:\\home\\vangelis\\Desktop\\rop_fun_output.txt` 

后来我又神经，在脚本里面写注释，然后被 wine 里面调 IDA 再调 python 说脚本里面的字符有问题，应该是编码格式问题，就把注释全删了

后来再改进一下，变成了 `nohup ./ida.exe -c -S".\Vangelis\test.py" ~/Desktop/Analysis_File/rop/rop > outfile 2>&1 &` 把标准输出和错误全部输出到 outfile 文件中

再后来觉得每次开 IDA 十分不舒服就查了一下运行 IDA 的参数，最后用的 `-B -S`
` -B     batch mode. IDA will generate .IDB and .ASM files automatically` 
因为最后会产生相应的 .idb 或者 .asm 所以最后处理一下，不处理其实也没什么

#### example 02 
寻找危险函数的被引用的地址

```
from idaapi import *
from idautils import *
import idc

idc.Wait()
danger_funcs = ["strcpy", "read", "gets"]

fp = open("Z:\\home\\vangelis\\Desktop\\Analysis_File\\rop\\rop_danger_call.txt", "w")
fp.write("check\n")

for func in danger_funcs :
    fp.write("#############\n")
    fp.write(func + "\n")
    addr = LocByName(func)
    if addr != BADADDR:
        cross_regs = CodeRefsTo(addr, 0)
        for ref in cross_regs:
            fp.write(str(hex(ref)) + "\n")

fp.close()
idc.Exit(0)

```

##### idc LocByName
经过尝试 LocByName 函数存在则返回函数地址，否则返回 0xffffffff (32位) 而 BADADDR 就是 0xffffffff (32位)
```
Python>hex(BADADDR)
0xffffffffL
Python>hex(LocByName("ABCD"))
0xffffffffL

$ grep -H "LocByName" ./*.py
./idc_bc695.py:def LocByName(name): return get_name_ea_simple(name)

$ grep -H "get_name_ea_simple" ./*.py
./idc.py:def get_name_ea_simple(name):

1879   │ def get_name_ea_simple(name):
1880   │     """
1881   │     Get linear address of a name
1882   │ 
1883   │     @param name: name of program byte
1884   │ 
1885   │     @return: address of the name
1886   │              BADADDR - No such name
1887   │     """
1888   │     return ida_name.get_name_ea(BADADDR, name)

  72   │ __EA64__ = ida_idaapi.BADADDR == 0xFFFFFFFFFFFFFFFFL
  73   │ WORDMASK = 0xFFFFFFFFFFFFFFFF if __EA64__ else 0xFFFFFFFF

```

##### idatils CodeRefsTo
找到 to 该地址的一个列表
```
Python>CodeRefsTo(0x806d290L, 0)
<generator object refs at 0x000000000C26B2D0>

  68   │ def CodeRefsFrom(ea, flow):
  69   │     """
  70   │     Get a list of code references from 'ea'
  71   │
  72   │     @param ea:   Target address
  73   │     @param flow: Follow normal code flow or not
  74   │     @type  flow: Boolean (0/1, False/True)
  75   │
  76   │     @return: list of references (may be empty list)
  77   │
  78   │     Example::
  79   │
  80   │         for ref in CodeRefsFrom(get_screen_ea(), 1):
  81   │             print ref
  82   │     """
  83   │     if flow == 1:
  84   │         return refs(ea, ida_xref.get_first_cref_from, ida_xref.get_next_cref_from)
  85   │     else:
  86   │         return refs(ea, ida_xref.get_first_fcref_from, ida_xref.get_next_fcref_from)
```

#### example 03
打印函数栈中的局部变量

```
from idaapi import *
from idautils import *
import idc
import types

if __name__ == "__main__":
    idc.Wait()
    fp = open("Z:\\home\\vangelis\\Desktop\\Analysis_File\\rop\\stack-member.txt", "w")
    fp.write("check\n")

    ea = BeginEA()
    for func in Functions(SegStart(ea), SegEnd(ea)):
        funcName = GetFunctionName(func)
        #  if(funcName != "overflow"):
            #  continue
        fp.write("###############\n")
        fp.write("Function Name is %s\n" % funcName)

        stack_frame = GetFrame(func)
        frame_size = GetStrucSize(stack_frame)

        fp.write("frame_size: %d\n" % frame_size)

        frame_counter = 0

        while frame_counter < frame_size:
            stack_var = GetMemberName(stack_frame, frame_counter)
            if type(stack_var) == type('a'):
                fp.write("[*] Function:%s -> Stack Variable:%s(%d bytes)\n" % (funcName, stack_var, GetMemberSize(stack_frame, frame_counter)))
                try :
                    frame_counter = frame_counter + GetMemberSize(stack_frame, frame_counter)
                except:
                    frame_counter += 1
            else :
                frame_counter += 1

    fp.close()
    idc.Exit(0)
```

注： stack frame 是从某一函数的 参数到栈顶 之间全部内容

##### idc GetMemberName
通过 stack id 和 offset 得到变量的名字
```
Python>stack = GetFrame(0x804887c) # "overflow" function's address
Python>hex(stack)
0xff00006cL
Python>GetMemberName(stack, 0xc)
var_C

$ grep -H "GetMemberName" ./*.py
./idc_bc695.py:def GetMemberName(id, member_offset): return get_member_name(id, member_offset)

5092   │ def get_member_name(sid, member_offset):
5093   │     """
5094   │     Get name of a member of a structure
5095   │
5096   │     @param sid: structure type ID
5097   │     @param member_offset: member offset. The offset can be
5098   │                           any offset in the member. For example,
5099   │                           is a member is 4 bytes long and starts
5100   │                           at offset 2, then 2,3,4,5 denote
5101   │                           the same structure member.
5102   │
5103   │     @return: None if bad structure type ID is passed
5104   │              or no such member in the structure
5105   │              otherwise returns name of the specified member.
5106   │     """
5107   │     s = ida_struct.get_struc(sid)
5108   │     if not s:
5109   │         return None
5110   │
5111   │     m = ida_struct.get_member(s, member_offset)
5112   │     if not m:
5113   │         return None
5114   │
5115   │     return ida_struct.get_member_name(m.id)
```

##### idc GetMemberSize
得到一个变量的结构所占字节数
```
5146   │ def get_member_size(sid, member_offset):
5147   │     """
5148   │     Get size of a member
5149   │
5150   │     @param sid: structure type ID
5151   │     @param member_offset: member offset. The offset can be
5152   │                           any offset in the member. For example,
5153   │                           is a member is 4 bytes long and starts
5154   │                           at offset 2, then 2,3,4,5 denote
5155   │                           the same structure member.
5156   │
5157   │     @return: None if bad structure type ID is passed,
5158   │              or no such member in the structure
5159   │              otherwise returns size of the specified
5160   │              member in bytes.
5161   │     """
5162   │     s = ida_struct.get_struc(sid)
5163   │     if not s:
5164   │         return None
5165   │
5166   │     m = ida_struct.get_member(s, member_offset)
5167   │     if not m:
5168   │         return None
5169   │
5170   │     return ida_struct.get_member_size(m)


5173   │ def get_member_flag(sid, member_offset):
5174   │     """
5175   │     Get type of a member
```



## 参考
> 脚本 | https://blog.csdn.net/ojshilu/article/details/12905405

> 博客 | https://blog.csdn.net/qq1084283172/article/details/64130118 

> part1-6 | https://unit42.paloaltonetworks.com/using-idapython-to-make-your-life-easier-part-1/

> 官方文档| https://www.hex-rays.com/products/ida/support/idapython_docs/


