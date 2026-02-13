# Release Notes (v2.0 브랜치)

이 문서는 `v2.0` 브랜치를 `master`에서 분기한 이후의 변경사항을 요약합니다.

- 기준(merge-base): `a520379ee8ef65c9c966e0bf6258738f1ebb43e3`
- 범위: `a520379..v2.0`
- 기간: 2026-01-03 ~ 2026-02-11

## 주요 변경사항

### WPF 구성 정리 및 신규 프로젝트

- `RatelLib.WPF` 프로젝트 추가
- `RatelSoft.Vision.Wpf` 프로젝트 추가
- 기존 일부 WPF 관련 코드/리소스 이동 및 정리

### Linux 호환성 개선(정리)

- Windows/WPF 의존 또는 사용되지 않는 코드/파일 정리
- Vision/Viewer 관련 코드 구조 일부 정리(삭제/이동 포함)

### 로깅 구조 변경

- `RatelSoft.Lib.Log` 사용을 제거하고, 클래스/모듈별 로그 사용으로 변경
- Vision 및 시나리오/네트워크 관련 코드 일부가 새로운 로그 사용 방식에 맞춰 수정됨

## 패키징 / 배포

### NuGet 패키지에 원본 DLL이 포함되는 문제 수정

- `nugetpack.py`에서 난독화(Obfuscar) 이후 산출물이 실제 `dotnet pack --no-build`가 참조하는 출력 폴더에 반영되도록 보완
- 관련 Obfuscar 설정 파일(`Obfuscar.xml`, `obfuscar8.xml`)도 함께 조정

### 배포 UX 개선

- NuGet 업로드 확인을 패키지별로 반복 질문하지 않고, 한 번의 확인으로 일괄 처리하도록 개선

## 인증(Auth) 관련

- 인증 동작/사용법 문서 업데이트: `Docs/Authentication-System.md`
- 앱에서 인증 상태를 조회할 수 있도록 `DongleInfo`에 런타임 인증 상태 필드를 추가하고, 백그라운드 인증 체크 루프에서 값이 채워지도록 수정
  - `IsAuthOk`, `AuthState`, `AuthCheckedAt`, `AuthMessage`
- 변경된/추가된 타입 정의 위치
  - `RatelSoft.Common/RatelSoft.LibInfo.cs`
  - `RatelVisionLib/Common/RatelSoft.LibInfo.cs`

## 최근 업데이트 (미릴리스)

### NuGet 패키징/업로드 스크립트 경로 및 버전 갱신 보완 (2026-02-13)

- `nugetpack.py` 개선:
  - 설정된 프로젝트들의 버전 필드 자동 갱신 추가
  - 버전 규칙 `2.yy.mmdd.build`로 적용 (`VersionPrefix`, `AssemblyVersion`, `FileVersion`, `InformationalVersion`)
  - 동일 날짜 기존 패키지/버전을 검사해 build 번호 자동 증가
  - 로컬 푸시(`C:\data\nuget`) 시 기존 동일 패키지 파일 정리 후 푸시하도록 보완
  - 버전 갱신된 `csproj`를 패키징 성공 후 저장소별 자동 `commit`/`push` 하도록 보완(옵션: `--no-git-commit`, `--no-git-push`)
  - 옵션 추가:
    - `--no-version-update` (버전 자동 갱신 비활성화)
    - `--keep-local-old` (로컬 피드 기존 패키지 유지)
- `upload_ratelsoft_packages.py` 개선:
  - 기본 탐색 경로를 `../RatelLib/artifacts/nupkg` 기준으로 고정
  - 기본 패턴을 `RatelSoft.*.nupkg`로 조정
  - `--package-dir` 옵션 추가
  - API 키 하드코딩 제거, 환경변수(`GITHUB_PACKAGES_TOKEN` 또는 `GITHUB_TOKEN`) 사용

### Script JS async 재현/안정화 (2026-02-13)

