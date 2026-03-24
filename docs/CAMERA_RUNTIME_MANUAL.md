# Camera Runtime Manual

`UnifiedCamera`는 카메라 연결, 단일 그랩, 연속 그랩, 프레임 조합, 공통 설정, 장비 전용 확장을 한 표면에서 다루기 위한 라이브러리다.
이 문서는 `ICamera`, `CameraBase`, `CameraFactory`, `CameraTypeRegistry`, `CameraAdapterBase`를 기준으로 독립 라이브러리 관점에서 사용 방법을 설명한다.

## 1. 핵심 타입

- `ICamera`
  - 공통 카메라 인터페이스다.
  - 연결, 그랩, 트리거, 폭/높이/노출/게인 설정, 이벤트를 제공한다.
- `CameraBase`
  - 공통 베이스 구현이다.
  - 상태 전이, 이벤트 전달, 프레임 버퍼링, 이미지 조합을 담당한다.
- `CameraFactory`
  - `CameraType` 또는 `kind` 문자열로 카메라를 만든다.
- `CameraTypeRegistry`
  - 기본 카메라와 사용자 카메라 생성자를 등록한다.
- `CameraSettings`
  - 폭, 노출, 게인, 라인레이트, 트리거, 조합 모드를 묶어 가진다.
- `GrabResult`
  - 그랩 결과, `Mat`, 폭/높이, 성공 여부, 타임스탬프, 에러 메시지를 담는다.
- `CameraAdapterBase`
  - 공통 카메라 API 위에 사용자 전용 편의 함수나 장비 전용 기능을 얹는 베이스 클래스다.

## 2. 카메라 생성

기본 생성은 `CameraFactory`를 사용한다.

```csharp
using RatelSoft.Utils.UnifiedCamera;

using var camera = CameraFactory.CreateCamera(CameraType.Basler);
```

문자열 `kind` 기반 생성도 가능하다.

```csharp
using var camera = CameraFactory.CreateCamera("basler");
```

연결까지 한 번에 하고 싶으면 `CreateAndConnectAsync()`를 쓸 수 있다.

```csharp
using RatelSoft.Utils.UnifiedCamera;
using CameraService.Contracts;

var settings = CameraSettings.Default with
{
    Exposure = 100,
    Width = 2048,
    LinesPerFrame = 1024,
    FramesPerImage = 1
};

using var camera = await CameraFactory.CreateAndConnectAsync(
    CameraType.Basler,
    cameraName: "CAM1",
    settings: settings);
```

## 3. 연결과 해제

### 3-1. 장치 목록 조회

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var cameras = await camera.GetAvailableCamerasAsync();

foreach (var name in cameras)
{
    Console.WriteLine(name);
}
```

### 3-2. 이름 없이 첫 번째 카메라 연결

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var ok = await camera.ConnectAsync();
if (!ok)
{
    throw new InvalidOperationException("No camera connected.");
}
```

### 3-3. 이름으로 특정 카메라 연결

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var ok = await camera.ConnectAsync("CAM1");
if (!ok)
{
    throw new InvalidOperationException("CAM1 connect failed.");
}
```

### 3-4. 해제

```csharp
await camera.DisconnectAsync();
```

## 4. 가장 단순한 1장 그랩

```csharp
using RatelSoft.Utils.UnifiedCamera;

using var camera = CameraFactory.CreateCamera(CameraType.Basler);
await camera.ConnectAsync("CAM1");

using var result = await camera.GrabOneAsync(timeoutMs: 5000);
if (!result.Success || result.Image == null)
{
    throw new InvalidOperationException(result.ErrorMessage);
}

Console.WriteLine($"Grab success: {result.Width}x{result.Height}");

await camera.DisconnectAsync();
```

## 5. 연속 그랩과 이벤트

`UnifiedCamera`는 `FrameGrabbed`와 `ImageGrabbed`를 모두 제공한다.

- `FrameGrabbed`
  - 개별 프레임 기준 이벤트다.
  - 라인 조합이나 버스트 조합에서 중간 단위를 보고 싶을 때 사용한다.
- `ImageGrabbed`
  - 최종 완성 이미지 기준 이벤트다.
  - `FramesPerImage == 1`이면 사실상 프레임마다 바로 발생한다.

```csharp
camera.FrameGrabbed += (_, frame) =>
{
    Console.WriteLine($"Frame: success={frame.Success}, size={frame.Width}x{frame.Height}, index={frame.FrameIndex}");
};

