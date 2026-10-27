# Tài liệu: Phân cụm phân cấp — AGNES (bottom-up Ward) & DIANA

Tài liệu này mô tả chi tiết file [`ward_clustering_1d.ipynb`](ward_clustering_1d.ipynb): bố cục
từng cell, các hàm, hai thuật toán được cài đặt và kết quả kiểm chứng. Mọi con số dưới đây là
output thật của notebook.

---

## 0. Tóm tắt

| Hạng mục | Giá trị |
|---|---|
| Dữ liệu | $A=1,\;B=2,\;C=4,\;D=8,\;E=9$ (1 chiều) |
| Khoảng cách | $d(x,y)=\lvert x-y\rvert$ |
| Thuật toán | ① AGNES (bottom-up Ward), ② DIANA (top-down) |
| Cắt cây tại $k=2$ | $\{A,B,C\}$ và $\{D,E\}$ — cả hai cây đều cho cùng kết quả |
| WCSS tại $k=2$ | $5.1667$ = cực tiểu toàn cục (duyệt đủ 15 cách chia) |
| Công cụ | `numpy` + `matplotlib` (không dùng `scipy`/`sklearn`) |
| Số cell | 18 (9 markdown + 9 code) |

---

## 1. Bài toán và dữ liệu

Cho 5 điểm trên trục số, khoảng cách là hiệu tuyệt đối. Ma trận khoảng cách (cell 02 in ra):

| $d$ | A | B | C | D | E |
|---|---|---|---|---|---|
| **A** | 0 | 1 | 3 | 7 | 8 |
| **B** | 1 | 0 | 2 | 6 | 7 |
| **C** | 3 | 2 | 0 | 4 | 5 |
| **D** | 7 | 6 | 4 | 0 | 1 |
| **E** | 8 | 7 | 5 | 1 | 0 |

Dữ liệu có **hai nhóm rõ rệt**: khoảng trống $C \to D$ bằng 4, lớn hơn hẳn các khoảng còn lại
(1, 1, 2, 1). Đây là lý do cả hai phương pháp đều ưu tiên tách $\{A,B,C\}$ khỏi $\{D,E\}$.

---

## 2. Bố cục notebook (18 cell)

| Cell | Loại | Nội dung | Output chính |
|---|---|---|---|
| 00 | md | Đề bài, bảng toạ độ, ma trận khoảng cách, yêu cầu | — |
| 01 | md | Tiêu chí Ward: WCSS, ΔWCSS, bảng so sánh AGNES/DIANA | — |
| 02 | code | Import, khai báo `LABELS`, `X`, `N`, ma trận `D` | In ma trận khoảng cách |
| 03 | md | Giới thiệu `Node` và các hàm dùng chung | — |
| 04 | code | `fmt_set`, `wcss`, `mean_of`, `ward_delta`, `Node` + kiểm chứng công thức | Bảng ΔWCSS 10 cặp, WCSS tổng = 50.8 |
| 05 | md | AGNES: ý tưởng, giả mã, tie-break, chi phí | — |
| 06 | code | Vòng lặp gộp → `agg_root`, `agg_steps` | Bảng 4 bước gộp |
| 07 | md | Giới thiệu công cụ vẽ | — |
| 08 | code | `draw_axis`, `draw_dendrogram`, `cut_tree`, `is_monotone` | — |
| 09 | code | Vẽ trục số + dendrogram AGNES | `figures/ward_bottom_up.png` |
| 10 | md | DIANA: splinter group, đường kính, bảng độ cao 8/3/1/1 | — |
| 11 | code | `diameter`, `avg_dissim`, `diana_split`, vòng lặp tách → `diana_root` | Bảng 4 lần tách |
| 12 | code | `draw_box_tree` + vẽ cây hộp kiểu DIANA | `figures/diana_tree.png` |
| 13 | md | Mục tiêu so sánh hai cây, ghi chú về ngưỡng cắt | — |
| 14 | code | Bảng cắt $k=1..5$, `find_node`, so sánh độ cao, kiểm chứng | Bảng so sánh + assert |
| 15 | code | Vẽ 2 cây cạnh nhau | `figures/ward_so_sanh.png` |
| 16 | md | Kết luận: vì sao giống nhau, diễn biến, chi phí | — |
| 17 | md | Hướng dẫn chạy | — |

