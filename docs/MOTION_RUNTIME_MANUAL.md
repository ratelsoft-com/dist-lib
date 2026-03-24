# Motion Runtime Manual

`UnifiedMotion`은 축 등록, 상태 폴링, 공통 모션 명령, 컨트롤러별 확장을 한 라이브러리 안에서 다루기 위한 런타임 계층이다.
이 문서는 `MotionManager`, `IMotionRuntime`, `MotionRuntime`, `MotionAdapterBase`를 기준으로 독립 라이브러리 관점에서 사용 방법을 설명한다.

## 1. 핵심 타입

- `MotionManager`
  - 축과 컨트롤러를 보관하고, `Open()` 이후 컨트롤러별 `SetInfo()` 폴링을 수행한다.
  - 실제 이동, 정지, 홈, 서보, 조그 명령의 진입점이다.
- `MotionAxis`
  - 축 상태와 파라미터를 담는 엔티티다.
  - `AxisNo`, `AxisName`, `InnerAxisNo`, `CurrentPos`, `ActSpeed`, `AmpFault`, `Values` 등을 가진다.
- `MotionControllerBase`
  - 장비별 컨트롤러 구현의 베이스다.
  - `AbsMove`, `IncMove`, `Stop`, `Home`, `SetInfo` 같은 동작을 구현한다.
- `IMotionRuntime`
  - 상태 조회와 공통 조작을 분리해 사용하는 런타임 인터페이스다.
  - 상태 스냅샷과 이벤트를 외부에 안전하게 노출한다.
- `MotionRuntime`
  - `MotionManager`를 감싼 런타임 어댑터다.
  - 상태 변경 감시와 `AxisStateChanged` 이벤트를 제공한다.
- `MotionAdapterBase`
  - 공통 모션 API 위에 컨트롤러 전용 기능을 얹는 사용자용 베이스 클래스다.

## 2. 기본 구성 순서

기본 흐름은 다음과 같다.

1. 컨트롤러 인스턴스를 만든다.
2. `MotionAxis`를 축 번호별로 등록한다.
3. `MotionManager.Open()`으로 실제 컨트롤러를 연다.
4. 필요하면 `MotionRuntime`을 만들어 상태 조회와 이벤트를 분리한다.
5. 장비 전용 기능이 있으면 `MotionAdapterBase` 파생 클래스를 만든다.

가장 단순한 구성 예시는 다음과 같다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

var motion = new MotionManager();
var controller = new VController(0);

motion.AddAxis(0, new MotionAxis
{
    AxisName = "X",
    InnerAxisNo = 0
}, controller);

motion.AddAxis(1, new MotionAxis
{
    AxisName = "Y",
    InnerAxisNo = 1
}, controller);

motion.Open();
```

컨트롤러를 여러 개 섞어 쓰는 경우도 동일하다.

```csharp
var motion = new MotionManager();

var vendorA = new VController(0);
var vendorB = new AnotherController(0);

motion.AddAxis(0, new MotionAxis { AxisName = "LoaderX", InnerAxisNo = 0 }, vendorA);
motion.AddAxis(1, new MotionAxis { AxisName = "LoaderY", InnerAxisNo = 1 }, vendorA);
motion.AddAxis(2, new MotionAxis { AxisName = "InspectZ", InnerAxisNo = 0 }, vendorB);

