# Unified Custom Device Guide

이 문서는 Unified 시리즈(`UnifiedCamera`, `UnifiedIO`, `UnifiedMotion`)에서 사용자 정의 디바이스를 추가하는 방법을 설명합니다.

## 핵심 개념

- 기존 `enum` 기반 API는 그대로 사용 가능합니다.
- 새 확장 방식은 `string kind + registry` 입니다.
- 사용자 코드는 런타임에 디바이스 생성자를 등록한 뒤 `Create("my.kind")`로 생성합니다.
- 등록은 앱 시작 시 1회 수행을 권장합니다.

## 1) UnifiedCamera 확장

### 1-1. BaslerCameraEx 작성

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";

    protected override GrabResult ConvertGrabResult(Basler.Pylon.IGrabResult grabResult)
    {
        var result = base.ConvertGrabResult(grabResult);
        // 사용자 후처리(메타데이터, 필터링 등)
        return result;
    }
}
```

### 1-2. Registry 등록

```csharp
CameraTypeRegistry.Register(
    kind: "mycompany.camera.basler-ex",
    creator: () => new BaslerCameraEx(),
    mappedType: CameraType.Custom);
```

### 1-3. 생성

```csharp
var camera = CameraFactory.CreateCamera("mycompany.camera.basler-ex");
// 또는 enum 경로 유지 시
var legacy = CameraFactory.CreateCamera(CameraType.Basler);
```

### 1-4. 왜 `BaslerCameraEx` 패턴이 중요한가

이 패턴의 핵심은 "새 Basler 드라이버를 다시 만드는 것"이 아니라,
기존 `BaslerCamera`의 연결/그랩/이벤트 파이프라인을 그대로 유지한 채 필요한 단계만 커스터마이즈하는 데 있습니다.

`BaslerCamera`는 이미 다음 책임을 가지고 있습니다.

- 카메라 탐색 및 연결
- StreamGrabber 이벤트 연결
- 기본 상태 전이와 에러 처리
- 단일 그랩/연속 그랩 제어
- `GrabResult` 생성과 `OnFrameReceived(...)` 호출

즉, 사용자 입장에서는 Basler SDK 초기화와 이벤트 wiring을 다시 작성하지 않고도
"설정 단계", "프레임 이벤트 단계", "결과 변환 단계"만 선택적으로 바꿀 수 있습니다.

### 1-5. 열려 있는 확장 포인트와 책임

`BaslerCameraEx : BaslerCamera` 구조에서 실제로 의미 있는 확장 포인트는 다음과 같습니다.

- `_camera`
  - Basler SDK 객체 직접 접근용 escape hatch
  - 정말 필요한 경우에만 사용
- `ApplySettingsAsync()`
  - 연결 직후 추가 파라미터를 적용하는 단계
  - UserSet, Trigger, Vendor Parameter, Counter/Timer류 설정에 적합
- `OnBaslerImageGrabbed(...)`
  - 프레임 수신 이벤트를 가로채는 단계
  - 추가 로깅, 조건부 드롭, 프레임 카운팅, 진단 코드 삽입에 적합
- `ConvertGrabResult(...)`
  - 최종 `GrabResult` 생성 방식을 바꾸는 단계
  - 이미지 후처리, 메타데이터 보강, 포맷 변환에 적합

중요한 점은 각 포인트의 용도가 다르다는 것입니다.
설정 변경은 `ApplySettingsAsync()`, 결과 가공은 `ConvertGrabResult()`에 넣는 식으로 책임을 맞춰야 유지보수가 쉬워집니다.

### 1-6. 언제 이 패턴을 쓰는가

다음처럼 "Basler 자체는 그대로 쓰되 일부만 후킹"하고 싶은 경우에 적합합니다.

- 연결 직후 Basler 파라미터를 추가 적용하고 싶을 때
- 받은 프레임에 사용자 후처리를 넣고 싶을 때
- `GrabResult`에 타임스탬프 외 사용자 메타데이터를 추가하고 싶을 때
- 기본 Basler 동작은 유지하면서 현장별 규칙만 덧붙이고 싶을 때

반대로, Basler 전체 동작을 완전히 새로 바꾸려는 목적이라면 상속보다 별도 구현이 더 나을 수 있습니다.

### 1-7. 예제 1: 연결 직후 사용자 설정 추가

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";

    protected override async Task ApplySettingsAsync()
    {
        await base.ApplySettingsAsync();

        if (_camera == null)
        {
            return;
        }

        await Task.Run(() =>
        {
            _camera.Parameters[Basler.Pylon.PLCamera.UserSetSelector].SetValue("Default");
            _camera.Parameters[Basler.Pylon.PLCamera.UserSetLoad].Execute();
            _camera.Parameters[Basler.Pylon.PLCamera.ChunkModeActive].SetValue(true);
        });
    }
}
```

