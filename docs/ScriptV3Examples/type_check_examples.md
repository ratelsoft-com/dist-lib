# type_check_examples

Source: RatelWPF\Scripts\type_check_examples.scr

```rscript
// ===================================================
// Script v3 - 타입 체크 및 변환 예제
// ===================================================

print("=== 타입 체크 및 변환 예제 시작 ===")
print("")

// 1. 타입 체크 기본
print("--- 1. 타입 체크 기본 ---")
let num = 123
let str = "Hello"
let flag = true

print("num = {num}")
print("  is_number: {}", is_number(num))
print("  is_string: {}", is_string(num))
print("  is_bool: {}", is_bool(num))
print("")

print("str = '{str}'")
print("  is_number: {}", is_number(str))
print("  is_string: {}", is_string(str))
print("  is_bool: {}", is_bool(str))
print("")

print("flag = {flag}")
print("  is_number: {}", is_number(flag))
print("  is_string: {}", is_string(flag))
print("  is_bool: {}", is_bool(flag))
print("")

// 2. typeof 함수
print("--- 2. typeof 함수 ---")
print("typeof(123) = '{}'", typeof(123))
print("typeof('hello') = '{}'", typeof("hello"))
print("typeof(true) = '{}'", typeof(true))
print("")

// 3. 타입 변환
print("--- 3. 타입 변환 ---")
let str_num = "456"
let converted = to_number(str_num)
print("문자열 '{str_num}' → 숫자: {}", converted)
print("숫자 × 2 = {}", converted * 2)
print("")

let num2 = 789
let str2 = to_string(num2)
print("숫자 {num2} → 문자열: '{str2}'")
print("문자열 길이: {}", length(str2))
print("")

print("to_bool(1) = {}", to_bool(1))
print("to_bool(0) = {}", to_bool(0))
print("to_bool('true') = {}", to_bool("true"))
print("to_bool('false') = {}", to_bool("false"))
print("")

// 4. 동적 타입 처리 함수
print("--- 4. 동적 타입 처리 ---")
func process_value(value)
{
    let type = typeof(value)
    print("값: {value}, 타입: {type}")

    if (is_number(value) == 1)
    {
        print("  → 숫자입니다. 제곱: {}", value * value)
    }
    else if (is_string(value) == 1)
    {
        print("  → 문자열입니다. 길이: {}", length(value))
        print("  → 대문자: {}", upper(value))
    }
    else if (is_bool(value) == 1)
    {
        if (value == true)
        {
            print("  → 참(true)입니다")
        }
        else
        {
            print("  → 거짓(false)입니다")
        }
    }
    print("")
}

process_value(42)
process_value("Script")
process_value(true)
process_value(false)

// 5. 안전한 숫자 변환
print("--- 5. 안전한 숫자 변환 ---")
func safe_to_number(value, default_val)
{
    if (is_number(value) == 1)
    {
        return value
    }

    if (is_string(value) == 1)
    {
        let str = trim(value)
        if (length(str) > 0)
        {
            return to_number(str)
        }
    }

    return default_val
}

print("safe_to_number(123, 0) = {}", safe_to_number(123, 0))
print("safe_to_number('456', 0) = {}", safe_to_number("456", 0))
print("safe_to_number('', 999) = {}", safe_to_number("", 999))
print("")

// 6. 입력 검증
print("--- 6. 입력 검증 ---")
func validate_input(value, expected_type)
{
    let type = typeof(value)
    if (type == expected_type)
    {
        print("검증 성공: 값={value}, 타입={type}")
        return 1
    }
    else
    {
        print("검증 실패: 값={value}, 예상={expected_type}, 실제={type}")
        return 0
    }
}

validate_input(100, "number")
validate_input("test", "string")
validate_input(100, "string")
print("")

// 7. 범위 체크 함수
print("--- 7. 범위 체크 ---")
func check_range(value, min_val, max_val)
{
    if (is_number(value) == 0)
    {
        print("오류: 숫자가 아닙니다 - {value}")
        return 0
    }

    if (value < min_val)
    {
        print("경고: {value}는 최소값 {min_val}보다 작습니다")
        return 0
    }

    if (value > max_val)
    {
        print("경고: {value}는 최대값 {max_val}보다 큽니다")
        return 0
    }

    print("OK: {value}는 범위 [{min_val}, {max_val}] 내에 있습니다")
    return 1
}

check_range(50, 0, 100)
check_range(-10, 0, 100)
check_range(150, 0, 100)
print("")

// 8. 데이터 정규화
print("--- 8. 데이터 정규화 ---")
func normalize_data(value)
{
    if (is_string(value) == 1)
    {
        let trimmed = trim(value)
        let lower = lower(trimmed)
        return lower
    }
    else if (is_number(value) == 1)
    {
        return round(value, 2)
    }
    else
    {
        return value
    }
}

print("normalize_data('  HELLO  ') = '{}'", normalize_data("  HELLO  "))
print("normalize_data(3.14159) = {}", normalize_data(3.14159))
print("normalize_data(true) = {}", normalize_data(true))
print("")

// 9. 설정 값 파싱
print("--- 9. 설정 값 파싱 ---")
func parse_config_value(key, value)
{
    print("설정: {key} = {value}")

    if (is_number(value) == 1)
    {
        print("  → 숫자 타입")
        return value
    }

    if (is_string(value) == 1)
    {
        let lower_val = lower(value)
        if (lower_val == "true")
        {
            print("  → 불린 타입 (true)")
            return true
        }
        else if (lower_val == "false")
        {
            print("  → 불린 타입 (false)")
            return false
        }
        else
        {
            print("  → 문자열 타입")
            return value
        }
    }

    return value
}

parse_config_value("timeout", 5000)
parse_config_value("enabled", "true")
parse_config_value("name", "system")
print("")

// 10. null 체크
print("--- 10. null 체크 ---")
func safe_print(value, default_msg)
{
    if (is_null(value) == 1)
    {
        print(default_msg)
    }
    else
    {
        print("값: {value}")
    }
}

let val1 = 123
safe_print(val1, "값이 없습니다")
print("")

// 11. 타입별 포맷팅
print("--- 11. 타입별 포맷팅 ---")
func format_value(value)
{
    if (is_number(value) == 1)
    {
        return "[숫자] " + to_string(round(value, 2))
    }
    else if (is_string(value) == 1)
    {
        return "[문자열] '" + value + "'"
    }
    else if (is_bool(value) == 1)
    {
        if (value == true)
        {
            return "[불린] 참"
        }
        else
        {
            return "[불린] 거짓"
        }
    }
    else
    {
        return "[알 수 없음]"
    }
}

print(format_value(3.14159))
print(format_value("Hello"))
print(format_value(true))
print(format_value(false))
print("")

print("=== 타입 체크 및 변환 예제 완료 ===")

```
