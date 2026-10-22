# Mura 검사 API 매뉴얼 (개발자용)

`RatelSoft.Vision`의 얼룩(Mura) 검사 라이브러리를 코드에서 사용하는 방법을 설명한다. 검사 원리와 파라미터 튜닝은 `MURA_USER_MANUAL.md`를 먼저 읽는다. 이 문서는 타입, 메서드, 입력 규칙, 반환값, 예외를 기준으로 쓰였다.

기준 버전: `RatelSoft.Vision` / `RatelSoft.Types` 2.26.x (2026-10-22 정리 반영)

## 1. 패키지와 네임스페이스

| 패키지 | 네임스페이스 | 내용 |
| --- | --- | --- |
| `RatelSoft.Types` | `RatelSoft.Vision.Mura` | `MuraConfig`, `SurConfig`, `CompareOption`, `AverageMethod`, `ThreadType`, `WB`, `MuraDefect`, `MuraUnitDefect`, `MuraFilterResult`, `ScratchParams`, `SubRect`(`RatelSoft.Types`) |
| `RatelSoft.Vision` | `RatelSoft.Vision.Mura` | `MuraEx`, `MuraInspectOptions<T>`, `ScratchAlgo` |
| `RatelSoft.Vision` | `RatelSoft.Vision.GPU` | `Cuda`, `GpuMat` |
| `RatelSoft.Vision` | `RatelSoft.Vision` | `RotateSizeException` |
| `RatelSoft.Vision.Wpf` | `RatelSoft.Vision.Wpf.Mura` | `MuraConfigEditWindow` |

```csharp
global using RatelSoft.Vision.Mura;
global using RatelSoft.Vision.GPU;
using OpenCvSharp;
```

설치:

```bash
dotnet add package RatelSoft.Vision
```

`RatelSoft.Types`, `RatelSoft.Lib`, `RatelSoft.Common`, `OpenCvSharp4`가 함께 설치된다. 대상 프레임워크는 .NET 8.0, Windows x64다. 패키지 피드 설정은 `README.md`를 참고한다.

## 2. 초기화

CPU 검사는 별도 초기화가 없다. GPU를 쓰려면 프로그램 시작 시 한 번 `Cuda.InitGPU()`를 호출한다.

```csharp
bool gpuReady = Cuda.InitGPU();          // deviceNo = 0
```

| 인자 | 기본값 | 의미 |
| --- | --- | --- |
| `deviceNo` | 0 | 사용할 CUDA 장치 번호 |
| `test` | false | true면 `c:\tmp\CudaMura.dll`도 탐색 후보에 넣는다(개발용) |

반환값이 false이면 GPU를 쓸 수 없다. 실패 원인은 로그(`RatelLog` 소스 `Cuda`)에 남는다. 성공 여부는 `Cuda.Init`으로도 읽을 수 있다. 런타임 배치는 8절을 본다.

로깅은 `RatelSoft.Common`의 `RatelLogger`를 쓴다. 검사 메서드에 `log` 인자를 넘기면 그 로거로, 넘기지 않으면 라이브러리 내부 로거(소스 `Mura`)로 기록된다.

```csharp
static readonly RatelLogger log = RatelLog.ForSource("MyInspection");
```

## 3. MuraConfig

검사 필터 하나의 설정이다. `INotifyPropertyChanged`를 구현하고 MessagePack(`[MessagePackObject(true)]`)과 System.Text.Json 직렬화를 지원한다.

### 3-1. 생성 메서드

```csharp
public static MuraConfig MakeCir4Config(int cx, int cy, double blackFactor, double whiteFactor,
    double angle = 0, double dpitch = 1, double minLevelPercent = 1.0, int diffCount = 2, bool fastAngle = false);
public static MuraConfig MakeCir8Config(int sizeX, int sizeY, double blackFactor, double whiteFactor,
    double angle = 0, double dpitch = 1, double minLevelPercent = 1.0, int diffCount = 2, bool fastAngle = false);
public static MuraConfig MakeHoriConfig(int cx, int cy, double blackFactor, double whiteFactor,
    double angle = 0, double dpitch = 1, double minLevelPercent = 1.0, int diffCount = 2, bool fastAngle = false);
public static MuraConfig MakeVertConfig(int cx, int cy, double blackFactor, double whiteFactor,
    double angle = 0, double dpitch = 1, double minLevelPercent = 1.0, int diffCount = 2, bool fastAngle = false);
```

