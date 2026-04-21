# RatelViewerEx Consumer Codebook

이 문서는 `RatelSoft.Vision.Wpf.RatelViewerEx`를 검사 UI에서 사용하는 복붙 중심 가이드다.

목표:
- 기존 `RatelViewer`의 메뉴/툴바/Profile/Histogram 기능 유지
- ROI는 기존 `Shape` 편집 방식 유지
- 불량(defect)은 대량 표시용 overlay로 등록
- defect 선택은 hit test로 처리

## 1) 언제 `RatelViewerEx`를 쓰나

`RatelViewer`는 ROI 편집 중심 뷰어다.

`RatelViewerEx`는 여기에 아래 요구가 추가될 때 쓴다.

- ROI 편집은 그대로 필요
- defect 표시 개수가 많음
- defect는 `Shape`로 편집하지 않고 표시/선택 위주로 사용
- defect 클릭 선택 이벤트가 필요

## 2) XAML

기본 사용은 `RatelViewer`와 거의 같다.

```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vision="clr-namespace:RatelSoft.Vision.Wpf;assembly=RatelSoft.Vision.Wpf">
    <Grid>
        <vision:RatelViewerEx x:Name="viewer"
                              ShowMenu="True"
                              ShowToolBar="True"
                              ShowStatusBar="True"/>
    </Grid>
</Window>
```

## 3) Mat 표시

`Mat` 사용 방식은 기존과 동일하다.

```csharp
viewer.Mat = mat;
```

ViewModel 바인딩도 가능하다.

## 4) Defect 등록

불량은 `AddRectangle(...)` 같은 편집용 API로 넣지 않고 `SetDefects(...)`로 한 번에 넣는다.

```csharp
using System.Windows;
using System.Windows.Media;
using RatelSoft.Vision.Wpf;

var defects = new[]
{
    new DefectOverlayItem
    {
        Id = "D-001",
        ShapeType = DefectShapeType.Rectangle,
        Bounds = new Rect(120, 80, 40, 20),
        Stroke = Brushes.Lime,
        Thickness = 1.5,
        Tag = "Scratch",
    },
    new DefectOverlayItem
    {
        Id = "D-002",
        ShapeType = DefectShapeType.Polygon,
        Points = new[]
        {
            new Point(300, 220),
            new Point(340, 230),
            new Point(330, 260),
            new Point(290, 250),
        },
        Stroke = Brushes.Yellow,
        Fill = new SolidColorBrush(Color.FromArgb(40, 255, 255, 0)),
        Thickness = 1,
        Tag = "Cluster",
    },
};

viewer.SetDefects(defects);
```

초기화:

```csharp
viewer.ClearDefects();
```

## 5) Defect 선택

defect는 drawing overlay로 그려지지만 클릭 선택은 가능하다.

```csharp
viewer.DefectClicked += (_, e) =>
{
    var item = e.Item;
    statusText.Text = $"{item.Id} selected";
};

viewer.DefectDoubleClicked += (_, e) =>
{
    var item = e.Item;
    MessageBox.Show($"Open defect detail: {item.Id}");
};

viewer.DefectSelectionChanged += (_, e) =>
{
    propertyGrid.SelectedObject = e.NewItem;
};
```

현재 선택된 defect:

```csharp
var selected = viewer.SelectedDefect;
```

ID로 선택:

```csharp
viewer.SelectDefect("D-002");
```

defect 상태 이벤트:

```csharp
viewer.DefectCollectionChanged += (_, e) =>
{
    defectCountText.Text = e.Items.Count.ToString();
};

viewer.DefectSelectionChanged += (_, e) =>
{
    statusText.Text = e.NewItem?.Id ?? "No defect";
};
```

## 6) 선택된 Defect 스타일 설정

선택 강조 스타일은 외부에서 설정할 수 있다.

- `SelectedDefectStroke`
- `SelectedDefectFill`
- `SelectedDefectThicknessDelta`

코드 설정:

```csharp
viewer.SelectedDefectStroke = Brushes.Cyan;
viewer.SelectedDefectFill = new SolidColorBrush(Color.FromArgb(64, 0, 255, 255));
viewer.SelectedDefectThicknessDelta = 3;
```

XAML 설정:

```xml
<vision:RatelViewerEx x:Name="viewer"
                      SelectedDefectStroke="Cyan"
                      SelectedDefectThicknessDelta="3">
    <vision:RatelViewerEx.SelectedDefectFill>
        <SolidColorBrush Color="#4000FFFF" />
    </vision:RatelViewerEx.SelectedDefectFill>
</vision:RatelViewerEx>
```

개별 defect의 기본 표시 스타일은 `DefectOverlayItem`에서 설정한다.

```csharp
var item = new DefectOverlayItem
{
    Id = "D-003",
    ShapeType = DefectShapeType.Rectangle,
    Bounds = new Rect(200, 120, 30, 18),
    Stroke = Brushes.Yellow,
    Fill = new SolidColorBrush(Color.FromArgb(32, 255, 255, 0)),
    Thickness = 1,
};
```

즉,

- 평상시 스타일: `DefectOverlayItem`
- 선택 강조 스타일: `RatelViewerEx`

로 나눠서 설정하면 된다.

## 7) ROI와 Defect 역할 분리

권장 패턴:

- ROI: 기존 `Rectangle`, `Line`, `Polygon` 편집 API 사용
- defect: `SetDefects(...)` 사용

예:

```csharp
viewer.MouseMode = MouseMode.DrawRect;

var roi = viewer.GetDefRectangle();
viewer.AddRectangle(roi, new Rect(50, 50, 200, 120));
```

즉,

- ROI는 편집용 `Shape`
- defect는 render-only overlay

로 나눠서 쓴다.

## 8) 메뉴/툴바/Profile/Histogram

`RatelViewerEx`는 `RatelViewer`를 상속하므로 기존 기능을 그대로 사용한다.

- 상단 메뉴
- Draw toolbar
- Zoom toolbar
- `Profile`
- `XProjection`
- `YProjection`
- `Histogram`

기존 `RatelViewer`에서 쓰던 `profileMenu`, `showHistogramMenu`, `MoveToToolBars(...)` 같은 패턴도 동일하게 적용 가능하다.

## 9) 소비자 변경 범위

기존 `RatelViewer`에서 `RatelViewerEx`로 바꿀 때 보통 바뀌는 부분은 아래뿐이다.

1. XAML 타입 이름
2. defect 등록 코드

예:

```xml
<vision:RatelViewer x:Name="viewer"/>
```

를

```xml
<vision:RatelViewerEx x:Name="viewer"/>
```

로 바꾸고,

```csharp
viewer.AddRectangle(...);
viewer.AddRectangle(...);
viewer.AddRectangle(...);
```

같은 defect 표시 코드를

```csharp
viewer.SetDefects(defects);
```

로 바꾸면 된다.

ROI 편집 코드는 대부분 그대로 유지할 수 있다.

## 10) 주의

- defect는 `Shape` 컬렉션에 들어가지 않는다.
- defect는 표시/선택 중심이다.
- ROI처럼 adorner 기반 자유 편집을 전제로 하지 않는다.
- defect를 편집 대상으로 올리고 싶으면 선택된 item만 별도 `Shape`로 승격하는 정책을 추천한다.
