# UnifiedCamera Camera/Board Extension Guide

이 문서는 `RatelSoft.Utils.UnifiedCamera`에 새로운 camera 또는 frame grabber board 타입을 추가할 때 따라야 하는 개발자 가이드다.
목표는 기존 Basler/MVS/Emulation 구현을 모르는 상태에서도 새 장비 구현, 등록, 연결, 그랩, 설정, 배포, 검증까지 빠뜨리지 않도록 하는 것이다.

## 1. 확장 지점 요약

`UnifiedCamera`의 확장 구조는 다음 타입을 중심으로 동작한다.

| 구분 | 타입/파일 | 역할 |
|---|---|---|
| 공통 API | `ICamera` (`Common.cs`) | 모든 카메라/보드 구현이 제공해야 하는 연결, 그랩, 트리거, 설정 API |
| 기본 구현 | `CameraBase` (`Common.cs`) | 상태 전이, 이벤트, 프레임 버퍼링, 이미지 조합, Dispose 공통 처리 |
| 생성 | `CameraFactory` (`Common.cs`) | `CameraType` 또는 `kind` 문자열로 카메라 생성 |
| 등록 | `CameraTypeRegistry` (`Common.cs`) | `kind -> Func<ICamera>` 생성자 매핑 |
| 장치 목록 | `CameraDescriptor`, `CameraDiscoveryExtensions` | 화면 표시/선택용 장치 메타데이터 |
| 선택 연결 | `CameraSelector`, `CameraConnectionExtensions` | `First`, `DisplayName`, `FriendlyName` 등 선택 규칙 |
| 외부 편의 계층 | `CameraAdapterBase` | 앱 전용 편의 메서드 또는 장비 전용 기능 래핑 |
| 공통 설정 | `CameraSettings` (`CameraService.Contracts`) | 폭, 높이, 노출, 게인, 트리거, 버퍼, 조합 설정 |
| 이미지 결과 | `GrabResult` | `OpenCvSharp.Mat`, 크기, timestamp, frame index, 오류 정보 |

새 타입 추가에는 두 가지 방식이 있다.

- 라이브러리를 수정하지 않는 외부 확장: `CameraBase` 파생 클래스를 만들고 `CameraTypeRegistry.Register("kind", ...)`로 등록한다.
- 라이브러리 내장 타입 추가: `CameraType` enum, `CameraTypeRegistry` 기본 등록, 구현 파일, 프로젝트 참조, 문서를 함께 수정한다.

가능하면 새 장비를 먼저 외부 확장 방식으로 검증한 뒤, 공용 배포가 필요할 때 내장 타입으로 승격하는 편이 안전하다.

## 2. Camera와 Board를 어떻게 모델링할지 결정

`UnifiedCamera`의 public surface는 `ICamera` 하나다. 따라서 frame grabber board가 있더라도 호출자에게는 하나의 `ICamera` 구현으로 노출하는 것이 기본 원칙이다.

권장 모델:

- USB/GigE 카메라: 카메라 1대를 `ICamera` 1개로 구현한다.
- CXP/CameraLink frame grabber board: 보드의 특정 channel 또는 특정 연결 카메라를 `ICamera` 1개로 구현한다.
- 멀티 채널 보드: channel마다 별도 `ICamera` 인스턴스를 만들 수 있게 `CameraDescriptor.DisplayName` 또는 `FullName`에 channel 정보를 포함한다.
- 한 보드에서 여러 카메라가 동시에 동작해야 하는 경우: board manager를 내부 구현체에 숨기고, 외부에는 channel별 `ICamera`를 제공한다.

예: `Euresys CXP Board #0 / Channel 1`을 하나의 카메라처럼 노출한다.

```text
kind: "euresys"
DisplayName: "Board0:Channel1:CameraA"
FullName: "Euresys Coaxlink 4C Board0 Channel1 CameraA Serial=..."
```

`CameraDescriptor`에 board 전용 필드가 부족하면 우선 `DisplayName`, `CanonicalName`, `FullName`, `DeviceSerialNumber`에 안정적인 식별 정보를 넣는다. 여러 소비자 앱에서 board 메타데이터를 구조적으로 사용해야 한다면 그때 `CameraDescriptor` 확장을 별도 변경으로 진행한다.

## 3. 새 구현 클래스 작성

