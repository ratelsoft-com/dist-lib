# 로그 시스템 (RatelLog)

## 목표
- MainWindow / ClientWindow(다중 인스턴스 포함) 별로 로그를 분리해서 UI(LogViewer)와 파일로 라우팅한다.
- Window를 추론하지 않고, 로그 이벤트에 WindowId(=Caller)를 **명시적으로 포함**하는 방식만 사용한다.
- 로깅 API는 가볍게 유지하면서도, 출력은 NLog의 타겟/필터 기능을 그대로 활용한다.

## 구성요소
- RatelLog: `RatelSoft.Utils.Common.Logging` 네임스페이스의 정적 로거. LogItem을 만들고 _sink로 전달한다.
- RatelLogger: RatelLog 위에 얹은 인스턴스형 로거(창/소스 고정용).
- LogItem: 이벤트 모델(시간, 레벨, 이벤트명, WindowId, SessionId, CorrelationId, 메시지, 예외 요약, 데이터, 호출자 정보, 스코프 등).
- NLog 어댑터: RatelLog의 기본 _sink로 등록되어 LogItem을 NLog LogEventInfo로 변환한다.
- LogViewer: WPF UI에서 NLogLogViewerTarget을 통해 실시간 표시한다.

## 전체 흐름
1. 앱 코드에서 RatelLog 또는 RatelLogger를 호출한다.
2. RatelLog가 LogItem을 생성하고 _sink로 전달한다.
3. 기본 _sink는 NLog 어댑터로, LogItem을 NLog LogEventInfo로 매핑한다.
4. NLog 설정(targets/rules/layouts)에 따라 파일/콘솔/UI로 출력된다.
5. UI는 NLogLogViewerTarget → LogViewer로 표시된다.

## MDI 라우팅 흐름 다이어그램
```
[MainWindow]                         [ClientWindow#1]
    | PushWindow("MainWindow")           | PushWindow("Client#1")
    v                                   v
AsyncLocal(Context)                AsyncLocal(Context)
    |                                   |
    +------------> RatelLog.CreateItem <+
                   WindowId 결정
                          |
                          v
                 Sink(ForwardToNLog)
                          |
                          v
       NLog Targets/Rules: ${event-properties:item=Caller}
            |                              |
   LogViewer(MainWindow)          LogViewer(Client#1)
            |                              |
    file_MainWindow.log            file_Client#1.log
```

## Window 분리 규칙
- WindowId는 명시적으로 설정한다.
  - `RatelLog.ForWindow("MainWindow")`
  - `RatelLog.ForWindowOrSource(windowId, source)`
- NLog 어댑터는 WindowId를 `LogEventInfo.Properties["Caller"]`에 넣는다.
- 분리/필터/파일명 규칙은 `${event-properties:item=Caller}`를 사용한다.

## AsyncLocal 컨텍스트 (ContextState)
- `RatelLog`는 `AsyncLocal<ContextState?> _context`로 **논리적 실행 흐름(Async/await 체인)**별 상태를 보관한다.
- `ContextState`는 `WindowId / SessionId / CorrelationId / Scope`를 가진다.
- 로그 생성 시(`CreateItem`) **명시 인자 > AsyncLocal 컨텍스트** 순으로 값을 채운다.
  - 예: `LogItem.WindowId = windowId ?? state.WindowId`
- 결과적으로, 라이브러리가 `windowId`를 직접 넘기지 않아도 `PushWindow`로 설정된 값이 자동으로 들어간다.

## PushWindow / PushContext 동작
- `PushWindow(windowId)`는 `PushContext`의 편의 래퍼로 **현재 AsyncLocal 컨텍스트의 WindowId를 임시로 설정**한다.
- 호출 시 이전 값을 저장하고, `Dispose()` 시 복원한다. (중첩 가능)
- `using` 범위 내에서 실행되는 모든 로그가 해당 WindowId를 가진다.

```csharp
var log = RatelLog.ForWindow("MainWindow");
using (RatelLog.PushWindow("MainWindow"))
{
    log.Info("Init", "MainWindow ready");
    await _service.LoadAsync(); // async 흐름에서도 WindowId 유지
}
```

