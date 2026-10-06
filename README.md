# FX3U Clone Structured Text Rules

Bộ quy tắc lập trình **Structured Text (ST) / Structured Project trên GX Works2** dành cho PLC **FX3U clone / LE3U** có hành vi tương tự CPU đã kiểm thử thực tế.

> [!WARNING]
> Đây **không phải** là tài liệu áp dụng cho mọi PLC Mitsubishi FX3U chính hãng.
> Các quy tắc dưới đây được xây dựng để tránh các lỗi runtime đã quan sát trên FX3U clone: chương trình compile/download bình thường nhưng thực thi sai, hoặc treo PLC.

## Mục tiêu

Ưu tiên theo thứ tự:

1. Chạy ổn định trên PLC thật.
2. Logic đơn giản, dễ monitor.
3. Tránh compiler/runtime feature mà clone không hỗ trợ đầy đủ.
4. Không coi `Compile OK` là bằng chứng `Runtime OK`.

---

# 1. Trạng thái tương thích đã kiểm thử

| Cấu trúc | Trạng thái | Ghi chú |
|---|---|---|
| ST cơ bản | ✅ Đã test | Chạy đúng |
| `BOOL := expression` | ✅ Đã test | Chạy đúng |
| `AND / OR / NOT` | ✅ Đã test | Chạy đúng |
| `IF / ELSIF / ELSE / END_IF` | ❌ CẤM | Hardware test bổ sung cho thấy runtime thiếu ổn định |
| Boolean self-latch | ✅ Đã test | Chạy đúng |
| `SET(EN, device)` native | ✅ Đã test | Chạy đúng |
| `RST(EN, device)` native | ✅ Đã test | Chạy đúng |
| `OUT_T(EN, TCx, Kx)` native | ✅ Đã test | Chạy đúng |
| User-defined FB đơn giản | ✅ Đã test | Chạy đúng |
| Nested user-defined FB 3 tầng | ✅ Đã test | Chạy đúng |
| Local FB instance | ✅ Đã test | Chạy đúng |
| Truyền I/O qua nested FB | ✅ Đã test | Chạy đúng |
| `IF cond THEN SET(TRUE,...)` | ❌ Không an toàn | Runtime sai trên PLC đã test |
| `IF cond THEN RST(TRUE,...)` | ❌ Không dùng | Cùng pattern lỗi, tránh hoàn toàn |
| IEC `TON` | ❌ CẤM | Không chạy đúng, có thể treo PLC |
| IEC `TOF` | ❌ CẤM | Không chạy đúng, có thể treo PLC |
| IEC `TP` | ❌ CẤM | Cùng nhóm IEC timer |
| IEC `RS / SR` | ❌ CẤM | IEC FB không tương thích |
| IEC `CTU / CTD` | ❌ CẤM | Cùng nhóm IEC FB/stateful FB |
| IEC Standard Function Block | ❌ CẤM | Có thể dùng vùng runtime/working memory clone không hỗ trợ |
| User FB lớn / nhiều local / state phức tạp | ⚠ Chưa đủ dữ liệu | Không suy rộng từ nested FB test |

---

# 2. Quy tắc quan trọng nhất: dùng EN trực tiếp

Nếu instruction có chân **EN**, hãy truyền điều kiện trực tiếp vào EN.

## ĐÚNG

```pascal
SET(StartBtn, MotorCmd);
RST(StopBtn, MotorCmd);
OUT_T(MotorCmd, TC0, K100);
```

## SAI / KHÔNG AN TOÀN TRÊN CLONE ĐÃ TEST

```pascal
IF StartBtn THEN
    SET(TRUE, MotorCmd);
END_IF;

IF StopBtn THEN
    RST(TRUE, MotorCmd);
END_IF;

IF MotorCmd THEN
    OUT_T(TRUE, TC0, K100);
END_IF;
```

### Lý do

Pattern:

```pascal
IF X0 THEN
    SET(TRUE, M110);
END_IF;
```

đã được test thực tế và cho hành vi runtime sai.

Trong một test:

```pascal
M130 := NOT X0;

IF X0 THEN
    SET(TRUE, M110);
END_IF;

M131 := NOT X0;

OUT_T(M8000, TC1, K20);
```

