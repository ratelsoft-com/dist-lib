# RatelSoft Library Consumer Guide (For AI Agents)

이 문서는 `RatelSoft.*` 라이브러리를 다른 프로젝트에서 사용할 때, AI Agent가 일관되게 코드를 작성하도록 하는 지침입니다.

## 1) 기본 원칙
- 라이브러리 이름/네임스페이스는 `RatelSoft.*` 기준으로 사용한다.
- 로깅은 `RatelLog` 정적 직접 호출보다 `RatelLogger` 인스턴스(`RatelLog.ForSource`)를 사용한다.
- WPF 로그뷰어를 쓸 때 `NLogLogViewerTarget` (`RatelSoft.Utils.Wpf.Logging`) 등록을 누락하지 않는다.
- Motion은 신규 프로젝트에서 `RatelSoft.Utils.UnifiedMotion` 우선 사용.
- IO는 `RatelSoft.Utils.UnifiedIO`, Camera는 `RatelSoft.Utils.UnifiedCamera` 우선 사용.
- 타입 위치를 찾을 때는 소스 탐색 전에 `Docs/RatelSoft_TYPE_INDEX.md`를 먼저 확인한다.
- 실전 예제가 필요하면 `Docs/RatelSoft_CODEBOOK_EsViewer_PRACTICAL.md`를 우선 참조한다.
- `RatelViewer`를 쓰는 UI 프로젝트는 `Docs/RatelSoft_CODEBOOK_RatelViewer.md`를 같이 확인한다.

## 2) 참조(패키지/프로젝트)
프로젝트 성격에 맞춰 아래를 선택한다.
- 공통 로깅/설정: `RatelSoft.Common`
- 기본 라이브러리 유틸/초기화: `RatelSoft.Lib`
- 타입 모델: `RatelSoft.Types`
- Motion: `RatelSoft.Utils.UnifiedMotion`
- IO: `RatelSoft.Utils.UnifiedIO`
- Camera: `RatelSoft.Utils.UnifiedCamera`
- Vision 알고리즘: `RatelSoft.Vision`

권장 대상 프레임워크
- WPF 소비 프로젝트는 `net8.0-windows` 기준으로 맞춘다.

로컬 NuGet 피드 사용 예시 (`NuGet.config`)
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="local-ratel" value="C:\data\nuget" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

## 3) 초기화 순서 권장
1. `RatelLib.InitLib(...)` 호출 (필요 시)
2. 로거 생성: `static RatelLogger log = RatelLog.ForSource("...")`
3. 설정 로드: `ConfigManager<T>` 또는 `DataStore<T>`
4. 하드웨어 모듈 연결 (Motion/IO/Camera)

예시
```csharp
RatelLib.InitLib(msg => Console.WriteLine(msg));
static RatelLogger log = RatelLog.ForSource("MyApp");

var config = new ConfigManager<AppSettings>(args, new ConfigManagerOptions("MYAPP_"));
```

## 4) 모듈 선택 가이드
- Motion 시뮬레이션: `VController`
- Motion 실장비(Ajin): `AjinController` + `MotFilePath`
- Motion 실장비(PowerPMac): `PPMacCore` + SSH 정보
- IO 시뮬레이션: `IODeviceFactory.Create(IODeviceType.Emulation)`
- IO EIO: `IODeviceFactory.Create(IODeviceType.Eio)` + `ConnectAsync("ip:port")`
- Camera 시뮬레이션: `CameraFactory.CreateCamera(CameraType.Emulation)`
- Camera Basler/MVS: 벤더 SDK DLL 준비 후 `CameraType.Basler`/`CameraType.Mvs`

## 5) AI Agent 구현 규칙
- 새 기능 추가 시 먼저 `Docs/RatelSoft_TYPE_INDEX.md`에서 타입 위치 확인 후, `Docs/RatelSoft_CODEBOOK.md`에서 사용 패턴 확인.
- Method 호출 전 `IsConnected`, `IsGrabbing`, `State` 등 상태를 확인.
- 비동기 API는 `await`로 호출하고, 취소 토큰을 가능한 전달.
- 예외 발생 시 `log.Error(ex, "...")` 패턴으로 로깅.
- Motion 축 추가 시 `InnerAxisNo` 매핑을 코드 주석으로 명시.

