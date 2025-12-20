# Ratel Library

## 개요

Ratel Library는 산업용 검사 및 제어 장비 개발을 위한 통합 라이브러리입니다. 특히 **Ratel.Vision**은 이미지 기반 품질 검사를 위한 고성능 얼룩(Mura) 검출 알고리즘을 제공합니다.

### 주요 특징

- **고성능 이미지 검사**: CPU/GPU 가속을 통한 대용량 이미지 실시간 처리
- **유연한 검사 설정**: 다양한 크기와 형태의 결함 검출 가능
- **멀티 스레드 지원**: 병렬 처리로 검사 속도 향상
- **WPF/WinForms UI 컴포넌트**: 이미지 뷰어 및 검사 결과 표시 도구 제공
- **OpenCV 통합**: OpenCvSharp 기반의 강력한 이미지 처리 기능

---

## 설치

### NuGet 패키지 설치

```bash
dotnet add package Ratel.Vision
```

또는 Visual Studio의 패키지 관리자 콘솔에서:

```powershell
Install-Package Ratel.Vision
```

### 요구 사항

- **.NET 6.0** 이상 또는 **.NET 8.0**
- **Windows 10** 이상
- **OpenCvSharp4** (자동 설치됨)

---

## Ratel.Vision - 얼룩(Mura) 검사 라이브러리

Ratel.Vision은 디스플레이 패널, 반도체, 필름 등 다양한 산업 분야에서 발생하는 **얼룩(Mura)** 결함을 자동으로 검출하는 라이브러리입니다.

### 핵심 기능

1. **다양한 검사 패턴**: 원형(Circle), 직사각형(Rectangle), 수평/수직(Horizontal/Vertical) 필터
2. **밝기/어둡기 결함 동시 검출**: White/Black 불량 구분
3. **GPU 가속**: CUDA를 이용한 대용량 이미지 고속 처리
4. **결함 병합**: 인접한 결함을 그룹화하여 과검출 방지
5. **마스크 영역 지원**: 특정 영역만 검사 또는 제외

---

## 빠른 시작

### 1. 라이브러리 초기화

```csharp
using Ratel;
using Ratel.Vision;
using Ratel.Vision.Mura;
using OpenCvSharp;

// 라이브러리 초기화 (프로그램 시작 시 한 번만 호출)
RatelLib.InitLib();
```

### 2. 기본 얼룩 검사

```csharp
// 이미지 로드
var mat = new Mat("input.png", ImreadModes.Grayscale);

// 검사 설정 생성 (32x32 크기, 밝기/어둡기 각 5 레벨 차이 검출)
var config = MuraConfig.MakeCir4Config(
    sizeX: 32,
    sizeY: 32,
    blackFactor: 5,      // 어두운 불량 민감도
    whiteFactor: 5,      // 밝은 불량 민감도
    diffCount: 2         // 주변과 다른 영역이 2개 이상일 때 불량으로 판정
);

// 얼룩 검사 실행
var result = MuraEx.InspectEx<MuraDefect>(
    mat,
    new[] { config },
    maxDefect: 10000,
    threadType: ThreadType.Multi
).GetDefectList();

// 결과 출력
Console.WriteLine($"검출된 불량: {result.Count}개");
foreach (var defect in result)
{
    Console.WriteLine($"위치: {defect.Rectangle}, 타입: {defect.WhiteBlack}");
}
```

### 3. 검사 결과 시각화

```csharp
// 결과 이미지 생성
Mat resultMat = mat.Channels() == 1
    ? mat.CvtColor(ColorConversionCodes.GRAY2BGR)
    : mat.Clone();

// 불량 영역에 빨간 사각형 표시
foreach (var defect in result)
{
    resultMat.Rectangle(
        defect.Rectangle.ToCV(),
        new Scalar(0, 0, 255),  // BGR: 빨강
        thickness: 2
    );
}

// 결과 저장
resultMat.SaveImage("result.png");
```