제네릭 오버로드 `MakeCir4Config<T>() where T : MuraConfig, new()` 등으로 파생 설정 타입을 만들 수 있다.

| 메서드 | `DefectPoints` | `SurPoints` | `MergeGroup` |
| --- | --- | --- | --- |
| `MakeCir4Config` | (0,0) | (0,−d), (0,d), (−d,0), (d,0) 모두 `Count` | "Cir" |
| `MakeCir8Config` | (0,0) | 상하좌우 4개 `Count` + 대각 4개 `NoCount` | "Cir" |
| `MakeHoriConfig` | (0,0) | (0,−d), (0,d) `Count` | "Hori" |
| `MakeVertConfig` | (0,0) | (−d,0), (d,0) `Count` | "Verti" |

`d`는 `dpitch`다. `MakeCir4Config`는 `dpitch >= StepX`(기본 5)이면 `dpitch / StepX`로, 나머지는 `dpitch >= 4`이면 정수로 내림한다. `minLevelPercent`는 각 `SurConfig.Percent`에 들어간다. `fastAngle`은 무시된다.

### 3-2. 속성

| 속성 | 타입 | 기본값 | 의미 |
| --- | --- | --- | --- |
| `Use` | bool | true | false면 검사에서 제외. 결과 딕셔너리에는 그룹 키만 빈 리스트로 생긴다 |
| `FilterName` | string | "NoName" | 결과 `MuraDefect.FilterName`에 복사된다. 설정 경고 로그의 키로도 쓰인다 |
| `SizeX`, `SizeY` | int | 32 | 필터 크기(픽셀) |
| `AverageMethod` | `AverageMethod` | Rect | `Rect`(박스 평균) 또는 `Gaussian` |
| `SigmaX`, `SigmaY` | double | 0 | Gaussian 표준편차. 0이면 OpenCV 자동값 |
| `StepX`, `StepY` | int | 5 | 필터를 나누는 칸 수. 비교 이동 간격 = `SizeX / StepX` |
| `DiffCount` | int | 2 | 조건을 만족해야 하는 주변점 수. `SurPoints.Count`보다 크면 검사 시 그 수로 클램프된다(설정 객체는 바뀌지 않음) |
| `BlackFactor` | double | 5 | 어두운 얼룩 임계 레벨(8비트 단위). 0이면 Black 검사 생략 |
| `WhiteFactor` | double | 5 | 밝은 얼룩 임계 레벨. 0이면 White 검사 생략 |
| `MinArea` | int | 1 | 최종 결과로 인정할 최소 `UnitDefectCount` |
| `Angle` | double | 0 | 필터 회전 각도(도) |
| `MergeGroup` | string | "Group1" | 결과 딕셔너리의 키 |
| `SubRect` | `SubRect` | 비어 있음 | 이 설정만의 검사 영역. 마스크와 AND된다 |
| `DefectPoints` | `List<SurConfig>` | 빈 목록 | 중심 블록 위치. 보통 (0,0) 하나 |
| `SurPoints` | `List<SurConfig>` | 빈 목록 | 주변 블록 위치와 비교 옵션 |
| `EdgeDiffCount` | int | −1 | 주변점 일부가 영상/마스크 밖일 때 쓸 DiffCount. −1이면 Edge 처리 없음 |
| `EdgeOffset` | double | 0 | Edge 처리 시 Factor에 더할 값 |
| `EdgeOffset2` | double | −1 | 유효 주변점이 `EdgeDiffCount` 미만일 때 Factor에 더할 값. −1이면 사용 안 함 |
| `EdgeUseDefectLevel` | bool | true | 무효 주변점을 중심값으로 치환할지 여부 |
| `Tag` | object | null | 사용자 데이터. 직렬화되지 않는다고 가정하지 말 것 |
| `PitchFactor`, `Level`, `FastAngle` | | | 호환용. 검사에 영향 없음. `FastAngle`은 `[Obsolete]` |

