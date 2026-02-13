# Script Console JavaScript Quickstart

## 개요

Script Console은 기존 ScriptV3(`.scr`)와 함께 JavaScript(`.js`) 실행을 지원합니다.

- 언어 선택: Script Console 하단 `Lang`
  - `ScriptV3`
  - `JavaScript`

## 빠른 시작

1. Script Console 열기
- `RatelWPF > File > Script Console`

2. 언어 선택
- `Lang = JavaScript`

3. 샘플 로드/실행
- `RatelWPF/Scripts/javascript_basic.js`를 `Load` 후 `Run`

## 기본 예제

```javascript
function add(a, b) {
    return a + b;
}

const result = add(1, 2);
print(`result=${result}`);
ratel.info("done");
```

## Host API (기본)

- `ratel.info(message)`
- `ratel.warn(message)`
- `ratel.error(message)`
- `ratel.sleep(ms)`
- `ratel.getVar(name)`
- `ratel.setVar(name, value)`
- `ratel.hasVar(name)`
- `ratel.run(path)` / `ratel.spawn(path)`
- `ratel.spawnFunc("functionName", ...args)` (한 파일 함수 비동기 실행)
- `ratel.await(handle)` / `ratel.awaitResult(handle)`
- `ratel.isRunning(handle)`
- `ratel.runOnce(key, path)` / `ratel.runOnceStop(key)`

장비/IO 함수는 호스트 앱 등록 상태에 따라 전역 함수로 사용 가능합니다.

- 예: `move_abs(...)`, `get_in(...)`, `set_out(...)`

### 공용변수 예제 (배열 포함)

```javascript
ratel.setVar("shared.items", [1, 2, 3]);
const items = ratel.getVar("shared.items");
if (ratel.hasVar("shared.items") === 1) {
  print(`items[0]=${items[0]}`);
}
```

## 현재 제한

- JavaScript 디버그(F10 Step)는 아직 미지원 (Run 모드 사용)
- ScriptV3 런타임 모니터 일부 항목은 JS 실행 항목과 표시 방식이 다를 수 있음
- `spawnFunc`는 현재 파일 기반 스크립트에서 함수명 문자열 호출을 권장합니다.
  - 권장: `ratel.spawnFunc("worker", 1, 2)`
  - 비권장: `ratel.spawnFunc(worker, 1, 2)` (런타임 제약으로 실패 가능)

## 관련 문서

- ScriptV3 매뉴얼: `SCRIPT_V3_MANUAL.md`
- Script 언어 방향: `SCRIPT_V3_LANGUAGE_DIRECTION.md`
