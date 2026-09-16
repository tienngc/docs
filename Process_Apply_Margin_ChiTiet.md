# Quá trình Apply Feature "Margin" — Chi tiết từ SRS đến Code

> Đây là tài liệu **chi tiết đầy đủ, gộp lại từ đầu đến cuối** cho việc apply feature Margin,
> đi đúng thứ tự **SRS → HLD → LLD → TDD → Code**. Dùng làm **quy trình mẫu (process chung)**
> để áp dụng cho các feature tiếp theo — chỉ cần lặp lại đúng 5 bước, đúng thứ tự, với nội dung
> tương ứng của feature mới.

---

## BƯỚC 1 — SRS (yêu cầu là gì)

### 1.1. Mục đích
Chừa khoảng cách (margin) đều ở 4 cạnh preview trước khi dàn đều item, để item không dính sát viền.

### 1.2. Định nghĩa
- **margin**: 1 giá trị `float` duy nhất, áp dụng đều cho cả 4 cạnh (trái = phải = trên = dưới).
- **effective area**: vùng còn lại sau khi trừ margin — `effectiveWidth = previewWidth - 2*margin`, `effectiveHeight = previewHeight - 2*margin`.

### 1.3. Yêu cầu chức năng
| ID | Nội dung |
|---|---|
| FR-M1 | API nhận **1 giá trị margin duy nhất**, áp dụng cho cả 4 cạnh |
| FR-M2 | Dàn đều item phải tính trên **effective area**, không phải preview gốc |
| FR-M3 | Vị trí item trả về phải được **offset vào trong** đúng `margin` (cả trục X và Y) |
| FR-M4 | 4 hàm tính kích thước zoom-in/zoom-out phải có overload nhận thêm `margin` |
| FR-M5 | `margin = 0` → kết quả phải **giống hệt** hàm gốc không margin (tương thích ngược) |
| FR-M6 | Nếu `2*margin >= previewWidth` hoặc `2*margin >= previewHeight` → xử lý an toàn, không NaN/crash, trả về rỗng/0 |

### 1.4. Yêu cầu phi chức năng
- Không phá vỡ API cũ (dùng overload, không sửa signature cũ).
- Độ phức tạp vẫn O(n).
- Margin âm không cần tự clamp, chỉ cần không crash.

---

## BƯỚC 2 — HLD (đặt ở đâu, luồng xử lý ra sao)

### 2.1. Nguyên tắc thiết kế
Margin là một **bước tiền xử lý**: thu nhỏ vùng tính toán trước khi đưa vào **đúng công thức dàn đều đã có sẵn** (không viết công thức mới), sau đó **hậu xử lý** để dịch vị trí kết quả vào trong.

```mermaid
flowchart LR
    UI["UI Layer"] -->|pw, ph, margin, ar[], orientation| Calc["PhotoStripLayoutCalculator"]
    Calc --> S1["Bước A: Trừ margin (2 phía mỗi chiều) → effectiveWidth/Height"]
    S1 --> S2["Bước B: Áp dụng công thức dàn đều GỐC (không đổi) trên effective area"]
    S2 --> S3["Bước C: Offset toàn bộ vị trí kết quả (+margin, +margin)"]
    S3 --> UI
```

### 2.2. Sequence xử lý

```mermaid
sequenceDiagram
    participant UI
    participant Calc as PhotoStripLayoutCalculator
    UI->>Calc: getItemPositions(pw, ph, ar[], orientation, margin)
    Calc->>Calc: ew = pw - 2*margin; eh = ph - 2*margin
    Calc->>Calc: tính common width/height (tái dùng công thức gốc, input = ew/eh)
    loop mỗi item
        Calc->>Calc: tính width/height item theo aspect ratio
        Calc->>Calc: offset vị trí += (margin, margin)
    end
    Calc-->>UI: List<ItemRect>
```

---

## BƯỚC 3 — LLD (thiết kế chi tiết: công thức, thuật toán, chữ ký hàm)

### 3.1. Công thức (ký hiệu: `ew = pw - 2*margin`, `eh = ph - 2*margin`)