새 장비 구현은 보통 `CameraBase`를 상속한다.

```csharp
using CameraService.Contracts;
using OpenCvSharp;
using RatelSoft.Utils.UnifiedCamera;

public sealed class MyBoardCamera : CameraBase
{
    private object? _device;
    private bool _isGrabbing;

    public override CameraType Type => CameraType.Custom;
    public override string Kind => "mycompany.myboard";

    public override async Task<bool> ConnectAsync(string? cameraName = null)
    {
        try
        {
            OnStateChanged(CameraState.Connected, "Connecting to MyBoard camera...");

            // 1. SDK 초기화
            // 2. board/camera 목록 조회
            // 3. cameraName이 있으면 정확히 매칭, 없으면 첫 번째 장치 선택
            // 4. 장치 open
            // 5. Settings 적용

            await ApplySettingsAsync().ConfigureAwait(false);
            OnStateChanged(CameraState.Connected, $"Connected to MyBoard camera: {cameraName ?? "Default"}");
            return true;
        }
        catch (Exception ex)
        {
            OnError("Failed to connect to MyBoard camera", ex);
            return false;
        }
    }

    public override async Task DisconnectAsync()
    {
        try
        {
            if (IsGrabbing)
                await StopGrabbingAsync().ConfigureAwait(false);

            // SDK close/dispose
            _device = null;
            OnStateChanged(CameraState.Disconnected, "Disconnected from MyBoard camera");
        }
        catch (Exception ex)
        {
            OnError("Error disconnecting from MyBoard camera", ex);
        }
    }

    public override async Task<bool> StartGrabbingAsync(int grabCount = -1)
    {
        try
        {
            if (!IsConnected || _device == null)
                return false;

            _grabCount = 0;
            await ApplySettingsAsync().ConfigureAwait(false);

            // SDK callback 등록 후 acquisition start
            _isGrabbing = true;
            OnStateChanged(CameraState.Grabbing, "Started grabbing");
            return true;
        }
        catch (Exception ex)
        {
            _isGrabbing = false;
            OnError("Failed to start grabbing", ex);
            return false;
        }
    }

    public override Task StopGrabbingAsync()
    {
        try
        {
            if (!_isGrabbing)
                return Task.CompletedTask;

            _isGrabbing = false;
            // SDK acquisition stop 및 callback 해제
            OnStateChanged(CameraState.Connected, "Stopped grabbing");
            return Task.CompletedTask;
        }
        catch (Exception ex)
        {
            OnError("Error stopping grabbing", ex);
            return Task.CompletedTask;
        }
    }

    public override async Task<GrabResult> GrabOneAsync(int timeoutMs = 1000)
    {
        try
        {
            if (!IsConnected || _device == null)
                return new GrabResult { Success = false, ErrorMessage = "Camera not connected" };

            // SDK에서 1장 취득
            // var sdkFrame = ...
            // return ConvertFrameToGrabResult(sdkFrame);

            return new GrabResult { Success = false, ErrorMessage = "Not implemented" };
        }
        catch (Exception ex)
        {
            OnError("Failed to grab single image", ex);
            return new GrabResult { Success = false, ErrorMessage = ex.Message };
        }
    }

    public override Task<List<string>> GetAvailableCamerasAsync()
    {
        // UI에 표시하고 ConnectAsync(cameraName)으로 다시 찾을 수 있는 안정적인 이름을 반환한다.
        return Task.FromResult(new List<string> { "Board0:Channel0" });
    }

    public override Task<bool> SoftwareTriggerAsync()
    {
        // SDK software trigger command
        return Task.FromResult(true);
    }

    public override Task<bool> SetTriggerModeAsync(bool enabled, TriggerSource source)
    {
        Settings = Settings with { TriggerEnabled = enabled, TriggerSource = source };
        // SDK trigger selector/source/mode 적용
        return Task.FromResult(true);
    }

    public override Task<bool> ResetAsync()
    {
        // SDK reset command
        return Task.FromResult(true);
    }

    public override Task<bool> GetInputStateAsync()
    {
        // Line input 또는 board input 상태 조회
        return Task.FromResult(false);
    }

    private Task ApplySettingsAsync()
    {
        // Settings.Width, LinesPerFrame, Exposure, Gain, LineRate, MaxBuffers 등을 SDK 파라미터로 매핑
        return Task.CompletedTask;
    }

    private GrabResult ConvertFrameToGrabResult(IntPtr data, int width, int height, int stride)
    {
        var mat = new Mat(height, width, MatType.CV_8UC1);
        // SDK 버퍼 수명과 무관하게 안전하게 복사한다.
        unsafe
        {
            Buffer.MemoryCopy(
                source: data.ToPointer(),
                destination: mat.Data.ToPointer(),
                destinationSizeInBytes: height * (long)mat.Step(),
                sourceBytesToCopy: height * (long)stride);
        }

        return new GrabResult
        {
            Success = true,
            Width = width,
            Height = height,
            Stride = (int)mat.Step(),
            Image = mat,
            Timestamp = DateTime.UtcNow,
            FrameIndex = Interlocked.Increment(ref _grabCount),
            ErrorMessage = string.Empty
        };
    }

    protected override void Dispose(bool disposing)
    {
        if (disposing && !_disposed)
        {
            try
            {
                if (IsGrabbing)
                    StopGrabbingAsync().GetAwaiter().GetResult();

                // SDK 리소스 정리
                _device = null;
            }
            catch (Exception ex)
            {
                _logger.Error(ex, "Error disposing MyBoard camera");
            }
        }

        base.Dispose(disposing);
    }
}
```

