# Hardware Test Matrix

Tài liệu này ghi lại các test đã thực hiện trên PLC FX3U clone/LE3U để xác định giới hạn thực tế của Structured ST/GX Works2.

> Mục tiêu là phân biệt lỗi source logic với lỗi compiler/runtime của clone.

---


# 0. Điều kiện cấu hình trước khi chạy test

Tất cả test ST trong tài liệu này phải được chạy với GX Works2 đã cấu hình:

```text
Tool
  > Device/Label Automatic-Assign Setting
    > Bit Range
      > M
        1024 to 3071
```

Mục đích là buộc GX Works2 chỉ tự động assign Local Label bit vào vùng:

```text
M1024 ... M3071
```

Nếu không giới hạn vùng này, kết quả hardware test có thể bị nhiễu bởi việc Local Label được tự động cấp vào vùng M mà PLC clone không hỗ trợ đúng.

Sau khi thay đổi setting phải **Rebuild All** trước khi chạy lại test.

---

# 1. Test ST execution cơ bản

## Code

```pascal
M100 := TRUE;
Y0 := X0;
```

## Kết quả

- `M100 = 1`
- `Y0` theo `X0`

## Kết luận

```text
Structured ST execution    PASS
Direct assignment          PASS
X/M/Y mapping              PASS
```

---

# 2. Test SET/RST bọc trong IF

## Code

```pascal
IF X0 THEN
    SET(TRUE, M100);
END_IF;

IF X1 THEN
    RST(TRUE, M100);
END_IF;

Y0 := M100;
```

## Kết quả

Không hoạt động đúng trên PLC đã test.

## Kết luận ban đầu

Không được dùng pattern:

```pascal
IF Condition THEN
    SET(TRUE, Device);
END_IF;
```

hoặc tương tự với RST.

---

# 3. Test Boolean latch

## Code

```pascal
M100 := (M100 OR X0) AND NOT X1;
Y0 := M100;
```

## Kết quả

PASS.

- pulse `X0` → `M100 = 1`
- thả `X0` → `M100` giữ 1
- pulse `X1` → `M100 = 0`

## Kết luận

```text
Boolean self-latch         PASS
Self reference             PASS
Single assignment          PASS
```

---

# 4. Test SET -> timer

## Code

```pascal
IF X0 THEN
    SET(TRUE, M110);
END_IF;

OUT_T(M110, TC2, K20);

M120 := M110;
M121 := TS2;
```

## Kết quả

Nếu chỉ pulse `X0` khoảng 0.5 s:

```text
M110 = 1
M120 = 1
M121 = 0
TS2  = 0
```

Nếu giữ `X0` đủ lâu:

```text
M121 = 1
TS2  = 1
```

## Ý nghĩa

`SET` có thể làm device hiển thị ON, nhưng pattern `IF X0 THEN SET(TRUE,...)` gây hành vi runtime bất thường cho phần logic phía sau.

---

# 5. Test copy sau IF + SET

## Code

```pascal
IF X0 THEN
    SET(TRUE, M110);
END_IF;

M120 := M110;
M121 := M110 AND M8000;

OUT_T(M110, TC0, K20);
OUT_T(M120, TC1, K20);
OUT_T(M121, TC2, K20);
```

## Kết quả sau pulse X0

```text
M110 = 1
M120 = 1
M121 = 1

các timer done = 0
```

## Kết luận

Lỗi không chỉ là việc timer đọc trực tiếp bit SET.

---

# 6. Test execution flow quanh IF + SET

## Code

```pascal
M130 := NOT X0;

OUT_T(M8000, TC0, K20);

IF X0 THEN
    SET(TRUE, M110);
END_IF;

M131 := NOT X0;

OUT_T(M8000, TC1, K20);

M140 := TS0;
M141 := TS1;
```

## Kết quả với X0 OFF

```text
M130 = 1
M131 = 0
TS0  = 1
TS1  = 1
```

## Ý nghĩa

- assignment trước block chạy;
- assignment sau block không cập nhật như mong đợi;
- `OUT_T(M8000,...)` sau block vẫn chạy.

Đây là bằng chứng mạnh rằng pattern `IF ... SET(TRUE,...)` làm generated runtime flow bị xử lý sai trên clone.

---

# 7. Test direct-EN SET

## Code

```pascal
M130 := NOT X0;

SET(X0, M110);

M131 := NOT X0;

OUT_T(M8000, TC0, K20);

M132 := NOT X0;
```

## Kết quả

Trước khi nhấn X0:

```text
M130 = 1
M131 = 1
M132 = 1
```

Sau pulse X0:

```text
M110 = 1
```

## Kết luận

```text
SET(EN, device)            PASS
Code after SET             PASS
Direct EN pattern          PASS
```

---

# 8. Test direct-EN SET + RST

## Code

