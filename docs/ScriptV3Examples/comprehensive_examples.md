# comprehensive_examples

Source: RatelWPF\Scripts\comprehensive_examples.scr

```rscript
// ===================================================
// Script v3 - 종합 예제 모음
// ===================================================

print("=== Script v3 종합 예제 시작 ===")
print("")

// ===================================================
// 예제 1: 온도 변환기
// ===================================================
print("--- 예제 1: 온도 변환기 ---")

func celsius_to_fahrenheit(celsius)
{
    return celsius * 9 / 5 + 32
}

func fahrenheit_to_celsius(fahrenheit)
{
    return (fahrenheit - 32) * 5 / 9
}

let temp_c = 25
let temp_f = celsius_to_fahrenheit(temp_c)
print("{temp_c}°C = {}°F", round(temp_f, 1))

let temp_f2 = 77
let temp_c2 = fahrenheit_to_celsius(temp_f2)
print("{temp_f2}°F = {}°C", round(temp_c2, 1))
print("")

// ===================================================
// 예제 2: 간단한 통계 계산
// ===================================================
print("--- 예제 2: 통계 계산 ---")

func calculate_statistics(a, b, c, d, e)
{
    let sum = a + b + c + d + e
    let avg = sum / 5
    let minimum = min(a, b, c, d, e)
    let maximum = max(a, b, c, d, e)

    print("데이터: {a}, {b}, {c}, {d}, {e}")
    print("  합계: {sum}")
    print("  평균: {}", round(avg, 2))
    print("  최소: {minimum}")
    print("  최대: {maximum}")
}

calculate_statistics(10, 25, 15, 30, 20)
print("")

// ===================================================
// 예제 3: 사칙연산 계산기
// ===================================================
print("--- 예제 3: 계산기 ---")

func calculator(operation, a, b)
{
    if (operation == "add")
    {
        return a + b
    }
    else if (operation == "sub")
    {
        return a - b
    }
    else if (operation == "mul")
    {
        return a * b
    }
    else if (operation == "div")
    {
        assert(b != 0, "0으로 나눌 수 없습니다")
        return a / b
    }
    else if (operation == "pow")
    {
        return pow(a, b)
    }
    else if (operation == "mod")
    {
        return a % b
    }
    else
    {
        print("알 수 없는 연산: {operation}")
        return 0
    }
}

print("10 + 5 = {}", calculator("add", 10, 5))
print("10 - 3 = {}", calculator("sub", 10, 3))
print("10 * 4 = {}", calculator("mul", 10, 4))
print("10 / 2 = {}", calculator("div", 10, 2))
print("2 ^ 8 = {}", calculator("pow", 2, 8))
print("10 % 3 = {}", calculator("mod", 10, 3))
print("")

// ===================================================
// 예제 4: 기하학 계산
// ===================================================
print("--- 예제 4: 기하학 계산 ---")

func circle_properties(radius)
{
    let area = pi() * pow(radius, 2)
    let circumference = 2 * pi() * radius
    print("원 (반지름={radius})")
    print("  면적: {}", round(area, 2))
    print("  둘레: {}", round(circumference, 2))
}

func rectangle_properties(width, height)
{
    let area = width * height
    let perimeter = 2 * (width + height)
    let diagonal = sqrt(pow(width, 2) + pow(height, 2))
    print("직사각형 (가로={width}, 세로={height})")
    print("  면적: {}", area)
    print("  둘레: {}", perimeter)
    print("  대각선: {}", round(diagonal, 2))
}

circle_properties(5)
print("")
rectangle_properties(4, 3)
print("")

// ===================================================
// 예제 5: 문자열 검증 및 처리
// ===================================================
print("--- 예제 5: 문자열 검증 ---")

func validate_username(username)
{
    let len = length(username)
    print("사용자명: '{username}'")

    if (len < 3)
    {
        print("  ✗ 너무 짧습니다 (최소 3자)")
        return 0
    }
    if (len > 20)
    {
        print("  ✗ 너무 깁니다 (최대 20자)")
        return 0
    }
    if (contains(username, " ") == 1)
    {
        print("  ✗ 공백을 포함할 수 없습니다")
        return 0
    }

    print("  ✓ 유효한 사용자명입니다")
    return 1
}

validate_username("jo")
validate_username("john_doe")
validate_username("very_long_username_that_exceeds_limit")
validate_username("user name")
print("")

// ===================================================
// 예제 6: 데이터 형식 변환
// ===================================================
print("--- 예제 6: 데이터 형식 변환 ---")

func format_bytes(bytes)
{
    if (bytes < 1024)
    {
        return to_string(bytes) + " B"
    }
    else if (bytes < 1048576)
    {
        let kb = bytes / 1024
        return to_string(round(kb, 2)) + " KB"
    }
    else if (bytes < 1073741824)
    {
        let mb = bytes / 1048576
        return to_string(round(mb, 2)) + " MB"
    }
    else
    {
        let gb = bytes / 1073741824
        return to_string(round(gb, 2)) + " GB"
    }
}

print("512 bytes = {}", format_bytes(512))
print("2048 bytes = {}", format_bytes(2048))
print("1048576 bytes = {}", format_bytes(1048576))
print("5368709120 bytes = {}", format_bytes(5368709120))
print("")

// ===================================================
// 예제 7: 시간 형식 변환
// ===================================================
print("--- 예제 7: 시간 형식 변환 ---")

func format_seconds(total_seconds)
{
    let hours = floor(total_seconds / 3600)
    let remaining = total_seconds % 3600
    let minutes = floor(remaining / 60)
    let seconds = remaining % 60

    return to_string(hours) + ":" + to_string(minutes) + ":" + to_string(seconds)
}

print("3661초 = {}", format_seconds(3661))
print("7325초 = {}", format_seconds(7325))
print("")

// ===================================================
// 예제 8: 패스워드 강도 체크
// ===================================================
print("--- 예제 8: 패스워드 강도 ---")

func check_password_strength(password)
{
    let len = length(password)
    let strength = 0

    print("패스워드: '{password}'")

    if (len >= 8)
    {
        strength = strength + 1
        print("  ✓ 길이 충분 (8자 이상)")
    }
    else
    {
        print("  ✗ 너무 짧음 (8자 미만)")
    }

    if (len >= 12)
    {
        strength = strength + 1
        print("  ✓ 강한 길이 (12자 이상)")
    }

    let has_upper = 0
    if (upper(password) != password)
    {
        has_upper = 1
    }

    let has_lower = 0
    if (lower(password) != password)
    {
        has_lower = 1
    }

    if (has_upper == 1 && has_lower == 1)
    {
        strength = strength + 1
        print("  ✓ 대소문자 혼합")
    }

    if (contains(password, "1") == 1 || contains(password, "2") == 1)
    {
        strength = strength + 1
        print("  ✓ 숫자 포함")
    }

    print("  강도: {strength}/4")
    return strength
}

check_password_strength("pass")
print("")
check_password_strength("Password123")
print("")

// ===================================================
// 예제 9: 간단한 암호화 (시저 암호 개념)
// ===================================================
print("--- 예제 9: 문자열 변환 ---")

func simple_encode(text)
{
    let encoded = upper(text)
    encoded = replace(encoded, "A", "4")
    encoded = replace(encoded, "E", "3")
    encoded = replace(encoded, "I", "1")
    encoded = replace(encoded, "O", "0")
    encoded = replace(encoded, "S", "5")
    return encoded
}

let original = "Hello World"
let encoded = simple_encode(original)
print("원본: '{original}'")
print("변환: '{encoded}'")
print("")

// ===================================================
// 예제 10: 진행률 표시
// ===================================================
print("--- 예제 10: 진행률 표시 ---")

func show_progress(current, total)
{
    let percent = current * 100 / total
    let bar_length = 20
    let filled = floor(bar_length * current / total)

    let bar = ""
    for i in 1..filled
    {
        bar = bar + "="
    }
    for i in 1..(bar_length - filled)
    {
        bar = bar + " "
    }

    print("[{bar}] {}%", round(percent, 0))
}

for i in 0..10
{
    show_progress(i, 10)
    sleep(100)
}
print("")

// ===================================================
// 예제 11: 랜덤 데이터 생성
// ===================================================
print("--- 예제 11: 랜덤 데이터 ---")

func generate_random_id(length)
{
    let chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    let id = ""

    for i in 1..length
    {
        let idx = random_int(36)
        let char = substring(chars, idx, 1)
        id = id + char
    }

    return id
}

print("랜덤 ID 생성:")
for i in 1..5
{
    print("  {}", generate_random_id(8))
}
print("")

// ===================================================
// 예제 12: 데이터 검증 및 정리
// ===================================================
print("--- 예제 12: 데이터 정리 ---")

func clean_and_validate(input)
{
    print("입력: '{input}'")

    let cleaned = trim(input)
    print("  공백 제거: '{cleaned}'")

    if (length(cleaned) == 0)
    {
        print("  ✗ 빈 문자열")
        return ""
    }

    cleaned = lower(cleaned)
    print("  소문자 변환: '{cleaned}'")

    if (contains(cleaned, "bad") == 1)
    {
        print("  ✗ 금지된 단어 포함")
        return ""
    }

    print("  ✓ 검증 통과")
    return cleaned
}

clean_and_validate("  Hello World  ")
print("")
clean_and_validate("   ")
print("")
clean_and_validate("bad word")
print("")

print("=== Script v3 종합 예제 완료 ===")

```
