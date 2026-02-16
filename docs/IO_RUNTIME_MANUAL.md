# IO Runtime Manual (RatelWPF 기반)

이 문서는 `RatelWPF`의 실제 코드 구조를 기준으로, UI와 분리된 IO 제어/조회 방법을 설명한다.

## 1. 핵심 구조

- 런타임(제어/조회): `IIoRuntime` / `IoRuntime`
  - 파일: `D:\Source\repos\jjack000\Library\RatelSoft.UnifiedIO\src\UnifiedIO\IoRuntime.cs`
- UI 어댑터(바인딩/CSV): `IoViewManager`
  - 파일: `D:\Source\repos\jjack000\Library\RatelLib\RatelWPF\Run\IoManager.cs`
- 앱 컨텍스트(조립): `RatelWpfContext`
  - 파일: `D:\Source\repos\jjack000\Library\RatelLib\RatelWPF\Run\RatelWpfContext.cs`

구조 원칙:
- `IIoRuntime`는 UI 없이 단독 동작해야 한다.
- `IoViewManager`는 표시용 컬렉션/맵 파일만 담당한다.

## 2. RatelWPF 초기화 패턴

```csharp
// RatelWPF/Run/RatelWpfContext.cs
private static readonly IIODevice IoDevice = new EmulatedIODevice(IoModel.inputCount, IoModel.outputCount);
private static readonly IIoRuntime IoRuntime = new IoRuntime(IoDevice);
public static IIoRuntime Io => IoRuntime;
public static IoViewManager IoView { get; private set; } = new IoViewManager(IoRuntime);

public static async Task InitializeAsync(CancellationToken token = default)
{
    IoView = new IoViewManager(IoRuntime, SynchronizationContext.Current);
    await IoView.InitializeAsync(token).ConfigureAwait(false);
}
```

## 3. 런타임 단독 사용(권장 API)

`IIoRuntime` 기본 API:
- `InitializeAsync()`, `RefreshAsync()`
- `GetInBit(int)`, `GetOutBit(int)`, `GetBit(string)`
- `SetInBitAsync(...)`, `SetOutBitAsync(...)`
- `ConfigureNameMap(inputMap, outputMap)`
- 이벤트: `ChannelStateChanged`, `MapChanged`

예제:

```csharp
using RatelSoft.Utils.UnifiedIO;

var dev = new EmulatedIODevice(inputCount: 256, outputCount: 256);
IIoRuntime io = new IoRuntime(dev);

await io.InitializeAsync();
io.ConfigureNameMap(
    new Dictionary<string, int> { ["DOOR_OPEN"] = 10, ["10"] = 10 },
    new Dictionary<string, int> { ["STACK_RED"] = 3, ["3"] = 3 });

await io.SetOutBitAsync("STACK_RED", true);
bool isDoorOpen = io.GetBit("DOOR_OPEN");
```

## 4. CSV 맵 + UI 어댑터 사용

`IoViewManager` 담당:
- CSV 로드/저장: `IoMapStore.LoadOrCreateDefault`, `SaveMap`, `ReloadMap`
- UI 컬렉션: `InItems`, `OutItems`
- 런타임 이름맵 반영: `ApplyMapToRuntime()`
- 런타임 이벤트 반영: `Runtime_ChannelStateChanged`

예제(`IOWindow.xaml.cs` 패턴):

```csharp
var src = showInput ? RatelWpfContext.IoView.InItems : RatelWpfContext.IoView.OutItems;
await RatelWpfContext.IoView.SetInBitAsync(pinNo, next);
await RatelWpfContext.IoView.SetOutBitAsync(pinNo, next);
```

CSV 상세 형식: `dist-lib/docs/IO_MAP_CSV_GUIDE.md`

## 5. Script Host 연동 패턴

`RatelWPF/Run/ScriptV3HostFunctionRegistrar.cs` 기준:
- `get_in(name)` -> `_io.GetBit(name)`
- `set_out(name, value)` -> `_io.SetOutBitAsync(name, value, token)`
- `wait_in(name, expected, timeout)` -> 폴링 + timeout

핵심:
- 스크립트는 `IoViewManager`가 아니라 `IIoRuntime`를 직접 사용한다.

## 6. 권장 구현 규칙

- 제어 로직/시나리오: `IIoRuntime`만 사용
- 화면 표시/페이징/색상: `IoViewManager` + `IoModel`
- enum은 고정 식별자로 유지, 이름/설명은 CSV로 관리