camera.ImageGrabbed += (_, image) =>
{
    Console.WriteLine($"Image: success={image.Success}, size={image.Width}x{image.Height}, index={image.FrameIndex}");
};

await camera.StartGrabbingAsync();
await Task.Delay(2000);
await camera.StopGrabbingAsync();
```

## 6. 상태와 에러 이벤트

```csharp
camera.StateChanged += (_, e) =>
{
    Console.WriteLine($"State={e.State}, Message={e.Message}");
};

camera.ErrorOccurred += (_, message) =>
{
    Console.WriteLine($"ERROR: {message}");
};
```

이 이벤트는 UI 없이도 서비스, 콘솔, 테스트 프로그램에서 그대로 사용할 수 있다.

## 7. CameraSettings

`CameraSettings`는 공통 설정 묶음이다.

주요 필드:

- `Width`
- `Exposure`
- `Gain`
- `LineRate`
- `TriggerEnabled`
- `TriggerSource`
- `MaxBuffers`
- `LinesPerFrame`
- `FramesPerImage`
- `CompositionMode`

예:

```csharp
using CameraService.Contracts;

camera.Settings = CameraSettings.Default with
{
    Width = 4096,
    Exposure = 100,
    Gain = 1.5,
    TriggerEnabled = false,
    FramesPerImage = 1,
    LinesPerFrame = 1024,
    CompositionMode = ImageCompositionMode.Vertical,
};
```

설정을 적용한 뒤 실제 카메라 값을 다시 읽고 싶으면 `ReadCameraSettingsAsync()`를 사용한다.

```csharp
await camera.ReadCameraSettingsAsync();
Console.WriteLine(camera.Settings.Exposure);
```

## 8. 폭, 높이, 노출, 게인 제어

공통 API는 폭, 높이, 노출, 게인을 별도 함수로도 제공한다.

### 8-1. 읽기

```csharp
int width = await camera.GetImageWidthAsync();
int height = await camera.GetImageHeightAsync();
int exposure = await camera.GetExposureAsync();
double gain = await camera.GetGainAsync();

Console.WriteLine($"Size={width}x{height}, Exposure={exposure}, Gain={gain}");
```

### 8-2. 쓰기

```csharp
await camera.SetImageWidthAsync(2048);
await camera.SetImageHeightAsync(1024);
await camera.SetExposureAsync(100);
await camera.SetGainAsync(2.0);
```

폭과 높이를 같이 적용할 수도 있다.

```csharp
await camera.SetImageSizeAsync(2048, 1024);
```

실제 장비는 스텝, 최소/최대값, 정렬 조건 때문에 요청값과 실제 적용값이 다를 수 있다. 설정 후 다시 읽어서 확인하는 편이 안전하다.

```csharp
await camera.SetExposureAsync(100);
var actualExposure = await camera.GetExposureAsync();
Console.WriteLine($"Applied exposure={actualExposure}");
```

## 9. 트리거 제어

```csharp
using CameraService.Contracts;

await camera.SetTriggerModeAsync(true, TriggerSource.Software);
await camera.SoftwareTriggerAsync();