motion.Open();
```

`MotionManager`는 같은 형식과 같은 `Index`를 가진 컨트롤러를 내부에서 재사용하므로, 축마다 같은 컨트롤러를 중복으로 열지 않는다.

## 3. MotionAxis 의미

`MotionAxis`는 상태와 설정을 함께 가진다.

주요 속성:

- `AxisNo`: 런타임 외부에서 사용하는 축 번호
- `AxisName`: 이름 기반 접근용 식별자
- `InnerAxisNo`: 실제 컨트롤러 내부 축 번호
- `CurrentPos`, `DestPos`, `ActSpeed`, `CmdSpeed`: 현재 상태
- `AmpEnable`, `AmpFault`, `HomeComplete`, `InPosition`: 상태 비트
- `Values`: 속도, 가감속, 이동 모드 같은 파라미터 묶음
- `Controller`: 해당 축을 관리하는 실제 컨트롤러 참조

예:

```csharp
var axis = motion[0];
Console.WriteLine($"{axis.AxisName}: pos={axis.CurrentPos}, moving={axis.IsMoving}");
```

이름으로도 접근할 수 있다.

```csharp
var axis = motion["X"];
Console.WriteLine(axis.InnerAxisNo);
```

주의:
- `MotionAxis.Controller`는 조회용이다.
- `Controller`의 설정은 런타임 내부에서만 수행된다.
- 외부에서 직접 명령을 내려도 되지만, 공통 명령은 `MotionManager` 또는 `IMotionRuntime`를 우선 사용하는 편이 구조상 안전하다.

## 4. MotionManager 직접 사용

`MotionManager`는 가장 단순한 진입점이다.

### 4-1. 이동

```csharp
await motion.MoveAbsAsync(0, 150000, timeout: 5000);
await motion.MoveIncAsync(0, -5000, timeout: 5000);
await motion.WaitMoveDoneAsync(0);
```

### 4-2. 속도와 모션 값 변경

```csharp
var values = motion[0].Values.Clone();
values.Speed = 20000;
values.Acc = 10000;
values.Dec = 10000;
values.Pos = 150000;
values.RelMove = false;

await motion.MoveAsync(0, values, timeout: 5000);
```

### 4-3. Jog, Stop, Home, Servo

```csharp
motion.ServoOn(0, true);
motion.Home(0);
motion.MoveJogP(0);
Thread.Sleep(500);
motion.Stop(0);
motion.AllStop();
```

### 4-4. 이름 기반 축 해석

```csharp
var axisNo = motion["Y"].AxisNo;
await motion.MoveAbsAsync(axisNo, 25000);
```

## 5. IMotionRuntime 사용

`IMotionRuntime`는 상태 조회와 제어를 외부 레이어에서 분리해서 쓸 때 적합하다.
UI, 서비스, 스크립트, 시퀀스 엔진에서 `MotionManager` 전체를 직접 들고 다니지 않고도 일관된 API를 사용할 수 있다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

var manager = new MotionManager();
var controller = new VController(0);

manager.AddAxis(0, new MotionAxis { AxisName = "X", InnerAxisNo = 0 }, controller);
manager.AddAxis(1, new MotionAxis { AxisName = "Y", InnerAxisNo = 1 }, controller);
manager.Open();

IMotionRuntime runtime = new MotionRuntime(manager);
```

### 5-1. 상태 조회

```csharp
var all = runtime.GetAllAxisSnapshots();
foreach (var axis in all)
{
    Console.WriteLine($"{axis.AxisNo}:{axis.AxisName} pos={axis.CurrentPos:F3} moving={axis.IsMoving}");
}

var x = runtime.GetAxisSnapshot("X");
Console.WriteLine(x.CurrentPos);
```

### 5-2. 축 번호 해석

```csharp
int axisNo = runtime.ResolveAxisNo("Y");
await runtime.MoveAbsAsync(axisNo, 30000);
```

### 5-3. 모션 값 읽기/쓰기

```csharp
var values = runtime.GetMotionValues(0);
values.Speed = 15000;
values.Acc = 8000;
values.Dec = 8000;
runtime.SetMotionValues(0, values);
```

### 5-4. 상태 이벤트

```csharp
runtime.AxisStateChanged += (_, e) =>
{
    var axis = e.Axis;
    Console.WriteLine(
        $"AxisChanged no={axis.AxisNo}, pos={axis.CurrentPos:F3}, moving={axis.IsMoving}, alarm={axis.AmpFault}");
};
```