이 예제의 의미:

- 연결/이벤트/에러 처리 구조는 그대로 유지
- 연결 직후 필요한 Basler 파라미터만 추가 적용
- 기존 `BaslerCamera` 개선 사항을 계속 상속받음

### 1-8. 예제 2: 프레임 수신 직후 추가 후처리

```csharp
using OpenCvSharp;
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";

    protected override GrabResult ConvertGrabResult(Basler.Pylon.IGrabResult grabResult)
    {
        var result = base.ConvertGrabResult(grabResult);
        if (!result.Success || result.Image == null)
        {
            return result;
        }

        var filtered = new Mat();
        Cv2.GaussianBlur(result.Image, filtered, new Size(3, 3), 0);

        result.Image.Dispose();
        result.Image = filtered;
        return result;
    }
}
```

이 예제의 의미:

- Basler grab 결과 생성은 기본 구현을 재사용
- 사용자 후처리만 마지막 단계에서 추가
- 이벤트 파이프라인은 건드리지 않음

### 1-9. 예제 3: 이벤트 단계에 진단 코드 추가

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";

    protected override void OnBaslerImageGrabbed(object? sender, Basler.Pylon.ImageGrabbedEventArgs e)
    {
        // 필요 시 grab 상태, 카운터, 간단한 진단 로깅 추가
        base.OnBaslerImageGrabbed(sender, e);
    }
}
```

이 오버라이드는 프레임 자체를 바꾸기보다는,
이벤트 시점에 진단 로직이나 현장 전용 추적 코드를 넣고 싶을 때 적합합니다.

### 1-10. 권장/비권장 패턴

권장:

- 설정 추가는 `ApplySettingsAsync()`에 넣기
- 결과 후처리는 `ConvertGrabResult()`에 넣기
- 이벤트 추적/모니터링은 `OnBaslerImageGrabbed(...)`에 넣기
- `_camera` 직접 접근은 필요한 지점으로 제한하기

비권장:

- `ConnectAsync()` 전체를 복사해서 재구현하기
- `_camera` 접근 코드를 여러 메서드/호출부에 흩뿌리기
- `ConvertGrabResult()`에서 연결 상태 제어까지 같이 처리하기

### 1-11. Motion 확장과 다른 점

Camera 쪽의 `BaslerCameraEx`는 "단일 장비 파이프라인을 부분 오버라이드하는 상속 패턴"입니다.
반면 Motion 쪽은 여러 컨트롤러가 축 단위로 섞여 사용되므로, 호출부 편의를 위해 어댑터 패턴이 더 중요합니다.

즉,

- Camera: 기존 장비 파이프라인 재사용 + 일부 단계 오버라이드
- Motion: 런타임 위에 사용자 어댑터를 얹어 전용 기능 노출

두 구조는 목적이 다르며, 문서도 이 차이를 기준으로 이해하는 것이 좋습니다.

## 2) UnifiedIO 확장

### 2-1. 사용자 IO 구현

```csharp
using RatelSoft.Utils.UnifiedIO;

public sealed class MyIoDevice : IODeviceBase
{
    public override IODeviceType Type => IODeviceType.Custom;
    public override string Kind => "mycompany.io.custom";
    public override int InputChannelCount => 16;
    public override int OutputChannelCount => 16;

    protected override Task<bool> OnConnectAsync(string? endpoint, CancellationToken cancellationToken)
        => Task.FromResult(true);

    protected override Task OnDisconnectAsync(CancellationToken cancellationToken)
        => Task.CompletedTask;

    protected override Task<bool[]> OnReadInputsAsync(CancellationToken cancellationToken)
        => Task.FromResult(new bool[InputChannelCount]);

    protected override Task<bool[]> OnReadOutputsAsync(CancellationToken cancellationToken)
        => Task.FromResult(new bool[OutputChannelCount]);

    protected override Task<bool> OnWriteOutputAsync(int channel, bool value, CancellationToken cancellationToken)
        => Task.FromResult(true);

    protected override Task<bool> OnClearInputCountAsync(int? channel, CancellationToken cancellationToken)
        => Task.FromResult(true);

    protected override Task<uint?> OnReadInputCountAsync(int channel, CancellationToken cancellationToken)
        => Task.FromResult<uint?>(null);
}
```

### 2-2. Registry 등록 및 생성

```csharp
IODeviceTypeRegistry.Register(
    kind: "mycompany.io.custom",
    creator: () => new MyIoDevice(),
    mappedType: IODeviceType.Custom);

var io = IODeviceFactory.Create("mycompany.io.custom");
```

## 3) UnifiedMotion 확장

### 3-1. 사용자 Controller 구현

```csharp
using RatelSoft.Utils.UnifiedMotion;

