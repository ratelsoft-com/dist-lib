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

장비/IO 함수는 호스트 앱 등록 상태에 따라 전역 함수로 사용 가능합니다.

- 예: `move_abs(...)`, `get_in(...)`, `set_out(...)`

## 현재 제한

- JavaScript 디버그(F10 Step)는 아직 미지원 (Run 모드 사용)
- ScriptV3 런타임 모니터 일부 항목은 JS 실행 항목과 표시 방식이 다를 수 있음

## 관련 문서

- ScriptV3 매뉴얼: `SCRIPT_V3_MANUAL.md`
- Script 언어 방향: `SCRIPT_V3_LANGUAGE_DIRECTION.md`

