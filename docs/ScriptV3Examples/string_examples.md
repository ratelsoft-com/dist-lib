# string_examples

Source: RatelWPF\Scripts\string_examples.scr

```rscript
// ===================================================
// Script v3 - 문자열 함수 예제
// ===================================================

print("=== 문자열 함수 예제 시작 ===")
print("")

// 1. 문자열 길이
print("--- 1. 문자열 길이 ---")
let text1 = "Hello World"
print("문자열: '{text1}'")
print("길이: {}", length(text1))
print("")

// 2. 부분 문자열 추출
print("--- 2. 부분 문자열 추출 ---")
let text2 = "Hello World"
print("원본: '{text2}'")
print("substring(0, 5): '{}'", substring(text2, 0, 5))
print("substring(6): '{}'", substring(text2, 6))
print("")

// 3. 대소문자 변환
print("--- 3. 대소문자 변환 ---")
let text3 = "Hello World"
print("원본: '{text3}'")
print("upper(): '{}'", upper(text3))
print("lower(): '{}'", lower(text3))
print("")

// 4. 공백 제거
print("--- 4. 공백 제거 ---")
let text4 = "   Hello World   "
print("원본: '{text4}'")
print("trim(): '{}'", trim(text4))
print("")

// 5. 문자열 치환
print("--- 5. 문자열 치환 ---")
let text5 = "Hello World"
print("원본: '{text5}'")
print("replace('World', 'Script'): '{}'", replace(text5, "World", "Script"))
print("replace('o', '0'): '{}'", replace(text5, "o", "0"))
print("")

// 6. 부분 문자열 포함 여부
print("--- 6. 부분 문자열 검사 ---")
let text6 = "Hello World"
print("문자열: '{text6}'")
if (contains(text6, "World") == 1)
{
    print("'World'를 포함합니다")
}
if (contains(text6, "Script") == 0)
{
    print("'Script'를 포함하지 않습니다")
}
print("")

// 7. 시작/끝 문자열 확인
print("--- 7. 시작/끝 확인 ---")
let text7 = "Hello World"
print("문자열: '{text7}'")
print("starts_with('He'): {}", starts_with(text7, "He"))
print("starts_with('Wo'): {}", starts_with(text7, "Wo"))
print("ends_with('ld'): {}", ends_with(text7, "ld"))
print("ends_with('He'): {}", ends_with(text7, "He"))
print("")

// 8. 문자열 위치 찾기
print("--- 8. 문자열 위치 찾기 ---")
let text8 = "Hello World"
print("문자열: '{text8}'")
print("index_of('World'): {}", index_of(text8, "World"))
print("index_of('o'): {}", index_of(text8, "o"))
print("index_of('Script'): {}", index_of(text8, "Script"))
print("")

// 9. 문자열 분할
print("--- 9. 문자열 분할 ---")
let csv = "apple,banana,cherry"
print("원본: '{csv}'")
let parts = split(csv, ",")
print("split(',') 결과: '{parts}'")
print("")

let sentence = "Hello World Script"
let words = split(sentence, " ")
print("원본: '{sentence}'")
print("split(' ') 결과: '{words}'")
print("")

// 10. 문자열 조합 예제
print("--- 10. 문자열 처리 함수 ---")
func format_name(first, last)
{
    let full = first + " " + last
    let upper_full = upper(full)
    let trimmed = trim(upper_full)
    return trimmed
}

let result = format_name("  john  ", "  doe  ")
print("format_name('  john  ', '  doe  ') = '{result}'")
print("")

// 11. 이메일 검증 (간단한 예제)
print("--- 11. 이메일 검증 ---")
func is_valid_email(email)
{
    if (contains(email, "@") == 0)
    {
        return 0
    }
    let at_pos = index_of(email, "@")
    if (at_pos <= 0)
    {
        return 0
    }
    let after_at = substring(email, at_pos + 1)
    if (contains(after_at, ".") == 0)
    {
        return 0
    }
    return 1
}

let email1 = "user@example.com"
let email2 = "invalid-email"
print("'{email1}' 유효: {}", is_valid_email(email1))
print("'{email2}' 유효: {}", is_valid_email(email2))
print("")

// 12. 문자열 반복 생성
print("--- 12. 문자열 반복 ---")
func repeat_string(str, count)
{
    let result = ""
    for i in 1..count
    {
        result = result + str
    }
    return result
}

let repeated = repeat_string("*", 10)
print("repeat_string('*', 10) = '{repeated}'")
print("")

// 13. 단어 카운트
print("--- 13. 단어 카운트 ---")
func count_words(text)
{
    let parts = split(text, " ")
    let count = 0
    let i = 0
    while (contains(parts, "|") == 1)
    {
        count = count + 1
        let pos = index_of(parts, "|")
        parts = substring(parts, pos + 1)
    }
    count = count + 1
    return count
}

let sentence2 = "The quick brown fox jumps"
print("문장: '{sentence2}'")
print("단어 수: {}", count_words(sentence2))
print("")

// 14. 센서 이름 파싱
print("--- 14. 센서 이름 파싱 ---")
func parse_sensor_name(full_name)
{
    if (contains(full_name, "_") == 0)
    {
        return full_name
    }
    let pos = index_of(full_name, "_")
    let prefix = substring(full_name, 0, pos)
    let suffix = substring(full_name, pos + 1)
    print("전체: '{full_name}'")
    print("  - 접두사: '{prefix}'")
    print("  - 접미사: '{suffix}'")
    return prefix
}

let sensor = "TEMP_SENSOR_01"
parse_sensor_name(sensor)
print("")

// 15. 문자열 패딩
print("--- 15. 문자열 패딩 ---")
func pad_left(str, width, pad_char)
{
    let len = length(str)
    if (len >= width)
    {
        return str
    }
    let padding = ""
    for i in 1..(width - len)
    {
        padding = padding + pad_char
    }
    return padding + str
}

let num = "42"
let padded = pad_left(num, 5, "0")
print("pad_left('{num}', 5, '0') = '{padded}'")
print("")

print("=== 문자열 함수 예제 완료 ===")

```