메서드: `Clone()`(속성 복사 새 객체), `CopyFrom(MuraConfig)`, `CopytTo(MuraConfig)`, `WeakCopy()`(MemberwiseClone, 리스트 공유), `UpdateData()`(모든 속성의 `PropertyChanged` 발생), `GetDefectPointsBox(x, y)`.

### 3-3. SurConfig와 CompareOption

```csharp
public SurConfig(double x, double y, CompareOption option = CompareOption.Count, double percent = 1, double diffValue = 0);
```

| 속성 | 의미 |
| --- | --- |
| `X`, `Y` | 중심 기준 위치. 단위는 필터 크기 배수(내부적으로 `X * StepX` 칸) |
| `CompareOption` | 아래 표 |
| `Percent` | 이 점의 임계 배율. 차이가 `Factor × Percent` 이상이어야 조건 만족 |
| `DiffValue`, `Level`, `Desc` | 예약. 검사에서 읽지 않음 |

| `CompareOption` | DiffCount 카운트 | 주변 평균 | 실패 조건 |
| --- | --- | --- | --- |
| `Count` | 포함 | 포함 | 없음 |
| `NoCount` | 제외 | 포함 | 없음 |
| `MustCount` | 포함 | 포함 | 차이 < 임계 |
| `MustNoCount` | 제외 | 제외 | 차이 < 임계 |
| `MustNot` | 제외 | 제외 | 차이 ≥ 임계 |

`Must*` 옵션은 모든 주변점이 영상과 마스크 안에 있는 위치에서만 통과할 수 있다. `EdgeUseDefectLevel = true`(기본)에서는 밖에 있는 주변점이 중심값으로 치환되어 차이 0으로 `Must` 조건이 실패한다. 따라서 `Must*`와 Edge 옵션(`EdgeDiffCount`, `EdgeOffset`, `EdgeOffset2`, `EdgeUseDefectLevel = true`)을 함께 설정하면 검사 진입 시 Warn 로그가 남는다(필터 이름당 프로세스에서 한 번).

### 3-4. SubRect

```csharp
public SubRect(int l = 0, int t = 0, int r = 0, int b = 0, bool inverse = false);
public SubRect(Rectangle rect, bool inverse = false);
```

좌/상/우/하 여백으로 영역을 정한다. `Inverse = true`면 그 영역을 제외한다. `IsEmpty`가 true(모두 0)면 적용하지 않는다. 적용 시 마스크를 복사해 영역 밖(또는 안)을 0으로 만든 뒤 검사하므로 원래 마스크는 바뀌지 않는다.

## 4. 검사 실행: MuraEx

### 4-1. InspectEx (CPU, 또는 GPU 자동 전환)

```csharp
public static Dictionary<string, List<T>> InspectEx<T>(
    Mat mat,
    IEnumerable<MuraConfig> config,
    int maxDefect = 10000,
    Mat mask = null,
    ThreadType threadType = ThreadType.Multi,
    int maxThreadCount = 4,
    MuraInspectOptions<T> muraOptions = null,
    RatelLogger? log = null)
    where T : MuraDefect, new();
```

| 인자 | 설명 |
| --- | --- |
| `mat` | 검사 영상. 아래 입력 규칙 참조. null이거나 비어 있으면 예외 |
| `config` | 설정 목록. `Use == false`인 항목은 건너뛴다 |
| `maxDefect` | 필터 하나의 단위 검출 상한. 넘치면 행 단위로 중단되어 결과가 잘린다. 과검출 시 메모리 보호용 |
| `mask` | `CV_8UC1`. 값이 255인 곳만 검사. null이면 전체 검사 |
| `threadType` | `Single`, `Multi`, `Gpu`. `Gpu`인데 `Cuda.Init == false`면 Warn 로그 후 `Multi`로 검사 |
| `maxThreadCount` | `Multi`일 때 동시에 처리할 설정 수. 각 설정 안에서 행 단위 병렬(최대 12)이 추가로 돈다 |
| `muraOptions` | 4-3절 |
| `log` | 로거. 설정 경고와 오류에 사용 |

반환: `MergeGroup`을 키로 하는 딕셔너리. 같은 그룹의 설정 결과는 한 리스트에 합쳐진다(이 단계에서 서로 병합되지는 않는다).

입력 규칙:

