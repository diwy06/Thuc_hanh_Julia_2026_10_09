# PHIẾU THỰC HÀNH — 09/10/2026

**Họ tên:** TRẦN HỮU DUY &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **MSSV:** N24DCDT022 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; **Lớp:** D24CQDT01-N  
**GitHub:** `https://github.com/TranHuuDuy-PTIT/Thuc_hanh_Julia_2026_10_09` &nbsp;&nbsp;&nbsp;&nbsp; **Commit SHA:** `a1c4b78`

---

## 1. Mô tả một luồng ứng dụng
*Ghi mục tiêu, đầu vào, đầu ra và sơ đồ truyền dữ liệu giữa các chức năng.*

- **Mục tiêu:** Điều phối giao từng đơn hàng độc lập từ kho (đỉnh 1) tới điểm giao rồi quay về kho trong khuôn viên trường bằng ngôn ngữ Julia. Lựa chọn tập đơn hàng tối ưu trong giới hạn ngân sách thời gian $B$ sao cho tổng lợi ích đạt được lớn nhất (mô hình Bài toán Ba lô 0/1).
- **Đầu vào:** Hai tệp CSV chuẩn hóa:
  - `don_hang.csv`: gồm các cột `id, diem_giao, thoi_gian_phuc_vu, loi_ich`.
  - `ban_do.csv`: gồm các cột `u, v` mô tả 7 cạnh nối vô hướng giữa 6 đỉnh trong khuôn viên.
- **Đầu ra:** Tập các đơn hàng được lựa chọn tối ưu, tuyến đường di chuyển ngắn nhất cho mỗi đơn, chuỗi trạng thái điều phối mô phỏng giao và hoàn tác trên Hàng đợi vòng (`CircularQueue`) và Ngăn xếp (`Stack`), cùng tệp kết quả chi tiết `ket_qua.csv` (9 cột).
- **Sơ đồ truyền dữ liệu:**  
  `[don_hang.csv, ban_do.csv]` $\longrightarrow$ `read_data` (kiểm tra toàn vẹn) $\longrightarrow$ `merge_sort` (chuẩn hóa danh mục) $\longrightarrow$ `bfs_routes` (tìm đường ngắn nhất & tính chi phí $\tau_i = 2h_i + s_i$) $\longrightarrow$ `knapsack` (quy hoạch động chọn đơn tối ưu $\le B$) $\longrightarrow$ `CircularQueue` & `Stack` (thao tác `deliver!` & `undo!`) $\longrightarrow$ `export_result` (`ket_qua.csv`).

---

## 2. Bảng truy vết đường và thời gian (MINI001)

| ID | Đường lượt đi | Số cạnh $h$ | $2h + \text{phục vụ}$ |
|:--:|:-------------:|:-----------:|:---------------------:|
| **1** | $1 \rightarrow 2$ | 1 | $2 \times 1 + 1 = 3$ |
| **2** | $1 \rightarrow 2 \rightarrow 4$ | 2 | $2 \times 2 + 1 = 5$ |
| **3** | $1 \rightarrow 2 \rightarrow 4 \rightarrow 6$ | 3 | $2 \times 3 + 1 = 7$ |

---

## 3. Minh chứng bốn hàm

### NV1 · Hoàn thiện `merge_sort(a, before)`
- **Ý tưởng/điều kiện áp dụng:** Áp dụng chiến lược Chia để trị (Divide and Conquer). Đệ quy chia đôi mảng cho đến khi kích thước $\le 1$, sau đó trộn (merge) hai nửa đã có thứ tự bằng hàm so sánh `before`. Đảm bảo tính ổn định (stable): khi hai phần tử bằng nhau (hòa) thì phần tử bên trái luôn được giữ trước. Áp dụng cho mọi mảng đối tượng có quan hệ thứ tự.
- **Lời gọi và kết quả kiểm thử:**  
  `julia TH_2026_10_09_Kiem_thu.jl --phase=sort`  
  **Kết quả:** ĐẠT 8/8 phép kiểm tra (bao gồm kiểm tra mảng rỗng, 1 phần tử, tăng/giảm dần, tính ổn định và comparator hòa).
