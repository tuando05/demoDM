# Tài liệu: Phân cụm phân cấp — Ward (2 hướng) & DIANA

Tài liệu này mô tả chi tiết file [`ward_clustering_1d.ipynb`](ward_clustering_1d.ipynb): bố cục
từng cell, các hàm, ba thuật toán được cài đặt và kết quả kiểm chứng. Mọi con số dưới đây là
output thật của notebook.

---

## 0. Tóm tắt

| Hạng mục | Giá trị |
|---|---|
| Dữ liệu | $A=1,\;B=2,\;C=4,\;D=8,\;E=9$ (1 chiều) |
| Khoảng cách | $d(x,y)=\lvert x-y\rvert$ |
| Thuật toán | ① bottom-up Ward, ② divisive Ward, ③ DIANA |
| Cắt cây tại $k=2$ | $\{A,B,C\}$ và $\{D,E\}$ — cả ba cách đều cho cùng kết quả |
| WCSS tại $k=2$ | $5.1667$ = cực tiểu toàn cục (duyệt đủ 15 cách chia) |
| Công cụ | `numpy` + `matplotlib` (không dùng `scipy`/`sklearn`) |
| Số cell | 21 (10 markdown + 11 code) |

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
(1, 1, 2, 1). Đây là lý do mọi phương pháp đều ưu tiên tách $\{A,B,C\}$ khỏi $\{D,E\}$.

---

## 2. Bố cục notebook (21 cell)

| Cell | Loại | Nội dung | Output chính |
|---|---|---|---|
| 00 | md | Đề bài, bảng toạ độ, ma trận khoảng cách, yêu cầu | — |
| 01 | md | Tiêu chí Ward: WCSS, ΔWCSS, bảng so sánh 2 hướng | — |
| 02 | code | Import, khai báo `LABELS`, `X`, `N`, ma trận `D` | In ma trận khoảng cách |
| 03 | md | Giới thiệu `Node` và các hàm dùng chung | — |
| 04 | code | `fmt_set`, `wcss`, `mean_of`, `ward_delta`, `Node` + kiểm chứng công thức | Bảng ΔWCSS 10 cặp, WCSS tổng = 50.8 |
| 05 | md | Ý tưởng, giả mã bottom-up, ghi chú tie-break, chi phí | — |
| 06 | code | Vòng lặp gộp → `agg_root`, `agg_steps` | Bảng 4 bước gộp |
| 07 | md | Giới thiệu công cụ vẽ | — |
| 08 | code | `draw_axis`, `draw_dendrogram`, `cut_tree`, `is_monotone` | — |
| 09 | code | Vẽ trục số + dendrogram bottom-up | `figures/ward_bottom_up.png` |
| 10 | md | Divisive Ward: giả mã, tie-break, chi phí $O(2^n)$ | — |
| 11 | code | `bipartitions`, `best_split`, vòng lặp tách → `top_root` | Bảng 4 bước tách |
| 12 | code | Vẽ trục số + dendrogram divisive Ward | `figures/ward_top_down.png` |
| 13 | md | DIANA: splinter group, đường kính, bảng độ cao 8/3/1/1 | — |
| 14 | code | `diameter`, `avg_dissim`, `diana_split`, vòng lặp tách → `diana_root` | Bảng 4 lần tách |
| 15 | code | `draw_box_tree` + vẽ cây hộp kiểu DIANA | `figures/diana_tree.png` |
| 16 | md | Mục tiêu so sánh ba cây | — |
| 17 | code | Bảng cắt $k=1..5$, `find_node`, so sánh độ cao, kiểm chứng | Bảng so sánh + assert |
| 18 | code | Vẽ 3 cây cạnh nhau | `figures/ward_so_sanh.png` |
| 19 | md | Kết luận: vì sao giống nhau, cách trình bày, chi phí | — |
| 20 | md | Hướng dẫn chạy | — |

---

## 3. Ký hiệu và công thức nền

Với cụm $C$ (tập chỉ số điểm):