- 사용자 보고 이슈 수정:
  - 증상: `async_examples.js` 실행 시 `--- 1. 기본 spawnFunc/await ---` 이후 정지/실패
  - 원인1: 백그라운드 작업이 매우 빠르게 완료되면 핸들이 즉시 정리되어
    `await(handle)`에서 `Task handle not found` 발생
  - 원인2: `spawnFunc`의 함수 객체 문자열화 결과가 `[native code]`인 경우
    wrapper 파싱 실패(`Unexpected identifier`)
- 조치:
  - 완료된 백그라운드 핸들 저장소 추가(`_completed`, 기본 512개 유지)로 `await` 유실 방지
  - `spawnFunc("functionName", ...args)` 경로를 파일 함수 추출 방식으로 안정화
  - `spawnFunc` wrapper 사전 파싱/오류 상세화로 문제 원인 즉시 노출
  - `spawnFunc` 실행 결과를 `await`로 받을 수 있도록 내부 결과 슬롯(`__ratel_internal_result__`) 적용
  - 백그라운드 task fault 시 `[JS:ERROR]` 로그 강화
- 재현/검증 도구 추가:
  - `../RatelLib/tools/ScriptJsRepro/ScriptJsRepro.csproj`
  - `dotnet run --project ../RatelLib/tools/ScriptJsRepro/ScriptJsRepro.csproj -- minimal`
  - `dotnet run --project ../RatelLib/tools/ScriptJsRepro/ScriptJsRepro.csproj -- file ../RatelLib/RatelWPF/Scripts/async_examples.js`

### ScriptV3 언어 방향 정리 (2026-02-12)

- ScriptV3 장기 방향을 "자체 DSL 확장"에서 "JavaScript 표준 문법 + Host API"로 정리
- 권장 엔진으로 `Jint` 채택 방향 문서화
- 사용자 문서 추가:
  - `docs/SCRIPT_V3_LANGUAGE_DIRECTION.md`
- 개발자 설계 문서 추가:
  - `../RatelLib/RatelSoft.Lib/docs/SCRIPT_V3_JS_ENGINE_DESIGN.md`
- 배포 포털 문서 링크 추가:
  - `README.md`의 문서 목록에 Script 언어 방향 문서 연결

### Script Console JavaScript 실행 모드 추가 (2026-02-12)

- 기존 ScriptV3는 유지하고, JavaScript 실행 경로를 신규 추가
- `RatelSoft.Lib`에 JS 엔진 추가 (`Jint` + `Esprima`)
  - `../RatelLib/RatelSoft.Lib/FlowScenario/ScriptJs/ScriptJsEngine.cs`
  - `../RatelLib/RatelSoft.Lib/FlowScenario/ScriptJs/ScriptJsHostApi.cs`
  - `../RatelLib/RatelSoft.Lib/FlowScenario/ScriptJs/ScriptJsPrimitives.cs`
- `ScriptConsoleWindow`를 공용 확장
  - 하단 `Lang` 선택(`ScriptV3` / `JavaScript`)
  - `.scr`, `.js` 로드/저장/목록/실행/중지 분기 지원
  - JS 모드에서는 Debug Step 비활성(런 모드만 지원)
- `RatelWPF` 호스트 등록 추가
  - `../RatelLib/RatelWPF/Run/ScriptJsHostFunctionRegistrar.cs`
  - 기존 모션/IO 함수들을 JS Host API에도 등록
- 예제/문서 추가
  - `../RatelLib/RatelWPF/Scripts/javascript_basic.js`
  - `../RatelLib/RatelWPF/Scripts/mainscr.js` (`mainscr.scr` 변환본)
  - `docs/SCRIPT_JS_QUICKSTART.md`
  - 개발 문서:
    - `../RatelLib/RatelSoft.Lib/docs/SCRIPT_JS_ENGINE_IMPLEMENTATION.md`
    - `../RatelLib/RatelSoft.Utils.Wpf/docs/SCRIPT_CONSOLE_LANGUAGE_MODE.md`
