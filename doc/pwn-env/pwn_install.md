

# Pwn环境搭建



## OWASP ZSC Shellcoder

安装
方法一
进入github下载页:https://github.com/Ali-Razmjoo/OWASP-ZSC 进行下载

运行installer.py，之后你就可以用”zsc”命令启动这个工具啦，当然你也可以在不安装的情况下直接直接执行zsc.py

方法二
wget https://github.com/Ali-Razmjoo/OWASP-ZSC/archive/master.zip -O owasp-zsc.zip && unzip owasp-zsc.zip && rm -rf owasp-zsc.zip && mv OWASP-ZSC-master owasp-zsc 
cd owasp-zsc && python installer.py

run
Logan$ zsc

---------------------------------------------->
Install capstone
---
cd ~
git clone https://github.com/aquynh/capstone
cd capstone
make
sudo make install
---------------------------------------------->

Install pwntools
sudo pacman -S python python-pip
pip install -U setuptools
pip install -U setuptools

cd ~
git clone https://github.com/Gallopsled/pwntools
cd pwntools
sudo python setup.py install
---------------------------------------------->

Install pwndbg

git clone https://github.com/pwndbg/pwndbg
cd pwndbg
sudo #./setup.sh

错误及解决
遇到了Python报错，没有找到setuptools，因为pwndbg使用Python3，所以自己手动安装一下
sudo python3 -m pip install setuptools
然后重新执行 ./setup.sh

如果没有装过其他插件的话应该就直接可以用了，shell中输入gdb能够看到pwndbg>，如果装过其他的插件，要修改一下配置文件，默认在home中:
sudo vim ./.gdbinit
看一下有没有这一行
source /home/logan/pwndbg/gdbinit.py
没有的话加上，把其他的注释掉，保存启动GDB，完活

---------------------------------------------->
Test pwntools
---	
python
import pwn----------(from pwn import *)
pwn.asm("xor eax,eax")

出现'1\xc0' 说明安装成功




---------------------------------------------->
Install LibcSearcher
git clone https://github.com/lieanu/LibcSearcher.git
cd LibcSearcher
python setup.py develop

---------------------------------------------->
Install one_gadget tools

sudo apt install ruby
gem install one_gadget

---------------------------------------------->
Install seccomp-tools

gem install seccomp-tools
---------------------------------------------->
pwntools 使用
	常用模块:
    asm : 汇编与反汇编，支持x86/x64/arm/mips/powerpc等基本上所有的主流平台
    dynelf : 用于远程符号泄漏，需要提供leak方法
    elf : 对elf文件进行操作
    gdb : 配合gdb进行调试
    memleak : 用于内存泄漏
    shellcraft : shellcode的生成器
    tubes : 包括tubes.sock, tubes.process, tubes.ssh, tubes.serialtube，分别适用于不同场景的PIPE
    utils : 一些实用的小功能，例如CRC计算，cyclic pattern等

1.连接

本地 ：sh = process("./level0")
远程：sh = remote("127.0.0.1",10001)
关闭连接：sh.close()  


2.IO模块

sh.send(data)  发送数据
sh.sendline(data)  发送一行数据，相当于在数据后面加\n
sh.recv(numb = 2048, timeout = dufault)  接受数据，numb指定接收的字节，timeout指定超时
sh.recvline(keepends=True)  接受一行数据，keepends为是否保留行尾的\n
sh.recvuntil("Hello,World\n",drop=fasle)  接受数据直到我们设置的标志出现
sh.recvall()  一直接收直到EOF
sh.recvrepeat(timeout = default)  持续接受直到EOF或timeout
sh.interactive()  直接进行交互，相当于回到shell的模式，在取得shell之后使用

3.汇编：

>>> asm('nop')
'\x90'
>>> asm('nop', arch='arm')
'\x00\xf0 \xe3'
>>> asm('push rax', arch = 'amd64', os = 'linux')
>>> asm('push rax', arch = 'i386', os = 'linux')

可以使用context来指定cpu类型以及操作系统

>>> context.arch      = 'i386'
>>> context.os        = 'linux'
>>> context.endian    = 'little'
>>> context.word_size = 32

使用disasm进行反汇编
>>> print disasm('6a0258cd80ebf9'.decode('hex'))
   0:   6a 02                   push   0x2
   2:   58                      pop    eax
   3:   cd 80                   int    0x80
   5:   eb f9                   jmp    0x0

