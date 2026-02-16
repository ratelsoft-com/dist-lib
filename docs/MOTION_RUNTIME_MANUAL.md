# Motion Runtime Manual (RatelWPF 기반)

이 문서는 `RatelWPF` 실코드를 기준으로, Motion 상태 조회와 조작을 UI와 분리해 사용하는 방법을 설명한다.

## 1. 핵심 구조

- 런타임(제어/조회): `IMotionRuntime` / `MotionRuntime`
  - 파일: `D:\Source\repos\jjack000\Library\RatelSoft.UnifiedMotion\src\UnifiedMotion\MotionRuntime.cs`
- 컨트롤러/축 등록: `MotionManager`, `MotionAxis`, `MotionControllerBase`
  - 파일: `D:\Source\repos\jjack000\Library\RatelSoft.UnifiedMotion\src\UnifiedMotion\MotionManager.cs`
  - 파일: `D:\Source\repos\jjack000\Library\RatelSoft.UnifiedMotion\src\UnifiedMotion\MotionAxis.cs`
- UI 어댑터: `MotionViewManager` / `MotionAxisItem`
  - 파일: `D:\Source\repos\jjack000\Library\RatelLib\RatelWPF\Run\MotionViewManager.cs`
- 앱 컨텍스트(조립): `RatelWpfContext`
  - 파일: `D:\Source\repos\jjack000\Library\RatelLib\RatelWPF\Run\RatelWpfContext.cs`

구조 원칙:
- 제어/조회는 `IMotionRuntime` 중심
- WPF 바인딩은 `MotionViewManager` 담당

## 2. RatelWPF 초기화 패턴

```csharp
// RatelWPF/Run/RatelWpfContext.cs
private static readonly MotionManager MotionManager = new MotionManager();
private static readonly IMotionRuntime MotionRuntime = new MotionRuntime(MotionManager);

public static MotionManager Motion => MotionManager;
public static IMotionRuntime MotionCore => MotionRuntime;
public static MotionViewManager MotionView { get; private set; } = new MotionViewManager(MotionRuntime);

public static async Task InitializeAsync(CancellationToken token = default)
{
    AxisManager.InitMotors(MotionManager); // 축 등록 + Open
    MotionView = new MotionViewManager(MotionRuntime, SynchronizationContext.Current);
    MotionView.Rebuild();
}
```

## 3. 축 등록(구성 단계)

`RatelWPF/Run/MotionManager.cs` 기준:

```csharp
var axisNames = Enum.GetNames(typeof(AxisNames));
var controller = new VController(0);

for (int i = 0; i < axisNames.Length; i++)
{
    motion.AddAxis(i, new MotionAxis
    {
        AxisNo = i,
        AxisName = axisNames[i],
        InnerAxisNo = i
    }, controller);
}

motion.Open();
```

## 4. 런타임 단독 사용(권장 API)

`IMotionRuntime` 주요 API:
- 상태: `GetAllAxisSnapshots()`, `GetAxisSnapshot(...)`, `ResolveAxisNo(...)`
- 파라미터: `GetMotionValues(...)`, `SetMotionValues(...)`
- 조작: `MoveAbsAsync`, `MoveIncAsync`, `WaitMoveDoneAsync`, `Jog/Stop/Home/Servo/AllStop`
- 이벤트: `AxisStateChanged`

예제:

```csharp
int axisNo = motion.ResolveAxisNo("Y");
var values = motion.GetMotionValues(axisNo);
values.Speed = 20000;
values.Acc = 10000;
values.Dec = 10000;
motion.SetMotionValues(axisNo, values);

await motion.MoveAbsAsync(axisNo, 150000, timeout: 5000);
var axis = motion.GetAxisSnapshot(axisNo);
Console.WriteLine($"{axis.AxisName} pos={axis.CurrentPos:F3}, moving={axis.IsMoving}");
```

## 5. WPF(Jog/Main) 연동 패턴

`RatelWPF/JogWindow.xaml.cs` 기준:
- 상태 바인딩: `Axes = RatelWpfContext.MotionView.Axes`
- 조작: `RatelWpfContext.MotionCore.ServoOn/Home/Stop/Jog`
- 이동: `RatelWpfContext.MotionView.MoveAbsAsync(SelectedAxis)`

역할 분리:
- `MotionCore`: 실제 제어/조회
- `MotionView`: UI 편의 계층

## 6. Script Host 연동 패턴

`RatelWPF/Run/ScriptV3HostFunctionRegistrar.cs` 기준:
- 축 해석: `_motion.ResolveAxisNo(axisName)`
- 이동: `_motion.MoveAbsAsync(...)`, `_motion.MoveIncAsync(...)`
- 상태: `_motion.GetAxisSnapshot(axisNo).CurrentPos`, `.IsMoving`
- 정지: `_motion.Stop(axisNo)`, `_motion.AllStop()`

핵심:
- 스크립트는 `MotionViewManager`가 아니라 `IMotionRuntime`를 직접 사용한다.

## 7. 단일 축 클래스 정책

축 상태 타입은 `MotionAxis` 단일 클래스 사용:
- 런타임 외부 반환은 `Clone()` 복사본
- 외부에서 내부 폴링 상태를 직접 오염시키지 않도록 보호

## 8. 권장 구현 규칙

- 장비/시나리오 로직은 `IMotionRuntime`만 사용
- UI 바인딩은 `MotionViewManager`/`MotionAxisItem`으로 제한
- 신규 프로젝트도 `Runtime + ViewAdapter` 패턴 유지