| 입력 | 처리 |
| --- | --- |
| 3채널 / 4채널 | 그레이로 변환(BGR2GRAY / BGRA2GRAY) |
| 그 외 다채널 | 예외 "Mat의 채널이 1이 아님" |
| `CV_8UC1` | 32F로 변환. 값 범위 0~255 |
| `CV_16UC1` | 32F로 변환하며 1/256 곱함. 0~255 범위로 환산되므로 Factor는 8비트 기준으로 설정 |
| `CV_32FC1` | 그대로 사용. 값 범위는 호출자 책임(8비트 레벨 단위로 맞출 것) |
| 그 외 | 예외 |

`Angle != 0`인 설정이 있고 영상의 폭 또는 높이가 30000을 넘으면 `RotateSizeException`.

입력 `mat`은 수정되지 않는다. 변환본은 내부에서 만든다.

### 4-2. InspectExGpu

```csharp
public static Dictionary<string, List<T>> InspectExGpu<T>(
    GpuMat gpuMat,
    IEnumerable<MuraConfig> config,
    int maxDefect = 10000,
    GpuMat gpuMask = null,
    MuraInspectOptions<T> scratchFilter = null,
    RatelLogger? log = null)
    where T : MuraDefect, new();
```

이미 GPU에 올린 영상을 반복 검사할 때 쓴다. `Cuda.Init == false`면 예외. 설정은 순차로 처리된다. `gpuMask`가 null이면 전체 255 마스크를 만든다. 입력 `GpuMat`의 수명은 호출자가 관리한다.

```csharp
using var gpuMat = new GpuMat();
gpuMat.Upload(mat);
using var gpuMask = new GpuMat(mask);
var groups = MuraEx.InspectExGpu<MuraDefect>(gpuMat, configs, gpuMask: gpuMask, log: log);
```

여러 스레드에서 GPU 검사를 동시에 호출하는 구조라면 실행 옵션을 정한다.

```csharp
MuraEx.SetGpuExecutionOptions(useLock: true, usePreprocessStream: false);   // 기본값, 가장 안전
var (useLock, useStream) = MuraEx.GetGpuExecutionOptions();
```

`useLock = true`면 GPU 전처리와 검사를 프로세스 전역 lock으로 직렬화한다. `false`는 장비와 드라이버에서 검증한 뒤 쓴다. `usePreprocessStream`은 네이티브 전처리에 CUDA 스트림을 쓸지 여부다.

### 4-3. MuraInspectOptions<T>

```csharp
public record MuraInspectOptions<T> where T : MuraDefect
{
    public Action<MuraInspectOptions<T>> FilterFunc { get; set; }
    public Mat Mat { get; set; }            // 콜백에 전달되는 전처리(회전) 영상, CPU
    public GpuMat GpuMat { get; set; }      // 콜백에 전달되는 전처리 영상, GPU
    public List<T> MuraDefects { get; set; } // 콜백에 전달되는 필터 결과
    public ThreadType ThreadType { get; set; }
    public bool UnitDefectRotate { get; set; } = false;
    public int MinLength { get; set; } = 20;      // 사용자 콜백용 값
    public int MinPercent { get; set; } = 30;
    public double Threshold { get; set; } = 10;
    [Obsolete] public int MaxCountForFastMerge { get; set; }   // 효과 없음
}
```

- `FilterFunc`: 설정 하나의 병합이 끝난 직후 호출된다. 인자로 받는 record에는 `Mat`(또는 `GpuMat`), `MuraDefects`, `ThreadType`이 채워져 있다. `MuraDefects` 리스트를 직접 수정(삭제, 속성 변경)하면 그것이 그 설정의 결과가 된다. `MinArea` 필터는 콜백 뒤에 적용된다.
- `UnitDefectRotate`: 각도 검사에서 `MuraDefect` 외에 단위 검출(`GetUnitDefects()`)의 좌표와 `Pts`까지 원본 좌표로 회전한다. 단위 검출을 그릴 때만 켠다.
- `MinLength`, `MinPercent`, `Threshold`: 라이브러리는 읽지 않는다. `FilterFunc` 구현에 전달하기 위한 값이다.

### 4-4. 결과 후처리