$$\mathrm{WCSS}(C)=\sum_{x\in C}\left(x-\bar x_C\right)^2,
\qquad \bar x_C=\frac{1}{|C|}\sum_{x\in C}x$$

**Phương sai tăng thêm khi gộp** (chính là khoảng cách Ward):

$$\Delta(A,B)=\mathrm{WCSS}(A\cup B)-\mathrm{WCSS}(A)-\mathrm{WCSS}(B)
=\frac{|A|\,|B|}{|A|+|B|}\left(\bar x_A-\bar x_B\right)^2\;\ge 0$$

Hai đại lượng đo "độ to" của cụm, dùng ở các thuật toán khác nhau:

- **Đường kính**: $\mathrm{diam}(C)=\max_{x,y\in C} d(x,y)$ — với dữ liệu 1 chiều chỉ là
  $\max C-\min C$ (độ dài đoạn thẳng chứa cụm).
- **Độ bất tương đồng trung bình** của điểm $i$ tới nhóm $G$:
  $D(i,G)=\frac{1}{|G|}\sum_{g\in G} d(i,g)$ (nếu $i\in G$ thì bỏ $i$ khỏi tổng).

**Quy ước độ cao** (rất quan trọng, đây là nguồn gốc mọi nhầm lẫn):

| Cây | Độ cao của nút |
|---|---|
| Bottom-up Ward | $\Delta$WCSS tại lần **gộp** |
| Divisive Ward | $\Delta$WCSS tại lần **tách** (cùng đơn vị với bottom-up) |
| DIANA | **đường kính** của cụm bị tách |

Lá luôn có độ cao 0.

---

## 4. Thuật toán ① — Bottom-up Ward (cell 05–06, 09)

### 4.1. Ý tưởng

Bắt đầu với $n$ cụm đơn. Mỗi bước gộp hai cụm có $\Delta$ Ward nhỏ nhất (gộp "rẻ" nhất),
lặp đến khi còn một cụm. Kết quả là một cây nhị phân (dendrogram).

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

## 5. Thuật toán ② — Divisive Ward (cell 10–12)

### 5.1. Ý tưởng

Bắt đầu với một cụm chứa tất cả. Mỗi bước chọn **một cụm** và tách thành hai phần sao cho
tổng WCSS sau khi tách nhỏ nhất — tức **giảm WCSS nhiều nhất**, vẫn đúng tinh thần Ward.

### 5.2. Giả mã

```
clusters = [ {A,B,C,D,E} ]
while còn cụm có ≥ 2 điểm:
    với mỗi cụm U (|U| ≥ 2):
        duyệt MỌI cách chia U = V ∪ W        (2^(|U|-1) - 1 cách)
        giữ cách có WCSS(V) + WCSS(W) nhỏ nhất  ⇒ gain(U) lớn nhất
    chọn cụm U* có gain lớn nhất
    tách U* theo cách tốt nhất
    độ cao(U*) = gain(U*) = WCSS(U*) - WCSS(V) - WCSS(W)
```

`bipartitions(members)` sinh các cách chia bằng bitmask, cố định bit 0 = 0 để mỗi cách chia
chỉ xuất hiện một lần (tránh trùng cặp bù $V|W$ và $W|V$).

### 5.3. Chạy trên dữ liệu

| Bước | Tách | $\Delta$WCSS | WCSS trước → sau | Số cách chia đã xét |
|---|---|---|---|---|
| 1 | $\{A,B,C,D,E\} \to \{A,B,C\} \mid \{D,E\}$ | 45.633 | 50.800 → 5.167 | 15 |
| 2 | $\{A,B,C\} \to \{A,B\} \mid C$ | 4.167 | 4.667 → 0.500 | 3 |
| 3 | $\{A,B\} \to A \mid B$ | 0.500 | 0.500 → 0.000 | 1 |
| 4 | $\{D,E\} \to D \mid E$ | 0.500 | 0.500 → 0.000 | 1 |

Tie-break khi nhiều cụm cùng gain: ưu tiên cụm lớn hơn, rồi cụm có chỉ số nhỏ hơn.
Thứ tự tách các nhánh độc lập không ảnh hưởng cây cuối.

