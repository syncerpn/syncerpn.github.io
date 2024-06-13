---
layout: post
title: level_1
---

## Lưu ý gì khi làm về thuật toán

Thuật toán là lời giải được áp dụng lên dữ liệu cho trước (đầu vào/input) của một vấn đề.
Trong giải thuật, việc giải được vấn đề chưa phải tất cả mà còn là giải được vấn đề trong bao lâu và dùng đến bao nhiêu tài nguyên.
Về cơ bản, các thuật toán dạng thử đến chết (thường gọi bruteforce) chắc chắn sẽ giải được vấn đề, nhưng thường chỉ được đánh giá là lựa chọn cuối cùng.
Do vậy, khi làm giải thuật cần lưu ý đến các vấn đề sau

### Giới hạn (constraints)

Giới hạn là tập xác định của đầu vào.
Nếu thuật toán chỉ được thực hiện trên đầu vào đơn giản, hãy đơn giản hóa thuật toán để tối ưu thời gian chạy và tài nguyên sử dụng.

### Độ phức tạp của giải thuật (complexity)

Độ phức tạp của giải thuật sẽ được đo bằng thời gian (time complexity) và tài nguyên sử dụng (space complexity).
Để đo được độ phức tạp, người ta thường không đo thời gian chạy thực tế tính bằng giây hay MB bộ nhớ cần dùng.
Đơn vị đo độ phức tạp là hàm big-O (ví dụ: `O(1)`, `O(logn)`, `O(n)`, etc.)
Ý nghĩa của hàm big-O là: khi đầu vào có độ phức tạp tăng dần đến vô hạn thì độ phức tạp của thuật toán sẽ phụ thuộc thế nào vào độ phức tạp của đầu vào.

Ví dụ đơn giản như sau: tìm 2 phần tử trong một `list` chứa `n` số nguyên sao cho tổng 2 số đó bằng 0.
Với bài toán ví dụ này, số lượng phần tử `n` chính là độ phức tạp của đầu vào.

#### Phương pháp bruteforce

Sử dụng 2 vòng lặp như sau sẽ giải được bài toán với độ phức tạp thời gian là `O(n^2)` và tài nguyên `O(1)`.

```python
def solve_bruteforce(nums: list, target: int) -> list:
	n = len(nums)
	for i in range(n-1):
		for j in range(i, n):
			if nums[i] + nums[j] == target:
				return [nums[i], nums[j]]

```

Các bước giải bằng bruteforce được giải thích như sau:
* Chạy qua (iterate) từng phần tử trong đầu vào `nums`
* Với mỗi phần tử `nums[i]`, tiếp tục chạy qua từng phần tử trong phần còn lại của `nums`, tạo ra một cặp `nums[i]` và `nums[j]`
* Kiểm tra tổng của cặp tạo ra; nếu trùng với giá trị yêu cầu thì bài toán đã được giải
* Nếu không trùng, tiếp tục kiểm tra các cặp số khác

Độ phức tạp thời gian `O(n^2)` có thể được giải thích như sau:
* Vòng lặp thứ nhất lặp qua `n-1` phần tử
* Vòng lặp thứ hai lặp qua `n-i` phần tử
* Đơn giản hóa, số lượng cặp tạo ra trong trường hợp xấu nhất là phụ thuộc vào `n` dưới dạng hàm bậc 2
* Do đó thời gian cần để hoàn thành cả hai vòng lặp sẽ phụ thuộc vào `n` theo hàm bậc 2: `n^2`

Độ phức tạp tài nguyên `O(1)` có thể được giải thích như sau:
* Chỉ 1 biến `n` để lưu số phần tử của đầu vào
* `nums[i]` và `nums[j]` chỉ là lấy giá trị từ đầu vào, không tốn thêm bộ nhớ
* Như vậy, tổng số lượng bộ nhớ cần thêm chỉ là 1
* Dù số lượng phần tử trong `nums` có tăng lên vô hạn, thuật toán vẫn chỉ cần 1 biến
* Khi độ phức tạp của thuật toán không phụ thuộc vào độ phức tạp của đầu vào, ta coi độ phức tạp của thuật toán là `O(1)`

#### Phương pháp hashtable/hashmap

Độ phức tạp thời gian của brute `O(n^2)` được coi là khá tệ.
Độ phức tạp này có thể được giảm xuống `O(n)` bằng cách sử dụng thêm tài nguyên mức `O(n)`.

```python
def solve_hashmap(nums: list, target: int) -> list:
	d = set() #python set (hoặc dict) để lưu thêm thông tin cho việc tăng tốc thời gian xử lý
	for a in nums:
		d.add(a)

	for a in nums:
		if target - a in d:
			return [a, target-a]

```

Các bước giải bằng hashmap được giải thích như sau:
* Chạy qua (iterate) từng phần tử trong đầu vào `nums`
* Thêm phần tử đó vào `set`
* Chạy qua (iterate) từng phần tử `a` trong đầu vào `nums`
* Tìm phần bù `target - a` trong `set` đã tạo; trả về cặp giá trị nếu tìm thấy

Độ phức tạp thời gian `O(n)` có thể được giải thích như sau:
* Vòng lặp thứ nhất lặp qua `n` phần tử của đầu vào
* Thêm vào `set` là lệnh tiêu tốn thời gian cố định `O(1)` không phụ thuộc `n`
* Vòng lặp thứ hai lặp qua `n` phần tử
* Truy cập `set` (hay bất kỳ kiểu dữ liệu hashmap/hashtable nào) sẽ tiêu tốn thời gian cố định `O(1)` không phụ thuộc `n`
* Do đó thời gian cần để hoàn thành thuật toán sẽ phụ thuộc vào `n` theo hàm bậc 1: `O(n * 1) + O(n * 1) = O(n) + O(n) = O(2n)`
* `O(2n)` tương đương `O(n)` theo tính chất của hàm big-O vì bản chất của big-O xét đến giá trị phụ thuộc `n` tăng đến vô cùng, và khi `n` tăng đến vô cùng thì `O(2n)` và `O(n)` cũng như nhau
* (tất nhiên trong thực tế, thời gian đo bằng giây để chạy thuật toán `O(2n)` sẽ vẫn lớn hơn `O(n)`)

Độ phức tạp tài nguyên `O(n)` có thể được giải thích như sau:
* Cần 1 biến `d` có kích cỡ phụ thuộc vào kích cỡ của đầu vào; do đó, độ phức tạp sẽ là `O(n)`
* Các bước còn lại không phát sinh thêm biến cần lưu
* Tổng kết lại, độ phức tạp là `O(n)`

### Các độ phức tạp thường gặp (từ thấp đến cao)

* `O(1)`: constant complexity; luôn tiêu tốn một mức cố định; độ phức tạp không phục thuộc vào đầu vào (ví dụ: hashmap lookup)
* `O(logn)`: log complexity; độ phức tạp phụ thuộc theo hàm log (ví dụ: binary search)
* `O(n)`: linear complexity; độ phức tạp tuyến tính (ví dụ: list iteration, two-pointer)
* `O(nlogn)`: log-linear complexity; độ phức tạp log tuyến tính (ví dụ: merge sort)
* `O(n^2)`: độ phức tạp phụ thuộc bình phương của đầu vào (ví dụ: bubble sort, 2 vòng lặp lồng nhau) 

### Thời gian hay tài nguyên quan trọng hơn?

Phần lớn giải thuật ưu tiên độ phức tạp thời gian nhỏ.