- 안정화 수정:
  - Script Console 초기화 중 `Lang` 콤보 `SelectionChanged` 선호출 시 발생 가능한
    `NullReferenceException` 가드 추가
  - `../RatelLib/RatelSoft.Utils.Wpf/Script/ScriptConsoleWindow.xaml.cs`
  - 상태머신 기반 수명주기 관리 추가:
    - `ScriptConsoleUiState` (`None/Initializing/Ready/Running/Closing/Closed`)
    - UI 이벤트/실행 동작을 상태 기반으로 가드
    - 실행 모니터 기반 `Ready/Running` 상태 자동 전이
  - 공용변수 연동 추가:
    - JS Host API `ratel.getVar`, `ratel.setVar`, `ratel.hasVar`
    - 저장소: `VariableManager` (`RatelLib.Variables`) 공유
    - 배열 지원: JS Array <-> `List<object>` (중첩 배열 포함)
  - async API 확장:
    - `ratel.spawn(path)` (run alias)
    - `ratel.spawnFunc(fnOrName, ...args)` (single-file 함수 비동기 실행)
    - `ratel.await(handle)` (awaitResult alias)
    - `ratel.isRunning(handle)`, `ratel.runOnce`, `ratel.runOnceStop`
  - async 예제 JS 변환:
    - `../RatelLib/RatelWPF/Scripts/async_examples.js`
    - worker 스크립트 추가:
      - `async_worker_basic.js`
      - `async_worker_task_a.js`
      - `async_worker_task_b.js`
      - `async_worker_long.js`
      - `async_worker_periodic.js`
      - `async_worker_producer.js`
      - `async_worker_consumer.js`
  - 안정화:
    - `spawnFunc` 인자 전달 방식을 `apply(__spawn_args)`에서 JS literal 직주입으로 변경
    - `async_examples.js`는 함수명 문자열 기반 `spawnFunc("fnName", ...)` 패턴으로 정리

### UI Window 상태머신 공통 지침 추가 (2026-02-12)

- 다른 프로젝트에서도 재사용 가능한 UI 수명주기 공통 지침을 정의
- 에이전트 운영 지침(`../RatelLib/AGENTS.md`)에 Window 상태머신 패턴 공통 규칙 추가
- 개발 가이드 문서 추가:
  - `../RatelLib/RatelSoft.Utils.Wpf/docs/WINDOW_LIFECYCLE_STATE_GUIDELINE.md`

### UnifiedMotion 독립성 정리 (2026-02-12)

- `RatelSoft.Utils.UnifiedMotion/UnifiedMotion/UnifiedMotion.csproj`에서 불필요한 프로젝트 참조 제거
  - 제거: `RatelSoft.Utils.UnifiedIO`, `RatelSoft.Utils.UnifiedCamera`
  - 유지: `RatelSoft.Common`
- `dist-lib/README.md`의 라이브러리 의존 관계 설명을 실제 구조와 동일하게 수정
  - `RatelSoft.Utils.UnifiedMotion -> RatelSoft.Common`

### 배포/문서 운영 정리 (2026-02-12)

- `dist-lib` 저장소를 서브모듈로 등록
  - `.gitmodules`
  - `dist-lib`
- 배포 포털 문서 추가: `dist-lib/README.md`
  - 라이브러리 목록/기능/의존 관계/문서 링크 정리
- 패키지 메타데이터 저장소 URL 통일
  - `Directory.Build.props`, `Directory.Build.targets`에서 `RepositoryUrl`, `PackageProjectUrl`을 `https://github.com/ratelsoft-com/dist-lib`로 통일
  - 주요 `RatelSoft.*` 패키지 csproj의 기존 URL도 동일 값으로 정리
- 라이브러리별 `LICENSE` 파일 추가
  - `RatelSoft.Common`, `RatelSoft.Lib`, `RatelSoft.Types`, `RatelSoft.Vision`, `RatelSoft.Vision.Wpf`
  - `RatelLib.WPF`, `RatelSoft.Utils.Wpf`, `RatelSoft.Utils.OpenCvAdapter`, `RatelSoft.Utils.TestLibApp.Template`
  - `RatelSoft.Utils.UnifiedCamera/UnifiedCamera`, `RatelSoft.Utils.UnifiedIO/UnifiedIO`, `RatelSoft.Utils.UnifiedMotion/UnifiedMotion`