## 6) 자주 발생하는 실수
- `RatelLog` 정적 호출만 남기는 패턴 (권장 패턴 아님)
- Motion `axisNo`와 `InnerAxisNo`를 혼동
- EIO 연결 시 endpoint 미지정 (`ConnectAsync()`만 호출)
- Camera 벤더 SDK 누락 상태에서 Basler/MVS 연결 시도
- `DataStore` 통신 전용 모드(`new DataStore<T>()`)에서 `Load/Save` 호출

## 7) 복붙 템플릿

### 7.1 Logger
```csharp
using RatelSoft.Common;

static RatelLogger log = RatelLog.ForSource("MyModule");
log.Info("startup", "module started");
```

### 7.1.1 NLog 설정 위치 규칙
- 코드 기반 구성: `ConfigureNLog()`에서 `RatelLogViewer` target 등록
- 파일 기반 구성: `nlog.config` 사용 시 `<extensions><add assembly="RatelSoft.Utils.Wpf" /></extensions>` 포함
- Caller 기반 분리 로그를 쓰면 `${event-properties:item=Caller}`를 레이아웃/필터에 같이 쓴다.

### 7.1.2 Global Using 권장 템플릿
```csharp
global using RatelSoft.Common;
global using RatelSoft.Lib;
global using RatelSoft.AI;
global using RatelSoft.Vision;
global using RatelSoft.Vision.Mura;
global using RatelSoft.Vision.GPU;
global using RatelSoft.Vision.Wpf;
global using RatelSoft.Utils.Wpf.Logging;
```

### 7.2 Config
```csharp
using RatelSoft.Common.Configuration;

var cfg = new ConfigManager<AppSettings>(args, new ConfigManagerOptions("MYAPP_"));
var s = cfg.Settings;
```

### 7.3 Motion
```csharp
using RatelSoft.Utils.UnifiedMotion;

var mm = new MotionManager();
mm.AddAxis(0, new MotionAxisStatus { AxisName = "X", InnerAxisNo = 0 }, new VController(0));
mm.Open();
await mm.MoveAbsAsync(0, 1000);
```

### 7.4 IO
```csharp
using RatelSoft.Utils.UnifiedIO;

var io = IODeviceFactory.Create(IODeviceType.Emulation);
await io.ConnectAsync();
await io.WriteOutputAsync(0, true);
```

### 7.5 Camera
```csharp
using RatelSoft.Utils.UnifiedCamera;

using var cam = CameraFactory.CreateCamera(CameraType.Emulation);
await cam.ConnectAsync();
var img = await cam.GrabOneAsync();
```

## 8) AI Agent용 프롬프트 템플릿
아래 문구를 다른 프로젝트의 AGENTS/README에 붙여 넣어 사용한다.

```text
This project depends on RatelSoft libraries.
Always consult `Docs/RatelSoft_CODEBOOK.md` first.
Use RatelLogger instance pattern (`RatelLog.ForSource`) for logging.
For hardware abstraction, prefer:
- Motion: RatelSoft.Utils.UnifiedMotion
- IO: RatelSoft.Utils.UnifiedIO
- Camera: RatelSoft.Utils.UnifiedCamera
Before calling device methods, verify connection/state properties.
When mapping motion axes, keep global axis and InnerAxisNo mapping explicit.
```

## 9) 유지보수 규칙
- API 변경 시 반드시 `Docs/RatelSoft_CODEBOOK.md`와 본 문서를 함께 업데이트한다.
- 신규 어셈블리 추가 시 "참조" 섹션과 "모듈 선택 가이드"에 반영한다.
- 로그 초기화 방식(`ConfigureNLog` 또는 `nlog.config`)이 바뀌면 템플릿과 예제를 같이 업데이트한다.
- `RatelViewer` DP/이벤트 변경 시 `Docs/RatelSoft_CODEBOOK_RatelViewer.md`를 함께 업데이트한다.
