# RatelSoft Library Codebook

이 문서는 `RatelSoft.*` 라이브러리를 빠르게 찾고 복사/붙여넣기용으로 쓰기 위한 참조 문서입니다.

## 0) 타입 이름으로 바로 찾기 (가장 먼저)
- 타입 역인덱스: `Docs/RatelSoft_TYPE_INDEX.md`
- RatelViewer 전용 코드북: `Docs/RatelSoft_CODEBOOK_RatelViewer.md`
- 실전 마이그레이션 예제: `Docs/RatelSoft_CODEBOOK_EsViewer_PRACTICAL.md`
- XML 레퍼런스 가이드: `Docs/RatelSoft_XML_REFERENCE_MANUAL.md`
- 예시: `CalcTextBox` 검색
  - Assembly: `RatelSoft.Utils.Wpf`
  - Namespace: `RatelSoft.Utils.Wpf`
  - File: `RatelSoft.Utils.Wpf/Controls/CalcTextBox.cs`

복붙 예제
```csharp
using RatelSoft.Utils.Wpf;

var box = new CalcTextBox
{
    EvaluateOnLostFocus = true,
    ShowErrorMessage = false,
    Minimum = 0,
    Maximum = 1000
};
```

## 1) 빠른 찾기 규칙
- Assembly 찾기: `rg -n "<AssemblyName>|<PackageId>" --glob "*.csproj"`
- Namespace/Class 찾기: `rg -n "namespace RatelSoft|class <ClassName>|interface <InterfaceName>"`
- Method 찾기: `rg -n "public .* <MethodName>\(" <folder>`
- Property 찾기: `rg -n "public .*\{ get;.*\}" <folder>`

마이그레이션 체크리스트 (레거시 `Ratel.*` -> `RatelSoft.*`)
1. 프로젝트 TFM을 `net8.0-windows`로 변경
2. 패키지를 `RatelSoft.*`로 교체
3. XAML 네임스페이스 교체
   - `Ratel.Vision.WPF` -> `RatelSoft.Vision.Wpf`
   - `Ratel.WPF` -> `RatelSoft.Utils.Wpf.Logging`
4. 코드 네임스페이스 교체
   - `Ratel.Vision.*` -> `RatelSoft.Vision.*`
   - `Ratel.GPU` -> `RatelSoft.Vision.GPU`
5. `RatelLib.Log` 사용 코드를 `RatelLogger` 패턴으로 교체
6. `NuGet.config`에서 로컬 피드(`C:\data\nuget`) 연결 확인

---

## 2) 초기화

### Assembly: `RatelSoft.Lib`
### Namespace: `RatelSoft.Lib`
### Class: `RatelLib` (`RatelSoft.Lib/Global.cs`)

주요 Property/Field
- `public static Action<string> RatelMessage`
- `public static bool UseActipro { get; set; }`
- `public static bool MotorSimul`
- `public static bool IoSimul`
- `public static string WorkingDir { get; set; }`
- `public static string RecipeDir { get; set; }`
- `public static string SystemSetupFile { get; set; }`
- `public static string BackupDir { get; set; }`
- `public static string CurRecipeFolder { get; set; }`
- `public static string DefRecipeFolder { get; set; }`
- `public static string FailedSceFile { get; set; }`
- `public static VariableManager Variables`
- `public static uint HID`
- `public static string LockKeyMsg`

주요 Method
- `public static void InitLib(Action<string> messageCallback = null)`
- `public static DongleInfo GetDongleInfo()`

복붙 예제
```csharp
using RatelSoft.Lib;

RatelLib.InitLib(msg => Console.WriteLine($"[RatelLib] {msg}"));
RatelLib.MotorSimul = true;
RatelLib.IoSimul = true;
```

### Assembly: `RatelSoft.Vision`
### Namespace: `RatelSoft.Vision`
### Class: `Init` (`RatelSoft.Vision/Ratel.Vision.Init.cs`)