- 문서 정책 반영
  - 루트 `README.md`에서 문서 기준 위치를 `dist-lib/docs`로 변경
  - `AGENTS.md`에 매뉴얼/가이드(`dist-lib/docs`) 및 릴리즈 노트 양방향 관리 규칙 명시

### LogViewer 기능 개선 (`RatelSoft.Utils.Wpf`)

- 로그 폴더 열기 동작 개선
  - 등록된 로그 폴더 우선 사용
  - 등록 폴더가 없으면 NLog `FileTarget`에서 로그 경로 자동 탐색
  - 탐색된 폴더를 자동 등록/생성 후 열기
- `AutoScroll`을 `DependencyProperty`로 추가하고 신규 로그 유입 시 자동 스크롤 지원
- 표시 필터 기능 추가: `경고만 표시` 토글(표시만 필터, 수집 로그는 유지)
- 검색 기능 개선: 일반 검색 + `Regex` 검색 옵션 지원
- 컨텍스트 메뉴 개선
  - `Auto Scroll` 체크 토글
  - `툴바 표시/숨김` 체크 토글
  - `경고만 표시` 체크 토글

### 다국어(Localization) 정식 구조 리팩터링

- 하드코딩 문자열 기반에서 `.resx` 기반 다국어 구조로 전환
- 공통 로컬라이저 추가: `RatelSoft.Common/Localization/ResourceLocalizationManager.cs`
- 로그뷰어 리소스 추가
  - `RatelSoft.Utils.Wpf/Resources/LogViewerStrings.resx` (기본/영어)
  - `RatelSoft.Utils.Wpf/Resources/LogViewerStrings.ko.resx` (한국어)
- `LogViewer`에서 `UiLanguage` 전환 시 리소스 문자열이 런타임에 즉시 반영되도록 변경
- 디자이너 표시 안정화
  - 디자인 타임 `d:DataContext` 설정
  - `RelativeSource`/`FallbackValue`/`d:Content` 보강

### 테스트/문서

- `RatelWPF`에서 한국어/영어 전환 테스트 버튼 추가
- 다국어 사용 가이드 문서 추가: `Docs/Localization.md`

## 버전

아래 목록은 `Update version to ...` 커밋 메시지 기준으로 정리했습니다.