khi `X0 = OFF`:

- `M130 = 1`
- `M131 = 0`  ← assignment sau block bị ảnh hưởng
- timer dùng `M8000` phía sau vẫn chạy

Khi đổi thành:

```pascal
SET(X0, M110);
```

thì chương trình hoạt động bình thường.

**Kết luận thực dụng:** không bọc native instruction có EN bằng `IF ... instruction(TRUE,...)`.

---

# 3. SET/RST native được phép dùng

Bản thân `SET` và `RST` **không bị lỗi** nếu dùng đúng dạng direct-EN.

## Mẫu chuẩn

```pascal
SET(SetCondition, StateBit);
RST(ResetCondition, StateBit);
```

Ví dụ:

```pascal
SET(
    StartBtn
    AND SafetyOK
    AND NOT StopBtn
    AND NOT Alarm,
    MotorCmd
);

RST(
    StopBtn
    OR Alarm
    OR EmergencyStop,
    MotorCmd
);
```

## Ưu tiên RESET

Nếu SET và RESET có khả năng TRUE cùng scan, hãy thiết kế điều kiện SET để loại các điều kiện reset:

```pascal
SET(
    StartBtn
    AND NOT StopBtn
    AND NOT Alarm,
    MotorCmd
);

RST(
    StopBtn OR Alarm,
    MotorCmd
);
```

Không nên dựa hoàn toàn vào thứ tự instruction để tạo safety priority.

---

# 4. Boolean latch là phương án thay thế an toàn

Boolean self-latch đã được test hoạt động:

```pascal
RunCmd :=
    (RunCmd OR StartCondition)
    AND NOT StopCondition;
```

Nhiều reset:

```pascal
RunCmd :=
    (RunCmd OR StartCondition)
    AND NOT StopBtn
    AND NOT Alarm
    AND NOT EmergencyStop;
```

Dùng khi muốn:

- tránh SET/RST;
- chỉ có một assignment cho device;
- logic dễ đọc và dễ monitor.

---

# 5. Tránh double-coil / multiple writer

GX Works2 có thể báo:

```text
C9300
'M100' is double-coil.
```

với code:

```pascal
IF X1 THEN
    M100 := FALSE;
ELSIF X0 THEN
    M100 := TRUE;
END_IF;
```

Dù logic con người thấy hai nhánh loại trừ nhau, compiler/consistency checker vẫn có thể coi đây là nhiều coil.

## Nên viết

```pascal
M100 := (M100 OR X0) AND NOT X1;
```

hoặc:

```pascal
SET(X0, M100);
RST(X1, M100);
```

### Rule

> Mỗi device writable nên có **một owner** và càng ít điểm ghi càng tốt.

---

# 6. Output Y chỉ nên có một nơi quyết định cuối

## Không nên

```pascal
Y0 := AutoRun;

// ...

Y0 := ManualRun;
```

## Nên

```pascal
Y0 :=
    (AutoRun OR ManualRun)
    AND SafetyOK
    AND NOT Alarm;
```

Tốt hơn nữa, tách command khỏi physical output:

```pascal
MotorEnable :=
    MotorCmd
    AND SafetyOK
    AND NOT Alarm;

Y0 := MotorEnable;
```

### Rule

> HMI/input không nên ghi thẳng output vật lý.  
> Command nội bộ → interlock/safety → Y output.

---

# 7. Timer: dùng native OUT_T

Dùng timer native FX:

```pascal
OUT_T(MotorRun, TC0, K100);

TimerDone := TS0;
```

Không dùng IEC `TON`, `TOF`, `TP`.

Không viết:

```pascal
IF MotorRun THEN
    OUT_T(TRUE, TC0, K100);
END_IF;
```

Mà viết:

```pascal
OUT_T(MotorRun, TC0, K100);
```

---

# 8. IEC Function Block: CẤM

Trên PLC đã test, các IEC FB không chỉ không hoạt động mà còn có thể **làm treo PLC**.

Nhóm cấm gồm:

```text
TON
TOF
TP
RS
SR
CTU
CTD
và các IEC Standard Function Block tương tự
```

## Lý do thực tế

