# Unified Custom Device Guide

이 문서는 `UnifiedCamera`, `UnifiedIO`, `UnifiedMotion`에서 사용자 정의 장비를 추가하거나, 기존 장비 구현 위에 현장 전용 기능을 얹는 방법을 설명한다.

핵심 원칙은 다음과 같다.

- 공통 동작은 라이브러리 기본 API를 유지한다.
- 장비별 차이는 `kind + registry`, 상속, 어댑터로 분리한다.
- 장비 내부 파이프라인 확장과 외부 호출 편의 확장을 구분한다.
- 공개 라이브러리 표면은 단순하게 두고, 현장 전용 기능은 사용자 코드에서 얹는다.

## 1. 확장 방식 개요

Unified 계열의 확장은 보통 세 가지 방식으로 나뉜다.

### 1-1. kind + registry

새 장비 종류를 등록해서 `Create("my.kind")`로 생성하는 방식이다.

적합한 경우:
- 새 장비 타입 추가
- 기본 enum 외에 현장 전용 식별자 추가
- 런타임에 장비 생성자를 주입하고 싶을 때

### 1-2. 상속

기존 장비 구현의 내부 파이프라인 일부를 바꾸는 방식이다.

적합한 경우:
- 연결 직후 설정 추가
- 수신 프레임 후처리
- 장비 SDK 객체에 직접 접근해야 하는 추가 동작

### 1-3. 어댑터

공통 런타임 위에 사용자 전용 기능을 얹는 방식이다.

적합한 경우:
- 호출부 반복 코드 제거
- 장비 전용 기능을 명시적 메서드로 노출
- 여러 장비가 섞여도 호출부를 단순하게 유지하고 싶을 때

정리:
- 새 장비 등록: `registry`
- 내부 동작 확장: `inheritance`
- 외부 사용 편의 확장: `adapter`

## 2. UnifiedCamera 확장

## 2-1. 카메라 등록

새 카메라를 문자열 `kind`로 등록할 수 있다.

```csharp
using RatelSoft.Utils.UnifiedCamera;

CameraTypeRegistry.Register(
    kind: "mycompany.camera.custom",
    creator: () => new MyCamera(),
    mappedType: CameraType.Custom,
    overwrite: true);

using var camera = CameraFactory.CreateCamera("mycompany.camera.custom");
```

직접 구현 예:

```csharp
using CameraService.Contracts;
using RatelSoft.Utils.UnifiedCamera;

public sealed class MyCamera : CameraBase
{
    public override CameraType Type => CameraType.Custom;
    public override string Kind => "mycompany.camera.custom";

    public override Task<bool> ConnectAsync(string? cameraName = null)
        => Task.FromResult(true);

    public override Task DisconnectAsync()
        => Task.CompletedTask;

    public override Task<bool> StartGrabbingAsync(int grabCount)
        => Task.FromResult(true);

    public override Task StopGrabbingAsync()
        => Task.CompletedTask;

    public override Task<GrabResult> GrabOneAsync(int timeoutMs = 1000)
    {
        return Task.FromResult(new GrabResult
        {
            Success = true,
            Width = 640,
            Height = 480,
            Timestamp = DateTime.Now
        });
    }

    public override Task<bool> SoftwareTriggerAsync()
        => Task.FromResult(true);

    public override Task<bool> SetTriggerModeAsync(bool enabled, TriggerSource source)
        => Task.FromResult(true);

    public override Task<bool> ResetAsync()
        => Task.FromResult(true);

    public override Task<bool> GetInputStateAsync()
        => Task.FromResult(false);

    public override Task<List<string>> GetAvailableCamerasAsync()
        => Task.FromResult(new List<string> { "MyCamera-1" });
}
```

## 2-2. BaslerCameraEx 패턴

카메라 확장에서 가장 중요한 패턴은 `BaslerCameraEx` 같은 상속 구조다.
이 패턴의 핵심은 "새 Basler 드라이버를 다시 작성하는 것"이 아니라, 기존 `BaslerCamera`의 연결, 에러 처리, 이벤트 배선, 그랩 파이프라인을 그대로 재사용하면서 필요한 단계만 바꾸는 데 있다.

