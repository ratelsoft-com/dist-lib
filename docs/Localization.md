# Localization Guide

이 문서는 현재 `RatelLib`의 WPF 다국어(로컬라이제이션) 표준 적용 방법을 정리합니다.

## 개요

현재 구조는 다음 기준을 따릅니다.

- 문자열 원본: `.resx` 리소스 파일
- 공통 로컬라이저: `RatelSoft.Common/Localization/ResourceLocalizationManager.cs`
- UI 바인딩: `Localizer[ResourceKey]` 인덱서 바인딩
- 언어 전환: `SetCulture("ko")`, `SetCulture("en")` 또는 컨트롤 DP(`UiLanguage`) 변경

---

## 현재 적용 예시 (LogViewer)

### 관련 파일

- 공통 매니저: `RatelSoft.Common/Localization/ResourceLocalizationManager.cs`
- 영어 리소스(기본): `RatelSoft.Utils.Wpf/Resources/LogViewerStrings.resx`
- 한국어 리소스: `RatelSoft.Utils.Wpf/Resources/LogViewerStrings.ko.resx`
- 컨트롤 코드: `RatelSoft.Utils.Wpf/Logging/LogViewer.xaml.cs`
- XAML 바인딩: `RatelSoft.Utils.Wpf/Logging/LogViewer.xaml`

### 컨트롤 코드 패턴

```csharp
public ResourceLocalizationManager Localizer { get; } =
    new ResourceLocalizationManager("RatelSoft.Utils.Wpf.Resources.LogViewerStrings", typeof(LogViewer));

public static readonly DependencyProperty UiLanguageProperty =
    DependencyProperty.Register(
        nameof(UiLanguage),
        typeof(string),
        typeof(LogViewer),
        new PropertyMetadata("ko", OnUiLanguageChanged));

private static void OnUiLanguageChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
    if (d is LogViewer control)
    {
        control.Localizer.SetCulture(e.NewValue as string);
    }
}
```

### XAML 바인딩 패턴

```xml
<TextBlock Text="{Binding RelativeSource={RelativeSource AncestorType=local:LogViewer}, Path=Localizer[SearchLabel]}"/>
<CheckBox Content="{Binding RelativeSource={RelativeSource AncestorType=local:LogViewer}, Path=Localizer[WarnOnlyLabel]}"/>
```

컨텍스트 메뉴처럼 `DataContext`가 분리되는 경우:

```xml
<MenuItem Header="{Binding PlacementTarget.Tag.Localizer[CopyLabel], RelativeSource={RelativeSource AncestorType=ContextMenu}}"/>
```

---

## 새 화면에 다국어 적용하는 방법

1. 리소스 파일 생성
- `FeatureStrings.resx` (기본 언어, 권장: 영어)
- `FeatureStrings.ko.resx` (한국어)

2. 리소스 키 추가
- 예: `Title`, `StartButton`, `StopButton`, `ErrorMessage`

3. 코드 비하인드/뷰모델에 로컬라이저 생성
- `new ResourceLocalizationManager("<기본네임스페이스>.Resources.FeatureStrings", typeof(현재타입))`

4. XAML에서 `Localizer[키]` 바인딩
- 고정 문자열 제거

5. 언어 전환 연결
- 앱/화면에서 `SetCulture("ko")`, `SetCulture("en")` 호출
- 또는 DP를 만들어 전환 트리거

---

## RatelWPF 테스트 예제

`RatelWPF/MainWindow.xaml.cs`:

```csharp
private void SetLogViewerKorean_Click(object sender, RoutedEventArgs e)
{
    logViewer.SetUiLanguage("ko");
}

private void SetLogViewerEnglish_Click(object sender, RoutedEventArgs e)
{
    logViewer.SetUiLanguage("en");
}
```

`RatelWPF/MainWindow.xaml`:

```xml
<Button Content="한국어 로그 UI" Click="SetLogViewerKorean_Click"/>
<Button Content="English Log UI" Click="SetLogViewerEnglish_Click"/>
```

---

## 디자이너(Design)에서 텍스트가 안 보일 때

다음 항목을 확인합니다.

- `Localizer`는 `InitializeComponent()` 이전(필드 초기화) 생성
- XAML 바인딩은 `RelativeSource AncestorType=...` 사용 (`ElementName`보다 디자이너 안정적)
- 필요하면 `d:Text`, `d:Content`, `FallbackValue` 추가
- `d:DataContext` 설정으로 디자인 타임 데이터 지정

---

## 권장 규칙

- 공통 로컬라이저 클래스는 `RatelSoft.Common` 유지
- 화면/기능별 리소스 파일은 해당 어셈블리의 `Resources` 폴더에 배치
- 기본 리소스는 영어, 지역별 리소스(`.ko`, `.ja`, `.zh`)를 추가하는 방식 권장
- 문자열 하드코딩 대신 리소스 키를 사용