- **Chi phí thời gian/bộ nhớ:** Thời gian $O(n \log n)$ trong mọi trường hợp (tốt nhất, trung bình, xấu nhất). Bộ nhớ phụ $O(n)$ do mảng đệm khi trộn.

### NV2 · Hoàn thiện `binary_search(a::Vector{Order}, id::Int)`
- **Ý tưởng/điều kiện áp dụng:** Tìm kiếm theo nguyên lý chia đôi khoảng tìm kiếm $[lo, hi]$. Tại mỗi bước tính $mid = lo + \lfloor(hi - lo)/2\rfloor$ và so sánh $id$ với $a[mid].id$ để thu hẹp một nửa không gian tìm kiếm. **Tiền điều kiện:** Mảng $a$ phải được sắp xếp tăng dần theo trường `id`. Trả về đối tượng `Order` nếu tìm thấy, hoặc `nothing` nếu không có.
- **Lời gọi và kết quả kiểm thử:**  
  `julia TH_2026_10_09_Kiem_thu.jl --phase=search`  
  **Kết quả:** ĐẠT 5/5 phép kiểm tra (tìm thấy đơn ở đầu, giữa, cuối mảng; trả về `nothing` khi ID không tồn tại).
- **Chi phí thời gian/bộ nhớ:** Thời gian $O(\log n)$; Bộ nhớ phụ $O(1)$ (cài đặt lặp tối ưu).

### NV3 · Hoàn thiện `bfs_routes(graph, source=1)`
- **Ý tưởng/điều kiện áp dụng:** Sử dụng hàng đợi vòng `CircularQueue` để duyệt theo chiều rộng (BFS) từ đỉnh nguồn (kho 1) trên đồ thị vô hướng không trọng số. Khởi tạo `distance = -1`, `distance[source] = 0`. Khi lấy đỉnh $u$ ra, duyệt các đỉnh kề $v$ chưa thăm (`distance[v] == -1`), cập nhật `distance[v] = distance[u] + 1`, `parent[v] = u` và đưa $v$ vào hàng đợi. Sau đó truy ngược `parent` từ đích về nguồn để dựng đường đi ít cạnh nhất `routes`.
- **Lời gọi và kết quả kiểm thử:**  
  `julia TH_2026_10_09_Kiem_thu.jl --phase=bfs`  
  **Kết quả:** ĐẠT 7/7 phép kiểm tra (khoảng cách 6 đỉnh, parent, tính hợp lệ của tuyến đường trên đồ thị 6 đỉnh, 7 cạnh).
- **Chi phí thời gian/bộ nhớ:** Thời gian $O(V + E)$ với $V = 6, E = 7$; Bộ nhớ phụ $O(V)$ cho `distance`, `parent` và hàng đợi `CircularQueue`.

### NV4 · Hoàn thiện `knapsack(orders, costs, budget)`
- **Ý tưởng/điều kiện áp dụng:** Áp dụng Quy hoạch động giải bài toán Ba lô 0/1. Bảng `table[i+1, b+1]` lưu lợi ích tối ưu khi xét $i$ đơn hàng đầu tiên với giới hạn ngân sách $b$. Hệ thức truy hồi:
  $$table[i+1, b+1] = table[i, b+1]$$
  Nếu $b \ge c_i$ và $table[i, b - c_i + 1] + v_i > table[i+1, b+1]$ thì cập nhật bằng $table[i, b - c_i + 1] + v_i$.  
  Khi bằng lợi ích, giữ phương án không chọn để đảm bảo nghiệm ổn định. Truy vết ngược từ `table[n+1, budget+1]` để xác định tập ID được chọn.
- **Lời gọi và kết quả kiểm thử:**  
  `julia TH_2026_10_09_Kiem_thu.jl --phase=dp`  
  **Kết quả:** ĐẠT 12/12 phép kiểm tra (đối chiếu chính xác từng ô bảng, tập ID chọn, tổng lợi ích và chi phí với thuật toán vét cạn `brute_force`).
