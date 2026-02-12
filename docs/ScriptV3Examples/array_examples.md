# array_examples

Source: RatelWPF\Scripts\array_examples.scr

```rscript
; ===========================================
; 배열 사용 예제 (Script v3)
; ===========================================

; -------------------------------------------
; 1. 배열 생성
; -------------------------------------------
print("=== 배열 생성 ===")

let numbers = [1, 2, 3, 4, 5]
print("숫자 배열: {}", join(numbers, ", "))

let names = ["Alice", "Bob", "Charlie"]
print("문자열 배열: {}", join(names, ", "))

let mixed = [1, "hello", true, 3.14]
print("혼합 배열: {}", join(mixed, ", "))

let empty = []
print("빈 배열 길이: {}", len(empty))

; -------------------------------------------
; 2. 배열 인덱스 접근
; -------------------------------------------
print("")
print("=== 배열 인덱스 접근 ===")

let arr = [10, 20, 30, 40, 50]
print("arr[0] = {}", arr[0])
print("arr[2] = {}", arr[2])
print("arr[4] = {}", arr[4])

; 값 변경
arr[2] = 300
print("arr[2] 변경 후: {}", arr[2])

; -------------------------------------------
; 3. foreach 순회
; -------------------------------------------
print("")
print("=== foreach 순회 ===")

let fruits = ["apple", "banana", "cherry"]
foreach fruit in fruits
{
    print("과일: {fruit}")
}

; break, continue 사용
print("")
print("=== break/continue 사용 ===")
let nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
foreach n in nums
{
    if (n == 3)
    {
        continue  ; 3 건너뛰기
    }
    if (n == 7)
    {
        break     ; 7에서 종료
    }
    print("숫자: {n}")
}

; -------------------------------------------
; 4. 배열 조작 함수
; -------------------------------------------
print("")
print("=== 배열 조작 ===")

let stack = [1, 2, 3]
print("초기 배열: {}", join(stack, ", "))

; push - 끝에 추가
push(stack, 4)
print("push(4) 후: {}", join(stack, ", "))

; pop - 끝에서 제거
let popped = pop(stack)
print("pop() 결과: {popped}, 배열: {}", join(stack, ", "))

; unshift - 앞에 추가
unshift(stack, 0)
print("unshift(0) 후: {}", join(stack, ", "))

; shift - 앞에서 제거
let shifted = shift(stack)
print("shift() 결과: {shifted}, 배열: {}", join(stack, ", "))

; -------------------------------------------
; 5. slice - 부분 배열
; -------------------------------------------
print("")
print("=== slice 사용 ===")

let data = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
print("원본: {}", join(data, ", "))
print("slice(2, 5): {}", join(slice(data, 2, 5), ", "))
print("slice(5): {}", join(slice(data, 5), ", "))
print("slice(-3): {}", join(slice(data, -3), ", "))

; -------------------------------------------
; 6. concat - 배열 합치기
; -------------------------------------------
print("")
print("=== concat 사용 ===")

let arr1 = [1, 2, 3]
let arr2 = [4, 5, 6]
let merged = concat(arr1, arr2)
print("concat 결과: {}", join(merged, ", "))

; -------------------------------------------
; 7. sort, reverse
; -------------------------------------------
print("")
print("=== sort, reverse ===")

let unsorted = [3, 1, 4, 1, 5, 9, 2, 6]
print("원본: {}", join(unsorted, ", "))
print("sort: {}", join(sort(unsorted), ", "))
print("reverse: {}", join(reverse(unsorted), ", "))

; -------------------------------------------
; 8. 유틸리티 함수
; -------------------------------------------
print("")
print("=== 유틸리티 함수 ===")

; array - 배열 생성
let zeros = array(5, 0)
print("array(5, 0): {}", join(zeros, ", "))

; range - 숫자 시퀀스
print("range(5): {}", join(range(5), ", "))
print("range(1, 6): {}", join(range(1, 6), ", "))
print("range(0, 10, 2): {}", join(range(0, 10, 2), ", "))

; -------------------------------------------
; 9. 실용 예제: 합계와 평균
; -------------------------------------------
print("")
print("=== 합계와 평균 계산 ===")

func sum_array(arr)
{
    let total = 0
    foreach x in arr
    {
        total = total + x
    }
    return total
}

func average(arr)
{
    if (len(arr) == 0)
    {
        return 0
    }
    return sum_array(arr) / len(arr)
}

let scores = [85, 92, 78, 95, 88]
print("점수: {}", join(scores, ", "))
print("합계: {}", sum_array(scores))
print("평균: {}", average(scores))

; -------------------------------------------
; 10. 실용 예제: 필터링
; -------------------------------------------
print("")
print("=== 필터링 예제 ===")

func filter_even(arr)
{
    let result = []
    foreach x in arr
    {
        if (x % 2 == 0)
        {
            push(result, x)
        }
    }
    return result
}

let all_nums = range(1, 11)
let evens = filter_even(all_nums)
print("1-10 중 짝수: {}", join(evens, ", "))

print("")
print("=== 배열 예제 완료 ===")

```
