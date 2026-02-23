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

### 3-3. MotionManager가 관리하는 Controller에서 전용 함수 호출

`UnifiedMotion`에서는 컨트롤러 인스턴스를 `MotionManager`가 관리하므로,
가능하면 외부에서 별도 인스턴스를 따로 운용하지 말고 `motion[axisNo].Controller` 경유로 접근하는 것을 권장합니다.

```csharp
using RatelSoft.Utils.UnifiedMotion;

public interface IMyMotionFeature
{
    string Func1();
}

public sealed class MyMotionController : VController, IMyMotionFeature
{
    public MyMotionController(int index, int maxMotorCount = 32) : base(index, maxMotorCount) { }

    public string Func1() => $"Func1 called. Index={Index}";
}

// 축 등록 이후
if (motion[0].Controller is IMyMotionFeature feature)
{
    var result = feature.Func1();
}
```

참고:
- `MotionAxis.Controller`는 조회(`get`)만 외부 공개되고, 설정(`set`)은 런타임 내부(`internal`)에서만 수행됩니다.
- 따라서 호출자는 컨트롤러 소유권/수명주기를 건드리지 않고, 전용 기능만 안전하게 확장해서 사용할 수 있습니다.

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