- **Chi phí thời gian/bộ nhớ:** Thời gian $O(n \times B)$; Bộ nhớ phụ $O(n \times B)$ cho bảng quy hoạch động.

---

## 4. Quyết định chọn đơn
*Điền ngân sách, tập chọn, tổng thời gian/lợi ích. Giải thích một ô DP và một bước truy vết; đối chiếu với vét cạn.*

- **Trên bộ dữ liệu mẫu MINI001:**
  - **Ngân sách:** $B = 8$. Tập chọn: **[1, 2]**. Tổng thời gian: $3 + 5 = 8 \le 8$. Tổng lợi ích: $6 + 8 = 14$.
  - **Giải thích ô DP $F(2, 8)$** (tương ứng `table[3, 9]` trong Julia do chỉ số từ 1):  
    Đơn 2 có $c_2 = 5, v_2 = 8$.
    - Phương án 1 (không lấy đơn 2): Kế thừa $F(1, 8) = 6$.
    - Phương án 2 (lấy đơn 2): Cho giá trị $F(1, 8 - 5) + 8 = F(1, 3) + 8 = 6 + 8 = 14$.
    - Do $14 > 6 \Rightarrow table[3, 9] = 14$.
  - **Bước truy vết:** Tại $i = 2, b = 8$: $table[3, 9] = 14 \neq table[2, 9] = 6 \rightarrow$ Chọn đơn 2, ngân sách còn lại $8 - 5 = 3$. Tại $i = 1, b = 3$: $table[2, 4] = 6 \neq table[1, 4] = 0 \rightarrow$ Chọn đơn 1. Tập chọn: **[1, 2]**.
  - **Đối chiếu vét cạn:** Xét toàn bộ $2^3 = 8$ tập con, phương án $\{1, 2\}$ đạt lợi ích 14 với chi phí 8. DP và vét cạn khớp hoàn toàn 100%.

- **Mở rộng trên bộ dữ liệu cá nhân N24DCDT022:**
  - **Ngân sách:** $B = 25$. Tập chọn: **[4, 5, 6, 7, 2]** (5 đơn). Tổng thời gian: $3 + 5 + 7 + 5 + 5 = 25 \le 25$. Tổng lợi ích: $12 + 13 + 13 + 9 + 6 = 53$. (Khớp 100% giữa DP, vét cạn và tham lam).

---

## 5. Giao và hoàn tác

| Trạng thái | Queue | Stack | Đã dùng | Đã nhận |
|:---|:---:|:---:|:---:|:---:|
| **Sau lập kế hoạch** | `[1, 2]` | `[]` | 0 | 0 |
| **Sau giao một đơn** | `[2]` | `[1]` | 3 | 6 |
| **Sau hoàn tác** | `[1, 2]` | `[]` | 0 | 0 |
| **Sau giao hết** | `[]` | `[1, 2]` | 8 | 14 |

---

## 6. Đầu ra và nhận xét

- **Đường dẫn CSV kết quả:** `data/MINI001/ket_qua.csv` và `data/N24DCDT022/ket_qua.csv`  
- **Link notebook:** `TH_2026_10_09_Pluto.jl`  
- **Lỗi đã phát hiện và cách sửa:**
  1. *Lỗi lệch chỉ số mảng 1-based trong Julia:* Bảng DP có kích thước $(n+1) \times (B+1)$, nên ô $F(i, b)$ phải được truy cập tại `table[i+1, b+1]`.
  2. *Lỗi tra cứu chi phí:* Tham số `costs` là kiểu `Dict` tra cứu theo ID đơn hàng, phải dùng cú pháp `costs[orders[i].id]` thay vì `costs[i]`.
  3. *Lỗi thứ tự duyệt kề:* Dùng `merge_sort` chuẩn hóa thứ tự duyệt danh sách kề theo tham số `tie` giúp thứ tự BFS hoàn toàn ổn định và tái lập được.