| 날짜 | 버전 | 커밋 | 메타(괄호) |
|---|---|---|---|
| 2026-02-11 | 2.26.0211.1 | d60daf0 | 32d9255 |
| 2026-02-10 | 2.26.0210.6 | dbebb0d | 17e3be2 |
| 2026-02-10 | 2.26.0210.1 | 08c3032 | 375260d |
| 2026-02-09 | 2.26.0209.15 | 4043357 | 7deb8fb+dirty |
| 2026-02-09 | 2.26.0209.13 | 41471ed | 0f67259 |
| 2026-02-09 | 2.26.0209.11 | bf7223b | b158c91 |
| 2026-02-09 | 2.26.0209.9 | 4ac7f5c | 9436f20+dirty |
| 2026-02-09 | 2.26.0209.7 | a0e7aba | a44cc15+dirty |
| 2026-02-09 | 2.26.0209.5 | 96370d1 | dd0309b |
| 2026-02-09 | 2.26.0209.2 | ee0782e | 79e476e+dirty |
| 2026-02-09 | 2.26.0209.1 | e2677c1 | 55931c7+dirty |
| 2026-02-08 | 2.26.0208.15 | bf79e42 | bf7758d+dirty |
| 2026-02-08 | 2.26.0208.13 | 44c0e37 | edf8ab2 |
| 2026-02-06 | 2.26.0206.5 | e1f8e7f | 9bcddb4 |
| 2026-02-06 | 2.26.0206.1 | 82fd93b | 0d699a4 |
| 2026-01-28 | 2.26.0128.6 | b779327 | 6e53cc8+dirty |
| 2026-01-28 | 2.26.0128.4 | 18e6502 | 4923528+dirty |
| 2026-01-28 | 2.26.0128.1 | c9c055d | 49f33ea |
| 2026-01-27 | 2.26.0127.28 | 1e9b9a7 | ed448e1 |
| 2026-01-27 | 2.26.0127.25 | ce20890 | 0727031 |
| 2026-01-27 | 2.26.0127.23 | fb70d31 | 54c6a65 |
| 2026-01-27 | 2.26.0127.19 | 7d0ed11 | 9f66112 |
| 2026-01-27 | 2.26.0127.17 | 263613b | de80c41+dirty |
| 2026-01-27 | 2.260127.15 | e469cce | e8eceaf+dirty |
| 2026-01-27 | 2.260127.13 | 99418c2 | 1f51f8b+dirty |
| 2026-01-27 | 2.260127.12 | 1f51f8b | 15c55b2+dirty |
| 2026-01-27 | 2.260127.11 | 15c55b2 | 4ef2495+dirty |
| 2026-01-27 | 2.260127.10 | 4ef2495 | 1f91586+dirty |
| 2026-01-27 | 2.260127.9 | 1f91586 | 1603f77+dirty |
| 2026-01-27 | 2.260127.8 | 1603f77 | 752b5e6 |
| 2026-01-27 | 2.260127.7 | 752b5e6 | 21439c9+dirty |
| 2026-01-27 | 2.260127.6 | 21439c9 | 565a412+dirty |
| 2026-01-27 | 2.260127.5 | 565a412 | 01d237f+dirty |
| 2026-01-27 | 2.260127.4 | 01d237f | 21fa2e0+dirty |
| 2026-01-10 | 2.260110.6 | be56051 | 92b99b1+dirty |
| 2026-01-10 | 2.260110.5 | 92b99b1 | dd1d7fa+dirty |
| 2026-01-10 | 2.260110.4 | dd1d7fa | bf27be7 |
| 2026-01-10 | 2.260110.1 | e6ffd04 | 63ee988+dirty |
| 2026-01-03 | 2.260103.7 | 63ee988 | 268ae84+dirty |
| 2026-01-03 | 2.260103.6 | 268ae84 | 466e35d+dirty |
| 2026-01-03 | 2.260103.5 | 466e35d | fd78710+dirty |

## 버전별 변경사항 (상세)

각 버전은 해당 `Update version to ...` 커밋을 기준으로, 직전 버전 범프 이후의 커밋 제목을 **모두** 나열합니다.

### 2.26.0211.1 (2026-02-11)
- 버전 범프 커밋: d60daf0
- 변경:
  - 32d9255 feat: add TotalLines property to CameraSettings for calculated line count

### 2.26.0210.6 (2026-02-10)
- 버전 범프 커밋: dbebb0d
- 변경:
  - 8fe4ed3 chore: apply current workspace updates and stabilize Mat formatter null/memory handling
  - b245b5a Move Script Console UI to Utils.Wpf and package Script v3 docs
  - 17e3be2 Remove old Script v3 manual path after docs migration

### 2.26.0210.1 (2026-02-10)
- 버전 범프 커밋: 08c3032
- 변경:
  - 74566fc docs(viewer): 소비자 문서 확장 및 Mat DP/XML 문서 기반 추가
  - cc15d7c docs(viewer): 숨김 UI 재구성 코드북 및 공개 API 한글 XML 주석 보강
  - 375260d xml 주석 추가

### 2.26.0209.15 (2026-02-09)
- 버전 범프 커밋: 4043357
- 변경: (버전만 갱신)

### 2.26.0209.13 (2026-02-09)
- 버전 범프 커밋: 41471ed
- 변경: (버전만 갱신)

### 2.26.0209.11 (2026-02-09)
- 버전 범프 커밋: bf7223b
- 변경: (버전만 갱신)

### 2.26.0209.9 (2026-02-09)
- 버전 범프 커밋: 4ac7f5c
- 변경: (버전만 갱신)

### 2.26.0209.7 (2026-02-09)
- 버전 범프 커밋: a0e7aba
- 변경: (버전만 갱신)

