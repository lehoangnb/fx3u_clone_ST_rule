# AGENTS.md

# FX3U Clone ST Coding Rules

Các rule trong file này là **bắt buộc** khi tạo, sửa hoặc review code ST/Structured Project cho PLC FX3U clone/LE3U thuộc nhóm đã kiểm thử.

## 1. Mặc định ưu tiên primitive native

Ưu tiên:

- Boolean assignment `:=`
- `AND / OR / NOT`
- `IF / ELSIF / ELSE` đơn giản
- `SET(EN, device)`
- `RST(EN, device)`
- `OUT_T(EN, TCx, Kx)`
- native FX instructions đã hardware-test
- X/Y/M/D/T trực tiếp hoặc Global Label ánh xạ rõ ràng

## 2. Direct EN là bắt buộc

Nếu instruction có EN, truyền condition trực tiếp vào EN.

### Đúng

```pascal
SET(StartBtn, RunCmd);
RST(StopBtn, RunCmd);
OUT_T(RunCmd, TC0, K100);
```

### Cấm

```pascal
IF StartBtn THEN
    SET(TRUE, RunCmd);
END_IF;
```

```pascal
IF StopBtn THEN
    RST(TRUE, RunCmd);
END_IF;
```

```pascal
IF RunCmd THEN
    OUT_T(TRUE, TC0, K100);
END_IF;
```

Pattern `IF condition THEN instruction(TRUE,...)` đã gây lỗi runtime trên PLC thật.

## 3. IEC Function Block bị cấm

Không dùng:

```text
TON
TOF
TP
RS
SR
CTU
CTD
```

hoặc IEC Standard FB tương tự.

Lý do:

- không chạy đúng trên PLC đã test;
- compiler/runtime có thể dùng vùng working memory cao/9000+;
- đã quan sát trường hợp làm treo PLC.

Không test IEC FB trên máy production.

## 4. User-defined FB được phép có điều kiện

Đã test thành công:

- user FB đơn giản;
- local FB instance;
- nested user FB 3 tầng;
- truyền I/O xuyên các tầng FB.

Không được suy rộng rằng mọi user FB lớn/phức tạp đều an toàn.

Nếu tạo cấu trúc mới với:

- rất nhiều local variables;
- state nội bộ lớn;
- nhiều nested instances;
- nhiều instance song song;
- native instruction chưa từng test trong FB;

thì phải hardware-test riêng.

## 5. Tránh double coil / multiple writer

Mỗi writable device phải có một owner rõ ràng.

Tránh nhiều assignment tới cùng device.

### Tránh

```pascal
IF A THEN
    M100 := TRUE;
ELSIF B THEN
    M100 := FALSE;
END_IF;
```

nếu GX Works2 báo C9300.

### Dùng

```pascal
M100 := (M100 OR A) AND NOT B;
```

hoặc:

```pascal
SET(A, M100);
RST(B, M100);
```

## 6. Output vật lý có một điểm quyết định cuối

Không để nhiều nơi ghi cùng Y.

### Đúng

```pascal
MotorOutput :=
    MotorCmd
    AND SafetyOK
    AND NOT Alarm;

Y0 := MotorOutput;
```

Safety/interlock phải xuất hiện ở đường cuối tới output vật lý.

## 7. Không dùng Compile OK làm tiêu chuẩn tương thích

```text
Compile OK != Runtime OK
```

Feature mới chỉ được coi là supported sau khi:

1. compile;
2. download;
3. PLC RUN ổn định;
4. hardware test input OFF/ON/pulse/hold/reset;
5. monitor state nội bộ;
6. không gây PLC hang.

## 8. Warning quan trọng phải được xử lý

Đặc biệt:

```text
C9300  double coil
C9026  WORD/DWORD return type
F1028  device out of range/reserved
C8028  invalid instruction argument
```

Không bỏ qua warning chỉ vì chương trình download được.

## 9. Kiến trúc khuyến nghị

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

HMI không ghi trực tiếp Y nếu không có lý do đặc biệt.

## 10. Latch chuẩn

### Native

```pascal
SET(
    StartCondition
    AND NOT StopCondition
    AND NOT Alarm,
    RunCmd
);

RST(
    StopCondition OR Alarm,
    RunCmd
);
```

### Boolean

```pascal
RunCmd :=
    (RunCmd OR StartCondition)
    AND NOT StopCondition
    AND NOT Alarm;
```

## 11. Timer chuẩn

```pascal
OUT_T(RunCmd, TC0, K100);
TimerDone := TS0;
```

Không dùng IEC timer.

## 12. Khi review code

Reject hoặc yêu cầu sửa nếu thấy:

- IEC FB;
- `IF ... SET(TRUE,...)`;
- `IF ... RST(TRUE,...)`;
- `IF ... OUT_T(TRUE,...)`;
- nhiều writer cho cùng Y/M/D state;
- safety chỉ nằm ở command mà không nằm ở output cuối;
- warning C9300;
- instruction chưa từng hardware-test nhưng được coi là supported.

## 13. Nguyên tắc cuối

```text
DIRECT EN
SINGLE OWNER
NATIVE FX
NO IEC FB
SIMPLE BOOLEAN
HARDWARE TEST
```
