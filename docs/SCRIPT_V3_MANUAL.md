# FlowScenario Script v3 (ANTLR) 매뉴얼

## 개요
Script v3는 ANTLR 기반 문법으로 `import`, `func`, `call`, `run`, `run_once`, `run_once_stop`, `spawn`, `await`를 지원합니다.

실행 창: `RatelWPF > File > Script Console`

참고: Script Console은 `Lang = JavaScript` 모드도 지원합니다.
- JS 가이드는 `SCRIPT_JS_QUICKSTART.md`를 참고하세요.

---

## 목차
1. [핵심 문법](#핵심-문법)
2. [제어문](#제어문)
3. [배열](#배열)
4. [예외 처리](#예외-처리)
5. [변수](#변수)
6. [연산자](#연산자)
7. [내장 함수](#내장-함수)
   - [기본 함수](#기본-함수)
   - [수학 함수](#수학-함수)
   - [삼각함수](#삼각함수)
   - [문자열 함수](#문자열-함수)
   - [배열 함수](#배열-함수)
   - [타입 체크 및 변환](#타입-체크-및-변환)
8. [모션 함수](#모션-함수-script-console)
9. [IO 함수](#io-함수-script-console)
10. [비동기 패턴](#비동기-패턴)
11. [예제 코드](#예제-코드)
12. [런타임 서비스 API (C#)](#런타임-서비스-api-c)

---

## 핵심 문법

### 함수 정의와 호출
```javascript
import "interlock_y_limit.scr" as interlock

func add(a, b)
{
    return a + b
}

let result = add(1, 2)      // 함수 호출
call interlock.check_interlock("Y", 500) // 모듈 함수 동기 호출
let h = run add(5, 6)       // 비동기 실행 핸들 생성 (run == spawn)
let r = await h             // 완료 대기
```

### 기본 규칙
- 함수 호출은 반드시 `()`를 사용합니다.
- 중괄호는 같은 줄 또는 줄바꿈 스타일 모두 지원합니다.

```javascript
// 스타일 1
if (condition) { print("ok") }

// 스타일 2
if (condition)
{
    print("ok")
}
```

---

## 제어문

### if-else
```javascript
if (x > 10)
{
    print("크다")
}
else if (x > 5)
{
    print("중간")
}
else
{
    print("작다")
}
```

### while
```javascript
let i = 0
while (i < 10)
{
    print("i = " + i)
    i = i + 1
}
```

### for
```javascript
// 기본: 1부터 10까지
for i in 1..10
{
    print(i)
}

// step 지정
for i in 0..100 step 10
{
    print(i)  // 0, 10, 20, 30, ...
}

// 역순
for i in 10..1 step -1
{
    print(i)  // 10, 9, 8, 7, ...
}
```

### break, continue, return
```javascript
func find_value(max)
{
    for i in 1..max
    {
        if (i == 5)
        {
            continue  // 5는 건너뛰기
        }
        if (i > 10)
        {
            break  // 루프 종료
        }
        if (i == 7)
        {
            return i  // 함수 종료 및 값 반환
        }
    }
    return 0
}
```

### foreach
배열의 각 요소를 순회합니다.
```javascript
let items = [1, 2, 3, 4, 5]
foreach item in items
{
    print("Item: {item}")
}

// break, continue 사용 가능
let numbers = [1, 2, 3, 4, 5]
foreach n in numbers
{
    if (n == 2)
    {
        continue  // 2는 건너뛰기
    }
    if (n == 4)
    {
        break     // 4에서 종료
    }
    print(n)      // 1, 3 출력
}
```

### end, stop
```javascript
end                    // 스크립트 정상 종료
stop "사용자 중단"     // 예외 발생 (ScriptStopException)
```

### strict 모드
```javascript
strict true   // strict 모드 활성화
// strict 모드에서는 변수를 let으로 선언하지 않으면 오류

let x = 10    // OK
x = 20        // OK (이미 선언됨)
y = 30        // 오류! let으로 선언되지 않음

strict false  // strict 모드 해제
```

---

## 배열

### 배열 리터럴
```javascript
let arr = [1, 2, 3, 4, 5]           // 숫자 배열
let names = ["Alice", "Bob"]        // 문자열 배열
let mixed = [1, "hello", true]      // 혼합 배열
let empty = []                       // 빈 배열
let nested = [[1, 2], [3, 4]]       // 중첩 배열
```

### 배열 인덱스 접근
```javascript
let arr = [10, 20, 30]
print(arr[0])        // 10
print(arr[1])        // 20
print(arr[2])        // 30

// 인덱스로 값 변경
arr[1] = 25
print(arr[1])        // 25
```

### 배열 순회
```javascript
let items = ["a", "b", "c"]

// foreach 사용
foreach item in items
{
    print(item)
}

// for 루프 사용
for i in 0..len(items) - 1
{
    print(items[i])
}
```

### 배열 조작
```javascript
let arr = [1, 2, 3]

// 요소 추가
push(arr, 4)              // [1, 2, 3, 4]
unshift(arr, 0)           // [0, 1, 2, 3, 4]

// 요소 제거
let last = pop(arr)       // last = 4, arr = [0, 1, 2, 3]
let first = shift(arr)    // first = 0, arr = [1, 2, 3]

// 부분 배열
let sub = slice(arr, 1, 3)  // [2, 3]

// 배열 합치기
let merged = concat([1, 2], [3, 4])  // [1, 2, 3, 4]

// 정렬 및 역순
let sorted = sort([3, 1, 2])    // [1, 2, 3]
let rev = reverse([1, 2, 3])    // [3, 2, 1]

// 문자열로 변환
let str = join([1, 2, 3], ", ")  // "1, 2, 3"
```

### 배열 유틸리티
```javascript
// 배열 생성
let zeros = array(5, 0)       // [0, 0, 0, 0, 0]
let nums = range(5)           // [0, 1, 2, 3, 4]
let nums2 = range(1, 5)       // [1, 2, 3, 4]
let nums3 = range(0, 10, 2)   // [0, 2, 4, 6, 8]

// 배열 길이
let size = len(arr)

// 배열 체크
if (is_array(arr))
{
    print("배열입니다")
}
```

---

## 예외 처리

### try-catch
```javascript
try
{
    let result = risky_operation()
    print("성공: {result}")
}
catch e
{
    print("오류 발생: {e}")
}
```

### try-catch-finally
```javascript
try
{
    let file = open_file("data.txt")
    process(file)
}
catch e
{
    print("처리 중 오류: {e}")
}
finally
{
    // 항상 실행됨 (성공/실패 무관)
    cleanup()
    print("정리 완료")
}
```

### throw
사용자 정의 예외를 발생시킵니다.
```javascript
func validate(x)
{
    if (x < 0)
    {
        throw "값은 0 이상이어야 합니다"
    }
    if (x > 100)
    {
        throw "값은 100 이하여야 합니다"
    }
    return x
}

try
{
    validate(-5)
}
catch e
{
    print("검증 실패: {e}")  // "검증 실패: 값은 0 이상이어야 합니다"
}
```

### 예외 처리 패턴
```javascript
// 여러 작업에서 예외 처리
func safe_divide(a, b)
{
    if (b == 0)
    {
        throw "0으로 나눌 수 없습니다"
    }
    return a / b
}

func calculate()
{
    try
    {
        let x = safe_divide(10, 0)
    }
    catch e
    {
        print("계산 오류: {e}")
        return 0  // 기본값 반환
    }
}

// 중첩 try-catch
try
{
    try
    {
        throw "내부 오류"
    }
    catch e
    {
        print("내부에서 처리: {e}")
        throw "외부로 전파"
    }
}
catch e
{
    print("외부에서 처리: {e}")
}
```

---

## 변수

### 로컬 변수
```javascript
let x = 10           // 새 변수 선언
x = x + 5            // 기존 변수 수정
let y = x * 2        // 다른 변수 선언
```

### 글로벌 변수 (시스템 변수)
```javascript
@DoorClosed = 1      // 글로벌 변수 설정
let status = @DoorClosed  // 글로벌 변수 읽기
```

### 시스템 변수 사용법
시스템 변수는 `@`로 시작하는 글로벌 저장소를 사용합니다.

- 이름 규칙: `@` + 영문/숫자/언더스코어 (`_`)
- 첫 글자는 영문 또는 `_`
- 예: `@DoorClosed`, `@tray_state_01`
- 불가: 공백, `-`, `.`, `[]`가 포함된 변수명

#### 1) 권장: 글로벌 배열로 직접 관리
배열 데이터는 이름 규칙으로 쪼개기보다, 글로벌 배열 변수 1개로 관리하는 것이 안전합니다.

```javascript
@tray_state = array(4, 0)
@tray_state[0] = 1
@tray_state[1] = 2
print("Tray0={}", @tray_state[0])
```

#### 2) 이름 규칙 기반(가상 배열) 관리 패턴
호스트 프로그램과의 약속이 "스칼라 키" 중심이면 prefix + index 키로 운용할 수 있습니다.

```javascript
@tray_0_state = 1
@tray_1_state = 2
@tray_2_state = 0
```

권장 규칙:
- prefix 고정: `@tray_`
- index 자릿수 고정(가능하면): `00`, `01`, `02` ...
- suffix 고정: `_state`, `_temp`, `_alarm`

#### 3) C# 프로그램과 데이터 교환
Script v3의 시스템 변수 저장소는 `RatelLib.Variables`(전역)입니다.

```csharp
// write
RatelLib.Variables.Set("@DoorClosed", 1);
RatelLib.Variables.Set("@tray_state", new List<object> { 1m, 0m, 0m });

// read
var door = RatelLib.Variables.Get("@DoorClosed");
var tray = RatelLib.Variables.Get("@tray_state");
```

참고:
- 저장소는 thread-safe로 동작합니다.
- 숫자형은 내부적으로 decimal로 정규화됩니다.
- 스크립트 엔진에 별도 글로벌 저장소를 주입하려면 `ExecuteAsync(..., globals: customManager)`를 사용하세요.

### 스코프 규칙
```javascript
let x = 10

if (true)
{
    let x = 20       // 새로운 x (shadowing)
    print(x)         // 20
}

print(x)             // 10 (원래 x)

// for 루프 변수도 블록 스코프
for i in 1..5
{
    let temp = i * 2
}
// 여기서는 i와 temp 모두 사용 불가
```

---

## 연산자

### 산술 연산자
```javascript
let a = 10 + 5       // 더하기 (15)
let b = 10 - 3       // 빼기 (7)
let c = 4 * 5        // 곱하기 (20)
let d = 20 / 4       // 나누기 (5)
let e = 10 % 3       // 나머지 (1)
```

### 비교 연산자
```javascript
x == 10              // 같음
x != 5               // 같지 않음
x > 5                // 초과
x >= 5               // 이상
x < 10               // 미만
x <= 10              // 이하
```

### 논리 연산자
```javascript
x > 5 && x < 10      // AND (그리고)
x == 0 || x == 1     // OR (또는)
!flag                // NOT (부정)
```

### 문자열 연결
```javascript
let msg = "Hello" + " " + "World"  // "Hello World"
let info = "Value: " + 123         // "Value: 123"
```

---

## 내장 함수

### 기본 함수

#### print(...)
메시지를 출력합니다. 문자열 보간을 지원합니다.
```javascript
print("Hello World")
let name = "Alice"
print("Name: " + name)

// 변수 보간
let x = 10
print("x = {x}")           // x = 10
print("Result: {}", x)     // Result: 10

// 글로벌 변수
print("Door: {@DoorClosed}")
```

#### sleep(duration)
지정된 시간만큼 대기합니다.
```javascript
sleep(1000)            // 1000ms (1초) 대기
sleep("500ms")         // 500밀리초
sleep("2sec")          // 2초
sleep("1min")          // 1분
sleep("1hour")         // 1시간
```

#### assert(condition, message)
조건이 false면 예외를 발생시킵니다.
```javascript
assert(x > 0, "x는 양수여야 합니다")
assert(status == 1, "시스템이 준비되지 않았습니다")
```

#### is_running(handle)
비동기 태스크가 실행 중인지 확인합니다.
```javascript
let h = spawn long_task()
while (is_running(h) == 1)
{
    print("실행 중...")
    sleep(100)
}
```

---

## 런타임 서비스 API (C#)

`ScriptV3RuntimeService`를 사용하면 C# 프로그램에서 스크립트 실행을 직접 제어할 수 있습니다.

위치:
- `RatelSoft.Lib/FlowScenario/ScriptV3/ScriptV3RuntimeService.cs`

핵심 기능:
- 스크립트 시작: `StartScript`, `StartScriptFile`
- 실행 목록 조회: `GetRuns`
- 실행 제어: `TryPause`, `TryResume`, `TryStep`, `TryStop`
- 이벤트: 시작/상태변경/완료/에러/출력

### 기본 사용 예제
```csharp
using RatelSoft.Lib;
using RatelSoft.Lib.FlowScenario.ScriptV3;

var engine = new ScriptV3Engine();
var runtime = new ScriptV3RuntimeService(engine);

runtime.RunStarted += (_, e) =>
{
    Console.WriteLine($"[START] {e.Info.Name} ({e.Info.RunId})");
};

runtime.RunStateChanged += (_, e) =>
{
    Console.WriteLine($"[STATE] {e.Info.Name} -> {e.Info.State}, line={e.Info.LastLine}");
};

runtime.RunOutput += (_, e) =>
{
    Console.WriteLine($"[OUT:{e.RunId}] {e.Message}");
};

runtime.RunFaulted += (_, e) =>
{
    Console.WriteLine($"[FAULT] {e.Info.Name}: {e.Exception.Message}");
};

runtime.RunCompleted += (_, e) =>
{
    Console.WriteLine($"[DONE] {e.Info.Name}: {e.Info.State}");
};
```

### 실행 시작
```csharp
string source = @"
print(""hello"")
sleep(500)
print(""done"")
";

// 글로벌 변수 저장소 공유 (필요 시)
var globals = RatelLib.Variables;
globals.Set("@DoorClosed", 1);

Guid runId = runtime.StartScript(
    source: source,
    sourcePath: "(inline)",
    name: "InlineSample",
    globals: globals);
```

파일 실행:
```csharp
Guid runId2 = runtime.StartScriptFile(
    filePath: @"C:\scripts\main.scr",
    name: "MainScr",
    globals: RatelLib.Variables);
```

### 실행 목록 조회
```csharp
var runs = runtime.GetRuns(includeCompleted: true);
foreach (var r in runs)
{
    Console.WriteLine($"{r.Name} | {r.State} | line={r.LastLine} | id={r.RunId}");
}
```

### 일시정지/재개/스텝/정지
```csharp
runtime.TryPause(runId);   // Running -> Paused
runtime.TryStep(runId);    // Paused 상태에서 statement 1회 진행
runtime.TryResume(runId);  // Paused -> Running
runtime.TryStop(runId);    // 취소 요청
```

주의:
- `TryStep`은 `Paused` 상태에서만 유효합니다.
- 상태 전이는 `ScriptRunState`를 기준으로 처리됩니다.
- `TryStop`은 취소 토큰 기반 정지 요청입니다.

### 상태 종류
`ScriptRunState`:
- `Created`
- `Running`
- `Paused`
- `Completed`
- `Canceled`
- `Faulted`

---

### 수학 함수

#### 기본 수학
```javascript
abs(-10)               // 절댓값: 10
floor(3.7)             // 내림: 3
ceil(3.2)              // 올림: 4
round(3.5)             // 반올림: 4
round(3.14159, 2)      // 소수점 2자리: 3.14
sqrt(16)               // 제곱근: 4
pow(2, 3)              // 거듭제곱: 8 (2^3)
```

#### 최소/최대
```javascript
min(3, 7, 2, 9)        // 최솟값: 2
max(3, 7, 2, 9)        // 최댓값: 9
```

#### 로그/지수
```javascript
log(10)                // 자연로그 ln(10)
log10(100)             // 상용로그: 2
exp(1)                 // e^1: 2.71828...
```

#### 난수
```javascript
random()               // 0.0 ~ 1.0 사이 실수
random_int(100)        // 0 ~ 99 사이 정수
random_int(10, 20)     // 10 ~ 19 사이 정수
```

#### 상수
```javascript
let circumference = 2 * pi() * radius  // π (3.14159...)
let val = e()                          // e (2.71828...)
```

---

### 삼각함수

모든 삼각함수는 **라디안(radian)** 단위를 사용합니다.

#### 기본 삼각함수
```javascript
sin(pi() / 2)          // 1
cos(0)                 // 1
tan(pi() / 4)          // 1
```

#### 역삼각함수
```javascript
asin(1)                // π/2
acos(0)                // π/2
atan(1)                // π/4
atan2(1, 1)            // π/4 (y, x 순서)
```

#### 각도 변환
```javascript
let rad = deg_to_rad(90)    // 90도 → 1.5708 라디안
let deg = rad_to_deg(pi())  // π 라디안 → 180도

// 예제: 각도를 이용한 삼각함수
let angle_deg = 45
let angle_rad = deg_to_rad(angle_deg)
let sin_val = sin(angle_rad)  // sin(45°) = 0.7071
```

---

### 문자열 함수

#### length(str)
문자열 길이를 반환합니다.
```javascript
length("Hello")        // 5
length("")             // 0
```

#### substring(str, start, [length])
부분 문자열을 추출합니다.
```javascript
substring("Hello World", 0, 5)   // "Hello"
substring("Hello World", 6)      // "World"
```

#### upper(str) / lower(str)
대소문자 변환
```javascript
upper("hello")         // "HELLO"
lower("WORLD")         // "world"
```

#### trim(str)
앞뒤 공백 제거
```javascript
trim("  hello  ")      // "hello"
```

#### replace(str, old, new)
문자열 치환
```javascript
replace("Hello World", "World", "Script")  // "Hello Script"
```

#### contains(str, substring)
부분 문자열 포함 여부
```javascript
contains("Hello World", "World")   // 1 (true)
contains("Hello World", "Script")  // 0 (false)
```

#### starts_with(str, prefix) / ends_with(str, suffix)
시작/끝 문자열 확인
```javascript
starts_with("Hello", "He")    // 1
ends_with("World", "ld")      // 1
```

#### index_of(str, substring)
부분 문자열의 위치 반환 (없으면 -1)
```javascript
index_of("Hello World", "World")  // 6
index_of("Hello World", "Script") // -1
```

#### split(str, separator)
문자열 분할 (결과는 | 구분자로 연결된 문자열)
```javascript
let parts = split("a,b,c", ",")   // "a|b|c"
```

---

### 배열 함수

#### len(arr)
배열 또는 문자열의 길이를 반환합니다.
```javascript
len([1, 2, 3])        // 3
len("hello")          // 5
len([])               // 0
```

#### push(arr, item)
배열 끝에 요소를 추가합니다. 새 길이를 반환합니다.
```javascript
let arr = [1, 2]
push(arr, 3)          // arr = [1, 2, 3], 반환값 3
```

#### pop(arr)
배열 끝의 요소를 제거하고 반환합니다.
```javascript
let arr = [1, 2, 3]
let last = pop(arr)   // last = 3, arr = [1, 2]
```

#### shift(arr)
배열 첫 번째 요소를 제거하고 반환합니다.
```javascript
let arr = [1, 2, 3]
let first = shift(arr)  // first = 1, arr = [2, 3]
```

#### unshift(arr, item)
배열 앞에 요소를 추가합니다. 새 길이를 반환합니다.
```javascript
let arr = [2, 3]
unshift(arr, 1)       // arr = [1, 2, 3], 반환값 3
```

#### slice(arr, start, [end])
배열의 일부분을 새 배열로 반환합니다.
```javascript
let arr = [0, 1, 2, 3, 4]
slice(arr, 1, 3)      // [1, 2]
slice(arr, 2)         // [2, 3, 4]
slice(arr, -2)        // [3, 4] (음수 인덱스 지원)
```

#### concat(arr1, arr2, ...)
여러 배열을 합쳐 새 배열을 반환합니다.
```javascript
concat([1, 2], [3, 4])       // [1, 2, 3, 4]
concat([1], [2], [3])        // [1, 2, 3]
```

#### reverse(arr)
역순으로 정렬된 새 배열을 반환합니다 (원본 변경 없음).
```javascript
reverse([1, 2, 3])    // [3, 2, 1]
```

#### sort(arr)
오름차순으로 정렬된 새 배열을 반환합니다 (원본 변경 없음).
```javascript
sort([3, 1, 2])       // [1, 2, 3]
```

#### join(arr, separator)
배열 요소를 문자열로 연결합니다.
```javascript
join([1, 2, 3], ", ")     // "1, 2, 3"
join(["a", "b"], "-")     // "a-b"
```

#### is_array(value)
값이 배열인지 확인합니다.
```javascript
is_array([1, 2])      // 1 (true)
is_array("hello")     // 0 (false)
```

#### array(size, [default])
지정된 크기의 배열을 생성합니다.
```javascript
array(5)              // [null, null, null, null, null]
array(3, 0)           // [0, 0, 0]
array(2, "x")         // ["x", "x"]
```

#### range(end) / range(start, end, [step])
숫자 시퀀스 배열을 생성합니다.
```javascript
range(5)              // [0, 1, 2, 3, 4]
range(1, 5)           // [1, 2, 3, 4]
range(0, 10, 2)       // [0, 2, 4, 6, 8]
range(5, 0, -1)       // [5, 4, 3, 2, 1]
```

---

### 타입 체크 및 변환

#### 타입 체크
```javascript
is_number(123)         // 1
is_string("hello")     // 1
is_bool(true)          // 1
is_null(x)             // 변수가 null이면 1

typeof(123)            // "number"
typeof("hello")        // "string"
typeof(true)           // "bool"
typeof(null)           // "null"
```

#### 타입 변환
```javascript
to_number("123")       // 123
to_string(123)         // "123"
to_bool(1)             // true
to_bool(0)             // false
```

---

## 모션 함수 (Script Console)

### 서보 제어
```javascript
servo_on("X")          // X축 서보 ON
servo_off("X")         // X축 서보 OFF
home("X")              // X축 원점 복귀
```

### 이동 명령 (동기)
```javascript
// 절대 위치 이동 (완료될 때까지 대기)
move_abs("X", 100, 5000)    // X축을 100 위치로, 타임아웃 5초

// 상대 위치 이동
move_inc("Y", 50, 3000)     // Y축을 현재 위치에서 +50 이동

// 이동 완료 대기
wait_move("X")
```

### 이동 명령 (비동기)
```javascript
// 이동 시작 (대기하지 않음)
start_move_abs("X", 100, 5000)
start_move_inc("Y", 50, 3000)

// 이동 중 확인
while (is_moving("X") == 1)
{
    print("X축 이동 중...")
    sleep(100)
}

// 정지 확인
if (is_stop("X") == 1)
{
    print("X축 정지됨")
}
```

### 위치 및 정지
```javascript
let pos = get_pos("X")     // 현재 위치 읽기
stop("X")                  // X축 정지
all_stop()                 // 모든 축 정지
```

---

## IO 함수 (Script Console)

### 입출력 읽기
```javascript
let door = get_in("DoorClosed")      // 입력 신호 읽기 (1/0)
let lamp = get_out("RedLamp")        // 출력 신호 읽기 (1/0)
```

### 입출력 설정
```javascript
set_in("DoorClosed", 1)              // 입력 시뮬레이션
set_out("RedLamp", 1)                // 출력 설정
```

### 입력 대기
```javascript
// DoorClosed가 1이 될 때까지 대기 (타임아웃 5초)
wait_in("DoorClosed", 1, 5000)

// 타임아웃 없이 대기
wait_in("StartButton", 1, 0)
```

---

## 비동기 패턴

### 기본 spawn/await
```javascript
func heavy_task(n)
{
    sleep(1000)
    return n * 2
}

let h = spawn heavy_task(10)    // 비동기 시작
print("다른 작업 수행...")
let result = await h            // 완료 대기
print("결과: " + result)        // 결과: 20
```

### 인터락 모니터링 패턴
```javascript
func interlock_loop(axis, max_pos)
{
    while (@interlock_run == 1)
    {
        let cur_pos = get_pos(axis)
        if (cur_pos > max_pos)
        {
            all_stop()
            trip("인터락 트립: 축={axis}, 위치={cur_pos}, 제한={max_pos}")
        }
        sleep(50)
    }
}

// 인터락 감시 시작
@interlock_run = 1
let ih = spawn interlock_loop("Y", 500)

// 이동 명령
start_move_abs("Y", 0, 10000)

// 이동 중 인터락 확인
while (is_moving("Y") == 1 && is_running(ih) == 1)
{
    sleep(100)
}

// 인터락 감시 종료
@interlock_run = 0
await ih

print("완료: 위치 = " + get_pos("Y"))
```

### 병렬 작업
```javascript
func task_a()
{
    for i in 1..5
    {
        print("Task A: " + i)
        sleep(100)
    }
}

func task_b()
{
    for i in 1..5
    {
        print("Task B: " + i)
        sleep(150)
    }
}

// 두 작업을 동시에 실행
let h1 = spawn task_a()
let h2 = spawn task_b()

// 모두 완료될 때까지 대기
await h1
await h2
print("모든 작업 완료")
```

---

## 예제 코드

### 1. 간단한 계산기
```javascript
func calculator(op, a, b)
{
    if (op == "add")
    {
        return a + b
    }
    else if (op == "sub")
    {
        return a - b
    }
    else if (op == "mul")
    {
        return a * b
    }
    else if (op == "div")
    {
        assert(b != 0, "0으로 나눌 수 없습니다")
        return a / b
    }
    else
    {
        print("알 수 없는 연산자: " + op)
        return 0
    }
}

print(calculator("add", 10, 5))  // 15
print(calculator("div", 20, 4))  // 5
```

### 2. 원의 면적 계산
```javascript
func circle_area(radius)
{
    assert(radius > 0, "반지름은 양수여야 합니다")
    return pi() * pow(radius, 2)
}

let r = 5
let area = circle_area(r)
print("반지름 {r}인 원의 면적: {area}")
```

### 3. 각도를 이용한 좌표 계산
```javascript
func polar_to_cartesian(distance, angle_deg)
{
    let angle_rad = deg_to_rad(angle_deg)
    let x = distance * cos(angle_rad)
    let y = distance * sin(angle_rad)
    print("극좌표 (r={distance}, θ={angle_deg}°)")
    print("직교좌표 (x={x}, y={y})")
}

polar_to_cartesian(10, 45)   // 45도 방향, 거리 10
polar_to_cartesian(5, 90)    // 90도 방향, 거리 5
```

### 4. 문자열 처리
```javascript
func format_message(name, age)
{
    let upper_name = upper(name)
    let trimmed = trim(upper_name)
    let msg = "이름: " + trimmed + ", 나이: " + age
    return msg
}

let result = format_message("  alice  ", 25)
print(result)  // 이름: ALICE, 나이: 25

// 문자열 검색
let text = "Hello World"
if (contains(text, "World") == 1)
{
    let pos = index_of(text, "World")
    print("'World'의 위치: " + pos)  // 6
}
```

### 5. 반복 패턴
```javascript
// 피보나치 수열
func fibonacci(n)
{
    if (n <= 1)
    {
        return n
    }
    let a = 0
    let b = 1
    for i in 2..n
    {
        let temp = a + b
        a = b
        b = temp
    }
    return b
}

for i in 0..10
{
    print("fib({i}) = {}", fibonacci(i))
}
```

### 6. 랜덤 게임
```javascript
func guess_number()
{
    let target = random_int(1, 101)  // 1~100 사이 난수
    print("1부터 100 사이의 숫자를 맞춰보세요!")

    let tries = 0
    let guess = 50  // 초기 추측값

    while (guess != target && tries < 10)
    {
        tries = tries + 1
        print("시도 {tries}: {guess}")

        if (guess < target)
        {
            print("더 큽니다!")
            guess = guess + random_int(1, 20)
        }
        else
        {
            print("더 작습니다!")
            guess = guess - random_int(1, 20)
        }

        sleep(500)
    }

    if (guess == target)
    {
        print("정답! {tries}번 만에 맞췄습니다!")
    }
    else
    {
        print("실패! 정답은 {target}였습니다.")
    }
}

call guess_number()
```

### 7. 타입 체크
```javascript
func print_type_info(value)
{
    let type = typeof(value)
    print("값: {value}, 타입: {type}")

    if (is_number(value) == 1)
    {
        print("  -> 숫자입니다")
    }
    else if (is_string(value) == 1)
    {
        print("  -> 문자열입니다 (길이: {})", length(value))
    }
}

print_type_info(123)
print_type_info("Hello")
print_type_info(true)
```

---

## import / call / run / run_once 차이

### import
파일을 모듈로 로드하고 별칭으로 참조합니다. 모듈 멤버는 `alias.name`, 함수는 `alias.func(...)` 형태로 사용합니다.

```javascript
import "utils.scr" as utils
let x = utils.PI
call utils.add(1, 2)
```

### call
함수 또는 외부 스크립트를 동기 실행합니다.

```javascript
call my_function(1, 2)     // 함수 동기 호출
call "other_script.scr"    // 외부 스크립트 실행
```

### run
함수 또는 외부 스크립트를 비동기 실행합니다. (`spawn`과 동일한 동작)

```javascript
let h = run my_function(1, 2)
run "other_script.scr"     // 백그라운드 실행
await h
```

### run_once
함수 또는 외부 스크립트를 비동기 실행하되, 동일 대상이 이미 실행 중이면 중복 실행하지 않습니다.

```javascript
; 같은 interlock 스크립트를 여러 번 호출해도 1개만 유지
run_once "interlock_template.scr"
run_once "interlock_template.scr"

; 필요하면 핸들을 받아 상태 확인/대기 가능
let ih = run_once "interlock_template.scr"
if (is_running(ih) == 1)
{
    print("interlock already running")
}
```

### run_once_stop
`run_once`로 실행 중인 대상을 중지 요청합니다. 중지 요청 성공 시 `1`, 대상이 없으면 `0`을 반환합니다.

```javascript
run_once "interlock_template.scr"
sleep(100)

let stopped = run_once_stop "interlock_template.scr"
print("stopped={}", stopped)
```

---

## 인터락 표준 템플릿

```javascript
func check_interlock(axis, max_pos)
{
    let cur_pos = get_pos(axis)
    if (cur_pos > max_pos)
    {
        all_stop()
        trip("인터락 트립: 축={axis}, 위치={cur_pos}, 제한={max_pos}")
    }
    return cur_pos
}

func interlock_monitor(axis, max_pos)
{
    while (@interlock_run == 1)
    {
        check_interlock(axis, max_pos)
        sleep(50)
    }
}

// 사용 예
@interlock_run = 1
let ih = spawn interlock_monitor("Y", 500)

// 작업 수행...
move_abs("Y", 100, 5000)

// 인터락 해제
@interlock_run = 0
await ih
```

---

## 카메라 함수 확장 포인트
- 기본 예약 함수: `cam_trigger(...)`, `cam_grab(...)`
- 기본 상태에서는 동작하지 않고, 호스트 앱에서 UnifiedCamera 어댑터를 등록하면 활성화됩니다.

---

## 샘플 파일
프로젝트의 `RatelWPF/Scripts` 폴더에서 다음 예제 파일을 확인할 수 있습니다:

- `math_examples.scr` - 수학 함수 사용 예제
- `string_examples.scr` - 문자열 처리 예제
- `array_examples.scr` - 배열 사용 예제
- `trycatch_examples.scr` - 예외 처리 예제
- `async_examples.scr` - 비동기 패턴 예제
- `type_check_examples.scr` - 타입 체크 예제
- `interlock_template.scr` - 인터락 템플릿
- `main.scr` - 종합 예제

---

## 확장 포인트
- 문법 파일: `RatelSoft.Lib/FlowScenario/ScriptV3/Grammar/ScriptV3.g4`
- 런타임 엔진: `RatelSoft.Lib/FlowScenario/ScriptV3/ScriptV3Engine.cs`
- 호스트 함수 등록: `RatelWPF/Run/ScriptV3HostFunctionRegistrar.cs`

---

## 버전 정보
- Script v3 엔진
- 수학/문자열/타입 함수 지원 (2025)
- 배열 지원 추가 (2026)
- try-catch-finally 예외 처리 (2026)
- foreach 문 지원 (2026)
- ANTLR 기반 파서
