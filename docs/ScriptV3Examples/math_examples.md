# math_examples

Source: RatelWPF\Scripts\math_examples.scr

```rscript
// ===================================================
// Script v3 - 수학 함수 예제
// ===================================================

print("=== 수학 함수 예제 시작 ===")
print("")

// 1. 기본 수학 함수
print("--- 1. 기본 수학 함수 ---")
let x = -15.7
print("x = {x}")
print("abs(x) = {}", abs(x))          // 절댓값: 15.7
print("floor(x) = {}", floor(abs(x))) // 내림: 15
print("ceil(x) = {}", ceil(abs(x)))   // 올림: 16
print("round(x) = {}", round(abs(x))) // 반올림: 16
print("")

// 2. 제곱근과 거듭제곱
print("--- 2. 제곱근과 거듭제곱 ---")
let num = 16
print("sqrt({num}) = {}", sqrt(num))   // 4
print("pow(2, 8) = {}", pow(2, 8))     // 256
print("pow(10, 3) = {}", pow(10, 3))   // 1000
print("")

// 3. 삼각함수 (라디안)
print("--- 3. 삼각함수 ---")
let angle_rad = pi() / 4  // 45도
print("각도: 45도 (π/4 라디안)")
print("sin(π/4) = {}", round(sin(angle_rad), 4))
print("cos(π/4) = {}", round(cos(angle_rad), 4))
print("tan(π/4) = {}", round(tan(angle_rad), 4))
print("")

// 4. 각도 변환
print("--- 4. 각도 변환 ---")
let deg = 90
let rad = deg_to_rad(deg)
print("{deg}도 = {} 라디안", round(rad, 4))
print("sin({deg}°) = {}", round(sin(rad), 4))
print("cos({deg}°) = {}", round(cos(rad), 4))
print("")

let back_deg = rad_to_deg(rad)
print("{} 라디안 = {}도", round(rad, 4), round(back_deg, 0))
print("")

// 5. 역삼각함수
print("--- 5. 역삼각함수 ---")
print("asin(1) = {} 라디안", round(asin(1), 4))
print("acos(0) = {} 라디안", round(acos(0), 4))
print("atan(1) = {} 라디안", round(atan(1), 4))
print("atan2(1, 1) = {} 라디안", round(atan2(1, 1), 4))
print("")

// 6. 로그와 지수
print("--- 6. 로그와 지수 ---")
print("e = {}", round(e(), 5))
print("log(e) = {}", round(log(e()), 5))
print("log10(100) = {}", log10(100))
print("exp(1) = {}", round(exp(1), 5))
print("exp(2) = {}", round(exp(2), 5))
print("")

// 7. 최소/최대
print("--- 7. 최소/최대 ---")
print("min(3, 7, 2, 9, 1) = {}", min(3, 7, 2, 9, 1))
print("max(3, 7, 2, 9, 1) = {}", max(3, 7, 2, 9, 1))
print("")

// 8. 난수
print("--- 8. 난수 ---")
print("random() = {}", round(random(), 4))
print("random() = {}", round(random(), 4))
print("random_int(100) = {}", random_int(100))
print("random_int(10, 20) = {}", random_int(10, 20))
print("")

// 9. 원의 면적 계산
print("--- 9. 원의 면적 계산 ---")
func circle_area(radius)
{
    return pi() * pow(radius, 2)
}

let radius = 5
let area = circle_area(radius)
print("반지름 {radius}인 원의 면적: {}", round(area, 2))
print("")

// 10. 극좌표를 직교좌표로 변환
print("--- 10. 극좌표 → 직교좌표 ---")
func polar_to_cartesian(distance, angle_deg)
{
    let angle_rad = deg_to_rad(angle_deg)
    let x = distance * cos(angle_rad)
    let y = distance * sin(angle_rad)
    return x
}

let dist = 10
let angle = 45
let cx = polar_to_cartesian(dist, angle)
let cy = dist * sin(deg_to_rad(angle))
print("극좌표 (r={dist}, θ={angle}°)")
print("직교좌표 x={}, y={}", round(cx, 2), round(cy, 2))
print("")

// 11. 피타고라스 정리
print("--- 11. 피타고라스 정리 ---")
func hypotenuse(a, b)
{
    return sqrt(pow(a, 2) + pow(b, 2))
}

let a = 3
let b = 4
let c = hypotenuse(a, b)
print("직각삼각형: a={a}, b={b}")
print("빗변 c = {}", c)
print("")

// 12. 거리 계산 (2D)
print("--- 12. 두 점 사이의 거리 ---")
func distance_2d(x1, y1, x2, y2)
{
    let dx = x2 - x1
    let dy = y2 - y1
    return sqrt(pow(dx, 2) + pow(dy, 2))
}

let dist_result = distance_2d(0, 0, 3, 4)
print("점 (0, 0)과 (3, 4) 사이의 거리: {}", dist_result)
print("")

print("=== 수학 함수 예제 완료 ===")

```