```csharp
public static List<T> GetDefectList<T>(this Dictionary<string, List<T>> dict);
public static Dictionary<string, List<T>> MergeRectGroup<T>(this Dictionary<string, List<T>> group) where T : MuraDefect, new();
public static List<T> MergeDefect<T>(List<T> defects) where T : MuraDefect;
```

- `GetDefectList`: 모든 그룹을 하나의 리스트로 합친다. 병합하지 않는다.
- `MergeRectGroup`: 그룹별로 `Rectangle`이 겹치는 결과를 하나로 합친다. 합쳐진 결과는 `DefectLevel`이 최대인 원소의 속성을 갖고, `Rectangle`/`OrgRect`/`Pts`는 합집합 사각형, 단위 검출은 모두 모인다. 각도는 고려하지 않는다. 판정은 입력 사각형 쌍의 교차로만 하므로 **결과 사각형끼리 겹칠 수 있다**(L자 결과의 외접 사각형 안에 독립 결과가 있는 경우). 겹침 없는 결과가 필요하면 결과 수가 줄지 않을 때까지 반복 호출한다.
- `MergeDefect`: 같은 `WhiteBlack`과 `MergeGroup`인 두 결과가 겹치면(각도 0: 사각형 교차, 각도 있음: 변 교차) 레벨이 낮은 쪽을 버린다. 각도 검사 결과용이다. 내부에서 `Tag`에 문자열을 쓰므로 `Tag`를 다른 용도로 쓴 결과에는 적용하지 말 것.

```csharp
var groups = MuraEx.InspectEx<MuraDefect>(mat, configs, mask: mask, log: log);
var merged = groups.MergeRectGroup();
var all = merged.GetDefectList();
```

### 4-5. 보조

```csharp
public static Mat MakeFilterImage(MuraConfig cfg);   // DefectPoints/SurPoints 배치를 그린 BGR Mat (편집 UI용)
```

## 5. 결과 타입

### 5-1. MuraDefect

| 속성 | 타입 | 의미 |
| --- | --- | --- |
| `Rectangle` | `System.Drawing.Rectangle` | 원본 영상 좌표의 외접 사각형. 각도 검사면 회전된 사각형의 외접 |
| `OrgRect` | `Rectangle` | 회전 전(회전 영상 좌표) 사각형. 각도 0이면 `Rectangle`과 같다 |
| `Pts` | `PointF[4]` | 네 꼭짓점. 각도 검사에서 실제 모양 |
| `Angle` | double | 설정의 각도 |
| `WhiteBlack` | `WB` | `White`, `Black`, `ND` |
| `DefectLevel` | double | 단위 검출 중 최대 레벨(중심 − 주변 평균) |
| `DefectMinLevel` | double | `min(단위 검출의 DefectMinLevel 최대값, DefectLevel)`. 가장 약한 경계의 차이 |
| `UnitDefectCount` | int | 묶인 단위 검출 수. 면적에 해당 |
| `FilterName`, `FilterSize` | string, `Size` | 검출한 설정의 이름과 크기 |
| `MergeGroup` | string | 그룹 |
| `DefectNo`, `Index` | int | 호출자가 번호를 매기는 용도. 라이브러리는 채우지 않는다 |
| `Tag` | object | 사용자 데이터. `MergeDefect`가 사용 |
| `FalseDefect` | bool | 호출자 용도 |
| `FilterInfo` | `List<FilterInfo>` | 호환용. 채워지지 않는다 |

메서드: `GetUnitDefects()`, `SetUnitDefects(list)`, `AddUnitDefect(d)`, `ClearUnitDefects()`, `NewUnitDefects()`, `FinalizeUnitDefects()`(`DefectMinLevel`, `UnitDefectCount` 재계산), `CopyTo(target, copyUnitDefects)`, `SetMaxFilterName()`, 정적 `FromMuraDefect<T>(d)`.

`T : MuraDefect, new()` 제약 덕분에 파생 타입을 결과로 받을 수 있다. 라이브러리는 `new T()`로 만들고 `MuraDefect` 속성만 채운다.

```csharp
public class MyDefect : MuraDefect
{
    public string ClassName { get; set; }
}
var groups = MuraEx.InspectEx<MyDefect>(mat, configs);
```

### 5-2. MuraUnitDefect

