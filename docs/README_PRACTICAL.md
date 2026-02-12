# README Practical - RatelSoft Usage Cookbook

이 문서는 `README.md`와 같은 내용을 실전 복붙 중심으로 확장한 버전입니다.
목표는 "프로젝트에 바로 붙여서 동작"입니다.

## 1) WPF 프로젝트 템플릿 (`net8.0-windows`)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows</TargetFramework>
    <UseWPF>true</UseWPF>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="RatelSoft.Common" Version="2.26.209.13" />
    <PackageReference Include="RatelSoft.Lib" Version="2.26.209.13" />
    <PackageReference Include="RatelSoft.Types" Version="2.26.209.13" />
    <PackageReference Include="RatelSoft.Utils.Wpf" Version="2.26.209.13" />
    <PackageReference Include="RatelSoft.Vision" Version="2.26.209.13" />
    <PackageReference Include="RatelSoft.Vision.Wpf" Version="2.26.209.13" />
  </ItemGroup>
</Project>
```

## 2) NuGet 피드 템플릿 (`C:\data\nuget`)

`NuGet.config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="local-ratel" value="C:\data\nuget" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```

## 3) Global Using 템플릿

```csharp
global using RatelSoft.Common;
global using RatelSoft.Lib;
global using RatelSoft.AI;
global using RatelSoft.Vision;
global using RatelSoft.Vision.Mura;
global using RatelSoft.Vision.GPU;
global using RatelSoft.Vision.Wpf;
global using RatelSoft.Utils.Wpf.Logging;
```

## 4) 로깅 진입점 템플릿 (`AppLog.cs`)

```csharp
public static class AppLog
{
    public static RatelLogger Log { get; } = RatelLog.ForSource("MyApp");
}
```

## 5) `App.xaml.cs` 초기화 템플릿

```csharp
protected override async void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);

    RatelLib.InitLib();
    AppLog.Log.Info("Library Init");
    Cuda.InitGPU();

    if (GetConsoleWindow() == IntPtr.Zero)
    {
        if (!AttachConsole(-1))
        {
            AllocConsole();
        }
    }

    // MainWindow 시작
    var main = new MainWindow();
    main.Show();
}
```

예외 처리:

```csharp
private void Application_DispatcherUnhandledException(object sender, System.Windows.Threading.DispatcherUnhandledExceptionEventArgs e)
{
    AppLog.Log.Error(e.Exception, "Application_DispatcherUnhandledException");
    e.Handled = true;
}
```

## 6) `nlog.config` 실전 템플릿

```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true">

  <extensions>
    <add assembly="RatelSoft.Utils.Wpf" />
  </extensions>

  <targets>
    <target xsi:type="File"
            name="Main"
            fileName="c:\tmp\log\app_${shortdate}.log"
            archiveFileName="c:\tmp\log\app_${shortdate}.{####}.log"
            archiveAboveSize="1048000"
            maxArchiveFiles="10000"
            archiveNumbering="Sequence"
            archiveEvery="Day"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}|${exception:format=tostring}" />

    <target xsi:type="Console"
            name="logconsole"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}|${exception:format=tostring}" />

    <target xsi:type="RatelLogViewer"
            name="logViewer"
            layout="${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}|${exception:format=message}" />
  </targets>

  <rules>
    <logger name="*" minlevel="Trace" writeTo="Main,logconsole,logViewer" />
  </rules>
</nlog>
```

## 7) XAML namespace 템플릿

```xml
xmlns:vision="clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf"
xmlns:wpf="clr-namespace:RatelSoft.Utils.Wpf.Logging;assembly=RatelSoft.Utils.Wpf"
```

## 8) Mura 검사 코드 템플릿

단일/멀티 스레드:

```csharp
var groups = MuraEx.InspectEx<MuraDefect>(
    mat,
    configs,
    maxDefect: 10000,
    mask: maskMat,
    threadType: ThreadType.Multi,
    maxThreadCount: Environment.ProcessorCount,
    log: AppLog.Log);

var defects = groups.GetDefectList();
```

스케일 축소 + 각도 루프 + 옵션:

```csharp
var groups = MuraEx.InspectEx<MuraDefect>(
    smallMat,
    rotateConfigs,
    maxDefect: 10000,
    mask: smallMask,
    threadType: threadType,
    maxThreadCount: maxThreadCount,
    muraOptions: scratchFilter,
    log: AppLog.Log);
```

GPU:

```csharp
var gpuMat = new GpuMat();
gpuMat.Upload(mat);
var gpuMask = new GpuMat(maskMat);

var groups = MuraEx.InspectExGpu<MuraDefect>(
    gpuMat,
    configs,
    maxDefect: 10000,
    gpuMask: gpuMask,
    log: AppLog.Log);
```

## 8-1) 이관된 UI 사용 템플릿

3D 프로파일:

```csharp
var win3d = new HeightMap3DWindow(curMat[roi]);
win3d.Owner = this;
win3d.ShowDialog();
```

Mura 필터 편집 창:

```csharp
using MuraConfigEditWindow = RatelSoft.Vision.Wpf.Mura.MuraConfigEditWindow;