---

## 3. Ký hiệu và công thức nền

Với cụm $C$ (tập chỉ số điểm):

$$\mathrm{WCSS}(C)=\sum_{x\in C}\left(x-\bar x_C\right)^2,
\qquad \bar x_C=\frac{1}{|C|}\sum_{x\in C}x$$

**Phương sai tăng thêm khi gộp** (chính là khoảng cách Ward, dùng cho AGNES):

$$\Delta(A,B)=\mathrm{WCSS}(A\cup B)-\mathrm{WCSS}(A)-\mathrm{WCSS}(B)
=\frac{|A|\,|B|}{|A|+|B|}\left(\bar x_A-\bar x_B\right)^2\;\ge 0$$

Hai đại lượng đo "độ to" của cụm, dùng ở hai thuật toán khác nhau:

- **Đường kính**: $\mathrm{diam}(C)=\max_{x,y\in C} d(x,y)$ — với dữ liệu 1 chiều chỉ là
  $\max C-\min C$ (độ dài đoạn thẳng chứa cụm). DIANA dùng đại lượng này.
- **Độ bất tương đồng trung bình** của điểm $i$ tới nhóm $G$:
  $D(i,G)=\frac{1}{|G|}\sum_{g\in G} d(i,g)$ (nếu $i\in G$ thì bỏ $i$ khỏi tổng).

**Quy ước độ cao** (rất quan trọng, đây là nguồn gốc mọi nhầm lẫn):

| Cây | Độ cao của nút |
|---|---|
| AGNES (bottom-up Ward) | $\Delta$WCSS tại lần **gộp** |
| DIANA | **đường kính** của cụm bị tách |

Lá luôn có độ cao 0.

---

## 4. Thuật toán ① — AGNES (cell 05–06, 09)

### 4.1. Ý tưởng

AGNES (*AGglomerative NESting*) bắt đầu với $n$ cụm đơn. Mỗi bước gộp hai cụm có $\Delta$ Ward
nhỏ nhất (gộp "rẻ" nhất), lặp đến khi còn một cụm. Kết quả là một cây nhị phân (dendrogram).

### 4.2. Giả mã

```
clusters = [ {A}, {B}, {C}, {D}, {E} ]
while len(clusters) > 1:
    tìm (U, V) sao cho Δ(U, V) nhỏ nhất
    clusters.remove(U); clusters.remove(V)
    clusters.append(U ∪ V)
    ghi lại: độ cao = Δ(U, V), hai con = (U, V)
```

### 4.3. Chạy trên dữ liệu

Bảng Δ của 10 cặp ban đầu (cell 04): `A-B 0.5`, `D-E 0.5`, `B-C 2.0`, `A-C 4.5`, `C-D 8.0`,
`C-E 12.5`, `B-D 18.0`, `A-D 24.5`, `B-E 24.5`, `A-E 32.0`.

| Bước | Gộp | $\Delta$WCSS | Tâm cụm mới | WCSS cụm mới |
|---|---|---|---|---|
| 1 | $A+B \to \{A,B\}$ | 0.500 | 1.50 | 0.500 |
| 2 | $D+E \to \{D,E\}$ | 0.500 | 8.50 | 0.500 |
| 3 | $\{A,B\}+C \to \{A,B,C\}$ | 4.167 | 2.33 | 4.667 |
| 4 | $\{A,B,C\}+\{D,E\} \to$ cả 5 điểm | 45.633 | 4.80 | 50.800 |

Tie-break: bước 1 có hai cặp cùng $\Delta = 0.5$; notebook chọn cặp có chỉ số nhỏ hơn
(`{A,B}` trước `{D,E}`). Với dữ liệu này, thứ tự đó không làm đổi cây cuối.

### 4.4. Cài đặt

