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
- `CameraSelector`
  - 카메라 이름 문자열을 직접 맞춰 넣지 않고, 첫 번째 물리 카메라, 첫 번째 emulator, `FriendlyName`, `FullName` 같은 기준으로 장치를 선택하기 위한 selector 타입이다.
- `CameraDescriptor`
  - 장치 목록 표시용 메타데이터다.
  - `DisplayName`, `CanonicalName`, `IsEmulator`, `FriendlyName`, `FullName`, `UserDefinedName`, `DeviceSerialNumber`를 담는다.

## 2. 벤더 DLL 배치 정책

`UnifiedCamera`의 벤더 DLL 정책은 벤더별로 다르다.

원칙:

- 라이브러리 프로젝트는 컴파일을 위해 벤더 DLL을 참조할 수 있다.
- 실제 실행 파일 옆에 어떤 DLL을 둘지는 사용자 프로젝트가 결정한다.
- Basler/MVS는 사용자가 자신이 준비한 `Basler.Pylon.dll`, `MvCameraControl.Net.dll` 경로를 명시해야 한다.
- Matrox는 패키지 내부 `buildTransitive\vendor\matrox\...` 경로에 기본 managed DLL이 함께 포함되며, 필요하면 사용자 경로로 override할 수 있다.

중요한 점:

- 단순히 앱 폴더 아래에 `dll` 폴더를 하나 만드는 것만으로는 충분하지 않을 수 있다.
- 빌드가 어떤 파일을 출력 폴더로 복사할지 알기 위해서는 경로를 명시해야 한다.
- 이 목적을 위해 `BaslerPylonManagedFile`, `MvsManagedFile`, `MatroxMilManagedFile`, `MatroxMilManagedFolder` 속성이 제공된다.

### 2-1. 가장 권장하는 방식

소비자 솔루션 루트에 `Directory.Build.props`를 두고, 사용자가 직접 준비한 DLL 경로를 적는다.

```xml
<Project>
  <PropertyGroup>
    <BaslerPylonManagedFile>$(MSBuildThisFileDirectory)vendor\basler\Basler.Pylon.dll</BaslerPylonManagedFile>
    <MvsManagedFile>$(MSBuildThisFileDirectory)vendor\mvs\MvCameraControl.Net.dll</MvsManagedFile>
    <MatroxMilManagedFile>$(MSBuildThisFileDirectory)vendor\matrox\10.70\Matrox.MatroxImagingLibrary.dll</MatroxMilManagedFile>
  </PropertyGroup>
</Project>
```

이렇게 하면:

- 사용자가 어떤 DLL을 쓸지 명확해진다.
- 패키지 소비 경로에서는 해당 DLL이 출력 폴더로 복사된다.
- 벤더 SDK 설치 경로와 무관하게, 프로젝트가 함께 배포할 DLL을 고정할 수 있다.

Matrox만 따로 보면 다음 규칙으로 동작한다.

- `MatroxMilManagedFile`이 비어 있지 않고 파일이 존재하면 그 경로를 우선 사용한다.
- `MatroxMilManagedFile`이 비어 있으면 패키지의 `buildTransitive\vendor\matrox\$(MatroxMilManagedFolder)\Matrox.MatroxImagingLibrary.dll`을 기본값으로 사용한다.
- `MatroxMilManagedFolder` 기본값은 현재 `10.70`이다.

### 2-2. Matrox 패키지 기본 DLL을 그대로 쓰는 방식

`UnifiedCamera` NuGet 패키지를 `PackageReference`로 소비하면, Matrox managed DLL은 패키지 내부 fallback을 바로 사용할 수 있다.

```xml
<Project>
  <PropertyGroup>
    <MatroxMilManagedFolder>10.70</MatroxMilManagedFolder>
  </PropertyGroup>
</Project>
```

이 경우 별도 `MatroxMilManagedFile`를 적지 않아도, 패키지에 포함된 `vendor\matrox\10.70\Matrox.MatroxImagingLibrary.dll`이 사용된다.

단, 실제 장비 구동에는 managed DLL만으로 충분하지 않을 수 있다.
Matrox MIL 런타임과 보드 드라이버, 관련 native DLL은 장비 PC에 정상 설치되어 있어야 한다.

