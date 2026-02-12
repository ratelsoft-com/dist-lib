# RatelSoft Codebook - EsViewer Practical Patterns

이 문서는 `EsViewer` 마이그레이션(`Ratel.*` -> `RatelSoft.*`)에서 실제로 사용된 패턴만 모아둔 실전 코드북입니다.

## 1) Project Baseline (`net8.0-windows`)

`EsViewer/EsViewer.csproj` 핵심 패턴:

```xml
<PropertyGroup>
  <TargetFramework>net8.0-windows</TargetFramework>
  <UseWPF>true</UseWPF>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="RatelSoft.Common" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Lib" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Types" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Utils.Wpf" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Vision" Version="2.26.209.13" />
  <PackageReference Include="RatelSoft.Vision.Wpf" Version="2.26.209.13" />
</ItemGroup>
```

## 2) Local NuGet Feed (`C:\data\nuget`)

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

## 3) Global Using Layout

`globalusing.cs`:

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

## 4) App Logger Entry (RatelLogger 통일)

```csharp
public static class AppLog
{
    public static RatelLogger Log { get; } = RatelLog.ForSource("EsViewer");
}
```

기존 `RatelLib.Log` 직접 사용 패턴은 `AppLog.Log`로 통일.

## 5) Startup + GPU Init + Debug Console

`App.xaml.cs` 핵심 흐름:

```csharp
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
```

## 6) NLog + WPF LogViewer Target

`nlog.config` 핵심:

```xml
<extensions>
  <add assembly="RatelSoft.Utils.Wpf" />
</extensions>

<target xsi:type="RatelLogViewer" name="logViewer"
        layout="${longdate}|${level}|${message}|${all-event-properties} ${exception:format=message}" />
```

주의:
- `xsi:type="WPFLogViewer"`가 아니라 `xsi:type="RatelLogViewer"`를 사용.

## 7) XAML Namespace Migration

Before:
- `clr-namespace:Ratel.Vision.WPF;assembly=Ratel.Vision`
- `clr-namespace:Ratel.WPF;assembly=Ratel`

After:
- `clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf`
- `clr-namespace:RatelSoft.Utils.Wpf.Logging;assembly=RatelSoft.Utils.Wpf`

예:

```xml
xmlns:vision="clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf"
xmlns:wpf="clr-namespace:RatelSoft.Utils.Wpf.Logging;assembly=RatelSoft.Utils.Wpf"
```

## 8) Mura Inspect API 호출 패턴

`MuraEx.InspectEx` 호출 시 `muraOptions` 파라미터명을 사용:

```csharp
var defectGroups = MuraEx.InspectEx<T>(
    smallMat,
    rotateConfig,
    maxDefectCount,
    mask: smallMask,
    threadType: threadType,
    maxThreadCount: maxThreadCount,
    muraOptions: scratchFilter,
    log: AppLog.Log);
```

## 9) 자동 복사 문서 폴더 처리

`RatelSoft.Common`의 buildTransitive target으로 소비 프로젝트에 `RatelSoftDocs`가 생성될 수 있음.

`.gitignore` 권장:

```gitignore
EsViewer/RatelSoftDocs/
```

## 10) Quick Verification Commands

```powershell
dotnet restore EsViewer/EsViewer.csproj
dotnet build EsViewer/EsViewer.csproj -c Debug
rg -n "Ratel\\.Vision|Ratel\\.WPF|RatelLib\\.Log" EsViewer --glob "*.cs" --glob "*.xaml"
```

## 11) Vision.Wpf 이관 항목

EsViewer 기준 이관 완료:
- `EsViewer/_3DProfileWindow` -> `RatelSoft.Vision.Wpf.HeightMap3DWindow`
- `EsViewer/Mura/MuraConfigEditWindow` -> `RatelSoft.Vision.Wpf.Mura.MuraConfigEditWindow`

호출 예:

```csharp
var win = new HeightMap3DWindow(curMat[roi]);
win.Owner = this;
win.ShowDialog();
```

```csharp
using MuraConfigEditWindow = RatelSoft.Vision.Wpf.Mura.MuraConfigEditWindow;
```