var win = new MuraConfigEditWindow
{
    MuraConfig = config.Clone()
};
if (win.ShowDialog() == true)
{
    config.CopyFrom(win.MuraConfig);
}
```

## 8-2) RatelViewer 소비자 구성 템플릿

메뉴/툴바/상태바 숨김 + `Mat` 바인딩:

```xml
<vision:RatelViewer x:Name="viewer"
                    ShowMenu="False"
                    ShowToolBar="False"
                    ShowStatusBar="False"
                    Mat="{Binding CurrentMat}" />
```

소비자 버튼으로 도형 모드 제어:

```csharp
viewer.MouseMode = MouseMode.DrawRect;    // 사각형
viewer.MouseMode = MouseMode.DrawLine;    // 선
viewer.MouseMode = MouseMode.DrawEllipse; // 타원
viewer.MouseMode = MouseMode.DrawPolygon; // 다각형
viewer.MouseMode = MouseMode.None;        // 종료
```

그리기 완료 이벤트:

```csharp
viewer.EndDrawShape += (_, e) =>
{
    if (e.Shape is Rectangle)
    {
        var rect = viewer.GetShapeRect(e.Shape);
        AppLog.Log.Info($"Rect: {rect.X},{rect.Y},{rect.Width},{rect.Height}");
    }
};
```

툴바를 소비자 UI로 이동:

```csharp
// hostToolBarTray: 소비자 윈도우의 ToolBarTray
viewer.MoveToToolBars(hostToolBarTray, startBand: 0);
```

## 9) 레거시 마이그레이션 규칙

치환 규칙:
- `Ratel.Vision.WPF` -> `RatelSoft.Vision.Wpf`
- `Ratel.Vision` -> `RatelSoft.Vision`
- `Ratel.Vision.Mura` -> `RatelSoft.Vision.Mura`
- `Ratel.GPU` -> `RatelSoft.Vision.GPU`
- `Ratel.WPF` -> `RatelSoft.Utils.Wpf.Logging`
- `WPFLogViewer` -> `RatelLogViewer`
- `RatelLib.Log` -> `AppLog.Log`

## 10) 자동 생성 문서 폴더 처리

`RatelSoft.Common` buildTransitive로 생성될 수 있는 폴더:
- `RatelSoftDocs`

`.gitignore`:

```gitignore
EsViewer/RatelSoftDocs/
```

## 11) 검증 명령

```powershell
dotnet restore EsViewer/EsViewer.csproj
dotnet build EsViewer/EsViewer.csproj -c Debug
rg -n "Ratel\.Vision|Ratel\.WPF|RatelLib\.Log|WPFLogViewer" EsViewer --glob "*.cs" --glob "*.xaml" --glob "*.config"
```

## 12) 참고 문서
- `README.md`
- `Docs/RatelSoft_CODEBOOK.md`
- `Docs/RatelSoft_CODEBOOK_RatelViewer.md`
- `Docs/RatelSoft_CODEBOOK_EsViewer_PRACTICAL.md`
- `Docs/RatelSoft_AI_CONSUMER_GUIDE.md`
- `Docs/RatelSoft_TYPE_INDEX.md`
- `Docs/RatelSoft_XML_REFERENCE_MANUAL.md`

## 13) EsViewer에서 실제 사용한 코드 정리

파일 기준:
- `EsViewer/EsViewer.csproj`
  - `net8.0-windows`, `RatelSoft.*` 패키지 참조
- `EsViewer/globalusing.cs`
  - RatelSoft 네임스페이스 전역 using
- `EsViewer/AppLog.cs`
  - `RatelLog.ForSource(...)` 기반 공통 로거
- `EsViewer/App.xaml.cs`
  - `RatelLib.InitLib()`, `Cuda.InitGPU()`, debug console attach/alloc
- `EsViewer/nlog.config`
  - `RatelLogViewer` 타깃 + `RatelSoft.Utils.Wpf` extension
- `EsViewer/MainWindow.xaml`
  - `RatelSoft.Vision.Wpf`, `RatelSoft.Utils.Wpf.Logging` xmlns
- `EsViewer/MainWindow.xaml.cs`
  - `MuraEx.InspectEx`, `MuraEx.InspectExGpu` 호출
- `EsViewer/GeoLensWindow.xaml.cs`
  - `AppLog.Log` 기반 처리 로그 + AI crop 흐름
- `EsViewer/Mura/*.xaml`, `EsViewer/Mura/*.cs`
  - `RatelSoft.Vision.Mura` 타입 기반 설정/편집/결과 UI

API 기준:
- 초기화: `RatelLib.InitLib`, `Cuda.InitGPU`
- 로깅: `RatelLog.ForSource`, `RatelLogger.Info/Error/Warn`
- 검사: `MuraEx.InspectEx`, `MuraEx.InspectExGpu`, `GetDefectList`
- UI: `RatelViewer`, `ImageMouseArgs`, `LogViewer`