### 2-3. 직접 앱 프로젝트에 참조 추가하는 방식

소스 프로젝트를 직접 참조하는 구조라면, 최상위 앱 프로젝트에 명시적으로 참조를 넣는 편이 가장 분명하다.

```xml
<ItemGroup>
  <Reference Include="Basler.Pylon">
    <HintPath>$(BaslerPylonManagedFile)</HintPath>
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

`Private=true`는 해당 DLL을 실행 폴더로 복사한다는 뜻이다.

### 2-4. 실제 예: `RatelWPF`처럼 프로젝트 내부 `CameraAssembly` 폴더를 쓰는 방식

예를 들어 앱 프로젝트가 아래처럼 DLL을 직접 보관한다고 가정한다.

```text
RatelWPF/
  CameraAssembly/
    pylon/
      2506/
        Basler.Pylon.dll
```

이 경우 앱 프로젝트는 다음처럼 설정할 수 있다.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <BaslerPylonManagedFile Condition="'$(BaslerPylonManagedFile)' == ''">
      CameraAssembly\pylon\2506\Basler.Pylon.dll
    </BaslerPylonManagedFile>
  </PropertyGroup>

  <ItemGroup>
    <Reference Include="Basler.Pylon" Condition="Exists('$(BaslerPylonManagedFile)')">
      <HintPath>$(BaslerPylonManagedFile)</HintPath>
      <Private>true</Private>
    </Reference>
  </ItemGroup>
</Project>
```

이 설정의 의미:

- `UnifiedCamera`는 Basler DLL을 자동 복사하지 않는다.
- `RatelWPF`가 자기 프로젝트 내부에 보관한 DLL을 직접 참조한다.
- 빌드 시 `Basler.Pylon.dll`이 `RatelWPF.exe` 출력 폴더로 복사된다.

이 방식은 설치 경로에 의존하지 않기 때문에 팀 내 테스트 환경을 맞추기 쉽다.

### 2-5. 설치 경로를 직접 사용하는 예

```xml
<Project>
  <PropertyGroup>
    <BaslerPylonManagedFile>C:\Program Files\Basler\pylon\Development\Assemblies\Basler.Pylon.dll</BaslerPylonManagedFile>
  </PropertyGroup>
</Project>
```

이 방식도 가능하지만, 배포 재현성은 프로젝트 내부 `vendor\...` 경로를 쓰는 방식보다 떨어진다.

## 3. 카메라 생성

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

selector 기반 overload도 제공된다.

```csharp
using var camera = await CameraFactory.CreateAndConnectAsync(
    CameraType.Basler,
    CameraSelector.FirstEmulator(),
    settings);
```

## 4. 연결과 해제

### 4-1. 장치 목록 조회

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var cameras = await camera.GetAvailableCamerasAsync();

foreach (var name in cameras)
{
    Console.WriteLine(name);
}
```

### 4-1-1. descriptor 목록 조회

표시 문자열 외에 raw 메타데이터가 필요하면 descriptor 목록을 조회한다.

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var descriptors = await camera.GetAvailableCameraDescriptorsAsync();

foreach (var item in descriptors)
{
    Console.WriteLine($"Display={item.DisplayName}");
    Console.WriteLine($"  Emulator={item.IsEmulator}");
    Console.WriteLine($"  Friendly={item.FriendlyName}");
    Console.WriteLine($"  Full={item.FullName}");
}
```

### 4-2. selector 기반 연결

Basler처럼 실카메라와 emulator가 섞이는 환경에서는 문자열 display name을 직접 맞춰 넣는 것보다 `CameraSelector`를 사용하는 편이 안전하다.

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);

await camera.ConnectAsync(CameraSelector.FirstPhysical());
await camera.DisconnectAsync();

await camera.ConnectAsync(CameraSelector.FirstEmulator());
await camera.DisconnectAsync();
```

지원되는 대표 selector:

- `CameraSelector.First()`
- `CameraSelector.FirstPhysical()`
- `CameraSelector.FirstEmulator()`
- `CameraSelector.DisplayName("CAM1")`
- `CameraSelector.FriendlyName("Basler Emulation (0815-0000)")`
- `CameraSelector.FullName("Emulation (0815-0000)")`
- `CameraSelector.UserDefinedName("CAM1")`

주의:

- Basler는 `FirstPhysical`, `FirstEmulator` 구분이 의미 있다.
- Matrox는 현재 emulator 개념이 없고 단일 descriptor 기반이라 `Default`, `First`, `DisplayName`, `FriendlyName`, `FullName`, `UserDefinedName` 중심으로 쓰는 편이 맞다.

### 4-3. 이름 없이 첫 번째 카메라 연결

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var ok = await camera.ConnectAsync();
if (!ok)
{
    throw new InvalidOperationException("No camera connected.");
}
```