GX Works2 có thể sinh instance/internal/runtime memory trong vùng working device cao, bao gồm các vùng 9000+ tùy cấu trúc. Firmware clone đã test không hỗ trợ đầy đủ cách runtime này.

Triệu chứng đã quan sát:

- compile thành công;
- download được;
- PLC có thể vào RUN;
- FB không chạy đúng;
- một số trường hợp PLC treo.

### Rule production

> Không dùng IEC Standard Function Block trên dòng clone này, kể cả project compile sạch.

Không thử IEC FB trên máy đang vận hành thực tế.

---

# 9. User-defined Function Block KHÔNG đồng nghĩa với IEC FB

Không được cấm oan toàn bộ FB.

User-defined FB đã được test thành công.

## Test đơn giản

```pascal
Alive := Enable;
OutValue := InValue + 1;
```

Kết quả:

```text
D100 = 100
D110 = 101
M100 = 1
```

## Nested 3 tầng đã test

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
InValue
+ 1     FB_LEAF
+ 10    FB_LEVEL2
+ 100   FB_LEVEL3
```

Với `D100 = 100`, kết quả thực tế:

```text
M100 = 1
D110 = 101
M110 = 1

M101 = 1
D111 = 211
M111 = 1
```

**Kết luận:** user-defined FB và nested user FB 3 tầng chạy được trên PLC đã test.

### Nhưng không suy rộng quá mức

Chưa được phép mặc định rằng user FB rất lớn, rất nhiều local/state, recursion-like structure hoặc nhiều instance phức tạp đều an toàn.

Mọi cấu trúc FB mới vượt quá phạm vi đã test phải được hardware-test riêng.

---

# 10. IF / ELSIF / ELSE / END_IF: CẤM

Sau các hardware test bổ sung, control-flow ST dùng:

```text
IF
ELSIF
ELSE
END_IF
```

được xác định là **thiếu ổn định trên PLC clone đã test**.

Triệu chứng không chỉ giới hạn ở `IF ... SET(TRUE,...)`. Hành vi runtime có thể thay đổi tùy cấu trúc, vị trí instruction và trạng thái scan. Vì vậy không tiếp tục coi `IF` cơ bản là an toàn.

## Rule production

> Không sử dụng `IF / ELSIF / ELSE / END_IF` trong ST production cho dòng clone này.

Không viết:

```pascal
IF Alarm THEN
    OutputEnable := FALSE;
ELSE
    OutputEnable := CommandEnable;
END_IF;
```

Hãy chuyển điều kiện thành Boolean expression trực tiếp:

```pascal
OutputEnable := CommandEnable AND NOT Alarm;
```

Không viết:

```pascal
IF A THEN
    IF B THEN
        SET(TRUE, Command);
    END_IF;
END_IF;
```

Hãy viết:

```pascal
Enable := A AND B;
SET(Enable, Command);
```

Nguyên tắc:

- tính condition bằng Boolean expression;
- truyền condition trực tiếp vào EN của native instruction;
- dùng single-assignment;
- dùng one-hot state bits nếu cần sequence/state machine;
- tránh toàn bộ structured conditional control-flow.

---

# 11. Không dùng nesting control-flow

Do `IF/ELSIF/ELSE/END_IF` đã bị cấm, mọi dạng nested conditional cũng bị cấm.

## Dùng Boolean expression

```pascal
Enable :=
    A
    AND B
    AND C
    AND D;

SET(Enable, Command);
```

Lợi ích:

- không phụ thuộc control-flow runtime thiếu ổn định;
- dễ monitor;
- dễ chuyển sang FBD/Ladder;
- gần semantics của native FX hơn.

---

# 12. Tách biểu thức phức tạp thành bit trung gian

```pascal
CanMove :=
    PowerOK
    AND ServoReady
    AND DoorUnlocked
    AND NOT Alarm
    AND NOT StopRequest;