bool enabled = await camera.GetTriggerModeAsync();
TriggerSource source = await camera.GetTriggerSourceAsync();
```

하드웨어 트리거로 바꾸는 예:

```csharp
await camera.SetTriggerModeAsync(true, TriggerSource.Hardware);
```

## 10. Area / LineScan 해석

현재 `UnifiedCamera`는 `Area`와 `LineScan`을 별도 enum으로 강제하지 않는다.
대신 `CameraSettings`의 조합으로 수집 방식을 표현한다.

핵심은 다음과 같다.

- `FramesPerImage == 1`
  - 단일 프레임이 바로 완성 이미지가 된다.
  - 일반적인 에어리 카메라 사용 방식과 가깝다.
- `FramesPerImage > 1`
  - 프레임을 버퍼링한 뒤 `CompositionMode`에 따라 최종 이미지를 만든다.
- `LinesPerFrame`
  - 한 프레임에서 의미 있는 세로 크기 또는 라인 수로 사용된다.
- `CompositionMode`
  - `Vertical`, `Horizontal`, `Average`, `Maximum`, `Minimum` 중 하나다.

### 10-1. 일반적인 에어리 카메라 예

```csharp
camera.Settings = CameraSettings.Default with
{
    Width = 2448,
    LinesPerFrame = 2048,
    FramesPerImage = 1,
    CompositionMode = ImageCompositionMode.Vertical,
};
```

이 경우 한 프레임이 곧 한 이미지다.

### 10-2. 라인 조합 예

```csharp
camera.Settings = CameraSettings.Default with
{
    Width = 4096,
    LinesPerFrame = 256,
    FramesPerImage = 8,
    CompositionMode = ImageCompositionMode.Vertical,
};
```

이 경우 세로 256 라인짜리 프레임 8개가 모여 최종 이미지가 된다.

### 10-3. 평균 합성 예

`Average`, `Maximum`, `Minimum`은 에어리 카메라에서도 유효하다.
즉, `Area`와 `CompositionMode`는 같은 축의 개념이 아니다.

```csharp
camera.Settings = CameraSettings.Default with
{
    Width = 2048,
    LinesPerFrame = 2048,
    FramesPerImage = 4,
    CompositionMode = ImageCompositionMode.Average,
};
```

이 경우 4장의 프레임을 평균 내서 최종 이미지 하나를 만든다.

정리:
- `Area` / `LineScan`은 사용 패턴을 설명하는 말이다.
- 실제 라이브러리 동작은 `LinesPerFrame`, `FramesPerImage`, `CompositionMode` 조합으로 결정된다.

## 11. CameraAdapterBase

장비 전용 기능을 쓰기 시작하면 호출부에서 `if (camera is BaslerCamera basler)` 같은 코드가 반복되기 쉽다.
`CameraAdapterBase`는 이 반복을 줄이기 위한 베이스 클래스다.

### 11-1. 기본 구조

```csharp
using CameraService.Contracts;
using RatelSoft.Utils.UnifiedCamera;

public abstract class CameraAdapterBase : IDisposable
{
    protected ICamera Camera { get; }

    protected CameraAdapterBase(ICamera camera)
    {
        Camera = camera ?? throw new ArgumentNullException(nameof(camera));
    }

    public Task<bool> ConnectAsync(string? cameraName = null)
        => Camera.ConnectAsync(cameraName);

    public Task<GrabResult> GrabOneAsync(int timeoutMs = 1000)
        => Camera.GrabOneAsync(timeoutMs);

    protected TCamera GetCamera<TCamera>() where TCamera : class, ICamera
    {
        if (Camera is not TCamera typed)
            throw new InvalidOperationException($"Camera is not {typeof(TCamera).Name}.");

        return typed;
    }
}
```

### 11-2. 간단한 사용자 어댑터 예

```csharp
public sealed class CameraAppAdapter : CameraAdapterBase
{
    public CameraAppAdapter(ICamera camera) : base(camera)
    {
    }

    public async Task<(int width, int height)> ReadSizeAsync()
    {
        var width = await GetImageWidthAsync();
        var height = await GetImageHeightAsync();
        return (width, height);
    }
}
```

### 11-3. Basler 전용 어댑터 예

```csharp
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

        return await GrabOneAsync(5000);
    }

    public BaslerCamera GetBasler()
    {
        return GetCamera<BaslerCamera>();
    }
}
```

이 패턴은 공통 카메라 API를 유지하면서 외부 호출 코드를 정리하는 데 적합하다.

## 12. Basler 실사용 예

### 12-1. 첫 번째 물리 카메라와 첫 번째 에뮬레이터 열기

Basler emulator를 테스트에 포함하려면 프로세스 시작 시 `PYLON_CAMEMU`를 설정하는 것이 일반적이다.

```csharp
Environment.SetEnvironmentVariable("PYLON_CAMEMU", "1", EnvironmentVariableTarget.Process);

using var physical = new BaslerCameraAdapter(CameraFactory.CreateCamera(CameraType.Basler));
using var emulator = new BaslerCameraAdapter(CameraFactory.CreateCamera(CameraType.Basler));

