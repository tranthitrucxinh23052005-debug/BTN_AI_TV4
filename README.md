# Báo cáo bàn giao — TV4 (Search Engineer)

**Module:** `path_validator.py` + `baseline_solver.py`


**Ngày:** 21/6


**Deadline gốc:** 1/7 — đã xong sớm, sẵn sàng bàn giao cho TV3 và TV5

---

## ⚠️ 1. Bug nghiêm trọng đã phát hiện và sửa: treo máy vô hạn

**Mức độ:** Nghiêm trọng — ảnh hưởng trực tiếp đến TV3 (tính reward khi train) và TV5 (vòng lặp real-time).

### Hiện tượng
`find_valid_path()` bị **treo vô hạn (không bao giờ trả về kết quả)** trên một số bàn cờ bế tắc — kể cả trên chính bộ test 4 case ban đầu. Cụ thể, test case "hai ô bị chặn hoàn toàn" `(0,2) → (2,0)` vốn phải trả về `None` thì lại làm chương trình treo, không hề có exception hay timeout tự nhiên.

### Nguyên nhân gốc
Trong hàm `is_walkable()`:

```python
# CODE CŨ - CÓ BUG
if r == -1 or r == rows or c == -1 or c == cols:
    return True
```

Phép `or` này trả `True` ngay khi **chỉ một trong hai trục** chạm viền ảo, kể cả khi trục còn lại đã văng quá xa khỏi giới hạn cho phép (ví dụ toạ độ `(-2, -1)`: vì `c == -1` đúng nên hàm trả `True` ngay, dù `r = -2` đã vượt quá viền ảo 1 ô theo thiết kế).

Hậu quả: không gian trạng thái mà BFS phải duyệt lớn hơn rất nhiều so với thiết kế ban đầu (vốn chỉ nên giới hạn trong viền mở rộng 1 ô quanh bàn cờ). Trên các bàn cờ không có lời giải, BFS phải quét cạn không gian này trước khi kết luận "bế tắc" — và vì không gian bị phình to bất thường, thời gian chạy trở nên cực lớn, biểu hiện ra ngoài như treo máy.

### Cách sửa
Viết lại điều kiện kiểm tra tường minh theo từng bước, đảm bảo cả hai trục đều được kiểm tra đúng giới hạn `[-1, rows]` / `[-1, cols]` trước khi kết luận:

```python
# CODE MỚI - ĐÃ SỬA
def is_walkable(board, r, c):
    rows, cols = board.shape
    r_ok = -1 <= r <= rows
    c_ok = -1 <= c <= cols
    if not (r_ok and c_ok):
        return False  # văng ra quá xa viền ảo

    r_in_real = 0 <= r < rows
    c_in_real = 0 <= c < cols
    if r_in_real and c_in_real:
        return board[r, c] == -1  # trong bàn cờ thật -> phải là ô trống

    return True  # viền ảo (kể cả góc ảo kép) -> luôn đi được
```

Lưu ý: vẫn **giữ nguyên luật cho phép "cắt góc" qua điểm ảo kép** (ví dụ `r=-1` và `c=-1` cùng lúc) vì đây là luật Onet chuẩn khi nối 2 ô ở góc chéo bàn cờ chỉ với 2 lần rẽ — chỉ sửa phần kiểm tra giới hạn khoảng cách, không thay đổi luật chơi.

### Bug phụ đi kèm (đã sửa cùng lúc)
`visited` set trước đây chỉ được đánh dấu **lúc pop khỏi queue**, không phải lúc push vào. Điều này khiến nhiều bản sao của cùng một trạng thái `(r, c, hướng, số_lần_rẽ)` có thể bị đẩy vào queue nhiều lần trước khi bản đầu tiên kịp bị lọc trùng — làm tăng thêm độ trễ không cần thiết. Đã sửa: đánh dấu `visited` **ngay khi push**, giúp BFS chạy tối ưu hơn đáng kể.

### Đã verify kỹ trước khi kết luận hết bug
Vì sửa `is_walkable` có rủi ro làm mất luật "đi qua góc ảo" hợp lệ của Onet thật, mình đã test riêng case nối 2 ô ở góc chéo bàn cờ (`(0,0) → (1,1)` trên board 2×2) để đảm bảo đường đi qua góc ảo kép vẫn hoạt động đúng — không bị cấm nhầm.

---

## ✅ 2. Bộ test case biên — nâng từ 4 lên 8 case

Theo yêu cầu "viết ít nhất 6 test case", đã bổ sung và tổ chức lại thành hàm `run_test_suite()`:

| # | Test case | Kỳ vọng |
|---|---|---|
| 1 | Hai ô liền kề cùng cột (0 lần rẽ) | Có đường |
| 2 | Hai ô liền kề cùng hàng (0 lần rẽ) | Có đường |
| 3 | Đi hình chữ L (1 lần rẽ) | Có đường |
| 4 | Đi vòng qua viền ngoài (2 lần rẽ) | Có đường |
| 5 | Hai ô bị chặn hoàn toàn | `None` (bế tắc) |
| 6 | 2 lần rẽ **không** qua viền (đi trong bàn cờ) | Có đường |
| 7 | Bàn cờ gần kín, chỉ còn 1 đường hẹp duy nhất | Có đường |
| 8 | Giới hạn `max_turns=0` chặt, đường duy nhất cần rẽ | `None` (đúng vì vượt giới hạn) |