## BeginScope 동작
- `BeginScope(name, value)`는 **현재 AsyncLocal 컨텍스트에 key/value를 추가**한다.
- 동일 키가 이미 있으면 기존 값을 저장해 두었다가 범위 종료 시 복원한다. (중첩 가능)
- `LogItem.Scope`로 스냅샷이 저장되고, NLog 이벤트에 `Scope.{key}`로 전달된다.

```csharp
var log = RatelLog.ForSource("Import");
using (RatelLog.BeginScope("Operation", "FileImport"))
{
    log.Info("Start", "Import begin");
    log.Warn("Slow", "Large file detected");
}
// NLog Properties: Scope.Operation = "FileImport"
```

## 라이브러리 주입 패턴 (권장)
### 1) RatelLogger 주입 (가장 단순)
- 앱(호출자)에서 WindowId/Source를 고정한 `RatelLogger`를 만들고 라이브러리에 전달한다.
- 라이브러리는 **static을 몰라도** 인스턴스 로거만 사용한다.

```csharp
// App (composition root)
var log = RatelLog.ForWindowOrSource("MainWindow", "NetLib");
var client = new NetClient(log);

// Library
public sealed class NetClient
{
    private readonly RatelLogger _log;
    public NetClient(RatelLogger log) => _log = log;

    public void Connect() => _log.Info("Connect", "Connecting...");
}
```


### 2) Source만 고정 + Window는 PushWindow로 주입
- 창 컨텍스트는 UI 레벨에서 `PushWindow`로 설정하고,
- 라이브러리는 `RatelLog.ForSource(...)`만 사용해도 된다.

```csharp
// App (UI)
using (RatelLog.PushWindow("MainWindow"))
{
    _client.Connect();
}

// Library (source만 고정)
private readonly RatelLogger _log = RatelLog.ForSource("NetLib");
```

### 3) 인터페이스로 감싸기 (테스트/DI 강화)
- `IRatelLogger` 같은 인터페이스를 두고 `RatelLogger`를 어댑트하면 DI/테스트가 쉬워진다.
- 내부 구현은 그대로 `RatelLogger`를 사용한다.

```csharp
public interface IRatelLogger
{
    LogItem Emit(
        LogLevel level,
        string @event,
        string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null,
        string? sessionId = null,
        string? correlationId = null);

    LogItem Info(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null);

    LogItem Warn(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null);

    LogItem Error(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null);
}

public sealed class RatelLoggerAdapter : IRatelLogger
{
    private readonly RatelLogger _inner;
    public RatelLoggerAdapter(RatelLogger inner) => _inner = inner;

    public LogItem Emit(LogLevel level, string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null,
        string? sessionId = null,
        string? correlationId = null)
        => _inner.Emit(level, @event, message, data, ex, sessionId, correlationId);

    public LogItem Info(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null)
        => _inner.Info(@event, message, data, ex);

    public LogItem Warn(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null)
        => _inner.Warn(@event, message, data, ex);

    public LogItem Error(string @event, string message,
        IReadOnlyDictionary<string, object?>? data = null,
        Exception? ex = null)
        => _inner.Error(@event, message, data, ex);
}
```

### DI 등록 예시
- 창별로 다른 WindowId를 주입하려면 **팩토리**를 등록하는 방식이 가장 단순하다.

```csharp
// Composition root
services.AddSingleton<Func<string, IRatelLogger>>(windowId =>
{
    var inner = RatelLog.ForWindowOrSource(windowId, "NetLib");
    return new RatelLoggerAdapter(inner);
});

// Window/VM 생성 시
public sealed class MainWindowViewModel
{
    private readonly IRatelLogger _log;
    public MainWindowViewModel(Func<string, IRatelLogger> loggerFactory)
    {
        _log = loggerFactory("MainWindow");
    }
}
```

```csharp
// 라이브러리
public sealed class NetClient
{
    private readonly IRatelLogger _log;
    public NetClient(IRatelLogger log) => _log = log;

    public void Connect() => _log.Info("Connect", "Connecting...");
}
```

## NLog 매핑 규칙 (RatelLog → NLog)
- LoggerName: `item.Source`(없으면 "RatelLog")
- Message: `"[Event] Message"` 형식으로 구성, 예외가 있으면 `"| Type: Message"` 추가
- Properties:
  - `Caller`: WindowId (분리/필터용)
  - `WindowId`, `Event`, `SessionId`, `CorrelationId`
  - `CallerMember`, `CallerFile`, `CallerLine`
  - `Scope.*` (BeginScope 값)
  - `Data` (추가 데이터 사전)
  - `ExceptionType`, `ExceptionMessage`, `ExceptionStack`

