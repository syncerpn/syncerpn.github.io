---
layout: post
title: level_1
---

## Lưu ý gì khi làm về thuật toán

Thuật toán là lời giải được áp dụng lên dữ liệu cho trước (đầu vào/input) của một vấn đề.
Trong giải thuật, việc giải được vấn đề chưa phải tất cả mà còn là giải được vấn đề trong bao lâu và dùng đến bao nhiêu tài nguyên.
Về cơ bản, các thuật toán dạng thử đến chết (thường gọi bruteforce) chắc chắn sẽ giải được vấn đề, nhưng thường chỉ được đánh giá là lựa chọn cuối cùng.
Do vậy, khi làm giải thuật cần lưu ý đến các vấn đề sau

## Giới hạn (constraints)

Giới hạn là tập xác định của đầu vào.
Nếu thuật toán chỉ được thực hiện trên đầu vào đơn giản, hãy đơn giản hóa thuật toán để tối ưu thời gian chạy và tài nguyên sử dụng.

## Độ phức tạp của giải thuật (complexity)

Độ phức tạp của giải thuật sẽ được đo bằng thời gian (time complexity) và tài nguyên sử dụng (space complexity).
Để đo được độ phức tạp, người ta thường không đo thời gian chạy thực tế tính bằng giây hay MB bộ nhớ cần dùng.
Đơn vị đo độ phức tạp là hàm big-O (ví dụ: O(1), O(logn), O(n), etc.)
Ý nghĩa của hàm big-O là: khi đầu vào có độ phức tạp tăng dần đến vô hạn thì độ phức tạp của thuật toán sẽ phụ thuộc thế nào vào độ phức tạp của đầu vào.

Ví dụ đơn giản như sau: tìm 2 phần tử trong một `list` chứa `n` số nguyên sao cho tổng 2 số đó bằng 0.
Với bài toán ví dụ này, số lượng phần tử `n` chính là độ phức tạp của đầu vào.
Phương pháp bruteforce sử dụng 2 vòng lặp như sau sẽ giải được bài toán với độ phức tạp thời gian là O(n^2) và tài nguyên O(1).

```python
def solve_bruteforce(nums: list, target: int) -> list:
	n = len(nums)
	for i in range(n-1):
		for j in range(i, n):
			if nums[i] + nums[j] == target:
				return [nums[i], nums[j]]

```

Độ phức tạp thời gian O(n^2) có thể được giải thích như sau:
* 