Mỗi test còn được tự động kiểm chứng bằng hàm `_validate_path_shape()` — không chỉ kiểm tra có/không có đường, mà còn xác minh path trả về **thực sự đúng luật**: các bước liên tiếp phải kề nhau, số lần rẽ không vượt `max_turns`, mọi ô giữa đường đều đi được.
<img width="407" height="419" alt="image" src="https://github.com/user-attachments/assets/119c2e49-e16e-42a0-882a-5c3f80e88960" />

**Kết quả:** `8/8 PASS`.

---

## ✅ 3. Benchmark tốc độ — đạt tiêu chí < 50ms

Đã viết `run_benchmark()`: sinh board ngẫu nhiên kích thước thật **8×6** (theo `config.py` của TV2), chạy 300 lần gọi `find_valid_path` với các cặp ô ngẫu nhiên.

| Chỉ số | Kết quả |
|---|---|
| Trung bình | **0.10 ms** |
| P50 | 0.08 ms |
| P95 | 0.26 ms |
| Tệ nhất | 0.45 ms |
| **Mục tiêu** | < 50 ms |
<img width="239" height="178" alt="image" src="https://github.com/user-attachments/assets/b14dcfae-8f1b-4fc6-a779-d2f94f183d20" />

→ **Đạt tiêu chí, nhanh hơn yêu cầu khoảng 100 lần.** Không cần nâng cấp lên A* ở giai đoạn này.

**Stress test bổ sung:** chạy toàn bộ cặp ô (946 cặp) trên board 8×6 với 92% ô đã lấp đầy (gần kín, dễ kích hoạt worst-case) — không có lần nào treo, thời gian tệ nhất 0.19ms.

---

## ✅ 4. `baseline_solver.py` — nâng cấp chạy full ván

### Trước đây
`baseline_choose_move()` chỉ chọn **một nước đi đầu tiên** tìm thấy rồi dừng — chưa đáp ứng tiêu chí "chạy được độc lập, chọn được nước đi hợp lệ **liên tục** đến khi bàn cờ hết nước đi".

### Đã thêm
- **`play_full_game(board)`**: lặp lại `baseline_choose_move` liên tục, tự "xoá" từng cặp ô đã nối (gán `-1`), in tiến trình từng bước, dừng khi thắng (xoá sạch) hoặc bế tắc. Trả về dict thống kê đầy đủ (`cleared`, `moves`, `remaining`, `steps`).
- **`generate_solvable_board()`**: sinh board ngẫu nhiên nhưng **đảm bảo luôn có lời giải lý thuyết** (mỗi ID xuất hiện đúng số chẵn lần) — dùng để test baseline trên dữ liệu sạch.
- **`apply_move()` / `is_board_cleared()`**: helper phụ trợ.

### Kết quả kiểm thử
<img width="289" height="221" alt="image" src="https://github.com/user-attachments/assets/a4e5b987-933e-4751-b8d8-431e70643616" />

<img width="287" height="260" alt="image" src="https://github.com/user-attachments/assets/bf32098c-5df1-446e-bb47-3c504e1fdd88" />

- Board test chuẩn 4×4: bế tắc sau 4 nước, còn lại đúng 1 ô — **đúng kỳ vọng**, vì ID `5` trong board này chỉ xuất hiện 1 lần (lẻ), không thể có cặp.
- Board 6×4 đảm bảo có lời giải: **xoá sạch 100% (12/12 nước)**.
- Stress test 20 ván ngẫu nhiên trên board 8×6 (đảm bảo có lời giải): **baseline tự xoá sạch 19/20 ván (95%)**.

**Lưu ý quan trọng cho TV3:** baseline này là thuật toán "tham lam" — chọn cặp hợp lệ đầu tiên tìm thấy, không nhìn trước hay backtrack. Vì vậy dù bàn cờ có lời giải lý thuyết, baseline vẫn có ~5% khả năng tự làm khó chính mình (chọn sai thứ tự khiến ô sau bị kẹt). Đây là đặc tính đã biết, không phải lỗi — và là điểm baseline RL agent (TV3) có thể vượt qua bằng cách học chiến lược ưu tiên (ví dụ ưu tiên xử lý "ô khó" trước, đúng như thiết kế reward trong tài liệu phân công).

---

## 📦 5. Cần báo ngay cho nhóm

- **TV3 (RL Engineer):** Reward tính sai có thể từng xảy ra nếu dùng bản `path_validator.py` cũ — vì khi `find_valid_path` treo, vòng lặp train cũng treo theo. Cần cập nhật bản mới trước khi train tiếp.
- **TV5 (Automation Engineer):** Vòng lặp real-time gọi `find_valid_path` mỗi bước — nếu dùng bản cũ, có rủi ro app đứng hình giữa ván khi gặp đúng board bế tắc dạng đặc biệt. Cập nhật bản mới để tránh.
- **TV6 (Tích hợp):** `find_valid_path(board, cellA, cellB) -> list[(r,c)] | None` — interface không đổi, chỉ sửa nội bộ, không ảnh hưởng `interfaces.md`.

---

## File đính kèm
- `path_validator.py` — đã sửa bug, có `run_test_suite()` (8 test) và `run_benchmark()`, chạy `python path_validator.py` để tự kiểm tra.
- `baseline_solver.py` — đã thêm `play_full_game()`, chạy `python baseline_solver.py` để xem demo 2 ván (board chuẩn + board đảm bảo có lời giải).

**Trạng thái: ĐÃ HOÀN THÀNH tiêu chí TV4, sẵn sàng bàn giao sớm hơn deadline (1/7).**