주요 Method
- `public static void InitActiPro()`

---

## 3) Log 설정

### Assembly: `RatelSoft.Common`
### Namespace: `RatelSoft.Common`
### Class: `RatelLog` / `RatelLogger` (`RatelSoft.Common/RatelLog.cs`)

권장 패턴
- `RatelLog` 정적 직접 호출보다 `RatelLogger` 인스턴스 사용
- 생성: `RatelLog.ForSource("YourSource")`

주요 타입
- `enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }`
- `record LogItem`
- `readonly struct RatelLogger`
- `interface IRatelLogger`

`RatelLogger` 주요 Method
- `Emit(...)`
- `Trace(...)`
- `Debug(...)`
- `Info(...)`
- `Warn(...)`
- `Error(...)`
- `Fatal(...)`

`RatelLog` 주요 Method
- `ForSource(string source)`
- `Configure(...)`
- `SetLevel(...)`
- `ForwardToNLog(LogItem item)`

WPF LogViewer Target 타입 위치
- Type: `NLogLogViewerTarget`
- Assembly: `RatelSoft.Utils.Wpf`
- Namespace: `RatelSoft.Utils.Wpf.Logging`
- File: `RatelSoft.Utils.Wpf/Logging/NLogLogViewerTarget.cs`

복붙 예제
```csharp
using RatelSoft.Common;

static RatelLogger log = RatelLog.ForSource("MyFeature");

log.Info("startup", "feature started");
try
{
    throw new InvalidOperationException("sample");
}
catch (Exception ex)
{
    log.Error(ex, "failed to run feature");
}
```

### NLog 코드 설정 템플릿 (`ConfigureNLog`)
```csharp
using System.IO;
using NLog;
using NLog.Config;
using NLog.Filters;
using NLog.Targets;
using RatelSoft.Utils.Wpf.Logging;

private void ConfigureNLog()
{
    LogManager.Setup().SetupExtensions(ext =>
    {
        ext.RegisterTarget<NLogLogViewerTarget>("RatelLogViewer");
    });

    var config = new LoggingConfiguration();

    try
    {
        Directory.CreateDirectory("logs");
    }
    catch
    {
        // ignore
    }

    var viewerTarget = new NLogLogViewerTarget { Name = "viewer" };

    var fileTarget = new FileTarget("file")
    {
        FileName = "${basedir}/logs/app_${date:format=yyyyMMdd}.log",
        CreateDirs = true,
        Layout = "${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}",
        ArchiveFileName = "${basedir}/logs/archives/app_{#}.log",
        ArchiveAboveSize = 10485760,
        MaxArchiveFiles = 30,
        ArchiveEvery = FileArchivePeriod.Day
    };

    var mainWindowFileTarget = new FileTarget("mainWindowFile")
    {
        FileName = "${basedir}/logs/mainwindow_${date:format=yyyyMMdd}.log",
        CreateDirs = true,
        Layout = "${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}",
        ArchiveFileName = "${basedir}/logs/archives/mainwindow_{#}.log",
        ArchiveAboveSize = 10485760,
        MaxArchiveFiles = 30,
        ArchiveEvery = FileArchivePeriod.Day
    };

    config.AddTarget(viewerTarget);
    config.AddTarget(fileTarget);
    config.AddTarget(mainWindowFileTarget);

    config.AddRule(NLog.LogLevel.Trace, NLog.LogLevel.Fatal, viewerTarget);
    config.AddRule(NLog.LogLevel.Info, NLog.LogLevel.Fatal, fileTarget);

    var mainWindowRule = new LoggingRule("*", NLog.LogLevel.Trace, mainWindowFileTarget);
    mainWindowRule.Filters.Add(new ConditionBasedFilter
    {
        Condition = "'${event-properties:item=Caller}' == 'MainWindow'",
        Action = FilterResult.Log
    });
    mainWindowRule.Filters.Add(new ConditionBasedFilter
    {
        Condition = "true",
        Action = FilterResult.Ignore
    });
    config.LoggingRules.Add(mainWindowRule);

    LogManager.Configuration = config;
}
```