---

## 주요 클래스 및 메서드

### MuraConfig - 검사 설정

얼룩 검사의 모든 파라미터를 정의하는 클래스입니다.

#### 주요 속성

```csharp
public class MuraConfig
{
    public int SizeX { get; set; }           // 필터 가로 크기
    public int SizeY { get; set; }           // 필터 세로 크기
    public double BlackFactor { get; set; }  // 어두운 불량 검출 민감도 (0~255)
    public double WhiteFactor { get; set; }  // 밝은 불량 검출 민감도 (0~255)
    public int DiffCount { get; set; }       // 주변과 다른 영역 개수 (보통 2~4)
    public double Angle { get; set; }        // 필터 회전 각도 (도 단위)
    public string MergeGroup { get; set; }   // 병합 그룹 이름
    public AverageMethod AverageMethod { get; set; }  // Rect 또는 Gaussian
}
```

#### 편의 생성 메서드

**1. 4방향 원형 필터** (가장 일반적)

```csharp
var config = MuraConfig.MakeCir4Config(
    sizeX: 64,           // 필터 가로 크기 (픽셀)
    sizeY: 64,           // 필터 세로 크기
    blackFactor: 10,     // 어두운 불량: 주변보다 10 레벨 이상 어두울 때 검출
    whiteFactor: 10,     // 밝은 불량: 주변보다 10 레벨 이상 밝을 때 검출
    diffCount: 2,        // 4방향 중 2방향 이상에서 차이가 있어야 불량
    dpitch: 1.0,         // 비교 위치 간격 (1.0 = 필터 크기만큼 떨어진 위치)
    angle: 0             // 회전 각도 (선택)
);
```

**2. 8방향 원형 필터** (더 정밀한 검사)

```csharp
var config = MuraConfig.MakeCir8Config(
    sizeX: 32,
    sizeY: 32,
    blackFactor: 5,
    whiteFactor: 5,
    diffCount: 3,        // 8방향 중 3방향 이상
    dpitch: 1.0,
    minLevelPercent: 0.25  // 최소 레벨 퍼센트 (선택)
);
```

**3. 수평 라인 검사**

```csharp
var config = MuraConfig.MakeHoriConfig(
    sizeX: 256,          // 가로로 긴 필터
    sizeY: 8,            // 세로로 짧은 필터
    blackFactor: 5,
    whiteFactor: 5,
    diffCount: 2,
    dpitch: 1.0
);
```

**4. 수직 라인 검사**

```csharp
var config = MuraConfig.MakeVertConfig(
    sizeX: 8,            // 가로로 짧은 필터
    sizeY: 256,          // 세로로 긴 필터
    blackFactor: 5,
    whiteFactor: 5,
    diffCount: 2,
    dpitch: 1.0
);
```

### MuraEx - 검사 실행

정적 클래스로 얼룩 검사를 수행합니다.

#### InspectEx 메서드

```csharp
public static Dictionary<string, List<T>> InspectEx<T>(
    Mat mat,                        // 검사할 이미지
    IEnumerable<MuraConfig> config, // 검사 설정 리스트
    int maxDefect = 10000,          // 최대 검출 개수
    Mat mask = null,                // 마스크 이미지 (선택)
    ThreadType threadType = ThreadType.Multi,  // 실행 모드
    int maxThreadCount = 4,         // 최대 스레드 수
    MuraInspectOptions<T> muraOptions = null,  // 추가 옵션
    NLog.Logger log = null          // 로거
) where T : MuraDefect, new()
```

**반환값**: `MergeGroup`별로 그룹화된 불량 리스트 딕셔너리

**예제**:

```csharp
var configs = new List<MuraConfig>
{
    MuraConfig.MakeCir4Config(32, 32, 5, 5, diffCount: 2),
    MuraConfig.MakeCir4Config(64, 64, 10, 10, diffCount: 3)
};

// Dictionary<string, List<MuraDefect>> 형태로 반환
var resultDict = MuraEx.InspectEx<MuraDefect>(
    mat,
    configs,
    threadType: ThreadType.Multi
);

// 모든 불량을 하나의 리스트로 가져오기
var allDefects = resultDict.GetDefectList();
```