SET(StartRequest AND CanMove, MoveCmd);
```

Ưu tiên các bit trung gian có ý nghĩa thay vì một biểu thức quá dài lặp lại nhiều nơi.

---

# 13. Edge detection: ưu tiên Boolean đơn giản

Nếu chưa xác minh instruction pulse cụ thể:

```pascal
StartPulse := StartBtn AND NOT PrevStartBtn;
PrevStartBtn := StartBtn;
```

Đảm bảo mỗi variable chỉ có một nơi ghi.

---

# 14. Special relay cơ bản

Có thể dùng trực tiếp:

```pascal
SystemRun := M8000;
InitPulse := M8002;
```

Timer native:

```pascal
OUT_T(M8000, TC0, K20);
```

Không cần:

```pascal
IF M8000 THEN
    OUT_T(TRUE, TC0, K20);
END_IF;
```

---

# 15. Kiến trúc command/state/output khuyến nghị

```text
INPUT / HMI REQUEST
        ↓
COMMAND
        ↓
STATE / SEQUENCE
        ↓
INTERLOCK / SAFETY
        ↓
PHYSICAL OUTPUT
```

Ví dụ:

```pascal
SET(
    StartRequest
    AND Ready
    AND NOT StopRequest
    AND NOT Alarm,
    RunCmd
);

RST(
    StopRequest
    OR Alarm
    OR NOT Ready,
    RunCmd
);

MachineActive :=
    RunCmd
    AND Ready
    AND NOT Alarm;

Y0 := MachineActive;
```

---

# 16. State machine: ưu tiên one-hot state bits

Không dùng `IF/ELSIF/ELSE` để triển khai state machine.

Ưu tiên mỗi state là một bit M và transition là Boolean condition + native SET/RST direct-EN.

Ví dụ:

```pascal
ToRun :=
    StateIdle
    AND StartBtn
    AND Ready
    AND NOT Alarm;

ToDone :=
    StateRun
    AND MoveDone;

ToIdle :=
    StateDone
    AND ResetBtn;

SET(ToRun, StateRun);
RST(ToRun, StateIdle);

SET(ToDone, StateDone);
RST(ToDone, StateRun);

SET(ToIdle, StateIdle);
RST(ToIdle, StateDone);
```

Yêu cầu:

- mỗi transition condition được tính bằng Boolean expression;
- không dùng `IF`;
- SET/RST phải dùng direct EN;
- thiết kế để chỉ một state hợp lệ tại một thời điểm;
- có logic initialization/recovery rõ ràng.

---

# 17. Device mapping và local memory

Ưu tiên các device native dễ monitor:

```text
X
Y
M
D
T
```

Các signal quan trọng nên có mapping rõ ràng:

```text
StartBtn     X0
StopBtn      X1
MotorCmd     M100
MotorOutput  Y0
State        D100
```

Không phụ thuộc mù quáng vào compiler-generated memory cho các chức năng quan trọng.

User-defined FB đã test chạy được, nhưng IEC FB đã chứng minh rằng **compiler-generated runtime memory vẫn có thể vượt khả năng firmware clone**.

---

# 18. Không bỏ qua warning của GX Works2

Đặc biệt chú ý:

```text
C9300  Double coil
C9026  WORD/DWORD return type
F1028  Device out of range / reserved
C8028  Invalid instruction argument
```

Trên clone:

> Warning phải được coi là lỗi tiềm tàng cho đến khi chứng minh ngược lại bằng hardware test.

Mục tiêu production:

```text
0 Error
0 critical Warning
0 Double coil
```

---

# 19. Compile OK ≠ Runtime OK

Đây là rule bắt buộc.

```text
GX Works2 Compile OK
        ≠
PLC clone Runtime OK
```

Một project có thể:

- compile;
- download;
- RUN;
- monitor được device;

nhưng vẫn thực thi sai một instruction hoặc FB cụ thể.

---

# 20. Quy trình kiểm thử instruction/FB mới

Mọi feature chưa từng dùng trên clone phải trải qua test tối thiểu.

## Test A — cô lập

Chỉ test instruction/FB cần kiểm tra.

## Test B — đối chứng

Tạo một nhánh bằng primitive đã biết hoạt động.

Ví dụ:

```text
M100 = reference
M110 = instruction under test
M120 = mismatch
```

## Test C — các trạng thái input

Phải test ít nhất:

- OFF;
- ON;
- pulse ngắn;
- giữ ON;
- reset;
- nhiều scan;
- power cycle nếu state/persistence có liên quan.

## Test D — monitor runtime

Không chỉ nhìn output cuối. Monitor:

- input;
- internal state;
- timer/counter current value nếu có;
- output;
- mismatch bit.

## Test E — fail-safe

Nếu feature làm treo CPU, đánh dấu **CẤM** và không lặp lại trên hệ thống production.

---

# 21. Mẫu code chuẩn

## Latch native

```pascal
SET(
    StartCondition
    AND NOT StopCondition
    AND NOT Alarm,
    RunCmd
);