### WPF 소비 프로젝트 최소 패키지 (예시)
```xml
<ItemGroup>
  <PackageReference Include="RatelSoft.Common" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Lib" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Types" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Utils.Wpf" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Vision" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Vision.Wpf" Version="2.26.209.13" />
</ItemGroup>
```

### `nlog.config` 템플릿 (파일 기반 설정)
```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true"
      throwConfigExceptions="false">

  <extensions>
    <add assembly="RatelSoft.Utils.Wpf" />
  </extensions>

  <targets>
    <target xsi:type="RatelLogViewer"
            name="viewer"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}" />

    <target xsi:type="File"
            name="file"
            fileName="${basedir}/logs/app_${date:format=yyyyMMdd}.log"
            createDirs="true"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}"
            archiveFileName="${basedir}/logs/archives/app_{#}.log"
            archiveAboveSize="10485760"
            maxArchiveFiles="30"
            archiveEvery="Day" />

    <target xsi:type="File"
            name="mainWindowFile"
            fileName="${basedir}/logs/mainwindow_${date:format=yyyyMMdd}.log"
            createDirs="true"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}"
            archiveFileName="${basedir}/logs/archives/mainwindow_{#}.log"
            archiveAboveSize="10485760"
            maxArchiveFiles="30"
            archiveEvery="Day" />
  </targets>

  <rules>
    <logger name="*" minlevel="Trace" writeTo="viewer" />
    <logger name="*" minlevel="Info" writeTo="file" />
    <logger name="*" minlevel="Trace" writeTo="mainWindowFile">
      <filters>
        <when condition="'${event-properties:item=Caller}' == 'MainWindow'" action="Log" />
        <when condition="true" action="Ignore" />
      </filters>
    </logger>
  </rules>
</nlog>
```

### WPF Debug Console 템플릿 (`App.xaml.cs`)
```csharp
using System;
using System.Runtime.InteropServices;
using System.Windows;
using NLog;
using RatelSoft.Lib;

public partial class App : Application
{
    private static readonly Logger log = LogManager.GetCurrentClassLogger();

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool AllocConsole();

    [DllImport("kernel32.dll")]
    private static extern IntPtr GetConsoleWindow();

    [DllImport("kernel32.dll")]
    private static extern bool AttachConsole(int dwProcessId);

    [DllImport("kernel32.dll")]
    private static extern bool FreeConsole();

    private void Application_Startup(object sender, StartupEventArgs e)
    {
        if (GetConsoleWindow() == IntPtr.Zero)
        {
            if (!AttachConsole(-1))
            {
                AllocConsole();
            }
        }

        Console.WriteLine("로그 출력 시작");
        Vars.Config = RatelConfig.Load();
        RatelLib.InitLib();
    }

    private void Application_DispatcherUnhandledException(object sender, System.Windows.Threading.DispatcherUnhandledExceptionEventArgs e)
    {
        MessageBox.Show(e.Exception.Message);
        log.Error(e.Exception);
        e.Handled = true;
    }

    protected override void OnExit(ExitEventArgs e)
    {
        FreeConsole();
        base.OnExit(e);
    }
}
```

---

## 4) Config / DataStore

### Assembly: `RatelSoft.Common`
### Namespace: `RatelSoft.Common.Configuration`

### Class: `ConfigManager<TSettings>` (`RatelSoft.Common/Config/ConfigManager.cs`)
주요 Property
- `public TSettings Settings { get; }`

주요 Method
- `public ConfigManager(string[] args, string environmentPrefix, IRatelLogger? log = null)`
- `public ConfigManager(string[]? args = null, ConfigManagerOptions? options = null, IRatelLogger? log = null)`
- `public bool HasFlag(string flag)`
- `public void SaveSettings(string? filePath = null)`
- `public void LogCurrentSettings()`