### 4-4. 이름으로 특정 카메라 연결

```csharp
using var camera = CameraFactory.CreateCamera(CameraType.Basler);
var ok = await camera.ConnectAsync("CAM1");
if (!ok)
{
    throw new InvalidOperationException("CAM1 connect failed.");
}
```

### 4-5. 해제

```csharp
await camera.DisconnectAsync();
```

## 5. 가장 단순한 1장 그랩

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

## 6. 연속 그랩과 이벤트

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

## 7. 상태와 에러 이벤트

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

## 8. CameraSettings

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

## 9. 폭, 높이, 노출, 게인 제어

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

Matrox 구현은 현재 `ExposureTime` 계열 제어가 추가되어 있어 노출 읽기/쓰기가 가능하다.
내부적으로는 `M_EXPOSURE_TIME`을 먼저 시도하고, 실패하면 `ExposureTime` feature fallback을 사용한다.

## 10. 트리거 제어

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

## 11. Area / LineScan 해석

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

### 11-1. 일반적인 에어리 카메라 예

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

### 11-2. 라인 조합 예

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

### 11-3. 평균 합성 예

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

## 12. CameraAdapterBase

장비 전용 기능을 쓰기 시작하면 호출부에서 `if (camera is BaslerCamera basler)` 같은 코드가 반복되기 쉽다.
`CameraAdapterBase`는 이 반복을 줄이기 위한 베이스 클래스다.

### 12-1. 기본 구조

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

    public Task<bool> ConnectAsync(CameraSelector selector)
        => Camera.ConnectAsync(selector);

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

### 12-2. 간단한 사용자 어댑터 예

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

### 12-3. Basler 전용 어댑터 예

```csharp
public sealed class BaslerCameraAdapter : CameraAdapterBase
{
    public BaslerCameraAdapter(ICamera camera) : base(camera)
    {
    }

    public async Task<GrabResult> ConnectAndGrabAsync(CameraSelector selector)
    {
        var ok = await ConnectAsync(selector);
        if (!ok)
        {
            throw new InvalidOperationException($"Connect failed: {selector.Mode}");
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

현재 소스에는 `MatroxCameraAdapter`도 포함되어 있다.
Matrox처럼 연결 전에 `DcfPath` 같은 전용 속성을 채워야 하는 장비는, 이런 전용 adapter로 준비 단계를 묶는 편이 안전하다.

## 13. Matrox 실사용 예

현재 Matrox 구현은 `CameraType.Matrox`로 생성 가능한 실구현이다.
다만 Camera Link grabber 특성상, 연결 전에 최소한 `DcfPath`를 지정해야 한다.

### 13-1. 기본 연결

```csharp
using RatelSoft.Utils.UnifiedCamera;

using var camera = (MatroxCamera)CameraFactory.CreateCamera(CameraType.Matrox);
camera.DcfPath = @"C:\Config\Matrox\MyCamera.dcf";

var ok = await camera.ConnectAsync();
if (!ok)
{
    throw new InvalidOperationException("Matrox connect failed.");
}
```

현재 기본 descriptor 이름은 `Board0:Channel0`이다.
`DcfPath`가 없거나 파일이 없으면 연결은 실패한다.

### 13-2. 보드/채널 메타데이터와 descriptor

```csharp
using var camera = (MatroxCamera)CameraFactory.CreateCamera(CameraType.Matrox);
camera.DcfPath = @"C:\Config\Matrox\LineCamera.dcf";
camera.SystemDescriptor = "M_SYSTEM_SOLIOS";
camera.BoardName = "Board0";
camera.ChannelName = "Channel0";