- **Một chức năng thay đổi ảnh hưởng đến chức năng sau như thế nào?**  
  Các mô-đun tạo thành một pipeline khép kín: Nếu `bfs_routes` tính sai số cạnh $h \rightarrow$ chi phí $\tau$ sai $\rightarrow$ bảng DP của `knapsack` nhận chi phí sai và chọn ra tập đơn vượt ngân sách hoặc sai lợi ích $\rightarrow$ hàng đợi điều phối giao sai $\rightarrow$ kết quả xuất ra `ket_qua.csv` sai lệch toàn bộ.

---

## Câu hỏi cuối buổi (Q1 – Q5)

### Q1. Vì sao không được dùng dãy đã sắp giảm theo lợi ích/$\tau$ để tìm kiếm nhị phân theo ID? Nêu tiền điều kiện của hàm tìm kiếm.
**Trả lời:**  
Dãy sắp giảm theo lợi ích/$\tau$ được sắp xếp theo khóa tỷ số (lợi ích trên chi phí) nhằm phục vụ chiến lược tham lam, không bảo toàn tính đơn điệu theo trường `id`. Tiền điều kiện bắt buộc của tìm kiếm nhị phân theo ID là mảng đầu vào phải được sắp xếp tăng dần (hoặc đơn điệu) theo trường `id`. Nếu áp dụng trên mảng chưa sắp theo ID, điều kiện so sánh giữa `id` cần tìm và phần tử giữa `a[mid].id` sẽ loại bỏ sai nửa không gian chứa phần tử cần tìm, khiến hàm trả về `nothing` dù phần tử thực sự có mặt trong danh sách.

### Q2. Với đơn ID 2 tới điểm 4, phục vụ 1 và bản đồ mẫu, giải thích từng thành phần của thời gian thực hiện $\tau$. BFS có tiếp tục cho thời gian ngắn nhất nếu các cạnh mang thời gian khác nhau không?
**Trả lời:**  
Với đơn ID 2 giao tới điểm 4, thời gian phục vụ $s = 1$:  
Tuyến đường ngắn nhất từ kho 1 tới đỉnh 4 trên bản đồ là $1 \rightarrow 2 \rightarrow 4$ với số cạnh $h = 2$. Thời gian thực hiện gồm:  
- Lượt đi từ kho tới điểm giao: $h = 2$  
- Lượt quay về lại kho xuất phát: $h = 2$  
- Thời gian phục vụ tại điểm giao: $s = 1$  
$\Rightarrow \tau = 2h + s = 2 \times 2 + 1 = 5$ đơn vị thời gian.  
Nếu các cạnh mang thời gian di chuyển khác nhau (đồ thị có trọng số), BFS thông thường **KHÔNG CÒN** đảm bảo tìm được đường đi ngắn nhất, vì BFS mở rộng theo số lượng cạnh ít nhất chứ không xét tổng trọng số thời gian. Khi đó, bài toán bắt buộc phải sử dụng thuật toán **Dijkstra** hoặc hàng đợi ưu tiên.

### Q3. Giải thích hai phương án tạo ô $F(2,8)$ và vị trí của ô đó trong table Julia. Vì sao nhánh lấy đơn phải dùng hàng trước?
**Trả lời:**  
- **Vị trí trong table Julia:** Do chỉ số mảng trong Julia bắt đầu từ 1, hàng và cột 1 đại diện cho $i=0$ và $b=0$. Do đó, ô $F(2, 8)$ nằm tại tọa độ `table[2+1, 8+1] = table[3, 9]`.
- **Hai phương án tạo ô:** Đơn thứ 2 có chi phí $c_2 = 5$ và lợi ích $v_2 = 8$:
  1. *Không lấy đơn 2:* Kế thừa giá trị tối ưu của 1 đơn đầu với ngân sách 8: $F(1, 8) = 6$.
  2. *Lấy đơn 2:* Nhận lợi ích $v_2 = 8$ cộng với lợi ích tối ưu của 1 đơn đầu với ngân sách còn lại $(8 - 5 = 3)$: $F(1, 3) + 8 = 6 + 8 = 14$.
  3. Giá trị ô là $\max(6, 14) = 14$.