핵심 규칙:

- SDK 콜백에서 프레임을 받으면 `GrabResult`로 변환한 뒤 `OnFrameReceived(result)`를 호출한다.
- `OnFrameReceived()`가 `FrameGrabbed`, `ImageGrabbed`, `FramesPerImage` 조합 처리를 담당한다.
- 장비 오류는 `throw`로 밖에 흘리기보다 `OnError(message, ex)`로 상태와 이벤트를 맞춘다.
- `ConnectAsync()` 실패 시 `false`를 반환하고, 이미 잡은 SDK 리소스는 정리한다.
- `DisconnectAsync()`는 grabbing 중이면 먼저 `StopGrabbingAsync()`를 호출한다.
- `Dispose()`는 SDK handle, callback, native buffer를 반드시 해제한다.

## 4. 등록 방식

### 4-1. 라이브러리 수정 없이 등록

앱 시작 시점에 한 번 등록한다.

```csharp
CameraTypeRegistry.Register(
    kind: "mycompany.myboard",
    creator: () => new MyBoardCamera(),
    mappedType: CameraType.Custom,
    overwrite: true);

using var camera = CameraFactory.CreateCamera("mycompany.myboard");
await camera.ConnectAsync("Board0:Channel0");
```

`mappedType: CameraType.Custom`은 기존 enum 기반 호출부와 최소한으로 호환하기 위한 선택이다. 여러 custom 장비를 동시에 쓸 수 있다면 enum보다는 `kind` 문자열을 기준으로 생성하는 편이 안전하다.

### 4-2. 라이브러리 내장 타입으로 추가

공용 패키지에 포함할 타입이라면 다음을 수정한다.

1. `Common.cs`의 `CameraType`에 새 enum 값을 추가한다.
2. `CameraTypeRegistry` static constructor에 기본 등록을 추가한다.
3. `src/UnifiedCamera/<Vendor>.cs`에 구현 클래스를 추가한다.
4. SDK managed DLL이 필요하면 `UnifiedCamera.csproj`에 MSBuild property와 `<Reference>`를 추가한다.
5. NuGet 소비자도 DLL 경로를 지정할 수 있게 `buildTransitive` targets 또는 소비자 문서를 갱신한다.
6. `dist-lib/docs/CAMERA_RUNTIME_MANUAL.md`와 이 문서의 예시 또는 체크리스트를 갱신한다.

예:

```csharp
public enum CameraType
{
    Basler,
    Matrox,
    Euresys,
    Mvs,
    Emulation,
    MyBoard,
    Custom = 999
}
```

```csharp
static CameraTypeRegistry()
{
    RegisterBuiltIn(CameraType.Basler, "basler", () => new BaslerCamera());
    RegisterBuiltIn(CameraType.Mvs, "mvs", () => new MvsCamera());
    RegisterBuiltIn(CameraType.MyBoard, "myboard", () => new MyBoardCamera());
}
```

## 5. 장치 검색과 선택 규칙