- `ward_delta(a, b)` — công thức Lance–Williams ở mục 3.
- `active` là danh sách `Node` đang hoạt động; mỗi vòng duyệt mọi cặp bằng `combinations`,
  chọn khoá `(round(delta, 10), a.members, b.members)` để tie-break ổn định.
- Cụm mới: `Node(members=hợp, height=delta, left=..., right=...)`.
- Tổng số cặp đã xét: $10 + 6 + 3 + 1 = 20$.

### 4.5. Độ phức tạp và nhận xét

- Ngây thơ: $O(n^3)$ (mỗi bước $O(n^2)$ cặp, $n$ bước). Có thể giảm còn $O(n^2\log n)$
  bằng priority queue.
- Ưu: không cần chọn $k$ trước; mỗi bước là quyết định tối ưu cục bộ rẻ nhất.
- Nhược: đã gộp sai thì **không sửa được**; nhạy với tie-break khi dữ liệu thưa.

---

## 5. Thuật toán ② — DIANA (cell 10–12)

DIANA (*DIvisive ANAlysis*, Kaufman & Rousseeuw) là thuật toán tách từ trên xuống cổ điển.
Nó khác AGNES ở **hướng đi**, **cách chọn nhóm để tách** và **thước đo độ cao**.

### 5.1. Tách một cụm — "splinter group"

1. Chọn **hạt giống**: điểm có độ bất tương đồng trung bình lớn nhất với các điểm còn lại
   trong cụm. Nhóm mới $S$ (splinter) = {hạt giống}, nhóm cũ $G$ = phần còn lại.
2. Với mỗi $j \in G$, tính:

$$D(j,S)=\frac{1}{|S|}\sum_{s\in S}d(j,s),\qquad
D(j,G)=\frac{1}{|G|-1}\sum_{g\in G,\,g\ne j}d(j,g)$$

3. Chọn $j$ có $D(j,G)-D(j,S)$ **lớn nhất**. Nếu hiệu này $>0$ (tức $j$ gần nhóm mới hơn
   nhóm cũ) thì chuyển $j$ sang $S$ và lặp lại bước 2.
4. Dừng khi hiệu lớn nhất $\le 0$ (hoặc $G$ chỉ còn 1 điểm).

> Một số tài liệu viết tiêu chí là $D(j,S)-D(j,G)$ và chọn **giá trị nhỏ nhất** — hoàn toàn
> tương đương với cách viết ở trên. Điều cốt lõi: chỉ chuyển điểm **gần nhóm mới hơn**.

### 5.2. Chọn cụm để tách tiếp

Chọn cụm có **đường kính** lớn nhất. Cụm bị tách có độ cao = đường kính của nó.

### 5.3. Chạy trên dữ liệu

Độ bất tương đồng trung bình trong cả 5 điểm: `A 4.75`, `B 4.00`, `C 3.50`, `D 4.50`, `E 5.25`
⇒ hạt giống là **E**.

**Lần 1** — tách $\{A,B,C,D,E\}$ (đường kính 8), $S=\{E\}$:

| Điểm $j$ | $D(j,S)$ | $D(j,G)$ | Hiệu $D(j,G)-D(j,S)$ | Hành động |
|---|---|---|---|---|
| D | 1.00 | $(7+6+4)/3 = 5.67$ | **+4.67** | chuyển sang $S$ |
| C | — | — | — | tính lại sau khi chuyển D |
| A | — | — | — | tính lại sau khi chuyển D |
| B | — | — | — | tính lại sau khi chuyển D |

Sau khi $S=\{D,E\}$, $G=\{A,B,C\}$:

| Điểm $j$ | $D(j,S)$ | $D(j,G)$ | Hiệu | Hành động |
|---|---|---|---|---|
| A | $(7+8)/2=7.50$ | $(1+3)/2=2.00$ | −5.50 | dừng |
| B | $(6+7)/2=6.50$ | $(1+2)/2=1.50$ | −5.00 | dừng |
| C | $(4+5)/2=4.50$ | $(3+2)/2=2.50$ | −2.00 | dừng |

Kết quả: $\{A,B,C\} \mid \{D,E\}$, độ cao = 8.