관련 타입
- `ConfigValueAttribute` (`CliArg`, `EnvVar`, `Default`, `ConfigOnly`)
- `ConfigManagerOptions` (`EnvironmentPrefix`, `ConfigFilePath`, `DotEnvFileName`, `YamlNamingConvention`)

복붙 예제
```csharp
using RatelSoft.Common.Configuration;

public class AppSettings
{
    [ConfigValue] public string WorkDir { get; set; } = "d:/data";
    [ConfigValue(CliArg = "--port", EnvVar = "MYAPP_PORT", Default = 5000)]
    public int Port { get; set; }
}

var cm = new ConfigManager<AppSettings>(
    args,
    new ConfigManagerOptions("MYAPP_") { ConfigFilePath = "config/app_settings.yaml" }
);

cm.LogCurrentSettings();
var settings = cm.Settings;
```

### Class: `DataStore<T>` (`RatelSoft.Common/Config/DataStore.cs`)
주요 Property
- `public T Data { get; set; }`
- `public string? FilePath { get; }`
- `public SerializationFormat Format { get; }`
- `public bool IsFileBased { get; }`
- `public bool EnableAutoBackup { get; set; }`
- `public int MaxBackups { get; set; }`
- `public bool HasChanges { get; }`
- `public bool FileExists { get; }`

주요 Method
- `Load()/LoadAsync()`
- `Save()/SaveAsync()`
- `Replace(T newData)`
- `Delete()`
- `ToCompactText()/FromCompactText()`
- `ToBytes()/FromBytes()`
- `ToUtf8Bytes()/FromUtf8Bytes()`
- `MarkAsClean()`
- `GetBackupFiles()/RestoreFromBackup()/DeleteAllBackups()`

복붙 예제
```csharp
using RatelSoft.Common.Configuration;

var store = new DataStore<AppSettings>("config/runtime.yaml");
store.EnableAutoBackup = true;
store.Data.WorkDir = "d:/work";
store.Save();
```

---

## 5) Network

### Assembly: `RatelSoft.Common`
### Namespace: `RatelSoft.Utils.Common.Networking`

### Class: `ProtobufTcpClient<TEnvelope>` (`RatelSoft.Common/Networking/ProtobufTcpClient.cs`)
주요 Property/Event
- `public bool IsConnected`
- `public event EventHandler<TEnvelope>? EventReceived`

주요 Method
- `ConnectAsync(string host, int port, CancellationToken token)`
- `Disconnect()`
- `SendAsync(TEnvelope envelope, CancellationToken token)`

### Class: `ControlServer<TEnvelope>` (`RatelSoft.Common/Networking/ControlServer.cs`)
주요 Property
- `public bool IsRunning`

주요 Method
- `StartAsync(int port, CancellationToken externalToken)`
- `Stop()`
- `Dispose()`

확장 포인트
- `protected abstract Task HandleEnvelopeAsync(...)`
- `protected virtual void OnClientConnected(...)`
- `protected virtual void OnClientDisconnected(...)`

### Class: `ControlServerSession<TEnvelope>`
주요 Property/Method
- `public string Endpoint`
- `public Task SendAsync(TEnvelope envelope, CancellationToken token)`

---

## 6) IO

### Assembly: `RatelSoft.Utils.UnifiedIO`
### Namespace: `RatelSoft.Utils.UnifiedIO`

주요 타입
- `enum IODeviceType { Eio, Contec, EtherCAT, ModbusTcp, Emulation }`
- `class IOState`
- `class IOStateChangedEventArgs`

### Interface: `IIODevice`
주요 Property
- `Type`, `IsConnected`, `InputChannelCount`, `OutputChannelCount`, `IsMonitoring`