판정을 통과한 칸 하나의 정보다. `Rectangle`(원본 좌표, 크기는 `SizeX/StepX × SizeY/StepY`), `DefectLevel`, `DefectMinLevel`, `WhiteBlack`, `FilterName`, `FilterSize`, `MergeGroup`, `Angle`, `Pts`. 각도 검사에서는 `UnitDefectRotate = true`일 때만 회전된다.

### 5-3. 직렬화

`MuraConfig`, `SurConfig`, `MuraDefect`, `MuraUnitDefect`는 MessagePack(키 = 속성 이름)과 System.Text.Json으로 저장할 수 있다.

```csharp
var json = JsonSerializer.Serialize(configs, new JsonSerializerOptions { WriteIndented = true });
var loaded = JsonSerializer.Deserialize<List<MuraConfig>>(json);
```

`MuraDefect`의 단위 검출 목록은 비공개 속성이라 직렬화되지 않는다. 필요하면 `GetUnitDefects()`를 따로 저장한다.

## 6. 예외와 로그

| 상황 | 동작 |
| --- | --- |
| `mat`이 null이거나 비어 있음 | `Exception("Mat is null or empty")` |
| 지원하지 않는 채널 수 | `Exception("Mat의 채널이 1이 아님")` |
| 지원하지 않는 타입 | `Exception("Mura algorithm doesn't support ...")` |
| 각도 검사 영상 크기 초과 | `RotateSizeException` |
| `ThreadType.Gpu`인데 GPU 미초기화 | Warn 로그 후 CPU `Multi`로 검사 (`InspectEx`). `InspectExGpu`는 예외 |
| 이미 Dispose된 Mat/mask | `Exception("Disposed Mat")` 등 |
| `Must*` + Edge 옵션 동시 설정 | Warn 로그. 검사는 진행 |

검사 중 예외는 로그에 기록된 뒤 다시 던져진다. 호출자가 잡아야 한다.

## 7. 스크래치 검사

```csharp
public static List<T> ScratchAlgo.InspectScratch<T>(Mat mat, Mat mask, ScratchParams param, int[] angles = null)
    where T : MuraDefect, new();
```

`angles`가 null이면 `{0, 90, 30, 60, -30, -60}`을 병렬로 검사한다. 결과 `Angle`에 검사 각도가 들어간다. 밝은 스크래치만 검출한다. 입력은 `CV_8UC1` 또는 `CV_32FC1`, 마스크는 `CV_8UC1`(255 유효).

`ScratchParams`:

| 속성 | 기본값 | 의미 |
| --- | --- | --- |
| `AngleDeg` | 0 | 단일 검사 각도(`ScratchAlgo.Inspect<T>` 사용 시) |
| `Thickness` | 32 | 스크래치 최대 두께(픽셀). `VStep` 배수로 내림 처리 |
| `MinLength` | 100 | 최소 길이(픽셀). `HStep` 배수로 내림 처리 |
| `Level` | 5 | 상하 픽셀보다 밝아야 하는 레벨 |
| `MinCount` | 4 | 스크래치로 인정할 최소 연결 조각 수 |
| `HStep`, `VStep` | 4 | 길이/두께 방향 분해능 |

## 8. GPU 런타임 배치

GPU 검사는 네이티브 `CudaMura.dll`과 같은 폴더의 `opencv_world4100.dll`, NVIDIA 드라이버가 필요하다. `Cuda.InitGPU()`는 다음 순서로 DLL을 찾는다.

1. 환경 변수 `CUDA_MURA_DLL_PATH` (파일 경로 또는 폴더 경로)
2. 환경 변수가 없으면 `%ProgramFiles%\RatelSoft\CudaMura\current\CudaMura.dll`. 이 파일이 있으면 사용자 환경 변수에 자동 등록한다
3. `test = true`일 때 `c:\tmp\CudaMura.dll`
4. 실행 파일 폴더의 `CudaMura.dll`
5. 실행 파일 폴더의 `runtimes\win-x64\native\CudaMura.dll`

로드된 DLL의 파일 버전 major가 2~20 범위가 아니거나, 설치된 CUDA 런타임 버전이 DLL이 요구하는 버전보다 낮으면 실패한다. 로그에 `Cuda.InitGPU loaded(<source>): <path>`로 실제 로드 경로가 남는다. 설치 프로그램 `RatelSoft_CudaMura_Runtime_<version>.exe`는 2번 경로에 설치한다.