var descriptors = await camera.GetAvailableCameraDescriptorsAsync();
foreach (var item in descriptors)
{
    Console.WriteLine($"Display={item.DisplayName}");
    Console.WriteLine($"Friendly={item.FriendlyName}");
    Console.WriteLine($"Full={item.FullName}");
    Console.WriteLine($"UserDefined={item.UserDefinedName}");
}
```

현재 Matrox descriptor는 다음 기준으로 만들어진다.

- `DisplayName`, `CanonicalName`: `Board0:Channel0`
- `FriendlyName`: `BoardName:ChannelName`
- `FullName`: `SystemDescriptor BoardName ChannelName DCF=<파일명>`
- `UserDefinedName`, `DeviceSerialNumber`: `DCF` 파일명 기반

즉, 화면에는 `DisplayName`을 보여주고, 내부 연결에는 `FriendlyName` 또는 `FullName` selector를 쓰는 구성이 가능하다.

### 13-3. selector 기반 연결

```csharp
using var camera = (MatroxCamera)CameraFactory.CreateCamera(CameraType.Matrox);
camera.DcfPath = @"C:\Config\Matrox\LineCamera.dcf";

await camera.ConnectAsync(CameraSelector.First());
await camera.DisconnectAsync();

await camera.ConnectAsync(CameraSelector.DisplayName("Board0:Channel0"));
await camera.DisconnectAsync();

await camera.ConnectAsync(CameraSelector.FullName("M_SYSTEM_SOLIOS Board0 Channel0 DCF=LineCamera.dcf"));
```

현재 Matrox selector는 아래 모드를 지원한다.

- `CameraSelector.Default()`
- `CameraSelector.First()`
- `CameraSelector.DisplayName(...)`
- `CameraSelector.FriendlyName(...)`
- `CameraSelector.FullName(...)`
- `CameraSelector.UserDefinedName(...)`

`FirstPhysical()`, `FirstEmulator()`는 Matrox 구현과는 맞지 않으므로 사용하지 않는 편이 낫다.

### 13-4. 노출 적용과 1장 그랩

```csharp
using var camera = (MatroxCamera)CameraFactory.CreateCamera(CameraType.Matrox);
camera.DcfPath = @"C:\Config\Matrox\LineCamera.dcf";

await camera.ConnectAsync();

await camera.SetExposureAsync(100);
var applied = await camera.GetExposureAsync();
Console.WriteLine($"Exposure={applied}");

using var result = await camera.GrabOneAsync(5000);
Console.WriteLine($"Grab={result.Success}, Size={result.Width}x{result.Height}");
```

현재 Matrox 구현은 1장 그랩과 노출 제어는 반영되어 있다.
반면 트리거, reset, gain, width/height 변경, 연속 그랩 파이프라인은 장비별 완성 구현으로 보기 어렵다.
따라서 Matrox를 붙이는 앱은 우선 `ConnectAsync` + `GrabOneAsync` + `Exposure` 중심으로 검증하는 편이 안전하다.

## 14. Basler 실사용 예

### 14-1. 첫 번째 물리 카메라와 첫 번째 에뮬레이터 열기

Basler emulator를 테스트에 포함하려면 프로세스 시작 시 `PYLON_CAMEMU`를 설정하는 것이 일반적이다.

```csharp
Environment.SetEnvironmentVariable("PYLON_CAMEMU", "1", EnvironmentVariableTarget.Process);

using var physical = new BaslerCameraAdapter(CameraFactory.CreateCamera(CameraType.Basler));
using var emulator = new BaslerCameraAdapter(CameraFactory.CreateCamera(CameraType.Basler));

using var physicalImage = await physical.ConnectAndGrabAsync(CameraSelector.FirstPhysical());
using var emulatorImage = await emulator.ConnectAndGrabAsync(CameraSelector.FirstEmulator());