### MuraDefect - 검출 결과

검출된 불량 정보를 담는 클래스입니다.

```csharp
public class MuraDefect
{
    public Rectangle Rectangle { get; set; }      // 불량 영역 (바운딩 박스)
    public Rectangle OrgRect { get; set; }        // 원본 불량 영역
    public WB WhiteBlack { get; set; }            // White(밝음) 또는 Black(어두움)
    public double DefectLevel { get; set; }       // 불량 레벨 (평균 밝기 차이)
    public double DefectMinLevel { get; set; }    // 최소 불량 레벨
    public List<Point> Pts { get; set; }          // 불량 윤곽선 좌표

    // 불량 정보를 문자열로 반환
    public override string ToString()
}
```

**예제**:

```csharp
foreach (var defect in result)
{
    Console.WriteLine($"좌표: ({defect.Rectangle.X}, {defect.Rectangle.Y})");
    Console.WriteLine($"크기: {defect.Rectangle.Width} x {defect.Rectangle.Height}");
    Console.WriteLine($"타입: {defect.WhiteBlack}");  // White 또는 Black
    Console.WriteLine($"불량 레벨: {defect.DefectLevel}");
}
```

### ThreadType - 실행 모드

```csharp
public enum ThreadType
{
    Single,  // 단일 스레드 (안전하지만 느림)
    Multi,   // 멀티 스레드 (권장)
    Gpu      // GPU 가속 (CUDA 설치 필요)
}
```

---

## 고급 사용법

### 1. 여러 크기의 불량 동시 검출

```csharp
var configs = new List<MuraConfig>
{
    // 작은 불량 검출 (민감도 높음)
    MuraConfig.MakeCir4Config(16, 16, blackFactor: 3, whiteFactor: 3, diffCount: 2),

    // 중간 불량 검출
    MuraConfig.MakeCir4Config(64, 64, blackFactor: 5, whiteFactor: 5, diffCount: 2),

    // 큰 불량 검출 (민감도 낮음)
    MuraConfig.MakeCir4Config(256, 256, blackFactor: 10, whiteFactor: 10, diffCount: 3)
};

var result = MuraEx.InspectEx<MuraDefect>(mat, configs)
    .GetDefectList();
```

### 2. 불량 병합 (MergeGroup)

동일한 `MergeGroup`을 가진 설정의 결과는 서로 병합됩니다.

```csharp
var configs = new List<MuraConfig>
{
    MuraConfig.MakeCir4Config(32, 32, 5, 5, diffCount: 2)
        { MergeGroup = "Small" },
    MuraConfig.MakeCir4Config(64, 64, 5, 5, diffCount: 2)
        { MergeGroup = "Small" },  // 위와 병합됨

    MuraConfig.MakeCir4Config(256, 256, 10, 10, diffCount: 3)
        { MergeGroup = "Large" }   // 별도 그룹
};

var resultDict = MuraEx.InspectEx<MuraDefect>(mat, configs);

// 그룹별로 결과 처리
var smallDefects = resultDict["Small"];
var largeDefects = resultDict["Large"];
```

### 3. 마스크를 이용한 부분 검사

특정 영역만 검사하거나 특정 영역을 제외할 수 있습니다.

```csharp
// 마스크 이미지 생성 (흰색: 검사, 검은색: 무시)
var mask = new Mat(mat.Size(), MatType.CV_8UC1, new Scalar(0));
mask.Rectangle(new Rect(100, 100, 500, 500), new Scalar(255), -1);

// 마스크 영역만 검사
var result = MuraEx.InspectEx<MuraDefect>(
    mat,
    new[] { config },
    mask: mask
).GetDefectList();
```