注意，asm需要binutils中的as工具辅助，如果是不同于本机平台的其他平台的汇编，
例如在我的x86机器上进行mips的汇编就会出现as工具未找到的情况，这时候需要安装其他平台的cross-binutils。


4.Shellcode生成器

>>> print shellcraft.i386.nop().strip('\n')
    nop
>>> print shellcraft.i386.linux.sh()
    /* push '/bin///sh\x00' */
    push 0x68
    push 0x732f2f2f
    push 0x6e69622f
...
结合asm可以可以得到最终的pyaload

from pwn import *
context(os='linux',arch='amd64')
shellcode = asm(shellcraft.sh())

或者
from pwn import *
shellcode = asm(shellcraft.amd64.linux.sh())


除了直接执行sh之外，还可以进行其它的一些常用操作例如提权、反向连接等等
5.ELF文件操作

>>> e = ELF('/bin/cat')
>>> print hex(e.address)  # 文件装载的基地址
0x400000
>>> print hex(e.symbols['write']) # 函数地址
0x401680
>>> print hex(e.got['write']) # GOT表的地址
0x60b070
>>> print hex(e.plt['write']) # PLT的地址
0x401680
>>> print hex(e.search('/bin/sh').next())# 字符串/bin/sh的地址


6整数pack与数据unpack

pack：p32，p64
unpack：u32，u64

from pwn import *
elf = ELF('./level0')
sys_addr = elf.symbols['system']
payload = 'a' * (0x80 + 0x8) + p64(sys_addr)
...

7.ROP链生成器

elf = ELF('ropasaurusrex')
rop = ROP(elf)
rop.read(0, elf.bss(0x80))
rop.dump()
# ['0x0000:        0x80482fc (read)',
#  '0x0004:       0xdeadbeef',
#  '0x0008:              0x0',
#  '0x000c:        0x80496a8']
str(rop)
# '\xfc\x82\x04\x08\xef\xbe\xad\xde\x00\x00\x00\x00\xa8\x96\x04\x08'

使用ROP(elf)来产生一个rop的对象，这时rop链还是空的，需要在其中添加函数

因为ROP对象实现了getattr的功能，可以直接通过func call的形式来添加函数，
rop.read(0, elf.bss(0x80))实际相当于rop.call('read', (0, elf.bss(0x80)))
通过多次添加函数调用，最后使用str将整个rop chain dump出来就可以了

    call(resolvable, arguments=()) : 添加一个调用，resolvable可以是一个符号，
    也可以是一个int型地址，
    注意后面的参数必须是元组否则会报错，即使只有一个参数也要写成元组的形式(在后面加上一个逗号)
    chain() : 返回当前的字节序列，即payload
    dump() : 直观地展示出当前的rop chain
    raw() : 在rop chain中加上一个整数或字符串
    search(move=0, regs=None, order=’size’) : 按特定条件搜索gadget
    unresolve(value) : 给出一个地址，反解析出符号
-----------------------------------------------------------------------

---------------------------ROP------------------------------------------->

ROP的全称为Return-oriented programming（返回导向编程）
这是一种高级的内存攻击技术可以用来绕过现代操作系统的各种通用防御

Control Flow Hijack 程序流劫持

比较常见的程序流劫持就是栈溢出，格式化字符串攻击和堆溢出了。通过程序流劫持，
攻击者可以控制PC指针从而执行目标代码。为了应对这种攻击，系统防御者也提出了
各种防御方法，最常见的方法有DEP（堆栈不可执行），ASLR（内存地址随机化），
Stack Protector（栈保护）等。但是如果上来就部署全部的防御，初学者可能会觉
得无从下手，所以我们先从最简单的没有任何保护的程序开始，


$ gcc -fno-stack-protector -z execstack -o level1 level1.c
这个命令编译程序。-fno-stack-protector和-z execstack这两个参数会
分别关掉DEP和Stack Protector
同时我们在shell中执行：
$ sudo -s (切换root用户)
$ echo 0 > /proc/sys/kernel/randomize_va_space (把ASLR关掉)


虽然我们关闭了ASLR，但这只能保证buf的地址在gdb的调试环境中不变，但当我们直接执行./level1的时候，
buf的位置会固定在别的地址上。怎么解决这个问题呢？

最简单的方法就是开启core dump这个功能。
$ ulimit -c unlimited
$ sudo sh -c 'echo "/tep/core.%t" > /proc/sys/kernel/core_pattern'