### 2-2-1. 기본 형태

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";
}
```

### 2-2-2. 왜 이 구조가 중요한가

`BaslerCamera`는 이미 다음 책임을 가진다.

- 카메라 탐색과 연결
- Grabber 이벤트 연결
- 상태 전이와 에러 처리
- 단일 그랩과 연속 그랩
- `GrabResult` 변환과 프레임 전달

즉, 사용자 입장에서는 Basler SDK 초기화와 이벤트 wiring을 다시 작성하지 않고도, 설정, 이벤트, 결과 변환 같은 확장 포인트만 선택적으로 오버라이드할 수 있다.

### 2-2-3. 주요 확장 포인트

- `_camera`
  - Basler SDK 객체 직접 접근용 escape hatch
- `ApplySettingsAsync()`
  - 연결 직후 추가 파라미터 적용
- `OnBaslerImageGrabbed(...)`
  - 이벤트 시점 진단과 추적 코드 삽입
- `ConvertGrabResult(...)`
  - 결과 이미지 변환과 후처리

### 2-2-4. 예제: 연결 직후 설정 추가

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

### 2-2-5. 예제: 결과 이미지 후처리

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

### 2-2-6. 예제: 이벤트 시점 진단 코드 추가

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraEx : BaslerCamera
{
    public override string Kind => "mycompany.camera.basler-ex";

    protected override void OnBaslerImageGrabbed(object? sender, Basler.Pylon.ImageGrabbedEventArgs e)
    {
        // tracing, counter, conditional logging
        base.OnBaslerImageGrabbed(sender, e);
    }
}
```

### 2-2-7. 권장 규칙

권장:
- 설정 추가는 `ApplySettingsAsync()`에 넣는다.
- 결과 후처리는 `ConvertGrabResult()`에 넣는다.
- 이벤트 추적은 `OnBaslerImageGrabbed(...)`에 넣는다.
- `_camera` 직접 접근은 필요한 지점으로 제한한다.

비권장:
- `ConnectAsync()` 전체를 복사해서 재구현한다.
- `_camera` 접근 코드를 여러 호출부에 흩뿌린다.
- 결과 변환 메서드에서 연결 상태 제어까지 같이 처리한다.

## 2-3. CameraAdapterBase 패턴

카메라 쪽은 상속만으로 끝나지 않는다.
실제 사용자 프로그램에서는 `Connect + Settings Read + GrabOne` 같은 반복 동작이나, 장비 전용 기능을 묶는 외부 진입점도 필요하다.

이 역할은 `CameraAdapterBase`가 맡는다.

### 2-3-1. 기본 예

```csharp
using RatelSoft.Utils.UnifiedCamera;

public sealed class BaslerCameraAdapter : CameraAdapterBase
{
    public BaslerCameraAdapter(ICamera camera) : base(camera)
    {
    }

    public async Task<GrabResult> ConnectAndGrabAsync(string cameraName)
    {
        var ok = await ConnectAsync(cameraName);
        if (!ok)
        {
            throw new InvalidOperationException($"Connect failed: {cameraName}");
        }

        await ReadCameraSettingsAsync();
        return await GrabOneAsync(5000);
    }
}
```

사용:

```csharp
using var adapter = new BaslerCameraAdapter(CameraFactory.CreateCamera(CameraType.Basler));
using var image = await adapter.ConnectAndGrabAsync("CAM1");
Console.WriteLine($"{image.Width}x{image.Height}");
```

### 2-3-2. 상속과 어댑터 차이

- `BaslerCameraEx`
  - 내부 파이프라인 확장
- `BaslerCameraAdapter`
  - 외부 호출 코드 정리

두 방식은 경쟁 관계가 아니라 서로 다른 계층이다.

## 2-4. Basler emulator 예제

```csharp
Environment.SetEnvironmentVariable("PYLON_CAMEMU", "1", EnvironmentVariableTarget.Process);

using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var names = await camera.GetAvailableCamerasAsync();

foreach (var name in names)
{
    Console.WriteLine(name);
}
```