최소 구현은 `GetAvailableCamerasAsync()`만 있으면 된다. 이 함수는 `ConnectAsync(cameraName)`에 그대로 넣어 다시 연결할 수 있는 이름을 반환해야 한다.

권장 이름 형식:

```text
Board0:Channel0
Board0:Channel1
BoardSerial123:CH0:CameraSerial456
```

장치 메타데이터가 필요하면 `CameraDescriptor`를 반환하는 전용 메서드를 구현하고 `CameraDiscoveryExtensions`에 분기 추가를 검토한다.

```csharp
public Task<List<CameraDescriptor>> GetAvailableCameraDescriptorsAsync()
{
    return Task.FromResult(new List<CameraDescriptor>
    {
        new CameraDescriptor
        {
            Type = CameraType.Custom,
            Kind = "mycompany.myboard",
            DisplayName = "Board0:Channel0",
            CanonicalName = "Board0:Channel0",
            FullName = "MyBoard Board0 Channel0 CameraA Serial=1234",
            DeviceSerialNumber = "1234",
            IsEmulator = false
        }
    });
}
```

새 selector mode가 필요 없다면 `CameraConnectionExtensions`를 수정하지 않아도 된다. `CameraSelector.DisplayName(value)`는 기본 fallback에서 `ConnectAsync(value)`로 연결된다.

`FirstPhysical`, `FirstEmulator`, `FriendlyName`, `FullName`, `UserDefinedName` 같은 selector를 새 장비에서도 지원하려면 `CameraConnectionExtensions.ConnectAsync()`에 새 타입 분기를 추가한다.

```csharp
if (camera is MyBoardCamera myBoardCamera)
{
    return myBoardCamera.ConnectAsync(selector);
}
```

그리고 구현체에 `ConnectAsync(CameraSelector selector)`를 추가해 selector별 매칭을 처리한다.

## 6. Settings 매핑 기준

`CameraSettings`는 공통 계약이므로 모든 장비가 같은 의미로 해석해야 한다.

| Settings 필드 | 장비 매핑 권장 |
|---|---|
| `Width` | 센서/ROI width. 0이면 장비 기본값 유지 |
| `LinesPerFrame` | frame height 또는 라인스캔 block height |
| `FramesPerImage` | `CameraBase` 이미지 조합에 사용할 프레임 수 |
| `CompositionMode` | `Vertical`, `Average` 등 `CameraBase` 조합 방식 |
| `Exposure` | 노출 시간. 단위는 장비 구현 문서에 명시 |
| `Gain` | 장비 gain. 지원하지 않으면 warning 후 기존값 유지 |
| `LineRate` | line rate 또는 free-run FPS. 장비별 해석을 문서화 |
| `TriggerEnabled` | trigger mode on/off |
| `TriggerSource` | `Software` 또는 `Hardware` |
| `MaxBuffers` | SDK stream buffer 개수 |

설정 함수 구현 원칙:

- `SetImageWidthAsync`, `SetImageHeightAsync`, `SetImageSizeAsync`는 SDK 적용 성공 후 `Settings`도 갱신한다.
- SDK가 요청값을 보정할 수 있으면, 설정 후 실제 값을 다시 읽어 `Settings`에 반영하는 편이 좋다.
- `ReadCameraSettingsAsync()`는 override하지 않아도 기본 구현이 `Get*` 메서드를 호출해 `Settings`를 갱신한다.
- 장비가 지원하지 않는 값은 조용히 성공 처리하지 말고 로그나 `false`로 호출자가 알 수 있게 한다.

## 7. GrabResult 변환 기준

`GrabResult.Image`는 `OpenCvSharp.Mat`이다. SDK native buffer 포인터를 그대로 참조하면 SDK가 버퍼를 재사용하는 순간 이미지가 깨질 수 있으므로 일반적으로 새 `Mat`에 복사한다.

권장 변환:

- Mono8: `MatType.CV_8UC1`
- Mono10/12/16: 필요하면 `CV_16UC1`로 보존하거나, 소비자 요구에 따라 Mono8로 변환
- Bayer/RGB/BGR: 호출부 기준을 정하고 `CV_8UC3` BGR로 통일하는 것을 권장
- `Stride`는 가능하면 `Mat.Step()` 값을 넣는다.
- `Timestamp`는 SDK timestamp가 신뢰 가능하면 SDK 값을, 아니면 `DateTime.UtcNow`를 사용한다.
- callback 연속 그랩에서는 `_grabCount`를 증가시키고 `FrameIndex`에 넣는다.