### 2.26.0209.5 (2026-02-09)
- 버전 범프 커밋: 96370d1
- 변경:
  - dd0309b 문서 업데이트

### 2.26.0209.2 (2026-02-09)
- 버전 범프 커밋: ee0782e
- 변경: (버전만 갱신)

### 2.26.0209.1 (2026-02-09)
- 버전 범프 커밋: e2677c1
- 변경: (버전만 갱신)

### 2.26.0208.15 (2026-02-08)
- 버전 범프 커밋: bf79e42
- 변경: (버전만 갱신)

### 2.26.0208.13 (2026-02-08)
- 버전 범프 커밋: 44c0e37
- 변경:
  - 9512e78 release note upudate
  - ed4ad6f 3d 파싱방법 수정
  - f9b8d3a scrpt v3
  - bb9ebf9 변수 스코프 수정
  - 6c9be27 스크립트 매뉴얼, 수학함수 제공
  - 10ed2c7 스크립트에 배열 및 try-catch 구조 지원
  - 5f67bdf 스크립틀르 v3만 남기고 삭제
  - f7f53b7 ScriptV3 개선: else-if 지원, async spawn 안정화, 콘솔 에디터 UI 확장
  - 1837b1e FlowScenario 글로벌 변수 관리 개선 및 v3 매뉴얼 시스템 변수 섹션 추가
  - 495faac Add ScriptV3 runtime service and WPF runtime monitor controls
  - cc235db 스크립트 디버깅 기능 추가중
  - 9ea685a script-v3: add run_once_stop and unified runtime monitoring
  - 1b012ed script-v3: replace include with import/module model
  - 04a624a script-v3: stabilize debug step/resume and sync script updates
  - a46cd11 merge agitated-hypatia into v2.0
  - 3444d93 cmake, vs2026 컴파일 가능, utf8 인코딩
  - fdc228f Add Mura stream benchmark and GPU execution controls
  - e59b9ef Add CUDA_MURA_DLL_PATH runtime policy, logging, and deploy/test tooling
  - edf8ab2 Add CudaMura runtime install flow and fixed-path InitGPU loading

### 2.26.0206.5 (2026-02-06)
- 버전 범프 커밋: e1f8e7f
- 변경:
  - 4818120 로그개선
  - 9bcddb4 다국어 관련기능 추가.

### 2.26.0206.1 (2026-02-06)
- 버전 범프 커밋: 82fd93b
- 변경:
  - 79b1732 OpenCvAdapter, TestLibApp 추가
  - ffb9727 obj 모델링 추가
  - 88f028c Helix Material 추가
  - 0d699a4 LogViewer 개선

### 2.26.0128.6 (2026-01-28)
- 버전 범프 커밋: b779327
- 변경: (버전만 갱신)

### 2.26.0128.4 (2026-01-28)
- 버전 범프 커밋: 18e6502
- 변경:
  - 4923528 RatelSoft.Utils.OpenCvAdapter, RatelSoft.TestLibApp.Template 추가 Changes to be committed:

### 2.26.0128.1 (2026-01-28)
- 버전 범프 커밋: c9c055d
- 변경:
  - bdd6690 ratelwpf mainwindow 수정
  - 85ac77c A1 Modeling
  - 49f33ea Hexlix Motion 작업중

### 2.26.0127.28 (2026-01-27)
- 버전 범프 커밋: 1e9b9a7
- 변경:
  - ed448e1 Log에 Excepton 표시

### 2.26.0127.25 (2026-01-27)
- 버전 범프 커밋: ce20890
- 변경: (버전만 갱신)

### 2.26.0127.23 (2026-01-27)
- 버전 범프 커밋: fb70d31
- 변경:
  - 15acc3d unifiedmotion 추가
  - 54c6a65 Unified* 네임스페이스 변경

### 2.26.0127.19 (2026-01-27)
- 버전 범프 커밋: 7d0ed11
- 변경: (버전만 갱신)

### 2.26.0127.17 (2026-01-27)
- 버전 범프 커밋: 263613b
- 변경: (버전만 갱신)