public sealed class MyMotionController : MotionControllerBase
{
    public MyMotionController(int index) : base(index) { }

    public override MotionControllerType ControllerType => MotionControllerType.Custom;
    public override string Kind => "mycompany.motion.custom";

    public override void SetInfo()
    {
        // 상태 동기화 구현
    }

    public override void AbsMove(int axisNo, double position) { }
    public override void IncMove(int axisNo, double position) { }
    public override void Stop(int axisNo) { }
    public override void Home(int axisNo) { }
}
```

### 3-2. Registry 등록 및 생성

```csharp
MotionControllerTypeRegistry.Register(
    kind: "mycompany.motion.custom",
    creator: index => new MyMotionController(index),
    mappedType: MotionControllerType.Custom);

var controller = MotionControllerFactory.Create("mycompany.motion.custom", index: 0);
```

### 3-3. 단순 캐스팅 방식의 한계

`UnifiedMotion`에서는 컨트롤러 인스턴스를 `MotionManager`가 소유하고 관리합니다.
따라서 사용자 코드가 컨트롤러를 별도로 새로 만들기보다는, 축에 바인딩된 컨트롤러에 접근하는 것이 구조상 맞습니다.

예를 들어 장비별 전용 기능이 인터페이스로 노출되어 있다면 다음처럼 접근할 수 있습니다.

```csharp
if (motion[0].Controller is IMyMotionFeature feature)
{
    var result = feature.Func1();
}
```

하지만 현장 코드에서는 이 패턴만으로는 다음 문제가 자주 발생합니다.

- 호출부마다 `is` / 캐스팅 코드가 반복됨
- 장비가 2~3종 섞이면 기능 분기 로직이 UI/시퀀스 코드로 퍼짐
- 공통 모션 기능(`MoveAbsAsync`, `Stop`, `Home`)과 장비 전용 기능이 서로 다른 진입점으로 분리됨

즉, 구조적으로는 맞지만 사용성 측면에서는 호출 코드가 거칠어집니다.

### 3-4. 권장 패턴: MotionAdapterBase + 사용자 어댑터

이 문제를 줄이기 위해 `UnifiedMotion` 라이브러리는 `MotionAdapterBase`를 제공합니다.
이 클래스는 `MotionManager` 위에 얇은 어댑터 계층을 두는 용도입니다.

역할은 다음과 같습니다.

- 공통 기능은 `MotionManager`로 그대로 위임
- 장비 전용 기능은 파생 클래스에서 메서드로 노출
- 내부적으로만 `Controller` / 인터페이스 캐스팅 수행

핵심 베이스 예시는 다음과 같습니다.

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

    protected TFeature GetFeature<TFeature>(int axisNo) where TFeature : class
    {
        var controller = Motion[axisNo].Controller
            ?? throw new InvalidOperationException($"Axis {axisNo} has no controller.");

        return controller as TFeature
            ?? throw new NotSupportedException(
                $"Axis {axisNo} controller {controller.GetType().Name} does not support {typeof(TFeature).Name}.");
    }
}
```

### 3-5. 사용자 컨트롤러 + 어댑터 구현 예시

컨트롤러는 전용 기능을 인터페이스로 노출합니다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

public interface IMyMotionFeature
{
    string Func1();
}

public sealed class MyMotionController : VController, IMyMotionFeature
{
    public MyMotionController(int index, int maxMotorCount = 32) : base(index, maxMotorCount)
    {
    }

    public string Func1()
    {
        return $"Func1 called. Index={Index}";
    }
}
```

그 위에 사용자용 어댑터를 얹습니다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

public sealed class MyMotionAdapter : MotionAdapterBase
{
    public MyMotionAdapter(MotionManager motion) : base(motion)
    {
    }

    public string Func1(int axisNo)
    {
        return GetFeature<IMyMotionFeature>(axisNo).Func1();
    }
}
```

### 3-6. 실제 사용 흐름

축 구성 단계에서는 기존처럼 컨트롤러를 등록합니다.

```csharp
var motion = new MotionManager();
var controller = new MyMotionController(0);

motion.AddAxis(0, new MotionAxis
{
    AxisName = "X",
    InnerAxisNo = 0
}, controller);

motion.Open();
```

사용 단계에서는 어댑터만 사용합니다.

```csharp
var adapter = new MyMotionAdapter(motion);

await adapter.MoveAbsAsync(0, 1000);
var result = adapter.Func1(0);
adapter.Stop(0);
```

이 패턴의 장점:

- 호출부가 컨트롤러 타입을 몰라도 됨
- 공통 기능과 장비 전용 기능을 동일한 객체에서 사용 가능
- 축 번호 기준으로 여러 컨트롤러를 자연스럽게 혼용 가능
- 캐스팅, 예외 메시지, 기능 지원 여부 판단을 한 곳에 모을 수 있음