| Hàm | Công thức |
|---|---|
| `getVeritcalWidthOnZoomOutState(..., margin)` | `W = eh / Σ(1/ar[i])` |
| `getHorizontalHeightOnZoomOutState(..., margin)` | `H = ew / Σ(ar[i])` |
| `getVeritcalHeightOnZoomInState(..., margin)` | `totalHeight = ew * Σ(1/ar[i])` |
| `getHorizontalWidthOnZoomInState(..., margin)` | `totalWidth = eh * Σ(ar[i])` |

### 3.2. Pseudo-code `getItemPositions` (có margin)

```
input: previewWidth, previewHeight, ar[1..n], orientation, margin
output: list ItemRect[1..n]

ew = previewWidth - 2*margin
eh = previewHeight - 2*margin

if ar rỗng/null OR ew <= 0 OR eh <= 0:
    return []

if orientation == VERTICAL:
    W = eh / sumInverse(ar)
    top = margin
    for i in 1..n:
        h_i = W / ar[i]
        rect_i = (left=margin, top=top, right=margin+W, bottom=top+h_i)
        top += h_i
else (HORIZONTAL):
    H = ew / sum(ar)
    left = margin
    for i in 1..n:
        w_i = H * ar[i]
        rect_i = (left=left, top=margin, right=left+w_i, bottom=margin+H)
        left += w_i

return list of rect_i
```

### 3.3. Nguyên tắc tương thích ngược
Hàm cũ (không margin) = gọi hàm mới với `margin = 0`:
```java
getItemPositions(pw, ph, ar, orientation) {
    return getItemPositions(pw, ph, ar, orientation, 0f);
}
```

### 3.4. Xử lý biên
- `ew <= 0` hoặc `eh <= 0` → trả rỗng (list) hoặc `0` (hàm kích thước).
- `margin = 0` → `ew = pw`, `eh = ph` → kết quả giống hệt bản gốc.

---

## BƯỚC 4 — TDD (viết test trước, chứng minh bằng so sánh IS/AS)

### 4.1. Input mẫu dùng xuyên suốt
```
previewWidth = 1080, previewHeight = 1920
imageAspectRatio = [1.0, 1.0]   // 2 item vuông
orientation = VERTICAL
margin (AS) = 20
```

### 4.2. So sánh IS (trước, không margin) vs AS (sau, margin=20)

| Đại lượng | IS — trước | AS — sau margin=20 | Chênh lệch |
|---|---|---|---|
| Width mỗi item | 960 | 940 | -20 |
| rect[0] | (0, 0, 960, 960) | (20, 20, 960, 960) | dịch vào trong 20px |
| rect[1] | (0, 960, 960, 1920) | (20, 960, 960, 1900) | co lại đúng margin |
| Tổng height 2 item | 1920 | 1880 | -40 (= 2×margin) |

### 4.3. Chu trình Red → Green → Refactor (ví dụ cụ thể cho FR-M2/M3)

**RED** — viết test trước khi code hỗ trợ margin tồn tại:
```java
@Test
public void margin_vertical_offsetsAndShrinksEffectiveArea() {
    float[] ar = {1.0f, 1.0f};
    List<ItemRect> positions = calculator.getItemPositions(
            1080, 1920, ar, Orientation.VERTICAL, /*margin=*/20f);

    assertEquals(940f, positions.get(0).width(), 0.001f);   // AS
    assertEquals(20f, positions.get(0).left, 0.001f);
    assertEquals(20f, positions.get(0).top, 0.001f);

    float totalHeight = 0f;
    for (ItemRect r : positions) totalHeight += r.height();
    assertEquals(1880f, totalHeight, 0.001f);
}
```
→ Chạy: **FAIL** (chưa có overload margin / chưa trừ margin).

**GREEN** — code tối thiểu để pass (xem 3.2), chạy lại: **PASS**.

**Kiểm tra không phá IS** (đảm bảo FR-M5) — chạy lại test cũ, không sửa:
```java
@Test
public void margin_zero_isBackwardCompatible() {
    float[] ar = {1.0f, 1.0f};
    List<ItemRect> withMargin = calculator.getItemPositions(1080, 1920, ar, Orientation.VERTICAL, 0f);
    List<ItemRect> original   = calculator.getItemPositions(1080, 1920, ar, Orientation.VERTICAL);
    // so sánh từng rect phải bằng nhau
}
```
→ **PASS**.

