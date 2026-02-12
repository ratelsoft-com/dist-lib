# trycatch_examples

Source: RatelWPF\Scripts\trycatch_examples.scr

```rscript
; ===========================================
; 예외 처리 예제 (Script v3)
; ===========================================

; -------------------------------------------
; 1. 기본 try-catch
; -------------------------------------------
print("=== 기본 try-catch ===")

try
{
    print("try 블록 시작")
    let x = 10 / 0  ; 0으로 나누기 오류
    print("이 줄은 실행되지 않음")
}
catch e
{
    print("오류 발생: {e}")
}

print("프로그램 계속 실행")

; -------------------------------------------
; 2. throw 사용
; -------------------------------------------
print("")
print("=== throw 사용 ===")

func validate_age(age)
{
    if (age < 0)
    {
        throw "나이는 음수일 수 없습니다"
    }
    if (age > 150)
    {
        throw "나이가 너무 큽니다"
    }
    return age
}

try
{
    let age = validate_age(-5)
}
catch e
{
    print("검증 실패: {e}")
}

try
{
    let age = validate_age(200)
}
catch e
{
    print("검증 실패: {e}")
}

try
{
    let age = validate_age(25)
    print("유효한 나이: {age}")
}
catch e
{
    print("검증 실패: {e}")
}

; -------------------------------------------
; 3. try-catch-finally
; -------------------------------------------
print("")
print("=== try-catch-finally ===")

func process_with_cleanup()
{
    try
    {
        print("리소스 열기...")
        print("작업 수행...")
        throw "작업 중 오류 발생"
    }
    catch e
    {
        print("오류 처리: {e}")
    }
    finally
    {
        print("리소스 정리 (항상 실행)")
    }
}

process_with_cleanup()

; 성공 시에도 finally 실행
print("")
print("=== 성공 시 finally ===")

try
{
    print("성공적인 작업 수행")
    let result = 10 + 20
    print("결과: {result}")
}
catch e
{
    print("오류: {e}")
}
finally
{
    print("finally 블록 실행됨")
}

; -------------------------------------------
; 4. 오류 변수 없이 catch
; -------------------------------------------
print("")
print("=== 오류 변수 생략 ===")

try
{
    throw "무시할 오류"
}
catch
{
    print("오류가 발생했지만 상세 정보 무시")
}

; -------------------------------------------
; 5. 중첩 try-catch
; -------------------------------------------
print("")
print("=== 중첩 try-catch ===")

try
{
    print("외부 try 시작")
    try
    {
        print("내부 try 시작")
        throw "내부 오류"
    }
    catch inner_e
    {
        print("내부 catch: {inner_e}")
        throw "외부로 전파된 오류"
    }
}
catch outer_e
{
    print("외부 catch: {outer_e}")
}

; -------------------------------------------
; 6. 실용 예제: 안전한 나눗셈
; -------------------------------------------
print("")
print("=== 안전한 나눗셈 ===")

func safe_divide(a, b)
{
    if (b == 0)
    {
        throw "0으로 나눌 수 없습니다"
    }
    return a / b
}

func calculate_ratio(x, y)
{
    try
    {
        let ratio = safe_divide(x, y)
        return ratio
    }
    catch e
    {
        print("계산 오류: {e}")
        return 0  ; 기본값 반환
    }
}

print("10 / 2 = {}", calculate_ratio(10, 2))
print("10 / 0 = {}", calculate_ratio(10, 0))

; -------------------------------------------
; 7. 실용 예제: 배열 안전 접근
; -------------------------------------------
print("")
print("=== 배열 안전 접근 ===")

func safe_get(arr, index, default_val)
{
    try
    {
        return arr[index]
    }
    catch e
    {
        return default_val
    }
}

let data = [10, 20, 30]
print("data[1] = {}", safe_get(data, 1, -1))
print("data[10] = {}", safe_get(data, 10, -1))

; -------------------------------------------
; 8. 실용 예제: 여러 작업 시도
; -------------------------------------------
print("")
print("=== 여러 작업 시도 ===")

func try_operations()
{
    let results = []

    ; 작업 1
    try
    {
        push(results, "작업1 성공")
    }
    catch e
    {
        push(results, "작업1 실패: " + e)
    }

    ; 작업 2 (실패)
    try
    {
        throw "의도적 오류"
    }
    catch e
    {
        push(results, "작업2 실패: " + e)
    }

    ; 작업 3
    try
    {
        push(results, "작업3 성공")
    }
    catch e
    {
        push(results, "작업3 실패: " + e)
    }

    return results
}

let ops = try_operations()
foreach op in ops
{
    print("- {op}")
}

; -------------------------------------------
; 9. 실용 예제: 재시도 패턴
; -------------------------------------------
print("")
print("=== 재시도 패턴 ===")

func unstable_operation()
{
    let chance = random()
    if (chance < 0.7)
    {
        throw "일시적 오류"
    }
    return "성공"
}

func retry(max_attempts)
{
    let attempt = 0
    while (attempt < max_attempts)
    {
        attempt = attempt + 1
        try
        {
            let result = unstable_operation()
            print("시도 {attempt}: {result}")
            return result
        }
        catch e
        {
            print("시도 {attempt}: 실패 - {e}")
            if (attempt < max_attempts)
            {
                sleep(100)
            }
        }
    }
    throw "최대 재시도 횟수 초과"
}

try
{
    let result = retry(5)
    print("최종 결과: {result}")
}
catch e
{
    print("모든 시도 실패: {e}")
}

print("")
print("=== 예외 처리 예제 완료 ===")

```