### 4. GPU 가속 사용

```csharp
// GPU 초기화 (프로그램 시작 시 한 번만)
Ratel.GPU.Cuda.InitGPU();

// GPU 모드로 검사
var result = MuraEx.InspectEx<MuraDefect>(
    mat,
    configs,
    threadType: ThreadType.Gpu
).GetDefectList();
```

**주의**: GPU 모드를 사용하려면 CUDA Toolkit 및 호환되는 NVIDIA GPU가 필요합니다.

### 5. Gaussian 필터 사용

더 부드러운 결과를 원할 때 가우시안 필터를 사용할 수 있습니다.

```csharp
var config = MuraConfig.MakeCir4Config(64, 64, 5, 5, diffCount: 2);
config.AverageMethod = AverageMethod.Gaussian;
config.SigmaX = 0.5;  // 가우시안 표준편차 (X)
config.SigmaY = 0.5;  // 가우시안 표준편차 (Y)

var result = MuraEx.InspectEx<MuraDefect>(mat, new[] { config })
    .GetDefectList();
```

### 6. 각도 회전 검사

특정 각도로 회전된 패턴의 불량을 검출할 때 사용합니다.

```csharp
var config = MuraConfig.MakeCir4Config(64, 64, 5, 5, diffCount: 2, angle: 45);
config.FastAngle = true;  // 빠른 각도 회전 (정확도 약간 낮음)

var result = MuraEx.InspectEx<MuraDefect>(mat, new[] { config })
    .GetDefectList();
```

---

## 실전 예제

### 예제 1: 디스플레이 패널 얼룩 검사

```csharp
using Ratel;
using Ratel.Vision;
using Ratel.Vision.Mura;
using OpenCvSharp;

class Program
{
    static void Main()
    {
        // 초기화
        RatelLib.InitLib();
        Thread.Sleep(1000);  // 라이브러리 초기화 대기

        // 이미지 로드
        var mat = new Mat("panel.png", ImreadModes.Grayscale);

        // 여러 크기의 얼룩 검출 설정
        var configs = new List<MuraConfig>
        {
            // 작은 점 결함
            MuraConfig.MakeCir4Config(32, 32,
                blackFactor: 5, whiteFactor: 5, diffCount: 2),

            // 중간 크기 얼룩
            MuraConfig.MakeCir4Config(128, 128,
                blackFactor: 3, whiteFactor: 3, diffCount: 2),

            // 큰 영역 얼룩
            MuraConfig.MakeCir4Config(512, 512,
                blackFactor: 1, whiteFactor: 1, diffCount: 3)
        };

        // 검사 실행
        var result = MuraEx.InspectEx<MuraDefect>(
            mat,
            configs,
            maxDefect: 100000,
            threadType: ThreadType.Multi
        ).GetDefectList();

        // 결과 출력 및 저장
        Console.WriteLine($"총 {result.Count}개 불량 검출");

        // White/Black 별로 분류
        var whiteDefects = result.Where(d => d.WhiteBlack == WB.White).ToList();
        var blackDefects = result.Where(d => d.WhiteBlack == WB.Black).ToList();

        Console.WriteLine($"- 밝은 불량: {whiteDefects.Count}개");
        Console.WriteLine($"- 어두운 불량: {blackDefects.Count}개");

        // 결과 이미지 생성
        SaveResultImage(mat, result, "panel_result.png");
    }

    static void SaveResultImage(Mat mat, List<MuraDefect> defects, string filename)
    {
        var resultMat = mat.CvtColor(ColorConversionCodes.GRAY2BGR);

        foreach (var defect in defects)
        {
            // 불량 타입에 따라 색상 선택
            var color = defect.WhiteBlack == WB.White
                ? new Scalar(0, 0, 255)    // 빨강: White 불량
                : new Scalar(255, 0, 0);   // 파랑: Black 불량

            resultMat.Rectangle(defect.Rectangle.ToCV(), color, 2);

            // 불량 레벨 표시 (선택)
            var text = $"{defect.DefectLevel:F1}";
            resultMat.PutText(text,
                new Point(defect.Rectangle.X, defect.Rectangle.Y - 5),
                HersheyFonts.HersheySimplex, 0.5, color, 1);
        }

        resultMat.SaveImage(filename);
        Console.WriteLine($"결과 저장: {filename}");
    }
}
```