RST(
    StopCondition
    OR Alarm,
    RunCmd
);
```

## Latch Boolean

```pascal
RunCmd :=
    (RunCmd OR StartCondition)
    AND NOT StopCondition
    AND NOT Alarm;
```

## Timer native

```pascal
OUT_T(RunCmd, TC0, K100);

TimerDone := TS0;
```

## Output

```pascal
MotorOutput :=
    RunCmd
    AND SafetyOK
    AND NOT Alarm;

Y0 := MotorOutput;
```

---

# 22. DO / DON'T

## DO

```pascal
SET(Condition, M100);

RST(Condition, M100);

OUT_T(Condition, TC0, K100);

M101 := A AND B AND NOT C;

M102 :=
    (M102 OR SetCondition)
    AND NOT ResetCondition;

Y0 :=
    Command
    AND SafetyOK
    AND NOT Alarm;
```

## DON'T

```pascal
IF Condition THEN
    SET(TRUE, M100);
END_IF;
```

```pascal
IF Condition THEN
    RST(TRUE, M100);
END_IF;
```

```pascal
IF Condition THEN
    OUT_T(TRUE, TC0, K100);
END_IF;
```

Không dùng IEC:

```text
TON
TOF
TP
RS
SR
CTU
CTD
```

---

# 23. Thứ tự ưu tiên khi chọn giải pháp

Ưu tiên:

1. Boolean expression đơn giản.
2. Native FX instruction với EN trực tiếp.
3. Native timer/counter đã test.
4. One-hot state machine bằng Boolean + SET/RST direct-EN.
5. User-defined FB đã hardware-test và không dùng IF/ELSE bên trong.
6. Nested user FB trong phạm vi đã test và không dùng IF/ELSE bên trong.
7. Cấu trúc user FB mới/phức tạp chỉ sau hardware test.

**Không sử dụng IF / ELSIF / ELSE / END_IF.**

**Không có IEC FB trong danh sách này.**

---

# 24. Rule ngắn gọn để nhớ

```text
DIRECT EN
+
SINGLE OWNER
+
NATIVE FX INSTRUCTION
+
SIMPLE BOOLEAN LOGIC
+
NO IEC FB
+
NO IF / ELSE
+
HARDWARE TEST
```

Hay nói cách khác:

> Viết ST càng gần semantics của Ladder native FX càng tốt: Boolean expression + direct EN + native instruction. Không dùng IF/ELSIF/ELSE/END_IF.

Ví dụ Ladder:

```text
X0 -------- [SET M100]
```

ST nên là:

```pascal
SET(X0, M100);
```

không phải:

```pascal
IF X0 THEN
    SET(TRUE, M100);
END_IF;
```

Timer Ladder:

```text
M100 ------- [T0 K100]
```

ST:

```pascal
OUT_T(M100, TC0, K100);
```

Output Ladder:

```text
M100 -- Safety -- /Alarm ---- (Y0)
```

ST:

```pascal
Y0 := M100 AND Safety AND NOT Alarm;
```

---

# 25. Phạm vi xác nhận

Các kết luận ✅/❌ trong tài liệu này phản ánh **PLC FX3U clone/LE3U đã test thực tế**, không phải tuyên bố rằng mọi clone trên thị trường đều có firmware giống nhau.

Với model/firmware clone khác:

1. bắt đầu từ các rule bảo thủ này;
2. chạy test matrix;
3. chỉ nới rule sau khi hardware test thành công.

Tham khảo thêm về lỗi Structured Project/output trên FX3U clone:

- https://industrialmonitordirect.com/blogs/knowledgebase/resolving-gx-works2-fx3u-output-coil-failures-on-clones
