---
layout: post
title: stack4
---
Kỹ thuật trong bài trước giới thiệu cách tìm và gán địa chỉ hàm để gọi đến hàm đó.
Tuy nhiên, việc gọi hàm thành công là do trong source code đã có sẵn dòng lệnh với mục đích tương tự.
Trong bài này, chúng ta sẽ tìm hiểu cách thức gọi hàm hoàn toàn ngoài ý muốn của chương trình.
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
  char buffer[64];

  gets(buffer);
}
```
Có thể thấy bên trong hàm `main` chỉ gọi `gets` để lấy giá trị người dùng nhập vào và gán cho biến `buffer`.
Chương trình sẽ thoát ngay sau đó mà không có thêm lệnh nào khác.
Tuy nhiên, nếu nhìn vào asm code của chương trình như dưới đây, ta sẽ thấy có thêm 2 lệnh được chạy.
```asm
0x08048408 <main+0>:    push   ebp
0x08048409 <main+1>:    mov    ebp,esp
0x0804840b <main+3>:    and    esp,0xfffffff0
0x0804840e <main+6>:    sub    esp,0x50
0x08048411 <main+9>:    lea    eax,[esp+0x10]
0x08048415 <main+13>:   mov    DWORD PTR [esp],eax
0x08048418 <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave
0x0804841e <main+22>:   ret
```
2 lệnh được nhắc đến là `leave` và `ret`, tương đương với `return` trong C.
Lưu ý rằng chạy một chương trình tương tự như việc gọi 1 hàm con.
Trước khi lệnh đầu tiên của hàm con được thực thi, CPU sẽ lưu lại địa chỉ của lệnh kế tiếp cần được thực hiện sau khi hàm con kết thúc.
Khi hàm con return về hàm đã gọi nó, địa chỉ này sẽ được `pop` vào `eip`, và lệnh tại đó được thực thi.
Ví dụ, hãy cùng xem stack ngay trước vào khi gọi hàm `gets` trong `main` của chương trình stack4.
Phân biệt theo màu: <span style="color:aqua">esp</span> và <span style="color:orangered">eip</span>.
<pre class="memory">
<span style="color:orangered">0x8048418</span> <main+16>:    call   0x804830c <gets@plt>
0x804841d <main+21>:    leave
...
0xbffff748:     0xb7fff8f8
0xbffff74c:     0xb7f0186e
<span style="color:aqua">0xbffff750</span>:     0xbffff760
0xbffff754:     0xb7ec6165
</pre>

<pre class="memory">
<span style="color:orangered">0x804830c</span> <gets@plt>:   jmp    DWORD PTR ds:0x80495fc
...
0xbffff748:     0xb7fff8f8
<span style="color:aqua">0xbffff74c</span>:     0x0804841d
0xbffff750:     0xbffff760
0xbffff754:     0xb7ec6165
</pre>
## Ref
```bash
user@protostar:~$ export GREENIE=$'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888\x0a\x0d\x0a\x0d'
user@protostar:~$ stack2
you have correctly modified the variable
```
