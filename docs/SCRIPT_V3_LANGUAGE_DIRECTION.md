# ScriptV3 Language Direction

## Summary

ScriptV3는 장기적으로 "자체 DSL 확장"보다 "JavaScript 표준 문법 + Ratel Host API" 방향을 권장합니다.

현재 상태(2026-02-12):
- Stage 0(병행 지원) 구현 완료
- Script Console에서 ScriptV3/JavaScript 선택 실행 가능

권장 구현:

- JavaScript 엔진: `Jint`
- 런타임 기능: `ratel.*` Host API
- 기존 ScriptV3 기능(`run`, `run_once`, `run_once_stop`)은 JS 함수로 동일 제공

## Why JavaScript

- 이미 널리 알려진 문법이라 학습 비용이 낮음
- C#/.NET 프로젝트에 내장하기 쉬움
- 배포 시 외부 런타임 의존(Python 설치 등)을 줄일 수 있음

## Python 대비 판단

- Python도 가능하지만, 임베딩 시 런타임/배포 복잡도가 상대적으로 큼
- 현재 RatelLib 배포/설치 경험을 고려하면 JS(Jint)가 운영 리스크가 낮음

## User-visible Changes

- 신규 스크립트는 `.js` 기준으로 작성 권장
- 기존 `.scr`은 즉시 제거하지 않고 병행 지원
- 장비 제어/실행 제어는 공통적으로 `ratel.*` API를 사용

## Example

```javascript
ratel.log.info("start");

const h = ratel.run("worker.js");
const ok = ratel.runOnce("interlock-main", "interlock.js");

if (ok) {
  ratel.log.info("interlock running");
}
```

## Rollout Policy

1. 병행 지원: `.scr` + `.js`
2. 신규 기능: JS 우선
3. 안정화 후: ScriptV3 DSL은 유지보수 모드

## Related Docs

- 개발 설계 문서: `RatelSoft.Lib/docs/SCRIPT_V3_JS_ENGINE_DESIGN.md`
- 기존 매뉴얼: `SCRIPT_V3_MANUAL.md`