### 3-7. 현장 예제: Alarm Reset 기능 추가

실제 현장에서는 `Move`, `Home` 같은 공통 기능보다,
컨트롤러 벤더 전용 기능이 더 자주 문제를 만듭니다.
대표적인 예가 `Alarm Reset`입니다.

이 기능을 호출부에서 직접 캐스팅하며 쓰기 시작하면 다음처럼 됩니다.

```csharp
if (motion[axisNo].Controller is IAlarmResetFeature feature)
{
    feature.AlarmReset(axisNo);
}
```

한두 번은 괜찮지만, 실제 코드에서는 아래 문제가 생깁니다.

- 화면/시퀀스/스크립트마다 같은 캐스팅 코드 반복
- 컨트롤러가 2~3종 섞이면 지원 여부 분기가 호출부로 퍼짐
- 공통 모션 API와 전용 유지보수 API가 분리되어 사용성 저하

그래서 `MotionAdapterBase` 위에 전용 진입점을 주는 방식이 더 적합합니다.

#### 3-7-1. 컨트롤러 인터페이스

```csharp
public interface IAlarmResetFeature
{
    void AlarmReset(int innerAxisNo);
}
```

#### 3-7-2. 사용자 컨트롤러 구현

```csharp
using RatelSoft.Utils.UnifiedMotion;

public sealed class MyMotionController : MotionControllerBase, IAlarmResetFeature
{
    public MyMotionController(int index) : base(index)
    {
    }

    public override MotionControllerType ControllerType => MotionControllerType.Custom;
    public override string Kind => "mycompany.motion.custom";

    public override void SetInfo()
    {
    }

    public override void AbsMove(int axisNo, double position)
    {
    }

    public override void IncMove(int axisNo, double position)
    {
    }

    public override void Stop(int axisNo)
    {
    }

    public override void Home(int axisNo)
    {
    }

    public void AlarmReset(int innerAxisNo)
    {
        // 벤더 SDK 또는 명령 프로토콜 호출
    }
}
```

#### 3-7-3. 사용자 어댑터 구현

```csharp
using RatelSoft.Utils.UnifiedMotion;

public sealed class MyMotionAdapter : MotionAdapterBase
{
    public MyMotionAdapter(MotionManager motion) : base(motion)
    {
    }

    public void AlarmReset(int axisNo)
    {
        var axis = GetAxis(axisNo);
        GetFeature<IAlarmResetFeature>(axisNo).AlarmReset(axis.InnerAxisNo);
    }
}
```

#### 3-7-4. 사용자 호출 코드

```csharp
var adapter = new MyMotionAdapter(motion);

adapter.AlarmReset(0);
await adapter.MoveAbsAsync(0, 1000);
adapter.Stop(0);
```

이렇게 하면 호출자는 더 이상 컨트롤러 타입, 인터페이스 캐스팅, 내부 축 번호(`InnerAxisNo`)를 신경 쓸 필요가 없습니다.
즉, 어댑터가 "현장 코드용 API 표면" 역할을 하게 됩니다.

### 3-8. 설계 규칙

`MotionAdapterBase`를 사용할 때는 다음 규칙을 권장합니다.

- 어댑터는 `MotionManager`의 대체물이 아니라 얇은 facade여야 합니다.
- 공통 모션 로직을 재구현하지 말고 `MotionManager`로 위임하세요.
- 장비 전용 기능은 컨트롤러에 두고, 어댑터는 이를 사용자 친화적인 메서드로 래핑하세요.
- 가능하면 컨트롤러 전용 기능은 인터페이스(`IMyMotionFeature`)로 노출하세요.
- `MotionAxis.Controller`는 조회(`get`)만 외부 공개되고, 설정(`set`)은 런타임 내부(`internal`)에서만 유지하세요.

## 4) 운영 권장사항

- `kind` 네이밍은 회사/제품 prefix를 붙이세요.
  - 예: `mycompany.camera.basler-ex`
- 등록은 앱 시작 시 1회만 수행하세요.
- 이미 등록된 `kind`를 바꾸려면 `overwrite: true`를 명시하세요.
- 기존 코드 호환이 필요하면 enum API를 그대로 유지하고, 신규 장비만 `kind` 경로로 추가하세요.

## 5) BaslerCameraEx 상속 구조 체크

현재 구조에서는 `BaslerCameraEx : BaslerCamera` 확장이 가능하도록 다음 확장 포인트가 열려 있습니다.

- `_camera` 접근: `protected`
- 오버라이드 가능:
  - `ApplySettingsAsync()`
  - `OnBaslerImageGrabbed(...)`
  - `ConvertGrabResult(...)`

따라서 Basler 기본 구현을 재사용하면서 사용자 로직을 덧붙이는 방식으로 안전하게 확장할 수 있습니다.