### 5.4. Độ phức tạp và nhận xét

- Duyệt đủ cách chia: $2^{n-1}-1$ ở lần tách gốc ⇒ $O(2^n)$ — chỉ hợp với $n$ nhỏ
  (bài này $n=5$: 15 cách, tổng 20 cách cho cả 4 bước).
- Ưu: nhìn được "bức tranh lớn" trước, các quyết định đầu dựa trên toàn bộ dữ liệu.
- Nhược: đã tách sai thì không sửa được; chi phí bùng nổ khi $n$ tăng.

---

## 6. Thuật toán ③ — DIANA (cell 13–15)

DIANA (*DIvisive ANAlysis*, Kaufman & Rousseeuw) là bản top-down cổ điển, khác divisive Ward ở
**cách tách** và **cách chọn cụm**, và dùng **đường kính** làm độ cao.

### 6.1. Tách một cụm — "splinter group"

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

### 6.2. Chọn cụm để tách tiếp

Chọn cụm có **đường kính** lớn nhất. Cụm bị tách có độ cao = đường kính của nó.

### 6.3. Chạy trên dữ liệu

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

### 6.4. Cài đặt

- `diameter(members)` — max $d$ trong cụm (1 chiều: $\max-\min$).
- `avg_dissim(i, group)` — $D(i,G)$.
- `diana_split(members)` — trả về `(splinter, rest, seed, moves)`; `moves` ghi lại các điểm
  bị chuyển để in giải thích.
- Vòng lặp chọn `max` theo khoá `(diameter, size, -min_member)` rồi tách.
- `draw_box_tree` vẽ cây dạng hộp: lá xếp cùng hàng dưới, nhãn `lần k · độ cao h`,
  tô xanh hai cụm tại mức cắt $k=2$.

### 6.5. Độ phức tạp và nhận xét

- Mỗi lần tách: $O(m^2)$ để tính các khoảng cách trung bình ⇒ tổng xấu nhất $O(n^3)$,
  nhưng **không duyệt $2^n$ cách chia** như divisive Ward — dùng được với $n$ lớn hơn nhiều.
- Ưu: không cần chọn $k$; bước đầu nhìn toàn cục; chi phí thực tế thấp.
- Nhược: vẫn là tham lam, tách sai không sửa được; kết quả phụ thuộc tie-break khi
  đường kính bằng nhau.

---

## 7. Cắt cây để lấy $k$ cụm (`cut_tree`, cell 08)

```
groups = [root]
while len(groups) < k:
    chọn nút có độ cao LỚN NHẤT trong các nút chưa phải lá
    thay nút đó bằng hai con của nó
```

Cách này dùng được cho cả ba cây vì độ cao luôn không giảm từ lá lên gốc (đã kiểm tra bằng
`is_monotone`).

**Ngưỡng cắt nào là "đúng"?** Không có ngưỡng duy nhất. Với cây Ward ở bài này, mức gộp lớn
nhất *bên dưới* gốc là 4.17 và lần gộp cuối là 45.63, nên **mọi** $h \in (4.17;\;45.63)$ đều
cho đúng 2 cụm $\{A,B,C\}$ và $\{D,E\}$. Hình `ward_so_sanh.png` vì vậy tô cả vùng đó
(thay vì chọn một con số cụ thể như 13.8 — con số đó chỉ là trung bình nhân của hai biên,
không mang ý nghĩa gì đặc biệt). Với DIANA, ngưỡng nằm giữa 3 và 8 (theo thang đường kính).

Kết quả (cell 17):

| $k$ | Cụm |
|---|---|
| 1 | $\{A,B,C,D,E\}$ |
| 2 | $\{A,B,C\}$, $\{D,E\}$ |
| 3 | $\{A,B\}$, $\{C\}$, $\{D,E\}$ |
| 4 | $\{A,B\}$, $\{C\}$, $\{D\}$, $\{E\}$ |
| 5 | từng điểm |

---