### 2.260127.15 (2026-01-27)
- 버전 범프 커밋: e469cce
- 변경: (버전만 갱신)

### 2.260127.13 (2026-01-27)
- 버전 범프 커밋: 99418c2
- 변경: (버전만 갱신)

### 2.260127.12 (2026-01-27)
- 버전 범프 커밋: 1f51f8b
- 변경: (버전만 갱신)

### 2.260127.11 (2026-01-27)
- 버전 범프 커밋: 15c55b2
- 변경: (버전만 갱신)

### 2.260127.10 (2026-01-27)
- 버전 범프 커밋: 4ef2495
- 변경: (버전만 갱신)

### 2.260127.9 (2026-01-27)
- 버전 범프 커밋: 1f91586
- 변경: (버전만 갱신)

### 2.260127.8 (2026-01-27)
- 버전 범프 커밋: 1603f77
- 변경: (버전만 갱신)

### 2.260127.7 (2026-01-27)
- 버전 범프 커밋: 752b5e6
- 변경: (버전만 갱신)

### 2.260127.6 (2026-01-27)
- 버전 범프 커밋: 21439c9
- 변경: (버전만 갱신)

### 2.260127.5 (2026-01-27)
- 버전 범프 커밋: 565a412
- 변경: (버전만 갱신)

### 2.260127.4 (2026-01-27)
- 버전 범프 커밋: 01d237f
- 변경:
  - cbeedbe 2.0 라이브러리 정리 #1
  - 32faf0b util 프로젝트 결합
  - d30d3fa 라이브러리 통합
  - d2c5059 util 삽입 완료
  - 5bc2fc4 RatelLog 사용으로 수정중
  - 21fa2e0 경고들 처리

### 2.260110.6 (2026-01-10)
- 버전 범프 커밋: be56051
- 변경: (버전만 갱신)

### 2.260110.5 (2026-01-10)
- 버전 범프 커밋: 92b99b1
- 변경: (버전만 갱신)

### 2.260110.4 (2026-01-10)
- 버전 범프 커밋: dd1d7fa
- 변경:
  - 8d8deeb nugetpack.py 에 dll 원본 들어가는 문제 해결
  - c63dccb auth 관련 수정
  - bf27be7 Release Note 작성

### 2.260110.1 (2026-01-10)
- 버전 범프 커밋: e6ffd04
- 변경: (버전만 갱신)

### 2.260103.7 (2026-01-03)
- 버전 범프 커밋: 63ee988
- 변경: (버전만 갱신)

### 2.260103.6 (2026-01-03)
- 버전 범프 커밋: 268ae84
- 변경: (버전만 갱신)

### 2.260103.5 (2026-01-03)
- 버전 범프 커밋: 466e35d
- 변경:
  - e86fcb4 v2.0 초기 작업 RatelLib.WPF, RatelLib.Vsion.WPF 클래스가 추가되었음.
  - 34c9e8b linux 가능하도록 정리
  - fd78710 RatelLib.Log를 없애고 클래스별 로그 사용

## 커밋 목록

| 날짜 | 커밋 | 요약 |
|---|---|---|
| 2026-01-03 | e86fcb4 | v2.0 초기 작업: `RatelLib.WPF`, `RatelSoft.Vision.Wpf` 추가 및 정리 |
| 2026-01-03 | 34c9e8b | Linux 가능하도록 정리 |
| 2026-01-03 | fd78710 | `RatelSoft.Lib.Log` 제거, 클래스별 로그 사용 |
| 2026-01-03 | 466e35d | Update version to 2.260103.5 |
| 2026-01-03 | 268ae84 | Update version to 2.260103.6 |
| 2026-01-03 | 63ee988 | Update version to 2.260103.7 |
| 2026-01-10 | e6ffd04 | Update version to 2.260110.1 |
| 2026-01-10 | 8d8deeb | `nugetpack.py` 원본 DLL 패키징 문제 수정 |
| 2026-01-10 | c63dccb | Auth 관련 수정 + 패키징 스크립트 개선 |