현재 Basler 구현은 emulator를 `[EMU] ` 접두사로 구분해서 반환한다.
식별 정보가 완전히 비어 있는 emulator placeholder 항목은 필터링된다.

## 3. UnifiedIO 확장

## 3-1. IO 장치 구현

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

## 3-2. 등록과 생성

```csharp
IODeviceTypeRegistry.Register(
    kind: "mycompany.io.custom",
    creator: () => new MyIoDevice(),
    mappedType: IODeviceType.Custom,
    overwrite: true);

var io = IODeviceFactory.Create("mycompany.io.custom");
```

## 4. UnifiedMotion 확장

Motion 쪽은 Camera와 성격이 다르다.
여러 컨트롤러가 축 단위로 섞여 쓰이기 때문에, 내부 파이프라인 상속보다 어댑터 패턴이 더 중요하다.

## 4-1. 컨트롤러 등록

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

## 4-2. MotionAdapterBase 패턴

`UnifiedMotion`에서 컨트롤러 인스턴스는 `MotionManager`가 소유한다.
따라서 사용자 코드는 새 컨트롤러를 따로 만들어 들고 있기보다, 축에 바인딩된 컨트롤러를 기준으로 전용 기능을 노출하는 편이 구조상 맞다.

가장 단순한 접근은 다음과 같다.

```csharp
if (motion[axisNo].Controller is IVendorFeature feature)
{
    feature.Func1();
}
```

하지만 이 패턴은 호출부마다 캐스팅이 반복된다. 그래서 `MotionAdapterBase`를 권장한다.

### 4-2-1. Func1 예제

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
var motion = new MotionManager();
var controller = new VendorController(0);

motion.AddAxis(0, new MotionAxis { AxisName = "X", InnerAxisNo = 0 }, controller);
motion.Open();

var adapter = new VendorMotionAdapter(motion);
var result = adapter.Func1(0);
await adapter.MoveAbsAsync(0, 1000);
```

### 4-2-2. Alarm Reset 예제

현장에서 더 자주 필요한 형태는 `Alarm Reset` 같은 벤더 전용 유지보수 기능이다.

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

이렇게 하면 호출자는 컨트롤러 타입이나 `InnerAxisNo`를 알 필요가 없다.

## 5. 어떤 방식을 선택할 것인가

### 5-1. Camera

- 새 장비 타입 추가
  - `CameraTypeRegistry.Register(...)`
- Basler 기본 동작 위에 연결/이벤트/이미지 처리 추가
  - `BaslerCameraEx : BaslerCamera`
- 외부 호출 코드를 정리하고 테스트/서비스용 API를 만들기
  - `CameraAdapterBase`

### 5-2. IO

- 새 장치 타입 추가
  - `IODeviceTypeRegistry.Register(...)`
- 프로토콜/채널 수/카운터 규칙이 다른 장치 구현
  - `IODeviceBase` 파생 클래스

### 5-3. Motion

- 새 컨트롤러 타입 추가
  - `MotionControllerTypeRegistry.Register(...)`
- 공통 모션 API는 유지하고 장비 전용 기능만 노출
  - `MotionAdapterBase`

## 6. 권장 구조

- 공개 라이브러리 표면은 공통 API 중심으로 유지한다.
- 장비 내부 파이프라인 수정은 상속으로 처리한다.
- 현장 전용 기능과 반복 호출은 어댑터로 정리한다.
- 문자열 `kind`는 충돌을 피하기 위해 회사/프로젝트 접두사를 붙인다.
  - 예: `mycompany.camera.basler-ex`
  - 예: `mycompany.io.custom`
  - 예: `mycompany.motion.custom`

## 7. 정리

Unified 계열의 확장은 한 가지 방식으로 해결되지 않는다.

- 생성 경로 확장: `registry`
- 내부 동작 확장: `inheritance`
- 외부 사용 편의 확장: `adapter`

이 세 축을 분리해서 설계하면, 기본 라이브러리를 건드리지 않고도 현장 전용 장비와 기능을 안정적으로 얹을 수 있다.
