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

#### example 01 linux ELF

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

在 IDA 文件里面用 powershell 打开，运行 `.\ida.exe -c -S"test.py" input_file` 即可， -c -S 是两个参数，

然后我神经刀，跑到 linux 里面来试行不行，我的 IDA 放在 ~/ 目录下，然后运行脚本的时候开始报错，后来觉得是因为用的 wine 调用的 ida.exe 所以在 -S 里面还是要用 windows 下反斜杠而不是斜杠 
```
# vangelis @ vangelis-PC in ~/IDA [22:47:42] C:2
$ ./ida.exe -c -S".\Vangelis\test.py" ~/Desktop/Analysis_File/rop/rop
```
同时在脚本里面也是要用映射过去的地址，所以用的 `Z:\\home\\vangelis\\Desktop\\rop_fun_output.txt` 

后来我又神经，在脚本里面写注释，然后被 wine 里面调 IDA 再调 python 说脚本里面的字符有问题，应该是编码格式问题，就把注释全删了

后来再改进一下，变成了 `nohup ./ida.exe -c -S".\Vangelis\test.py" ~/Desktop/Analysis_File/rop/rop > outfile 2>&1 &` 把标准输出和错误全部输出到 outfile 文件中

再后来觉得每次开 IDA 十分不舒服就查了一下 IDA - 的参数，最后用的 -B -S 

#### example 02 Windows PE

