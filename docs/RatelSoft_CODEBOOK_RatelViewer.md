# RatelViewer Consumer Codebook

이 문서는 `RatelSoft.Vision.Wpf.RatelViewer`를 소비자 프로젝트에서 소스 없이 바로 사용하는 복붙 중심 가이드다.

## 1) 최소 사용 (XAML)

```xml
<Window x:Class="Sample.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vision="clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf">
  <Grid>
    <vision:RatelViewer x:Name="viewer" />
  </Grid>
</Window>
```

## 2) `Mat` DependencyProperty 바인딩

`RatelViewer.Mat`은 DependencyProperty이므로 ViewModel 바인딩 가능하다.

```xml
<vision:RatelViewer x:Name="viewer"
                    Mat="{Binding CurrentMat}" />
```

```csharp
using OpenCvSharp;

public class MainViewModel
{
    public Mat? CurrentMat { get; set; }
}
```

주의:
- `Mat` 갱신은 UI 스레드에서 바인딩 반영되도록 처리한다.
- 같은 `Mat` 인스턴스를 내부 데이터만 바꾼 경우에는 바인딩 변경이 발생하지 않으므로 인스턴스를 교체하거나 `viewer.Refresh()`를 호출한다.

## 3) Menu/ToolBar/StatusBar 숨김 + 소비자 UI 구성

### 3.1 내부 크롬 숨기기

```xml
<vision:RatelViewer x:Name="viewer"
                    ShowMenu="False"
                    ShowToolBar="False"
                    ShowStatusBar="False" />
```

### 3.2 외부 ToolBarTray로 이동

```xml
<DockPanel>
  <ToolBarTray x:Name="hostToolBars" DockPanel.Dock="Top" />
  <vision:RatelViewer x:Name="viewer"
                      ShowMenu="False"
                      ShowToolBar="False" />
</DockPanel>
```

```csharp
private void InitViewerHost()
{
    // 내부 Enhance/Draw/Zoom 툴바를 외부 tray로 이동
    viewer.MoveToToolBars(hostToolBars, startBand: 0);
}
```

### 3.3 외부 메뉴로 재배치

`RatelViewer`의 주요 메뉴 필드는 public으로 노출되어 있다(`profileMenu` 등).

```xml
<DockPanel>
  <Menu x:Name="hostMenu" DockPanel.Dock="Top">
    <MenuItem Header="_View" x:Name="viewMenu" />
  </Menu>
  <vision:RatelViewer x:Name="viewer" ShowMenu="False" />
</DockPanel>
```

```csharp
private void InitViewerMenus()
{
    // Profile 메뉴를 호스트 메뉴에 재사용
    viewMenu.Items.Add(viewer.profileMenu);
}
```

### 3.4 메뉴/툴바 전체 재구성 템플릿 (복붙용)

아래 코드는 `RatelViewer` 내부 메뉴/툴바를 숨긴 뒤, 소비자 창의 메뉴/툴바에 완전히 재배치하는 예시다.

```xml
<Window x:Class="Sample.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vision="clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf"
        Title="Viewer Host" Height="900" Width="1400"
        Loaded="Window_Loaded">
  <DockPanel>
    <Menu DockPanel.Dock="Top">
      <MenuItem Header="_File" x:Name="HostFileMenu" />
      <MenuItem Header="_Profiles" x:Name="HostProfileMenu" />
      <MenuItem Header="_Draw">
        <MenuItem Header="Rect" Click="DrawRect_Click" />
        <MenuItem Header="Line" Click="DrawLine_Click" />
        <MenuItem Header="Ellipse" Click="DrawEllipse_Click" />
        <MenuItem Header="Polygon" Click="DrawPolygon_Click" />
        <Separator />
        <MenuItem Header="Stop" Click="DrawStop_Click" />
        <MenuItem Header="Clear ROI" Click="ClearRoi_Click" />
      </MenuItem>
      <MenuItem Header="_View">
        <MenuItem Header="Fit Screen" Click="FitScreen_Click" />
        <MenuItem Header="100%" Click="Zoom100_Click" />
      </MenuItem>
    </Menu>

    <ToolBarTray x:Name="HostToolBarTray" DockPanel.Dock="Top">
      <ToolBar Header="Consumer Draw">
        <Button Content="Rect" Click="DrawRect_Click" />
        <Button Content="Line" Click="DrawLine_Click" />
        <Button Content="Ellipse" Click="DrawEllipse_Click" />
        <Button Content="Poly" Click="DrawPolygon_Click" />
        <Button Content="Stop" Click="DrawStop_Click" />
      </ToolBar>
    </ToolBarTray>

    <vision:RatelViewer x:Name="viewer"
                        ShowMenu="False"
                        ShowToolBar="False"
                        ShowStatusBar="False" />
  </DockPanel>
</Window>
```