**Lần 2** — tách $\{A,B,C\}$ (đường kính 3): trung bình là `A 2.0`, `B 1.5`, `C 2.5`
⇒ hạt giống **C**; $A$: $3$ vs $1$ (−2), $B$: $2$ vs $1$ (−1) ⇒ dừng.
Kết quả $\{A,B\} \mid \{C\}$, độ cao = 3.

**Lần 3, 4** — $\{A,B\}$ và $\{D,E\}$ (cùng đường kính 1) ⇒ tách $A|B$ và $D|E$, độ cao = 1.

Tóm lại đúng như hình minh hoạ:

| Lần | Cụm bị tách | Hai cụm con | Độ cao (đường kính) |
|---|---|---|---|
| 1 | $\{A,B,C,D,E\}$ | $\{A,B,C\}$ và $\{D,E\}$ | 8 |
| 2 | $\{A,B,C\}$ | $\{A,B\}$ và $\{C\}$ | 3 |
| 3 | $\{A,B\}$ | $A$ và $B$ | 1 |
| 4 | $\{D,E\}$ | $D$ và $E$ | 1 |

### 5.4. Cài đặt

- `diameter(members)` — max $d$ trong cụm (1 chiều: $\max-\min$).
- `avg_dissim(i, group)` — $D(i,G)$.
- `diana_split(members)` — trả về `(splinter, rest, seed, moves)`; `moves` ghi lại các điểm
  bị chuyển để in giải thích.
- Vòng lặp chọn `max` theo khoá `(diameter, size, -min_member)` rồi tách.
- `draw_box_tree` vẽ cây dạng hộp: lá xếp cùng hàng dưới, nhãn `lần k · độ cao h`,
  tô xanh hai cụm tại mức cắt $k=2$.

### 5.5. Độ phức tạp và nhận xét

- Mỗi lần tách: $O(m^2)$ để tính các khoảng cách trung bình ⇒ tổng xấu nhất $O(n^3)$,
  nhưng chỉ dùng ma trận khoảng cách, không cần duyệt mọi cách chia.
- Ưu: không cần chọn $k$; bước đầu nhìn toàn cục; chi phí thực tế thấp.
- Nhược: vẫn là tham lam, tách sai không sửa được; kết quả phụ thuộc tie-break khi
  đường kính bằng nhau.

---

## 6. Cắt cây để lấy $k$ cụm (`cut_tree`, cell 08)

```
groups = [root]
while len(groups) < k:
    chọn nút có độ cao LỚN NHẤT trong các nút chưa phải lá
    thay nút đó bằng hai con của nó
```

Cách này dùng được cho cả hai cây vì độ cao luôn không giảm từ lá lên gốc (đã kiểm tra bằng
`is_monotone`).

**Ngưỡng cắt nào là "đúng"?** Không có ngưỡng duy nhất. Với cây AGNES ở bài này, mức gộp lớn
nhất *bên dưới* gốc là 4.17 và lần gộp cuối là 45.63, nên **mọi** $h \in (4.17;\;45.63)$ đều
cho đúng 2 cụm $\{A,B,C\}$ và $\{D,E\}$. Hình `ward_so_sanh.png` vì vậy tô cả vùng đó
(thay vì chọn một con số cụ thể như 13.8 — con số đó chỉ là trung bình nhân của hai biên,
không mang ý nghĩa gì đặc biệt). Với DIANA, ngưỡng nằm giữa 3 và 8 (theo thang đường kính).

Kết quả (cell 14):

| $k$ | Cụm |
|---|---|
| 1 | $\{A,B,C,D,E\}$ |
| 2 | $\{A,B,C\}$, $\{D,E\}$ |
| 3 | $\{A,B\}$, $\{C\}$, $\{D,E\}$ |
| 4 | $\{A,B\}$, $\{C\}$, $\{D\}$, $\{E\}$ |
| 5 | từng điểm |

---

## 7. Kiểm chứng trong notebook