Console.WriteLine($"physical={physicalImage.Width}x{physicalImage.Height}");
Console.WriteLine($"emulator={emulatorImage.Width}x{emulatorImage.Height}");
```

현재 Basler 구현은 에뮬레이터 장치를 `[EMU] ` 접두사로 구분해서 반환한다. 식별 정보가 완전히 비어 있는 에뮬레이터 placeholder 항목은 필터링된다.

권장 연결 규칙:

- 목록 표시용: `GetAvailableCamerasAsync()`
- 목록 + 메타데이터 표시용: `GetAvailableCameraDescriptorsAsync()`
- 실제 선택/연결용: `CameraSelector`

즉, UI는 목록 문자열을 보여줘도 되지만, 내부 연결은 가능하면 아래 selector를 쓰는 편이 낫다.

```csharp
await camera.ConnectAsync(CameraSelector.FirstPhysical());
await camera.ConnectAsync(CameraSelector.FirstEmulator());
await camera.ConnectAsync(CameraSelector.FriendlyName("Basler Emulation (0815-0000)"));
await camera.ConnectAsync(CameraSelector.FullName("Emulation (0815-0000)"));
```

Basler emulator canonical display name 정책:

- display name은 `"[EMU] " + FriendlyName`을 우선 사용한다.
- `FriendlyName`이 없으면 `FullName`을 사용한다.
- 연결 시에는 prefix 유무, `FriendlyName`, `FullName`을 모두 유연하게 허용한다.

예를 들어 아래 입력은 같은 emulator를 가리킬 수 있다.

- `[EMU] Basler Emulation (0815-0000)`
- `Basler Emulation (0815-0000)`
- `Emulation (0815-0000)`

### 14-2. 노출 100 적용 후 한 장 그랩

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

### 14-3. selector와 raw 목록을 함께 쓰는 예

실제 앱에서는 장치 목록을 화면에 보여주되, 연결은 selector로 수행하는 구성이 가장 안정적이다.

```csharp
Environment.SetEnvironmentVariable("PYLON_CAMEMU", "2", EnvironmentVariableTarget.Process);

using var probe = new BaslerCamera();
var names = await probe.GetAvailableCamerasAsync();

foreach (var name in names)
{
    Console.WriteLine(name);
}

using var physical = new BaslerCamera();
using var emulator = new BaslerCamera();

await physical.ConnectAsync(CameraSelector.FirstPhysical());
await emulator.ConnectAsync(CameraSelector.FirstEmulator());

using var physicalImage = await physical.GrabOneAsync(5000);
using var emulatorImage = await emulator.GrabOneAsync(5000);

Console.WriteLine($"physical={physicalImage.Width}x{physicalImage.Height}");
Console.WriteLine($"emulator={emulatorImage.Width}x{emulatorImage.Height}");
```

## 15. 사용자 카메라 등록 패턴

### 15-1. 새 kind 등록

```csharp
using RatelSoft.Utils.UnifiedCamera;

CameraTypeRegistry.Register(
    kind: "mycompany.camera.basler-ex",
    creator: () => new BaslerCameraEx(),
    mappedType: CameraType.Custom,
    overwrite: true);
```

### 15-2. 등록한 kind로 생성

```csharp
using var camera = CameraFactory.CreateCamera("mycompany.camera.basler-ex");
```

## 16. 상속과 어댑터의 역할 차이

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

## 17. 권장 규칙

- 공통 기능은 `ICamera` / `CameraAdapterBase`로 사용한다.
- 장비 내부 파이프라인을 바꿀 때만 상속을 사용한다.
- 요청값을 설정한 뒤 실제 장비값을 다시 읽어 검증하는 습관을 갖는다.
- 라인/에어리 구분은 강제 enum보다 현재 `CameraSettings` 조합을 먼저 기준으로 해석한다.
- Matrox는 연결 전에 `DcfPath`를 반드시 검증하고, selector는 descriptor 기반 문자열로 맞춘다.
- `GrabResult.Image`는 `Mat`이므로 수명 관리를 명확히 한다.
- 이벤트 핸들러에서 받은 이미지를 오래 들고 있을 경우 `Clone()` 후 별도로 관리한다.

## 18. 정리

`UnifiedCamera`의 기준 구조는 다음과 같다.

- 공통 카메라 표면: `ICamera`
- 상태/조합/이벤트를 포함한 베이스 구현: `CameraBase`
- 카메라 생성: `CameraFactory`, `CameraTypeRegistry`
- 외부 사용 편의 계층: `CameraAdapterBase`
- 장비 내부 파이프라인 확장: 사용자 파생 카메라 클래스

이 구조를 유지하면 공통 카메라 코드와 장비 전용 기능을 분리하면서도, 콘솔 프로그램, 서비스, UI 프로그램에서 같은 런타임 모델을 재사용할 수 있다.