주요 Method
- `ConnectAsync`, `DisconnectAsync`
- `ReadAllAsync`, `ReadInputAsync`, `ReadInputsAsync`
- `ReadOutputAsync`, `ReadOutputsAsync`, `WriteOutputAsync`, `ToggleOutputAsync`
- `ClearInputCountAsync`, `ReadInputCountAsync`
- `SimulateInputAsync`
- `StartMonitoringAsync`, `StopMonitoringAsync`

주요 구현체
- `EmulatedIODevice`
- `EioDevice` (`endpoint`는 `ip` 또는 `ip:port` 필수)
- `IODeviceFactory.Create(IODeviceType)`

복붙 예제
```csharp
using RatelSoft.Utils.UnifiedIO;

var io = IODeviceFactory.Create(IODeviceType.Emulation);
await io.ConnectAsync();
await io.WriteOutputAsync(0, true);
await io.SimulateInputAsync(1, true);
bool di1 = await io.ReadInputAsync(1);
await io.DisconnectAsync();
```

---

## 7) Motion

### Assembly: `RatelSoft.Utils.UnifiedMotion`
### Namespace: `RatelSoft.Utils.UnifiedMotion`

### Class: `MotionManager`
주요 Property
- `PollInterval`
- `Status`
- 인덱서 `this[int axisNo]`, `this[string axisName]`
- `Count`

주요 Method
- `AddAxis(int axisNo, MotionAxisStatus axis, MotionControllerBase controller)`
- `Open()`, `Close()`
- `MoveAsync`, `MoveAbsAsync`, `MoveIncAsync`
- `MoveJogP`, `MoveJogM`, `SetSpeed`, `Stop`, `Home`, `ServoOn`, `AllStop`
- `WaitMoveDoneAsync`
- `GetResponse(...)`

### Class: `MotionAxisStatus`
주요 Property
- 축/상태: `AxisName`, `AxisNo`, `InnerAxisNo`, `ControllerIndex`
- 신호: `AmpEnable`, `PlusLimit`, `MinusLimit`, `Vel0`, `HomeComplete`, `AmpFault`, `InPosition`, `IsActivated`
- 위치/속도: `CmdSpeed`, `ActSpeed`, `CmdPos`, `CurrentPos`, `DestPos`, `UmMul`
- 편의: `IsMoving`, `CurrentPosUm`, `CurrentPosMm`
- 이동 파라미터: `MotionValues Values`

주요 Method
- `MoveAsync`, `MoveAbsAsync`, `MoveIncAsync`, `WaitMoveDoneAsync`

### Class: `MotionValues`
주요 Property
- `Speed`, `Acc`, `Dec`, `Jerk`, `Pos`, `RelPos`, `RelMove`

### Controller
- `VController` (가상)
- `AjinController` (Ajin, `MotFilePath`, `HomeCompleteBit`)
- `PPMacCore` (PowerPMac, `IPAddress`, `Username`, `Password`)

복붙 예제 (Virtual)
```csharp
using RatelSoft.Utils.UnifiedMotion;

var manager = new MotionManager();
var ctrl = new VController(index: 0);

manager.AddAxis(0, new MotionAxisStatus
{
    AxisName = "X",
    InnerAxisNo = 0,
    UmMul = 1,
    Values = new MotionValues { Speed = 10000, Acc = 5000, Dec = 5000 }
}, ctrl);

manager.Open();
await manager.MoveAbsAsync(0, 50000);
await manager.WaitMoveDoneAsync(0);
manager.Close();
```

중요 주의
- `AddAxis`의 `InnerAxisNo`는 실제 컨트롤러 축 번호와 정확히 매핑해야 함.

---

## 8) Camera

### Assembly: `RatelSoft.Utils.UnifiedCamera`
### Namespace: `RatelSoft.Utils.UnifiedCamera`

핵심 타입
- `enum CameraType { Basler, Matrox, Euresys, Mvs, Emulation }`
- `class GrabResult`
- `class CameraEventArgs`
- `interface ICamera`
- `abstract class CameraBase`
- `static class CameraFactory`