| Kiểm chứng | Cách làm | Kết quả |
|---|---|---|
| Công thức Ward | so `ward_delta` với `WCSS(A∪B)-WCSS(A)-WCSS(B)` cho cả 10 cặp | khớp đến $10^{-12}$ |
| Hai cách cho cùng cây | `assert` kết quả cắt $k=2$ của hai cây | đều là `((0,1,2),(3,4))` |
| Tối ưu toàn cục | duyệt đủ 15 cách chia 2 cụm, lấy min WCSS | 5.1667 = WCSS của cách cắt cây |
| Cây hợp lệ | `is_monotone` kiểm tra độ cao không giảm từ lá lên gốc | cả hai cây hợp lệ |
| Độ cao cùng nút | `find_node` tìm nút theo tập điểm | 45.63 / 8; 4.17 / 3; 0.50 / 1 … |

---

## 8. So sánh AGNES và DIANA

| Tiêu chí | AGNES (bottom-up Ward) | DIANA (top-down) |
|---|---|---|
| Hướng | gộp dần | tách dần |
| Xuất phát | $n$ cụm đơn | 1 cụm chứa tất cả |
| Tiêu chí mỗi bước | $\Delta$WCSS nhỏ nhất | splinter group (gần nhóm mới hơn) |
| Chọn cụm để xử lý | mọi cặp cụm | cụm có đường kính lớn nhất |
| Độ cao nút | $\Delta$WCSS | đường kính cụm |
| Độ cao ở bài này | 45.63 / 4.17 / 0.50 | 8 / 3 / 1 |
| Chi phí | $O(n^3)$ ngây thơ | $O(n^3)$ xấu nhất, thực tế nhanh |
| Kết quả ở bài này | $\{A,B,C\}$ và $\{D,E\}$ | giống |

**Vì sao topology giống nhau?** Cấu trúc cây do dữ liệu quyết định: khoảng trống $C\to D=4$
áp đảo các khoảng còn lại, nên cả hai hướng đều để $\{A,B,C\}$ tách khỏi $\{D,E\}$ sau cùng.
Khác nhau chỉ ở **thang đo độ cao**, nên không được so sánh trực tiếp 8 với 45.63.

---

## 9. Hình ảnh sinh ra

| File | Cell | Nội dung |
|---|---|---|
| `figures/ward_bottom_up.png` | 09 | Trục số tô màu 2 cụm + dendrogram AGNES |
| `figures/diana_tree.png` | 12 | Cây hộp DIANA (8/3/1/1) đúng kiểu hình minh hoạ |
| `figures/ward_so_sanh.png` | 15 | 2 cây cạnh nhau: cùng topology, hai thang đo |

---

## 10. Chạy và tái tạo

```powershell
# cài môi trường
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt

# chạy toàn bộ notebook, ghi đè output
.\.venv\Scripts\python.exe -m jupyter nbconvert --to notebook --execute --inplace ward_clustering_1d.ipynb
```

Hoặc mở `ward_clustering_1d.ipynb` bằng VS Code / Jupyter rồi **Run All**.
Notebook không có phụ thuộc ẩn: chỉ cần `numpy`, `matplotlib`, `ipykernel`.

---

## 11. Mở rộng và hạn chế

- **Sang 2 chiều**: thay $\lvert x-y\rvert$ bằng khoảng cách Euclid. `ward_delta` chuyển thành
  $\frac{|A||B|}{|A|+|B|}\lVert \bar{\mathbf x}_A-\bar{\mathbf x}_B\rVert^2$; DIANA dùng ma
  trận khoảng cách nên gần như không phải sửa gì.
- **Chi phí**: với $n$ lớn, dùng AGNES + priority queue hoặc DIANA.
- **Tie-break**: khi nhiều phương án cùng chi phí/đường kính, notebook chọn theo chỉ số nhỏ
  (đã ghi chú trong từng mục). Kết quả trên dữ liệu này không phụ thuộc tie-break.
- **Hạn chế chung của phân cụm phân cấp**: quyết định tham lam không sửa được; dữ liệu có
  nhiễu hoặc cụm lồng nhau có thể cho cây "đẹp" nhưng không tối ưu.
