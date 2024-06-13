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
* `bin()`: trả về dạng nhị phân kiểu ký tự
* `enumerate()`: iterate iterable, trả về cả index và phần tử trong chuỗi
* `int()`: trả về số `int` của tham số
* `len()`: trả về độ dài `list`/`str`/iterable
* `list()`: convert tham số, trả về `list`
* `max()`: trả về giá trị lớn nhất của một `list`
* `min()`: trả về giá trị nhỏ nhất của một `list`
* `ord()`: trả về mã ascii dạng số
* `print()`: in ra màn hình
* `range()`: iterate một khoảng (tham số a đến tham số b)
* `set()`: convert tham số, trả về `set`
* `sorted()`: sort và trả về kết quả đã sort
* `str()`: trả về dạng `str` của tham số
* `sum()`: trả về tổng tất cả phần tử của `list`/iterable
* `tuple()`: trả về dang `tuple` của tham số
* `zip()`: dùng để iterate nhiều `list`/iterable một lúc

### Type built-in

Type mặc định của Python. Sau đây la một số type thông dụng

* `int`: trong python, số bit ko giới hạn
* `float`
* `bool`: chỉ có `True` hoặc `False`
* `list`: định nghĩa bằng `[]`; `list` trong python tương tự array, nhưng có thể chứa đa dạng và lẫn lộn nhiều kiểu phần tử (ví dụ: `[1, "2"]` vừa chứa giá trị `int`, vừa chứa giá trị `str`)
* `tuple`: tượng tự `list`, nhưng ít sử dụng hơn; có thể hiểu đơn giản là để gom các giá trị thành một cặp/một bộ giá trị
* `str`: chuỗi ký tự định nghĩa bằng `""` hoặc `''`
* `dict`: định nghĩa bằng `{}` hoặc `{key: value}`; `dict` trong python là hashtable/map trong nhiều ngôn ngữ khác; được sử dụng rất nhiều
* `set`: định nghĩa bằng `{value1, value2}`; tương tự `list` nhưng đảm bảo các giá trị trong `set` là duy nhất

```python
l = [1, 2, 1.4, "i", "<3", [5, 6]] #a list contains various elements of different types
d = {1: "hello", 3: "world", 5:6} #dict
t = (1, 2) #tuple, pair of two values
s = {"a", "b"} #set, with two values
```