SDK callback 예:

```csharp
private void OnSdkFrameGrabbed(object? sender, SdkFrameEventArgs e)
{
    GrabResult? result = null;

    try
    {
        result = ConvertFrameToGrabResult(e.Data, e.Width, e.Height, e.Stride);
    }
    catch (Exception ex)
    {
        result = new GrabResult
        {
            Success = false,
            ErrorMessage = ex.Message,
            Timestamp = DateTime.UtcNow
        };
        _logger.Error(ex, "Failed to convert SDK frame");
    }

    OnFrameReceived(result);
}
```

## 8. 트리거 구현 기준

공통 API는 단순하다.

```csharp
await camera.SetTriggerModeAsync(true, TriggerSource.Software);
await camera.SoftwareTriggerAsync();
await camera.SetTriggerModeAsync(true, TriggerSource.Hardware);
await camera.SetTriggerModeAsync(false, TriggerSource.Software);
```

장비 구현에서는 다음을 맞춘다.

- software trigger: SDK trigger source를 software로 설정하고 command 실행
- hardware trigger: SDK trigger source를 Line0/Line1 등 실제 input line으로 설정
- trigger off: acquisition/free-run 모드로 복귀
- `GetTriggerModeAsync()`: 가능하면 SDK 값을 읽고, 실패하면 `Settings.TriggerEnabled` 반환
- `GetTriggerSourceAsync()`: 필요하면 override해서 SDK 값을 읽는다. 기본 구현은 `Settings.TriggerSource`를 반환한다.

## 9. SDK DLL과 프로젝트 설정

새 vendor managed DLL이 필요한 경우 `UnifiedCamera.csproj`에 다음 패턴을 추가한다.

```xml
<PropertyGroup>
  <MyBoardManagedFile Condition="'$(MyBoardManagedFile)' == ''"></MyBoardManagedFile>
</PropertyGroup>

<ItemGroup>
  <Reference Include="MyBoard.Sdk" Condition="'$(MyBoardManagedFile)' != ''">
    <HintPath>$(MyBoardManagedFile)</HintPath>
    <Private>false</Private>
  </Reference>
</ItemGroup>
```

정책:

- 라이브러리는 컴파일 참조만 담당한다.
- 실제 실행 폴더 복사는 소비자 앱이 `Directory.Build.props` 또는 앱 프로젝트 `<Reference Private="true">`로 결정한다.
- NuGet 패키지에 벤더 DLL을 포함하지 않는다면, 소비자 문서에 `MyBoardManagedFile` 예시를 반드시 추가한다.

소비자 예:

```xml
<Project>
  <PropertyGroup>
    <MyBoardManagedFile>$(MSBuildThisFileDirectory)vendor\myboard\MyBoard.Sdk.dll</MyBoardManagedFile>
  </PropertyGroup>
</Project>
```

## 10. Adapter를 추가해야 하는 경우

`ICamera` 구현은 장비를 움직이는 내부 파이프라인이고, `CameraAdapterBase`는 호출부 편의를 위한 외부 래퍼다.

다음 경우 adapter를 만든다.

- 앱에서 `Connect -> 설정 -> GrabOne -> 결과 검증` 같은 반복 절차가 많다.
- 특정 장비 전용 기능을 앱 코드에서 명확한 이름으로 쓰고 싶다.
- 호출부에서 `if (camera is MyBoardCamera typed)`가 반복된다.

예:

```csharp
public sealed class MyBoardCameraAdapter : CameraAdapterBase
{
    public MyBoardCameraAdapter(ICamera camera) : base(camera)
    {
    }

    public async Task<GrabResult> ConnectChannelAndGrabAsync(string channelName)
    {
        var ok = await ConnectAsync(channelName).ConfigureAwait(false);
        if (!ok)
            throw new InvalidOperationException($"Connect failed: {channelName}");

        return await GrabOneAsync(5000).ConfigureAwait(false);
    }

    public MyBoardCamera GetMyBoardCamera()
        => GetCamera<MyBoardCamera>();
}
```

Adapter는 새 타입 등록을 대체하지 않는다. 생성/등록은 여전히 `CameraTypeRegistry` 또는 직접 생성으로 처리한다.