`MotionRuntime`은 내부적으로 `MotionManager.Status`를 주기적으로 비교해서 바뀐 축만 이벤트로 내보낸다. 이벤트에 포함되는 `MotionAxis`는 복사본이므로 외부 코드가 내부 상태를 오염시키지 않는다.

## 6. 서비스/시퀀스/스크립트 패턴

`IMotionRuntime`는 특정 UI 프레임워크 없이도 그대로 사용할 수 있다.

### 6-1. 서비스 클래스 예제

```csharp
public sealed class LoaderMotionService
{
    private readonly IMotionRuntime _motion;

    public LoaderMotionService(IMotionRuntime motion)
    {
        _motion = motion;
    }

    public async Task MoveLoadPositionAsync(CancellationToken token = default)
    {
        int x = _motion.ResolveAxisNo("LoaderX");
        int z = _motion.ResolveAxisNo("LoaderZ");

        await _motion.MoveAbsAsync(z, 0, timeout: 5000, token);
        await _motion.MoveAbsAsync(x, 125000, timeout: 5000, token);
    }

    public MotionAxis GetXAxis() => _motion.GetAxisSnapshot("LoaderX");
}
```

### 6-2. 상태 폴링 루프 예제

```csharp
while (!token.IsCancellationRequested)
{
    var axis = runtime.GetAxisSnapshot("Y");
    Console.WriteLine($"Y pos={axis.CurrentPos:F3}, inpos={axis.InPosition}, fault={axis.AmpFault}");
    await Task.Delay(100, token);
}
```

### 6-3. 비상 정지 예제

```csharp
try
{
    await runtime.MoveAbsAsync(0, 100000, timeout: 3000, token);
}
catch (OperationCanceledException)
{
    runtime.AllStop();
    throw;
}
catch
{
    runtime.AllStop();
    throw;
}
```

## 7. MotionAdapterBase

현장 코드에서는 공통 모션 API만으로 끝나지 않는 경우가 많다.
예를 들면 다음과 같은 컨트롤러 전용 기능이 필요할 수 있다.

- Alarm Reset
- Vendor Parameter Read/Write
- Homing 모드 전환
- Servo pack 특수 진단
- 통신 버퍼 초기화

이 기능을 호출부에서 매번 캐스팅하면 금방 지저분해진다.

```csharp
if (motion[axisNo].Controller is IVendorFeature feature)
{
    feature.Func1();
}
```

그래서 `UnifiedMotion`은 `MotionAdapterBase` 기반의 사용자 어댑터 패턴을 제공한다.

### 7-1. 기본 구조

```csharp
using RatelSoft.Utils.UnifiedMotion;

public abstract class MotionAdapterBase
{
    protected MotionAdapterBase(MotionManager motion)
    {
        Motion = motion ?? throw new ArgumentNullException(nameof(motion));
    }

    protected MotionManager Motion { get; }

    public Task MoveAbsAsync(int axisNo, double position, int timeout = 0, CancellationToken token = default)
        => Motion.MoveAbsAsync(axisNo, position, timeout, token);

    public void Stop(int axisNo)
        => Motion.Stop(axisNo);

    protected TFeature GetFeature<TFeature>(int axisNo)
        where TFeature : class
    {
        var controller = Motion[axisNo].Controller
            ?? throw new InvalidOperationException($"Axis {axisNo} has no controller.");

        return controller as TFeature
            ?? throw new NotSupportedException(
                $"Axis {axisNo} controller {controller.GetType().Name} does not support {typeof(TFeature).Name}.");
    }
}
```

### 7-2. Func1 예제

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

사용:

```csharp
var adapter = new VendorMotionAdapter(motion);
var result = adapter.Func1(0);
await adapter.MoveAbsAsync(0, 1000);
```

### 7-3. Alarm Reset 예제

이 패턴이 가장 유용한 대표 예가 `Alarm Reset`이다.

