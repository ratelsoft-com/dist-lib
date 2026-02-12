# async_examples

Source: RatelWPF\Scripts\async_examples.scr

```rscript
// ===================================================
// Script v3 - 비동기 패턴 예제
// ===================================================

print("=== 비동기 패턴 예제 시작 ===")
print("")

// 1. 기본 spawn/await
print("--- 1. 기본 spawn/await ---")
func count_task(name, max)
{
    for i in 1..max
    {
        print("{name}: {i}")
        sleep(100)
    }
    return max
}

let handle1 = spawn count_task("작업A", 5)
print("작업A가 백그라운드에서 실행 중...")
sleep(300)
print("메인 스레드는 다른 작업 수행 가능")
let result1 = await handle1
print("작업A 완료. 결과: {result1}")
print("")

// 2. 병렬 실행
print("--- 2. 병렬 실행 ---")
func task_a()
{
    for i in 1..3
    {
        print("  Task-A [{i}]")
        sleep(150)
    }
    return "A완료"
}

func task_b()
{
    for i in 1..3
    {
        print("  Task-B [{i}]")
        sleep(200)
    }
    return "B완료"
}

print("두 작업을 동시에 시작...")
let ha = spawn task_a()
let hb = spawn task_b()

print("메인 스레드: 대기 중...")
let ra = await ha
let rb = await hb
print("결과: {ra}, {rb}")
print("")

// 3. is_running으로 상태 확인
print("--- 3. 실행 상태 확인 ---")
func long_running_task(seconds)
{
    print("  긴 작업 시작 ({seconds}초)...")
    sleep(seconds * 1000)
    print("  긴 작업 완료!")
    return seconds
}

let h = spawn long_running_task(2)
print("작업 실행 중...")

let count = 0
while (is_running(h) == 1)
{
    count = count + 1
    print("  대기 중... ({count})")
    sleep(500)
}

let result = await h
print("작업 완료. 결과: {result}")
print("")

// 4. 여러 작업 동시 모니터링
print("--- 4. 여러 작업 모니터링 ---")
func worker(id, duration)
{
    print("  Worker-{id} 시작")
    sleep(duration)
    print("  Worker-{id} 완료")
    return id
}

let h1 = spawn worker(1, 500)
let h2 = spawn worker(2, 1000)
let h3 = spawn worker(3, 1500)

print("세 작업 모니터링 중...")
while (is_running(h1) == 1 || is_running(h2) == 1 || is_running(h3) == 1)
{
    let status1 = is_running(h1)
    let status2 = is_running(h2)
    let status3 = is_running(h3)
    print("  상태: W1={status1}, W2={status2}, W3={status3}")
    sleep(300)
}

await h1
await h2
await h3
print("모든 작업 완료")
print("")

// 5. 조건부 대기
print("--- 5. 조건부 대기 ---")
@counter = 0

func increment_counter()
{
    for i in 1..10
    {
        @counter = @counter + 1
        print("  카운터: {@counter}")
        sleep(200)
    }
    return @counter
}

let h_counter = spawn increment_counter()

print("카운터가 5가 될 때까지 대기...")
while (@counter < 5 && is_running(h_counter) == 1)
{
    sleep(100)
}

print("카운터가 5에 도달! 현재: {@counter}")
print("작업 완료 대기...")
await h_counter
print("최종 카운터: {@counter}")
print("")

// 6. 타임아웃 패턴 (수동 구현)
print("--- 6. 타임아웃 패턴 ---")
func slow_operation()
{
    print("  느린 작업 시작...")
    sleep(3000)
    print("  느린 작업 완료")
    return "완료"
}

let h_slow = spawn slow_operation()
let timeout_ms = 1000
let elapsed = 0

print("최대 {timeout_ms}ms 대기...")
while (is_running(h_slow) == 1 && elapsed < timeout_ms)
{
    sleep(100)
    elapsed = elapsed + 100
}

if (is_running(h_slow) == 1)
{
    print("타임아웃! 작업이 아직 실행 중입니다")
}
else
{
    let result_slow = await h_slow
    print("작업 완료: {result_slow}")
}
print("")

// 7. 생산자-소비자 패턴 (단순화)
print("--- 7. 생산자-소비자 패턴 ---")
@queue_size = 0
@producer_done = 0

func producer()
{
    for i in 1..5
    {
        @queue_size = @queue_size + 1
        print("  [생산] 항목 추가. 큐 크기: {@queue_size}")
        sleep(300)
    }
    @producer_done = 1
    print("  [생산] 완료")
}

func consumer()
{
    while (@producer_done == 0 || @queue_size > 0)
    {
        if (@queue_size > 0)
        {
            @queue_size = @queue_size - 1
            print("  [소비] 항목 처리. 큐 크기: {@queue_size}")
            sleep(500)
        }
        else
        {
            sleep(100)
        }
    }
    print("  [소비] 완료")
}

let h_prod = spawn producer()
let h_cons = spawn consumer()

print("생산자-소비자 실행 중...")
await h_prod
await h_cons
print("생산자-소비자 완료")
print("")

// 8. 반복 작업 with 취소 플래그
print("--- 8. 반복 작업 with 취소 ---")
@keep_running = 1

func periodic_task()
{
    let iteration = 0
    while (@keep_running == 1)
    {
        iteration = iteration + 1
        print("  주기 작업 #{iteration}")
        sleep(200)
    }
    print("  주기 작업 종료")
    return iteration
}

let h_periodic = spawn periodic_task()
print("주기 작업 시작...")
sleep(1000)

print("작업 중지 요청...")
@keep_running = 0
let iterations = await h_periodic
print("주기 작업 완료. 반복 횟수: {iterations}")
print("")

// 9. 계산 작업 백그라운드 실행
print("--- 9. 무거운 계산 ---")
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

func calculate_fibonacci(max)
{
    print("  피보나치 계산 시작...")
    let result = fibonacci(max)
    print("  피보나치({max}) = {result}")
    return result
}

let h_fib = spawn calculate_fibonacci(30)
print("계산이 백그라운드에서 진행 중...")
print("메인: 다른 작업 수행 가능")
let fib_result = await h_fib
print("계산 완료: {fib_result}")
print("")

// 10. 에러 시뮬레이션
print("--- 10. 조건 기반 종료 ---")
@error_flag = 0

func monitored_task()
{
    for i in 1..10
    {
        if (@error_flag == 1)
        {
            print("  오류 감지! 작업 중단")
            return -1
        }
        print("  작업 진행 중... {i}/10")
        sleep(200)
    }
    return 10
}

let h_mon = spawn monitored_task()
sleep(500)

print("오류 발생 시뮬레이션...")
@error_flag = 1

let mon_result = await h_mon
if (mon_result == -1)
{
    print("작업이 오류로 인해 중단되었습니다")
}
else
{
    print("작업 정상 완료: {mon_result}")
}
print("")

print("=== 비동기 패턴 예제 완료 ===")

```
