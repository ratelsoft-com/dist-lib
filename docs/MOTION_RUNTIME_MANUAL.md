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

## 7. 장비 전용 기능 확장 패턴

기본 `UnifiedMotion` API는 공통 모션 기능만 제공합니다.
현장에서는 여기에 컨트롤러 전용 기능을 추가해야 하는 경우가 많습니다.

예:
- 특정 컨트롤러의 유지보수 명령
- 벤더 전용 진단/리셋 기능
- 특수 파라미터 읽기/쓰기

단순 접근 방식은 다음과 같습니다.

```csharp
if (motion[axisNo].Controller is IVendorFeature feature)
{
    feature.Func1();
}
```

이 방식은 동작은 맞지만, 호출부마다 캐스팅 코드가 반복되고 공통 기능과 전용 기능의 사용 방식이 분리됩니다.

그래서 `UnifiedMotion`은 `MotionAdapterBase` 기반 확장을 권장합니다.

### 7-1. MotionAdapterBase 역할

`MotionAdapterBase`는 `MotionManager` 위에 얇게 올라가는 사용자용 베이스 클래스입니다.

- 공통 API: `MoveAbsAsync`, `MoveIncAsync`, `Stop`, `Home`, `ServoOn`, `AllStop` 등은 그대로 위임
- 조회/검증: `GetAxis(...)`, `GetController(...)`, `GetFeature<TFeature>(...)`
- 장비 전용 기능: 사용자 파생 클래스에서 추가

즉, 런타임의 주체는 계속 `MotionManager`이고, 어댑터는 사용자 편의용 진입점만 제공합니다.

### 7-2. 권장 구현 구조

```csharp
public interface IVendorFeature
{
    string Func1();
}

public sealed class VendorController : VController, IVendorFeature
{
    public VendorController(int index) : base(index)
    {
    }

    public string Func1() => "ok";
}

public sealed class VendorMotionAdapter : MotionAdapterBase
{
    public VendorMotionAdapter(MotionManager motion) : base(motion)
    {
    }

    public string Func1(int axisNo)
    {
        return GetFeature<IVendorFeature>(axisNo).Func1();
    }
}
```

사용 예:

```csharp
var adapter = new VendorMotionAdapter(motion);
await adapter.MoveAbsAsync(axisNo, 1000);
var result = adapter.Func1(axisNo);
```

Alarm Reset 같이 벤더 전용 유지보수 기능도 같은 방식으로 노출하는 것을 권장합니다.

```csharp
public interface IAlarmResetFeature
{
    void AlarmReset(int innerAxisNo);
}

public sealed class VendorMotionAdapter : MotionAdapterBase
{
    public VendorMotionAdapter(MotionManager motion) : base(motion)
    {
    }

    public void AlarmReset(int axisNo)
    {
        var axis = GetAxis(axisNo);
        GetFeature<IAlarmResetFeature>(axisNo).AlarmReset(axis.InnerAxisNo);
    }
}
```

사용자는 다음처럼 호출하면 된다.

```csharp
var adapter = new VendorMotionAdapter(motion);
adapter.AlarmReset(axisNo);
```

### 7-3. 여러 컨트롤러 혼용 시 장점

현장 장비는 축마다 서로 다른 컨트롤러에 매핑되는 경우가 흔합니다.
어댑터 패턴을 사용하면 호출부는 항상 `axisNo` 기준으로만 접근하면 됩니다.

- 어떤 컨트롤러가 바인딩되어 있는지는 런타임이 관리
- 기능 지원 여부 판단은 어댑터 내부에서 처리
- UI/시퀀스/스크립트 코드는 장비 타입 분기에서 자유로워짐

### 7-4. 구현 원칙

- 어댑터는 `MotionManager`를 감싸는 얇은 facade로 유지
- 모션 실행 로직을 어댑터에 재구현하지 않음
- 전용 기능은 컨트롤러 인터페이스로 정의하고 어댑터에서 래핑
- 예외 메시지와 기능 지원 검사는 어댑터 내부로 집중

## 8. 단일 축 클래스 정책

축 상태 타입은 `MotionAxis` 단일 클래스 사용:
- 런타임 외부 반환은 `Clone()` 복사본
- 외부에서 내부 폴링 상태를 직접 오염시키지 않도록 보호

추가로 `MotionAxis.Controller`는 외부 조회가 가능하지만 설정은 런타임 내부에서만 수행된다.
이 값은 사용자 어댑터가 축에 연결된 실제 컨트롤러를 확인하는 용도로 사용한다.

## 9. 권장 구현 규칙

- 장비/시나리오 로직은 `IMotionRuntime`만 사용
- UI 바인딩은 `MotionViewManager`/`MotionAxisItem`으로 제한
- 신규 프로젝트도 `Runtime + ViewAdapter` 패턴 유지
- 장비 전용 기능은 `MotionAdapterBase` 파생 클래스로 별도 진입점 제공