### 예제 2: 라인 스크래치 검출

```csharp
static void DetectLineScratches()
{
    var mat = new Mat("film.png", ImreadModes.Grayscale);

    var configs = new List<MuraConfig>
    {
        // 수평 스크래치 (긴 가로선)
        MuraConfig.MakeHoriConfig(
            sizeX: 200,
            sizeY: 5,
            blackFactor: 3,
            whiteFactor: 3,
            diffCount: 2
        ) { MergeGroup = "Horizontal" },

        // 수직 스크래치 (긴 세로선)
        MuraConfig.MakeVertConfig(
            sizeX: 5,
            sizeY: 200,
            blackFactor: 3,
            whiteFactor: 3,
            diffCount: 2
        ) { MergeGroup = "Vertical" }
    };

    var resultDict = MuraEx.InspectEx<MuraDefect>(
        mat,
        configs,
        threadType: ThreadType.Multi
    );

    Console.WriteLine($"수평 스크래치: {resultDict["Horizontal"].Count}개");
    Console.WriteLine($"수직 스크래치: {resultDict["Vertical"].Count}개");
}
```

### 예제 3: Edge 영역 보정 검사

이미지 가장자리는 주변 픽셀이 부족하여 오검출이 발생할 수 있습니다.

```csharp
var config = MuraConfig.MakeCir4Config(64, 64, 5, 5, diffCount: 2);

// Edge 영역 보정 설정
config.EdgeDiffCount = 3;           // 가장자리에서는 더 엄격한 기준 (3방향 이상)
config.EdgeUseDefectLevel = true;   // 가장자리에서 DefectLevel 사용
config.EdgeOffset = 10;             // 가장자리로부터 10픽셀 이내

var result = MuraEx.InspectEx<MuraDefect>(mat, new[] { config })
    .GetDefectList();
```

---

## 파라미터 튜닝 가이드

### BlackFactor / WhiteFactor

- **값이 작을수록**: 민감도 높음 (작은 차이도 검출) → 과검출 위험
- **값이 클수록**: 민감도 낮음 (큰 차이만 검출) → 미검출 위험
- **권장값**: 3 ~ 10 (이미지 품질에 따라 조정)

### DiffCount

- **값이 작을수록**: 느슨한 판정 (노이즈에 약함)
- **값이 클수록**: 엄격한 판정 (실제 불량만 검출)
- **권장값**:
  - 4방향 필터: 2 ~ 3
  - 8방향 필터: 3 ~ 5

### 필터 크기 (SizeX, SizeY)

- **작은 필터 (16x16 ~ 64x64)**: 작은 점 결함 검출
- **중간 필터 (64x64 ~ 256x256)**: 일반적인 얼룩 검출
- **큰 필터 (256x256 ~ 512x512)**: 넓은 영역의 불균일 검출

**팁**: 여러 크기의 필터를 동시에 사용하면 다양한 크기의 불량을 한 번에 검출할 수 있습니다.

### dpitch (Distance Pitch)

- **1.0**: 필터 크기만큼 떨어진 위치와 비교
- **0.8**: 필터 크기의 80% 거리
- **1.2**: 필터 크기의 120% 거리
- **권장값**: 0.8 ~ 1.2

---

## WPF 이미지 뷰어 사용

Ratel.Vision은 검사 결과를 표시할 수 있는 WPF 컨트롤을 제공합니다.

### RatelViewer