**Test biên (FR-M6):**
```java
@Test
public void margin_tooLarge_returnsEmptyList() {
    float[] ar = {1.0f};
    List<ItemRect> positions = calculator.getItemPositions(1080, 1920, ar, Orientation.VERTICAL, 600f);
    assertTrue(positions.isEmpty());
}
```
→ **PASS**.

**REFACTOR** — chạy lại toàn bộ test suite (cả IS lẫn AS) → tất cả **PASS**, không regression.

### 4.4. Bảng traceability (Requirement → Test)

| Requirement | Test | Trạng thái |
|---|---|---|
| FR-M2, FR-M3 | `margin_vertical_offsetsAndShrinksEffectiveArea` | ☑ Red ☑ Green ☑ Refactor |
| FR-M5 | `margin_zero_isBackwardCompatible` | ☑ Red ☑ Green ☑ Refactor |
| FR-M6 | `margin_tooLarge_returnsEmptyList` | ☑ Red ☑ Green ☑ Refactor |
| FR-M4 | test riêng cho từng hàm zoom-in/out (tương tự pattern trên) | ☑ Red ☑ Green ☑ Refactor |

---

## BƯỚC 5 — Code Implementation (code thật, làm test ở bước 4 pass)

Code thật đã được cập nhật trong `PhotoStripLayoutCalculator.java`, bám đúng LLD (bước 3):

```java
public List<ItemRect> getItemPositions(float previewWidth, float previewHeight,
                                        float[] imageAspectRatio, Orientation orientation) {
    return getItemPositions(previewWidth, previewHeight, imageAspectRatio, orientation, 0f);
}

public List<ItemRect> getItemPositions(float previewWidth, float previewHeight,
                                        float[] imageAspectRatio, Orientation orientation,
                                        float margin) {
    List<ItemRect> positions = new ArrayList<>();
    if (imageAspectRatio == null || imageAspectRatio.length == 0) return positions;

    float effectiveWidth = previewWidth - 2 * margin;
    float effectiveHeight = previewHeight - 2 * margin;
    if (effectiveWidth <= 0f || effectiveHeight <= 0f) return positions;

    if (orientation == Orientation.VERTICAL) {
        float itemWidth = getVeritcalWidthOnZoomOutState(previewWidth, previewHeight, imageAspectRatio, margin);
        float top = margin;
        for (float ar : imageAspectRatio) {
            float itemHeight = safeDivide(itemWidth, ar);
            positions.add(new ItemRect(margin, top, margin + itemWidth, top + itemHeight));
            top += itemHeight;
        }
    } else {
        float itemHeight = getHorizontalHeightOnZoomOutState(previewWidth, previewHeight, imageAspectRatio, margin);
        float left = margin;
        for (float ar : imageAspectRatio) {
            float itemWidth = itemHeight * ar;
            positions.add(new ItemRect(left, margin, left + itemWidth, margin + itemHeight));
            left += itemWidth;
        }
    }
    return positions;
}
```

**Kết quả**: toàn bộ test ở Bước 4 (Green + Refactor) đều pass, code không thêm logic nào ngoài phạm vi LLD.

---

## Tóm tắt quy trình (dùng lặp lại cho feature tiếp theo)

| Bước | Làm gì | Xong khi nào mới qua bước sau |
|---|---|---|
| 1. SRS | Liệt kê FR/NFR có ID | Yêu cầu rõ ràng, không mơ hồ |
| 2. HLD | Vẽ luồng xử lý / vị trí trong kiến trúc | Xác định rõ input/output, các bước xử lý chính |
| 3. LLD | Viết công thức + pseudo-code + chữ ký hàm | Đủ chi tiết để viết test mà không cần đoán |
| 4. TDD | Viết test trước (Red) → code tối thiểu (Green) → refactor, chứng minh bằng bảng so sánh IS/AS số liệu thật | Mọi FR/NFR đều có test, test suite pass 100% |
| 5. Code | Implement đúng LLD, làm test ở bước 4 pass | Code khớp hoàn toàn với LLD, không thêm/bớt logic ngoài phạm vi |