임시로 다른 DLL을 쓰려면:

```powershell
$env:CUDA_MURA_DLL_PATH = "D:\work\drop\CudaMura.dll"
```

## 9. WPF 설정 편집 창

`RatelSoft.Vision.Wpf`의 `MuraConfigEditWindow`는 `MuraConfig` 하나를 편집한다.

```csharp
using MuraConfigEditWindow = RatelSoft.Vision.Wpf.Mura.MuraConfigEditWindow;

var win = new MuraConfigEditWindow { MuraConfig = config.Clone(), Owner = this };
if (win.ShowDialog() == true)
{
    config.CopyFrom(win.MuraConfig);
}
```

| 속성 | 의미 |
| --- | --- |
| `MuraConfig` | 편집 대상. 복사본을 넘기고 OK일 때만 반영하는 패턴을 권장 |
| `CanNewFilter` | false면 "Make New" 영역을 숨긴다 |

창은 필터 종류 선택(Circle4/Circle8/Horizontal/Vertical), 비교 블록 배치 그림(`Draw`), 테스트 영상에 대한 단일 필터 검사(`Test`)를 제공한다.

## 10. 전체 예제

```csharp
using OpenCvSharp;
using RatelSoft.Common;
using RatelSoft.Vision.GPU;
using RatelSoft.Vision.Mura;

static readonly RatelLogger log = RatelLog.ForSource("MuraSample");

static MuraConfig Named(MuraConfig c, string name, string group)
{
    c.FilterName = name;
    c.MergeGroup = group;
    return c;
}

static void Run(string path)
{
    bool gpu = Cuda.InitGPU();

    using var mat = new Mat(path, ImreadModes.Grayscale);
    using var mask = new Mat(mat.Size(), MatType.CV_8UC1, Scalar.All(0));
    mask.Rectangle(new Rect(50, 50, mat.Width - 100, mat.Height - 100), Scalar.All(255), -1);

    var configs = new List<MuraConfig>
    {
        Named(MuraConfig.MakeCir4Config(32, 32, 8, 8, diffCount: 2), "Spot32", "Small"),
        Named(MuraConfig.MakeCir4Config(128, 128, 4, 4, diffCount: 2), "Mura128", "Middle"),
        Named(MuraConfig.MakeHoriConfig(512, 16, 3, 3, diffCount: 2), "HLine", "Line"),
    };

    var groups = MuraEx.InspectEx<MuraDefect>(
        mat, configs,
        maxDefect: 20000,
        mask: mask,
        threadType: gpu ? ThreadType.Gpu : ThreadType.Multi,
        maxThreadCount: Environment.ProcessorCount,
        log: log);

    var defects = groups.MergeRectGroup().GetDefectList();
    foreach (var d in defects.OrderByDescending(d => d.DefectLevel))
    {
        log.Info($"{d.MergeGroup}/{d.FilterName} {d.WhiteBlack} {d.Rectangle} level={d.DefectLevel:F1}/{d.DefectMinLevel:F1} area={d.UnitDefectCount}");
    }
}
```

## 11. 변경 이력 (API에 영향 있는 것)

- 2026-10-22: 16비트 입력 스케일 1/255 → 1/256. `DiffCount` 클램프가 설정 객체를 수정하지 않음. `Must*` + Edge 설정 경고 추가. `MergeRectGroup`이 공간형 병합 사용(결과 동일). `MuraInspectOptions.MaxCountForFastMerge` Obsolete. Obsolete `MergeGroup` 확장 메서드 제거.
- 2026-02: `CUDA_MURA_DLL_PATH`와 고정 런타임 경로, `SetGpuExecutionOptions` 추가.
- 2026-01: 네임스페이스 `Ratel.Vision.Mura` → `RatelSoft.Vision.Mura`, 로거 `NLog.Logger` → `RatelLogger`.

## 12. 관련 문서

- `MURA_USER_MANUAL.md`: 검사 원리, 파라미터 의미, 튜닝
- `README_PRACTICAL.md`: 소비자 프로젝트 구성 템플릿
- `RatelSoft_CODEBOOK_RatelViewer.md`: 결과 표시용 뷰어