```csharp
using System.Windows;
using RatelSoft.Vision.Wpf;

namespace Sample;

public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }

    private void Window_Loaded(object sender, RoutedEventArgs e)
    {
        BuildViewerChrome();
        HookViewerEvents();
    }

    private void BuildViewerChrome()
    {
        // 1) 내부 툴바를 소비자 ToolBarTray로 이동
        viewer.MoveToToolBars(HostToolBarTray, startBand: 1);

        // 2) 내부 메뉴를 소비자 Menu로 이동
        MoveMenuItem(viewer.fileMenu, HostFileMenu);
        MoveMenuItem(viewer.profileMenu, HostProfileMenu);

        // 필요 시 내장 메뉴 항목 가시성 조정
        viewer.fileSaveRawDataMenu.Visibility = Visibility.Collapsed;
        viewer.showHistogramMenu.Visibility = Visibility.Visible;
    }

    private static void MoveMenuItem(System.Windows.Controls.MenuItem source, System.Windows.Controls.MenuItem targetRoot)
    {
        if (source == null || targetRoot == null)
            return;

        // 이미 다른 부모에 붙어 있으면 먼저 분리
        if (source.Parent is System.Windows.Controls.MenuItem oldParentItem)
        {
            oldParentItem.Items.Remove(source);
        }
        else if (source.Parent is System.Windows.Controls.Menu oldParentMenu)
        {
            oldParentMenu.Items.Remove(source);
        }

        targetRoot.Items.Add(source);
    }

    private void HookViewerEvents()
    {
        viewer.StartDrawShape += (_, e) => { /* 시작점 처리 */ };
        viewer.EndDrawShape += (_, e) => { /* 완료 ROI 처리 */ };
        viewer.OnShapeEdited += (_, e) => { /* 편집 반영 */ };
    }

    private void DrawRect_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawRect;
    private void DrawLine_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawLine;
    private void DrawEllipse_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawEllipse;
    private void DrawPolygon_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawPolygon;
    private void DrawStop_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.None;
    private void ClearRoi_Click(object sender, RoutedEventArgs e) => viewer.RemoveElements("^_.*", isRegEx: true);
    private void FitScreen_Click(object sender, RoutedEventArgs e) => viewer.SetZoom(ZoomRatio.FitScreen);
    private void Zoom100_Click(object sender, RoutedEventArgs e) => viewer.SetZoom(1.0);
}
```

## 4) 도형 그리기/조작 코드북

### 4.1 그리기 스타일 지정

```csharp
private void ConfigureDrawStyle()
{
    viewer.GetDrawRectangle = () => new Rectangle
    {
        Name = "roi_rect",
        Stroke = Brushes.Lime,
        StrokeThickness = 2
    };

    viewer.GetDrawLine = () => new Line
    {
        Name = "roi_line",
        Stroke = Brushes.Orange,
        StrokeThickness = 2
    };
}
```

### 4.2 소비자 버튼으로 그리기 모드 제어

```csharp
private void DrawRect_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawRect;
private void DrawLine_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawLine;
private void DrawEllipse_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawEllipse;
private void DrawPoly_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.DrawPolygon;
private void DrawStop_Click(object sender, RoutedEventArgs e) => viewer.MouseMode = MouseMode.None;
```

### 4.3 그리기 완료/편집 이벤트 처리

```csharp
private void HookViewerEvents()
{
    viewer.StartDrawShape += (_, e) =>
    {
        // 시작점: e.Point (이미지 좌표)
    };

    viewer.EndDrawShape += (_, e) =>
    {
        if (e.Shape is Rectangle)
        {
            var rect = viewer.GetShapeRect(e.Shape);
            // rect: 이미지 좌표 기준
        }
        else if (e.Shape is Line line)
        {
            var x1 = RatelViewer.GetLeft(line);
            var y1 = RatelViewer.GetTop(line);
            var x2 = RatelViewer.GetRight(line);
            var y2 = RatelViewer.GetBottom(line);
        }
    };

    viewer.OnShapeEdited += (_, e) =>
    {
        var edited = viewer.GetShapeRect(e.Shape);
        // 편집 후 좌표 반영
    };
}
```

### 4.4 도형 추가/조회/삭제

```csharp
private void AddOverlays()
{
    viewer.AddRectangle(
        new Rectangle { Name = "result_1", Stroke = Brushes.Red, StrokeThickness = 2 },
        new Rect(100, 80, 220, 140));

    viewer.AddLine(
        new Line { Name = "axis_x", Stroke = Brushes.Cyan, StrokeThickness = 1 },
        new[] { new Point(0, 120), new Point(4096, 120) });
}

private void ClearResultOverlays()
{
    viewer.RemoveElements("^result_", isRegEx: true);
}
```

## 5) 좌표/줌/중심 이동

```csharp
// FitScreen
viewer.SetZoom(ZoomRatio.FitScreen);

// 100%
viewer.SetZoom(1.0);

// 특정 이미지 좌표를 화면 중앙으로
viewer.PointToCenter(new Point(2048, 1024));
```

좌표 변환:
- `viewer.ClientToImage(...)`
- `viewer.ImageToClient(...)`

## 6) 소비자 체크리스트

1. `RatelSoft.Vision.Wpf` 패키지 참조 확인
2. `RatelViewer.Mat` 바인딩 또는 코드 설정 방식 확정
3. 내장 UI 숨김 여부(`ShowMenu`, `ShowToolBar`, `ShowStatusBar`) 결정
4. 도형 모드 전환 버튼과 `EndDrawShape` 처리 루틴 연결
5. 오버레이 네이밍 규칙(예: `result_*`, `roi_*`) 정의 후 `RemoveElements` 정리 정책 수립