## 11. 내장 타입 추가 체크리스트

구현:

- `CameraBase` 파생 클래스를 추가했다.
- `Type`과 `Kind`가 일관된다.
- `ConnectAsync()`가 이름이 없을 때 첫 장치, 이름이 있을 때 정확한 장치를 선택한다.
- `DisconnectAsync()`가 grabbing 상태를 먼저 중지한다.
- `StartGrabbingAsync()`가 callback 등록, buffer 설정, acquisition start를 수행한다.
- `StopGrabbingAsync()`가 acquisition stop 후 callback을 해제한다.
- `GrabOneAsync()`가 timeout을 반영하고 SDK buffer를 안전하게 반환/해제한다.
- SDK callback에서 `OnFrameReceived()`를 호출한다.
- `GetAvailableCamerasAsync()` 결과를 `ConnectAsync(name)`에 다시 사용할 수 있다.
- `SetTriggerModeAsync()`, `SoftwareTriggerAsync()`, `GetInputStateAsync()`가 장비 기준으로 구현되어 있다.
- `Get/SetImageWidth/Height/Exposure/Gain`이 `Settings`와 SDK 값을 동기화한다.
- `Dispose()`에서 SDK handle, callback, native buffer, thread를 해제한다.

등록/배포:

- 외부 확장이라면 앱 시작 시 `CameraTypeRegistry.Register()`가 호출된다.
- 내장 타입이라면 `CameraType` enum과 `CameraTypeRegistry` 기본 등록이 수정됐다.
- SDK DLL이 필요하면 csproj property/reference와 소비자 설정 예시가 있다.
- `dist-lib/docs` 문서 링크가 갱신됐다.

검증:

- 장치가 없을 때 `GetAvailableCamerasAsync()`가 빈 목록을 반환하고 예외로 죽지 않는다.
- 잘못된 이름으로 `ConnectAsync(name)` 호출 시 `false`를 반환하고 `ErrorOccurred`가 발생한다.
- 연결/해제 10회 반복에서 handle leak이 없다.
- `GrabOneAsync()` 10회 반복에서 `Mat` 크기와 stride가 정상이다.
- 연속 그랩 시작/중지 10회 반복에서 callback 중복 등록이 없다.
- `FramesPerImage > 1`일 때 `ImageGrabbed`가 조합 결과 기준으로 발생한다.
- `TriggerEnabled=true`에서 software trigger 1회당 기대 프레임 수만 발생한다.
- 앱 종료 또는 `Dispose()` 시 native SDK 예외가 발생하지 않는다.

## 12. 흔한 실수

- SDK callback에서 `FrameGrabbed` 이벤트를 직접 발생시키는 것: `OnFrameReceived()`를 호출해야 공통 조합 로직이 동작한다.
- SDK native buffer를 `Mat`이 직접 참조하게 두는 것: 대부분의 SDK는 callback 이후 버퍼를 재사용하므로 안전하게 복사해야 한다.
- `StopGrabbingAsync()`에서 callback 해제를 빼먹는 것: start/stop 반복 후 이벤트가 중복 발생한다.
- `ConnectAsync()` 실패 후 partially opened handle을 해제하지 않는 것: 다음 연결이 실패하거나 장비가 busy 상태로 남는다.
- `Settings`만 바꾸고 SDK 파라미터를 적용하지 않는 것: 호출자는 성공으로 보지만 실제 장비는 바뀌지 않는다.
- `CameraType.Custom`에 여러 장비를 모두 매핑하는 것: enum 생성은 마지막 등록에 덮일 수 있으므로 여러 custom 장비는 `kind` 문자열 생성이 안전하다.
- board channel을 표시 이름에만 넣고 실제 선택 로직에서 무시하는 것: 멀티 채널 보드에서 항상 첫 channel만 연결되는 문제가 생긴다.

## 13. 권장 kind 이름

`kind`는 설정 파일, 로그, 테스트 코드에 남는 안정적인 식별자다. 소문자와 점/하이픈을 사용해 충돌을 피한다.

권장 예:

```text
basler
mvs
emulation
mycompany.myboard
mycompany.myboard-cxp
mycompany.camera-usb
```

공용 배포 타입은 짧은 vendor 이름을 쓰고, 사내/프로젝트 전용 확장은 회사 또는 프로젝트 prefix를 붙인다.