```xml
<Window xmlns:ratel="clr-namespace:Ratel.Vision;assembly=Ratel.Vision">
    <ratel:RatelViewer x:Name="imageViewer" />
</Window>
```

```csharp
// 이미지 표시
imageViewer.SetImage(mat);

// 불량 영역 오버레이
foreach (var defect in result)
{
    imageViewer.AddRectangle(defect.Rectangle, Colors.Red);
}
```

---

## 성능 최적화

### 1. 적절한 ThreadType 선택

```csharp
// 작은 이미지 (< 2048x2048): Single 또는 Multi
var smallResult = MuraEx.InspectEx<MuraDefect>(
    smallMat, configs, threadType: ThreadType.Multi);

// 큰 이미지 (> 4096x4096): Gpu 권장
var largeResult = MuraEx.InspectEx<MuraDefect>(
    largeMat, configs, threadType: ThreadType.Gpu);
```

### 2. maxThreadCount 조정

```csharp
// CPU 코어 수에 따라 조정 (보통 4~8)
var result = MuraEx.InspectEx<MuraDefect>(
    mat, configs,
    threadType: ThreadType.Multi,
    maxThreadCount: Environment.ProcessorCount
);
```

### 3. 불필요한 설정 비활성화

```csharp
config.Use = false;  // 사용하지 않는 설정은 비활성화
```

---

## 문제 해결

### Q: 검사 결과가 너무 많아요 (과검출)

**A**: 다음을 시도해보세요:
1. `BlackFactor` / `WhiteFactor` 값을 증가 (예: 5 → 10)
2. `DiffCount` 값을 증가 (예: 2 → 3)
3. 필터 크기를 증가 (예: 32 → 64)

### Q: 불량이 검출되지 않아요 (미검출)

**A**: 다음을 시도해보세요:
1. `BlackFactor` / `WhiteFactor` 값을 감소 (예: 10 → 5)
2. `DiffCount` 값을 감소 (예: 3 → 2)
3. 여러 크기의 필터를 동시에 사용
4. 이미지 전처리 적용 (히스토그램 평활화 등)

### Q: GPU 모드가 작동하지 않아요

**A**:
1. CUDA Toolkit이 설치되어 있는지 확인
2. `Ratel.GPU.Cuda.Init` 상태 확인
3. NVIDIA GPU 드라이버 최신 버전 확인
4. CPU 모드로 대체 사용: `threadType: ThreadType.Multi`

### Q: 가장자리에서 오검출이 많아요

**A**: Edge 보정 옵션을 사용하세요:
```csharp
config.EdgeDiffCount = 3;
config.EdgeUseDefectLevel = true;
```

---

## API 레퍼런스

전체 API 문서는 [docs](./docs) 폴더를 참조하세요.

### 주요 네임스페이스

- `Ratel` - 라이브러리 초기화 및 공통 유틸리티
- `Ratel.Vision` - 이미지 처리 및 뷰어 컴포넌트
- `Ratel.Vision.Mura` - 얼룩 검사 알고리즘
- `Ratel.GPU` - GPU 가속 기능

---

## 라이센스 및 인증

Ratel Library는 하드웨어 동글 기반 라이센스 시스템을 사용합니다.

---

## 개발 환경

|항목|사양|비고|
|----|----|----|
|개발 언어|C# 10.0 or higher||
|개발 Platform|.NET 6.0 / .NET 8.0||
|OS|Windows 10 or higher||
|개발 Tool|Visual Studio 2022 or higher||
|이미지 처리|OpenCvSharp4||
|GPU 가속|CUDA Toolkit 12.0+|선택 사항|

---

## 변경 이력

### v25.1220.2 (2024-12-20)
- Library nuget 배포 시작

---

## 지원 및 문의

- **이슈 리포트**: [GitHub Issues](https://github.com/ratelsoft-com/dist-lib/issues)
- **리포지토리**: https://github.com/ratelsoft-com/dist-lib.git

---

**Copyright © 2024 RatelSoft. All rights reserved.**