var names = await physical.GetAvailableCamerasAsync();
var firstPhysical = names.First(n => !n.StartsWith("[EMU] "));
var firstEmulator = names.First(n => n.StartsWith("[EMU] "));

using var physicalImage = await physical.ConnectAndGrabAsync(firstPhysical);
using var emulatorImage = await emulator.ConnectAndGrabAsync(firstEmulator);

Console.WriteLine($"physical={physicalImage.Width}x{physicalImage.Height}");
Console.WriteLine($"emulator={emulatorImage.Width}x{emulatorImage.Height}");
```

현재 Basler 구현은 에뮬레이터 장치를 `[EMU] ` 접두사로 구분해서 반환한다. 식별 정보가 완전히 비어 있는 에뮬레이터 placeholder 항목은 필터링된다.

### 12-2. 노출 100 적용 후 한 장 그랩

```csharp
Environment.SetEnvironmentVariable("PYLON_CAMEMU", "1", EnvironmentVariableTarget.Process);

using var camera = CameraFactory.CreateCamera(CameraType.Basler);
await camera.ConnectAsync("CAM1");

await camera.SetExposureAsync(100);
var applied = await camera.GetExposureAsync();
Console.WriteLine($"Exposure={applied}");

using var result = await camera.GrabOneAsync(5000);
Console.WriteLine($"Grab={result.Success}, Size={result.Width}x{result.Height}");
```

장비가 스텝 제약을 가지면 읽어온 값이 `100`이 아닌 인접 값일 수 있다.

## 13. 사용자 카메라 등록 패턴

### 13-1. 새 kind 등록

```csharp
using RatelSoft.Utils.UnifiedCamera;

CameraTypeRegistry.Register(
    kind: "mycompany.camera.basler-ex",
    creator: () => new BaslerCameraEx(),
    mappedType: CameraType.Custom,
    overwrite: true);
```

### 13-2. 등록한 kind로 생성

```csharp
using var camera = CameraFactory.CreateCamera("mycompany.camera.basler-ex");
```

## 14. 상속과 어댑터의 역할 차이

카메라 쪽에는 확장 방식이 두 가지가 있다.

- 상속
  - 내부 파이프라인을 확장할 때 사용한다.
  - 예: `BaslerCameraEx : BaslerCamera`
  - 연결 직후 설정, 프레임 처리, `GrabResult` 변환을 바꾸고 싶을 때 적합하다.
- 어댑터
  - 외부 호출 코드를 정리할 때 사용한다.
  - 예: `BaslerCameraAdapter : CameraAdapterBase`
  - `Connect + Grab + 설정 읽기` 같은 반복 시나리오를 묶고 싶을 때 적합하다.

즉,
- 내부 동작 확장: 상속
- 외부 사용 편의 확장: 어댑터

## 15. 권장 규칙

- 공통 기능은 `ICamera` / `CameraAdapterBase`로 사용한다.
- 장비 내부 파이프라인을 바꿀 때만 상속을 사용한다.
- 요청값을 설정한 뒤 실제 장비값을 다시 읽어 검증하는 습관을 갖는다.
- 라인/에어리 구분은 강제 enum보다 현재 `CameraSettings` 조합을 먼저 기준으로 해석한다.
- `GrabResult.Image`는 `Mat`이므로 수명 관리를 명확히 한다.
- 이벤트 핸들러에서 받은 이미지를 오래 들고 있을 경우 `Clone()` 후 별도로 관리한다.

## 16. 정리

`UnifiedCamera`의 기준 구조는 다음과 같다.

- 공통 카메라 표면: `ICamera`
- 상태/조합/이벤트를 포함한 베이스 구현: `CameraBase`
- 카메라 생성: `CameraFactory`, `CameraTypeRegistry`
- 외부 사용 편의 계층: `CameraAdapterBase`
- 장비 내부 파이프라인 확장: 사용자 파생 카메라 클래스

이 구조를 유지하면 공통 카메라 코드와 장비 전용 기능을 분리하면서도, 콘솔 프로그램, 서비스, UI 프로그램에서 같은 런타임 모델을 재사용할 수 있다.