```pascal
SET(X0, M110);
RST(X1, M110);

M120 := M110;

OUT_T(M110, TC0, K20);

M121 := TS0;
```

## Kết quả

PASS.

Trình tự đúng:

```text
pulse X0   -> M110 = 1
release X0 -> M110 giữ 1
wait >2 s  -> TS0 = 1
pulse X1   -> M110 = 0, TS0 = 0
```

## Kết luận

```text
Native SET direct EN       PASS
Native RST direct EN       PASS
SET state -> OUT_T         PASS
```

---

# 9. Test user-defined FB đơn giản

## FB_LEAF

```pascal
Alive := Enable;
OutValue := InValue + 1;
```

Input:

```text
D100 = 100
Enable = TRUE
```

Kết quả:

```text
M100 = 1
D110 = 101
M110 = 1
```

## Kết luận

```text
User-defined FB            PASS
FB input/output            PASS
Local FB instance          PASS
```

---

# 10. Test nested user FB 3 tầng

Cấu trúc:

```text
MAIN
 ↓
FB_LEVEL3
 ↓
FB_LEVEL2
 ↓
FB_LEAF
```

Phép tính:

```text
FB_LEAF   +1
FB_LEVEL2 +10
FB_LEVEL3 +100
```

Với `D100 = 100`:

Kết quả thực tế:

```text
M100 = 1
D110 = 101
M110 = 1

M101 = 1
D111 = 211
M111 = 1
```

## Kết luận

```text
Nested user FB 3 levels    PASS
Nested local instance      PASS
I/O propagation            PASS
```

Không được suy rộng rằng mọi user FB lớn/phức tạp đều an toàn.

---

# 11. Test IEC Function Blocks

Đã thử lại nhóm IEC FB như:

```text
TON
TOF
RS
SR
```

## Kết quả

FAIL.

Triệu chứng:

- IEC FB không chạy đúng;
- có trường hợp làm treo PLC;
- runtime/compiler-generated memory không tương thích với clone;
- liên quan tới vùng working memory cao/9000+ trên cấu hình đã quan sát.

## Kết luận

```text
IEC TON                    FAIL / FORBIDDEN
IEC TOF                    FAIL / FORBIDDEN
IEC RS                     FAIL / FORBIDDEN
IEC SR                     FAIL / FORBIDDEN
IEC Standard FB            FORBIDDEN
```

Không tiếp tục test IEC FB trên máy production.

---

# 12. Hardware test bổ sung: IF / ELSE / END_IF

Sau các test bổ sung trên PLC thật, `IF / ELSIF / ELSE / END_IF` được quan sát là **thiếu tính ổn định**, không chỉ trong pattern `IF ... SET(TRUE,...)`.

Do hành vi thay đổi theo cấu trúc và vị trí code, nhóm control-flow này được hạ từ "PASS cơ bản" xuống **FORBIDDEN cho production**.

## Kết luận

```text
IF / ELSIF / ELSE / END_IF    UNSTABLE / FORBIDDEN
Boolean expression            PASS
Direct-EN native instruction  PASS
```

Từ thời điểm này, test matrix coi mọi ví dụ IF ở các mục trước là **bằng chứng tái hiện lỗi**, không phải pattern được phép dùng.

---

# 13. Compatibility summary

| Feature | Result |
|---|---|
| ST basic execution | ✅ PASS |
| Direct BOOL assignment | ✅ PASS |
| Boolean latch | ✅ PASS |
| IF / ELSIF / ELSE / END_IF | ❌ UNSTABLE / FORBIDDEN |
| Native OUT_T | ✅ PASS |
| Native SET with direct EN | ✅ PASS |
| Native RST with direct EN | ✅ PASS |
| SET state driving OUT_T | ✅ PASS |
| IF + SET(TRUE,...) | ❌ FAIL |
| User-defined FB | ✅ PASS |
| Nested user FB 3 levels | ✅ PASS |
| IEC TON/TOF | ❌ FAIL |
| IEC RS/SR | ❌ FAIL |
| IEC Standard FB | ❌ FORBIDDEN |
| Large/complex user FB | ⚠ NOT YET FULLY VALIDATED |

---

# 14. Regression rule

Nếu đổi:

- model PLC clone;
- firmware;
- GX Works2 version;
- CPU parameter;
- compiler settings;

thì không được mặc định kết quả vẫn giống nhau.

Tối thiểu phải chạy lại:

1. ST basic test;
2. Boolean latch;
3. direct-EN SET/RST;
4. native OUT_T;
5. user FB basic;
6. nested user FB nếu project dùng;
7. tuyệt đối không test IEC FB trên hệ thống production.

---

# 15. Production decision

Rule cuối cùng:

```text
DIRECT EN
NATIVE FX
NO IEC FB
NO IF / ELSE
SINGLE OWNER
HARDWARE VERIFIED
```