```csharp
public interface IAlarmResetFeature
{
    void AlarmReset(int innerAxisNo);
}

public sealed class VendorController : MotionControllerBase, IAlarmResetFeature
{
    public VendorController(int index) : base(index)
    {
    }

    public override MotionControllerType ControllerType => MotionControllerType.Custom;
    public override string Kind => "mycompany.motion.custom";

    public override void SetInfo() { }
    public override void AbsMove(int axisNo, double position) { }
    public override void IncMove(int axisNo, double position) { }
    public override void Stop(int axisNo) { }
    public override void Home(int axisNo) { }

    public void AlarmReset(int innerAxisNo)
    {
        // vendor SDK call
    }
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

사용:

```csharp
var adapter = new VendorMotionAdapter(motion);
adapter.AlarmReset(0);
```

호출부는 이제 컨트롤러 타입, `InnerAxisNo`, 인터페이스 캐스팅을 알 필요가 없다.

### 7-4. 여러 컨트롤러 혼용 예제

```csharp
var motion = new MotionManager();

motion.AddAxis(0, new MotionAxis { AxisName = "X", InnerAxisNo = 0 }, new VendorController(0));
motion.AddAxis(1, new MotionAxis { AxisName = "Y", InnerAxisNo = 1 }, new VendorController(0));
motion.AddAxis(2, new MotionAxis { AxisName = "StageZ", InnerAxisNo = 0 }, new AnotherController(0));

motion.Open();

var adapter = new VendorMotionAdapter(motion);
adapter.AlarmReset(0);
await adapter.MoveAbsAsync(1, 10000);
```

지원되지 않는 축에 기능을 호출하면 `MotionAdapterBase.GetFeature<TFeature>()`가 명확한 예외를 발생시킨다.

## 8. 사용자 컨트롤러 등록 패턴

컨트롤러를 직접 생성해 써도 되고, 레지스트리 기반으로 생성할 수도 있다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

public sealed class MyMotionController : MotionControllerBase
{
    public MyMotionController(int index) : base(index) { }

    public override MotionControllerType ControllerType => MotionControllerType.Custom;
    public override string Kind => "mycompany.motion.custom";

    public override void SetInfo() { }
    public override void AbsMove(int axisNo, double position) { }
    public override void IncMove(int axisNo, double position) { }
    public override void Stop(int axisNo) { }
    public override void Home(int axisNo) { }
}
```

```csharp
MotionControllerTypeRegistry.Register(
    kind: "mycompany.motion.custom",
    creator: index => new MyMotionController(index),
    mappedType: MotionControllerType.Custom,
    overwrite: true);

var controller = MotionControllerFactory.Create("mycompany.motion.custom", index: 0);
```

## 9. 권장 규칙

- 축 등록과 컨트롤러 수명 관리는 `MotionManager`에 집중한다.
- 외부 레이어는 가능하면 `IMotionRuntime` 또는 `MotionAdapterBase`를 사용한다.
- 공통 모션 동작은 어댑터에 재구현하지 않는다.
- 컨트롤러 전용 기능은 인터페이스로 정의하고 어댑터에서 노출한다.
- `MotionAxis.Controller` 직접 사용은 예외적인 진단이나 어댑터 내부로 제한한다.
- 장비/업무 시퀀스는 축 번호보다 축 이름을 우선 사용하고, 최종 실행 직전에 `ResolveAxisNo()`로 변환하는 편이 유지보수에 유리하다.

## 10. 정리

`UnifiedMotion`의 기준 구조는 다음과 같다.

- 축과 컨트롤러의 실제 소유자: `MotionManager`
- 상태 스냅샷과 이벤트 분리 계층: `IMotionRuntime` / `MotionRuntime`
- 컨트롤러 전용 기능의 사용자 진입점: `MotionAdapterBase`

이 구조를 유지하면 공통 모션 API와 장비 전용 기능을 한 프로젝트 안에서 섞어 쓰더라도, 호출부는 단순하게 유지할 수 있다.