## 기본 동작 및 커스터마이즈
- 기본값: RatelLog는 NLog 어댑터를 _sink로 등록한다.
- NLog 설정을 하지 않으면 출력되지 않을 수 있다. (NLog의 타겟/룰은 별도로 구성 필요)
- 커스터마이즈:
  - `RatelLog.Configure(sink, minLevel)`로 _sink를 교체한다. NLog를 유지하려면 sink 내부에서 `RatelLog.ForwardToNLog(item)`를 호출한다.
  - `RatelLog.Emitted`를 구독하면 출력 파이프에 손대지 않고 로그를 관찰할 수 있다. (UI/테스트/디버그용)

## 사용 예시 (코드 기반 NLog 구성)
App의 OnStartup()에서 한 번 호출한다.

```csharp
private static void ConfigureNLog()
{
    LogManager.Setup().SetupExtensions(ext =>
    {
        ext.RegisterTarget<NLogLogViewerTarget>("RatelLogViewer");
    });

    var config = LogManager.Configuration ?? new LoggingConfiguration();

    var viewerTarget = new NLogLogViewerTarget
    {
        Name = "viewer"
    };

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

    config.AddTarget(viewerTarget);
    config.AddTarget(fileTarget);

    config.AddRule(NLog.LogLevel.Trace, NLog.LogLevel.Fatal, viewerTarget);
    config.AddRule(NLog.LogLevel.Info, NLog.LogLevel.Fatal, fileTarget);

    LogManager.Configuration = config;
}
```

## 사용 예시 (nlog.config)
```xml
<?xml version="1.0" encoding="utf-8" ?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      throwConfigExceptions="true">
  <extensions>
    <!-- RatelLogViewer 커스텀 타겟이 들어있는 어셈블리를 NLog가 로드하도록 명시 -->
    <add assembly="RatelSoft.Utils.Wpf" />
  </extensions>
  <targets>
    <target name="logconsole" xsi:type="Console" />
    <target name="ratelViewer" xsi:type="RatelLogViewer" />
  </targets>

  <rules>
    <logger name="*" minlevel="Info" writeTo="logconsole" />
    <logger name="*" minlevel="Debug" writeTo="ratelViewer" />
  </rules>
</nlog>
```

## LogViewer 사용 예시
```xml
<Window x:Class="..."
        ...
        xmlns:logging="clr-namespace:RatelSoft.WPF.Utils.Logging;assembly=RatelSoft.Utils.Wpf"
        ...>

    <logging:LogViewer Grid.Row="5" Margin="0"/>
</Window>
```

## LogViewer 기능
- 실시간 검색: Level/Logger/Message에서 대소문자 구분 없이 검색
- Clear: 현재 표시된 로그 삭제
- 내보내기: 로그를 텍스트로 저장 후 자동 열기
  - 파일명: `logs_export_yyyyMMdd_HHmmss.txt`
  - 위치: `{AppDir}/logs/`
- 파일 열기: 로그 폴더를 탐색기로 열기

## 파일 분리 규칙 (예시)
- 레이아웃: `${longdate}|${level}|${logger}|${event-properties:item=Caller}|${message}`
- 조건(필터): `'${event-properties:item=Caller}' == 'MainWindow'`

## 주의사항
- WindowId/Caller가 비어있으면 창별 분리/필터가 깨진다. 반드시 WindowId를 명시 전달한다.
- `Caller`는 창 식별 용도로 사용한다. 코드 호출 위치는 `CallerMember/CallerFile/CallerLine`을 사용한다.
- 여러 sink가 필요한 경우에는 `RatelLog.Configure`에서 직접 fan-out 하거나, `RatelLog.Emitted`를 보조 관찰용으로 사용한다.
- _sink를 교체(Configure)하면 기본 NLog 어댑터가 제거되므로, 필요 시 `RatelLog.ForwardToNLog`를 함께 호출한다.
- Emitted 핸들러가 같은 출력 대상(NLog 등)으로 다시 전달하면 중복 로그가 발생할 수 있다.