- **Lý do nhánh lấy đơn phải dùng hàng trước ($i-1$):** Vì đây là bài toán ba lô 0/1, mỗi đơn hàng chỉ được chọn giao tối đa một lần. Việc tham chiếu tới hàng $i-1$ bảo đảm đơn thứ $i$ chưa từng được chọn trong phần ngân sách còn lại $(b - c_i)$. Nếu lấy từ cùng hàng $i$, đơn $i$ có thể được cộng dồn nhiều lần, làm bài toán biến thành bài toán ba lô không giới hạn (Unbounded Knapsack).

### Q4. Sau khi giao ID 1 rồi hoàn tác ngay, FIFO, ngăn xếp, tổng thời gian và tổng lợi ích phải thay đổi thế nào? Nêu một bất biến kiểm tra việc không mất hoặc lặp đơn.
**Trả lời:**  
- **Khi giao ID 1:** Đơn 1 được lấy ra khỏi đầu hàng đợi FIFO (`dequeue!`), ngăn xếp Stack đẩy đơn 1 vào đỉnh (`push!`), tổng thời gian tăng thêm chi phí của đơn 1 (tăng từ 0 lên 3), tổng lợi ích tăng thêm lợi ích của đơn 1 (tăng từ 0 lên 6).
- **Khi hoàn tác ngay:** Đơn 1 được lấy ra khỏi đỉnh ngăn xếp (`pop!`), chèn ngược lại vào đầu hàng đợi FIFO (`enqueue_front!`), tổng thời gian giảm 3 (về 0), tổng lợi ích giảm 6 (về 0). Hệ thống trở về trạng thái nguyên vẹn ban đầu.
- **Bất biến kiểm tra (Invariants):**
  1. *Bảo toàn danh sách đơn:* $\text{set}(Queue) \cup \text{set}(Stack) = \text{set}(selected\_ids)$ tại mọi thời điểm.
  2. *Không trùng lặp đơn:* $\text{set}(Queue) \cap \text{set}(Stack) = \emptyset$.
  3. *Bảo toàn chi phí và lợi ích:* $\text{spent} = \sum_{id \in Stack} costs[id]$ và $\text{earned} = \sum_{id \in Stack} benefit[id]$.

### Q5. Cho biết độ phức tạp theo $n, V, E, B$ của sắp xếp trộn, tìm kiếm nhị phân, BFS cốt lõi và DP bảng đầy đủ. Vì sao không thể khẳng định mọi BST đều tra cứu trong $O(\log n)$?
**Trả lời:**  
- **Bảng độ phức tạp:**
  - Sắp xếp trộn (Merge Sort): Thời gian $O(n \log n)$, Bộ nhớ phụ $O(n)$.
  - Tìm kiếm nhị phân (Binary Search): Thời gian $O(\log n)$, Bộ nhớ phụ $O(1)$.
  - BFS cốt lõi: Thời gian $O(V + E)$, Bộ nhớ phụ $O(V)$.
  - DP bảng đầy đủ: Thời gian $O(n \times B)$, Bộ nhớ phụ $O(n \times B)$.
- **Lý do không thể khẳng định mọi BST đều tra cứu trong $O(\log n)$:**  
  Thời gian tìm kiếm trên cây tìm kiếm nhị phân (BST) phụ thuộc trực tiếp vào chiều cao $h$ của cây ($O(h)$). Chiều cao $h$ chỉ đạt $\approx \log_2 n$ khi cây ở trạng thái cân bằng. Trong trường hợp xấu nhất (ví dụ khi chèn các phần tử theo thứ tự đã sắp xếp tăng dần hoặc giảm dần), cây bị suy biến hoàn toàn thành một danh sách liên kết có chiều cao $h = O(n)$. Khi đó, chi phí tìm kiếm suy biến thành $O(n)$. Để đảm bảo luôn đạt $O(\log n)$, cần sử dụng các cây tự cân bằng như AVL Tree hoặc Red-Black Tree.