### Interface: `ICamera` 주요 Property
- `Type`, `State`, `IsConnected`, `IsGrabbing`, `GrabCount`
- `Settings` (`CameraService.Contracts.CameraSettings`)
- `BufferCount`

### Interface: `ICamera` 주요 Method
- 연결: `ConnectAsync`, `DisconnectAsync`
- 그랩: `StartGrabbingAsync`, `StopGrabbingAsync`, `GrabOneAsync`
- 트리거: `SoftwareTriggerAsync`, `SetTriggerModeAsync`, `GetTriggerModeAsync`, `GetTriggerSourceAsync`
- 장치/설정: `ResetAsync`, `GetInputStateAsync`, `GetImageWidthAsync`, `SetExposureAsync` 등
- 탐색: `GetAvailableCamerasAsync`

### 구현체
- `BaslerCamera`
- `MvsCamera`
- `EmulationCamera`

관련 설정 타입 (Assembly: `RatelSoft.Utils.UnifiedCamera`)
- Namespace: `CameraService.Contracts`
- `record CameraSettings`
- `enum CameraState`, `TriggerSource`, `ImageCompositionMode`

복붙 예제
```csharp
using RatelSoft.Utils.UnifiedCamera;
using CameraService.Contracts;

using var cam = CameraFactory.CreateCamera(CameraType.Emulation);
cam.Settings = CameraSettings.Default with
{
    Width = 2048,
    LinesPerFrame = 1024,
    TriggerEnabled = false
};

await cam.ConnectAsync();
await cam.StartGrabbingAsync();
var one = await cam.GrabOneAsync();
await cam.StopGrabbingAsync();
await cam.DisconnectAsync();
```

---

## 9) RatelViewer (WPF 소비자 핵심)

### Assembly: `RatelSoft.Vision.Wpf`
### Namespace: `RatelSoft.Vision.Wpf`
### Class: `RatelViewer` (`RatelSoft.Vision.Wpf/WPF/RatelViewer.xaml.cs`)

주요 Property
- `Mat` (`DependencyProperty`)
- `ShowMenu`, `ShowToolBar`, `ShowStatusBar` (`DependencyProperty`)
- `MouseMode`
- `Shapes`

주요 Event
- `StartDrawShape`, `EndDrawShape`
- `OnShapeEdited`
- `ImageMouseDown`, `ImageMouseMove`, `ImageMouseUp`

주요 Method
- `MoveToToolBars(ToolBarTray toolBarTray, int startBand = 1)`
- `AddRectangle`, `AddEllipse`, `AddLine`, `AddPolygon`, `AddPolyline`
- `GetShapeRect`, `GetElements`, `RemoveElements`, `SelectShape`
- `SetZoom`, `PointToCenter`, `ClientToImage`, `ImageToClient`

복붙 예제
```xml
<vision:RatelViewer x:Name="viewer"
                    ShowMenu="False"
                    ShowToolBar="False"
                    ShowStatusBar="False"
                    Mat="{Binding CurrentMat}" />
```

```csharp
viewer.MouseMode = MouseMode.DrawRect;
viewer.EndDrawShape += (_, e) =>
{
    if (e.Shape is Rectangle)
    {
        var rect = viewer.GetShapeRect(e.Shape);
        log.Info($"ROI: {rect.X},{rect.Y},{rect.Width},{rect.Height}");
    }
};
```

상세 패턴 문서:
- `Docs/RatelSoft_CODEBOOK_RatelViewer.md`

## 10) 추천 검색 순서 (실무)
1. 먼저 이 문서에서 카테고리/클래스 이름 파악
2. `Docs/RatelSoft_TYPE_INDEX.md`에서 타입 이름 역검색
3. 해당 파일에서 Method 시그니처와 예외 조건 확인
4. 샘플은 이 문서 복붙 후 실제 프로젝트 설정값으로 치환