## 8. Kiểm chứng trong notebook

| Kiểm chứng | Cách làm | Kết quả |
|---|---|---|
| Công thức Ward | so `ward_delta` với `WCSS(A∪B)-WCSS(A)-WCSS(B)` cho cả 10 cặp | khớp đến $10^{-12}$ |
| Ba cách cho cùng cây | `assert` kết quả cắt $k=2$ của ba cây | đều là `((0,1,2),(3,4))` |
| Tối ưu toàn cục | duyệt đủ 15 cách chia 2 cụm, lấy min WCSS | 5.1667 = WCSS của cách cắt cây |
| Dendrogram hợp lệ | `is_monotone` kiểm tra độ cao không giảm từ lá lên gốc | cả ba cây hợp lệ |
| Độ cao cùng nút | `find_node` tìm nút theo tập điểm | 45.63 / 45.63 / 8 … |

---

## 9. So sánh ba phương pháp

| Tiêu chí | Bottom-up Ward | Divisive Ward | DIANA |
|---|---|---|---|
| Hướng | gộp dần | tách dần | tách dần |
| Tiêu chí mỗi bước | $\Delta$WCSS nhỏ nhất | tổng WCSS sau tách nhỏ nhất | splinter group (gần nhóm mới hơn) |
| Chọn cụm để xử lý | mọi cặp cụm | mọi cụm, gain lớn nhất | cụm có đường kính lớn nhất |
| Độ cao nút | $\Delta$WCSS | $\Delta$WCSS | đường kính cụm |
| Độ cao ở bài này | 45.63 / 4.17 / 0.50 | 45.63 / 4.17 / 0.50 | 8 / 3 / 1 |
| Chi phí | $O(n^3)$ | $O(2^n)$ (duyệt đủ) | $O(n^3)$ xấu nhất, thực tế nhanh hơn nhiều |
| Kết quả ở bài này | $\{A,B,C\}$ và $\{D,E\}$ | giống | giống |

**Vì sao topology giống nhau?** Cấu trúc cây do dữ liệu quyết định: khoảng trống $C\to D=4$
áp đảo các khoảng còn lại, nên mọi tiêu chí hợp lý đều để $\{A,B,C\}$ tách khỏi $\{D,E\}$
sau cùng. Khác nhau chỉ ở **thang đo độ cao**, nên không được so sánh trực tiếp 8 với 45.63.

---

## 10. Hình ảnh sinh ra

| File | Cell | Nội dung |
|---|---|---|
| `figures/ward_bottom_up.png` | 09 | Trục số tô màu 2 cụm + dendrogram bottom-up |
| `figures/ward_top_down.png` | 12 | Trục số + dendrogram divisive Ward |
| `figures/diana_tree.png` | 15 | Cây hộp DIANA (8/3/1/1) đúng kiểu hình minh hoạ |
| `figures/ward_so_sanh.png` | 18 | 3 cây cạnh nhau: cùng topology, ba thang đo |

---

## 11. Chạy và tái tạo

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

## 12. Mở rộng và hạn chế

- **Sang 2 chiều**: thay $\lvert x-y\rvert$ bằng khoảng cách Euclid. `ward_delta` chuyển thành
  $\frac{|A||B|}{|A|+|B|}\lVert \bar{\mathbf x}_A-\bar{\mathbf x}_B\rVert^2$; DIANA dùng ma
  trận khoảng cách nên gần như không phải sửa gì.
- **Chi phí**: với $n$ lớn, dùng bottom-up + priority queue hoặc DIANA; tránh divisive Ward
  kiểu duyệt đủ $2^n$.
- **Tie-break**: khi nhiều phương án cùng chi phí/đường kính, notebook chọn theo chỉ số nhỏ
  (đã ghi chú trong từng mục). Kết quả trên dữ liệu này không phụ thuộc tie-break.
- **Hạn chế chung của phân cụm phân cấp**: quyết định tham lam không sửa được; dữ liệu có
  nhiễu hoặc cụm lồng nhau có thể cho cây "đẹp" nhưng không tối ưu.
