# Mura 검사 사용자 매뉴얼

`RatelSoft.Vision`의 얼룩(Mura) 검사 라이브러리를 사용하는 방법을 설명한다.
이 문서는 검사 레시피를 만들고 튜닝하는 사용자 관점에서 쓰였다. 검사 원리, 각 파라미터의 의미, 파라미터 설정과 튜닝 방법을 중심으로 다루며, 내부 구현은 다루지 않는다.

기준 버전: `RatelSoft.Vision` / `RatelSoft.Types` 2.26.x

## 1. 개요

### 1-1. 얼룩(Mura) 검사란

얼룩은 디스플레이 패널, 필름, 유리, 시트처럼 균일해야 하는 면에서 주변보다 조금 밝거나 어두운 영역이 넓게 퍼져 있는 결함이다. 점 결함처럼 경계가 뚜렷하지 않고, 밝기 차이가 몇 레벨에 불과한 경우가 많아 단순한 밝기 임계값으로는 찾기 어렵다.

이 라이브러리는 **일정 크기의 창(필터)으로 영상을 뭉개서 본 뒤, 창의 중심과 주변 창의 평균 밝기를 비교**하는 방식으로 얼룩을 찾는다. 사람이 멀리서 패널을 보며 "이 부분이 주변보다 조금 어둡다"라고 판단하는 과정을 수치화한 것이라고 생각하면 된다.

### 1-2. 라이브러리가 제공하는 것

- 밝은 얼룩(White)과 어두운 얼룩(Black)을 동시에 검출한다.
- 필터 크기를 바꿔 작은 얼룩부터 넓은 불균일까지 검출한다. 여러 크기의 필터를 한 번에 돌릴 수 있다.
- 수평/수직 라인성 얼룩, 임의 각도의 얼룩을 검출한다.
- 마스크로 검사 영역을 제한한다.
- CPU 멀티스레드와 GPU(CUDA) 실행을 지원한다.
- 검출 결과는 위치(사각형), 밝기 차이(레벨), 종류(White/Black)로 제공된다.

### 1-3. 패키지 구성

| 패키지 | 역할 |
| --- | --- |
| `RatelSoft.Types` | `MuraConfig`(검사 설정), `MuraDefect`(검사 결과) 등 데이터 타입 |
| `RatelSoft.Vision` | 검사 실행(`MuraEx`), GPU 초기화(`Cuda`), 스크래치 검사 |
| `RatelSoft.Vision.Wpf` | 필터 설정 편집 창(`MuraConfigEditWindow`), 이미지 뷰어 |
| CudaMura 런타임 | GPU 실행에 필요한 네이티브 DLL. GPU를 쓸 때만 설치한다 |

네임스페이스는 `RatelSoft.Vision.Mura`(검사와 설정 타입), `RatelSoft.Vision.GPU`(GPU)이다.

## 2. 검사 원리

파라미터를 올바르게 설정하려면 검사가 어떤 순서로 진행되는지 알아야 한다. 아래는 사용자 관점에서 본 동작이다.

### 2-1. 블록 평균 영상

검사는 원본 픽셀을 하나씩 보지 않는다. 먼저 필터 크기(`SizeX` × `SizeY`)로 영상을 평균한다. 32×32 필터라면 32×32 픽셀 안의 평균 밝기가 하나의 값이 된다. 이 과정에서 픽셀 노이즈와 작은 점 결함은 사라지고, 필터 크기 정도의 밝기 변화만 남는다.

그래서 **필터 크기는 "찾고 싶은 얼룩의 크기"**로 정한다. 얼룩보다 훨씬 작은 필터는 얼룩 내부를 평탄한 영역으로 보고, 얼룩보다 훨씬 큰 필터는 얼룩을 평균에 묻어 버린다.

평균을 내는 방식은 `AverageMethod`로 고른다.

- `Rect`: 창 안을 똑같은 가중치로 평균한다. 빠르다. 기본값이다.
- `Gaussian`: 창 중심에 가중치를 둔다. 경계가 부드러운 얼룩에 유리하다.

### 2-2. 비교 간격(Step)

평균 영상은 필터 크기만큼 성큼성큼 이동하며 만들지 않는다. 필터를 `StepX` × `StepY`로 나눈 작은 칸 단위로 이동한다. 기본값 5이면 32×32 필터는 약 6.4픽셀 간격으로 평균값을 만든다. 따라서 얼룩이 필터 격자에 딱 맞지 않아도 검출된다.

Step 값은 보통 기본값을 그대로 둔다. 값을 키우면 더 촘촘하게 검사하지만 검사 시간과 결과 개수가 늘어난다.

### 2-3. 중심과 주변의 비교

평균 영상의 각 위치에서 **중심 블록**과 **주변 블록**의 밝기를 비교한다. 주변 블록을 어디에 둘지는 필터 종류가 결정한다.

```
Circle4 (4방향)          Circle8 (8방향)          Horizontal         Vertical
      [S]                 [s] [S] [s]                [S]
  [S] [C] [S]             [S] [C] [S]                [C]            [S] [C] [S]
      [S]                 [s] [S] [s]                [S]

C: 중심 블록   S: 판정에 참여하는 주변 블록   s: 평균 계산에만 참여하는 주변 블록
```

- `Circle4`: 상하좌우 4곳과 비교한다. 가장 일반적인 얼룩용이다.
- `Circle8`: 상하좌우 4곳으로 판정하고, 대각선 4곳은 주변 평균에만 넣는다. 주변 평균이 안정적이라 레벨 값이 덜 흔들린다.
- `Horizontal`: 위아래 2곳과만 비교한다. **가로로 긴** 얼룩이나 라인을 찾는다. 필터를 가로로 길게(`SizeX` 크게) 만든다.
- `Vertical`: 좌우 2곳과만 비교한다. **세로로 긴** 얼룩용이다. 필터를 세로로 길게 만든다.

주변 블록까지의 거리는 `dpitch`로 정한다. 1이면 중심 바로 옆 블록(필터 한 칸 거리)이다.

### 2-4. 판정 규칙

중심 밝기에서 주변 밝기를 뺀 값이 **차이**다.

- 차이가 `WhiteFactor` 이상인 주변 블록 수가 `DiffCount` 이상이면 **White(밝은) 얼룩** 후보다.
- 주변 밝기에서 중심 밝기를 뺀 값이 `BlackFactor` 이상인 주변 블록 수가 `DiffCount` 이상이면 **Black(어두운) 얼룩** 후보다.

예를 들어 `Circle4`, `WhiteFactor = 5`, `DiffCount = 2`이면 "4방향 중 2방향 이상에서 주변보다 5레벨 이상 밝다"가 조건이다.

밝기 단위는 8비트 영상 기준 0~255 레벨이다. 16비트 영상은 자동으로 8비트 범위로 환산되므로 Factor는 항상 8비트 기준으로 정한다.

`DiffCount`가 작으면 한쪽 경계만 있는 얼룩(예: 영상 가장자리나 패턴 경계에 붙은 얼룩)도 잡히지만 오검출도 늘어난다. `DiffCount`를 주변 블록 수와 같게 하면 사방이 모두 다른 "섬" 형태만 잡는다.

`DiffCount`가 주변 블록 수보다 크면 주변 블록 수로 검사한다. 모든 필터에 `DiffCount = 4`를 쓰면 `Circle4`는 4방향 모두, 주변 블록이 2개인 `Horizontal`/`Vertical`은 자동으로 2방향으로 검사된다. 레시피의 값은 바뀌지 않는다.

### 2-5. 불량 레벨

판정을 통과한 위치마다 두 가지 레벨이 계산된다.

- `DefectLevel`: 중심 밝기와 주변 평균 밝기의 차이. 얼룩이 얼마나 뚜렷한지 나타낸다.
- `DefectMinLevel`: 조건을 만족한 주변 블록 중 가장 작은 차이. 얼룩의 "가장 약한 쪽 경계"다.

등급 판정이나 재검 기준에는 보통 `DefectLevel`을 쓴다. 약한 경계까지 확실한 얼룩만 고르고 싶으면 `DefectMinLevel`을 함께 본다.

### 2-6. 병합과 최종 결과

판정을 통과한 위치는 칸 단위로 생기므로 얼룩 하나에서 수십~수천 개가 나온다. 라이브러리는 서로 닿아 있는 것들을 하나로 묶어 `MuraDefect` 하나로 만든다.

- `Rectangle`: 묶인 영역의 사각형(원본 영상 좌표).
- `WhiteBlack`: 묶음 안에서 더 강한 쪽(White/Black).
- `DefectLevel`: 묶음 안의 최대 레벨.
- `UnitDefectCount`: 묶인 칸 수. 얼룩의 면적에 해당한다.

`MinArea`보다 칸 수가 적은 묶음은 버린다. 노이즈성 작은 검출을 제거하는 가장 간단한 방법이다.

### 2-7. 여러 필터와 병합 그룹

`InspectEx`에는 설정을 여러 개 넘길 수 있다. 각 설정은 독립적으로 검사되고, 결과는 설정의 `MergeGroup` 이름별로 묶여 반환된다. 같은 그룹에 넣은 필터들의 결과는 한 리스트에 합쳐진다.

이 구조는 다음 용도로 쓴다.

- 크기별로 그룹을 나눠 "작은 얼룩", "큰 얼룩"을 따로 집계한다.
- 라인성 필터와 원형 필터를 그룹으로 분리해 결함 종류를 구분한다.
- 같은 그룹 안의 겹치는 결과를 `MergeRectGroup`으로 한 번 더 합친다.

### 2-8. 각도, 마스크, 부분 영역

- `Angle`: 필터를 회전시켜 검사한다. 비스듬한 라인이나 방향성 얼룩에 쓴다. 내부적으로 영상을 회전하므로 검사 시간이 늘어난다. 회전 후 영상 크기가 30000×30000을 넘으면 검사할 수 없다.
- 마스크: 흰색(255)인 곳만 검사한다. 패널 외곽, 마킹 영역, 정렬 마크를 제외할 때 쓴다. 마스크 밖과 접한 위치는 주변 블록이 일부 없는 상태로 판정된다(2-9 참고).
- `SubRect`: 설정 단위로 검사 영역을 좌/상/우/하 여백으로 제한한다. `Inverse`를 켜면 그 영역을 제외한다. 마스크를 만들지 않고 필터별로 영역을 다르게 하고 싶을 때 편하다.

### 2-9. 가장자리 처리

영상 가장자리나 마스크 경계에서는 주변 블록 일부가 영상 밖에 있다. 이런 위치는 기본적으로 **있는 주변 블록만으로** 판정한다. 4방향 필터가 모서리에 오면 2방향만 비교하므로 `DiffCount = 2`이면 통과하기 쉬워진다. 가장자리 오검출이 많으면 3-4절의 Edge 파라미터로 기준을 바꾼다.

## 3. 파라미터 레퍼런스

### 3-1. 필터 생성 메서드

가장 간단한 설정 방법은 `MuraConfig`의 정적 메서드를 쓰는 것이다. 메서드가 필터 종류에 맞는 비교 블록 배치를 만들어 준다.

```csharp
MuraConfig.MakeCir4Config(cx, cy, blackFactor, whiteFactor, angle: 0, dpitch: 1, minLevelPercent: 1.0, diffCount: 2);
MuraConfig.MakeCir8Config(sizeX, sizeY, blackFactor, whiteFactor, angle: 0, dpitch: 1, minLevelPercent: 1.0, diffCount: 2);
MuraConfig.MakeHoriConfig(cx, cy, blackFactor, whiteFactor, angle: 0, dpitch: 1, minLevelPercent: 1.0, diffCount: 2);
MuraConfig.MakeVertConfig(cx, cy, blackFactor, whiteFactor, angle: 0, dpitch: 1, minLevelPercent: 1.0, diffCount: 2);
```

| 인자 | 의미 | 권장 범위 |
| --- | --- | --- |
| `cx`, `cy` (`sizeX`, `sizeY`) | 필터 가로/세로 크기(픽셀) | 8 ~ 512. 찾을 얼룩 크기에 맞춘다 |
| `blackFactor` | 어두운 얼룩 임계 레벨. 0이면 Black 검사를 하지 않는다 | 2 ~ 15 |
| `whiteFactor` | 밝은 얼룩 임계 레벨. 0이면 White 검사를 하지 않는다 | 2 ~ 15 |
| `angle` | 필터 회전 각도(도, 시계 방향 양수) | 0이 기본. 필요할 때만 |
| `dpitch` | 주변 블록까지의 거리(필터 크기 배수) | 1이 기본. 0.5 ~ 2 |
| `minLevelPercent` | 주변 블록별 임계 배율(`Percent`) | 1.0이 기본 |
| `diffCount` | 조건을 만족해야 하는 주변 블록 수 | Circle4: 2~4, Circle8: 2~4, Hori/Vert: 1~2 |

메서드는 `MergeGroup`을 각각 `"Cir"`, `"Hori"`, `"Verti"`로 설정한다. 그룹을 나누려면 생성 후 `MergeGroup`을 바꾼다.

`dpitch`는 4 이상이면 정수로 내림된다(Circle4는 5 이상이면 5로 나눈 값). 특별한 이유가 없으면 1을 쓴다.

### 3-2. MuraConfig 속성

| 속성 | 기본값 | 의미 | 값을 키우면 |
| --- | --- | --- | --- |
| `Use` | true | 이 설정을 검사에 포함할지 여부 | - |
| `FilterName` | "NoName" | 설정 이름. 결과의 `FilterName`에 기록되어 어떤 필터가 잡았는지 알 수 있다 | - |
| `SizeX`, `SizeY` | 32 | 필터 크기(픽셀) | 더 큰 얼룩을 찾고, 작은 얼룩은 묻힌다 |
| `AverageMethod` | Rect | 블록 평균 방식(Rect/Gaussian) | - |
| `SigmaX`, `SigmaY` | 0 | Gaussian일 때 표준편차. 0이면 자동 | 평균이 더 넓게 퍼진다 |
| `StepX`, `StepY` | 5 | 필터를 몇 칸으로 나눠 이동할지 | 더 촘촘히 검사. 시간과 결과 수 증가 |
| `DiffCount` | 2 | 조건을 만족해야 하는 주변 블록 수 | 엄격해진다(오검출 감소, 미검출 증가) |
| `BlackFactor` | 5 | 어두운 얼룩 임계 레벨. 0이면 검사 안 함 | 둔감해진다 |
| `WhiteFactor` | 5 | 밝은 얼룩 임계 레벨. 0이면 검사 안 함 | 둔감해진다 |
| `MinArea` | 1 | 최종 불량으로 인정할 최소 칸 수 | 작은 검출이 제거된다 |
| `Angle` | 0 | 필터 회전 각도 | - |
| `MergeGroup` | "Group1" | 결과를 묶을 그룹 이름 | - |
| `SubRect` | 비어 있음 | 이 설정만의 검사 영역 여백(좌/상/우/하, Inverse) | - |
| `EdgeDiffCount` | -1 | 가장자리용 DiffCount. -1이면 가장자리 특별 처리 없음 | 3-4절 |
| `EdgeOffset` | 0 | 가장자리에서 Factor에 더할 값 | 3-4절 |
| `EdgeOffset2` | -1 | 주변 블록이 EdgeDiffCount보다 적을 때 Factor에 더할 값. -1이면 사용 안 함 | 3-4절 |
| `EdgeUseDefectLevel` | true | 비교할 수 없는 주변 블록을 중심값과 같다고 볼지 여부 | 3-4절 |
| `DefectPoints`, `SurPoints` | 생성 메서드가 채움 | 중심 블록과 주변 블록 배치. 직접 편집은 3-3절 | - |
| `Tag` | null | 사용자 임의 데이터 | - |

다음 속성은 호환성을 위해 남아 있을 뿐 검사에 영향이 없다: `PitchFactor`, `FastAngle`, `Level`.

### 3-3. 주변 블록 배치 직접 정의

생성 메서드가 만드는 배치로 부족하면 `DefectPoints`(중심)와 `SurPoints`(주변)를 직접 편집한다. 각 항목은 `SurConfig`이며, 좌표 단위는 **필터 크기 배수**다. `(0, -1)`은 중심에서 위로 필터 한 칸 떨어진 블록이다.

```csharp
var cfg = new MuraConfig { SizeX = 64, SizeY = 16, WhiteFactor = 4, BlackFactor = 4, DiffCount = 2 };
cfg.DefectPoints.Add(new SurConfig(0, 0));
cfg.SurPoints.Add(new SurConfig(0, -1.5));                        // 위, 1.5칸
cfg.SurPoints.Add(new SurConfig(0,  1.5));                        // 아래, 1.5칸
cfg.SurPoints.Add(new SurConfig(-1, 0, CompareOption.NoCount));   // 좌, 평균에만 사용
cfg.SurPoints.Add(new SurConfig( 1, 0, CompareOption.NoCount));   // 우, 평균에만 사용
```

`SurConfig`의 속성:

| 속성 | 의미 |
| --- | --- |
| `X`, `Y` | 중심 기준 위치(필터 크기 배수) |
| `CompareOption` | 이 블록이 판정에 참여하는 방식(아래 표) |
| `Percent` | 이 블록에만 적용할 임계 배율. Factor × Percent 이상 차이 나야 한다. 1이면 Factor 그대로 |

`CompareOption`:

| 값 | 판정 카운트 | 주변 평균 | 설명 |
| --- | --- | --- | --- |
| `Count` | 포함 | 포함 | 기본. 조건을 만족하면 DiffCount에 더한다 |
| `NoCount` | 제외 | 포함 | 레벨 계산의 기준에만 쓴다. Circle8의 대각선이 이 방식이다 |
| `MustCount` | 포함 | 포함 | 반드시 조건을 만족해야 한다. 아니면 그 위치는 불량이 아니다 |
| `MustNoCount` | 제외 | 제외 | 반드시 조건을 만족해야 하지만 레벨에는 쓰지 않는다 |
| `MustNot` | 제외 | 제외 | 반드시 조건을 만족하지 **않아야** 한다. 특정 방향으로는 차이가 없어야 하는 결함에 쓴다 |

`Must` 계열은 "위아래는 반드시 어둡고 좌우는 달라도 상관없다"처럼 방향별 조건을 줄 때 쓴다. `DiffCount`는 `Count`와 `MustCount` 블록만 센다.

`Must` 계열은 **모든 주변 블록이 영상과 마스크 안에 있는 위치에서만 판정된다.** 영상 가장자리나 마스크 경계처럼 `Must` 블록 하나라도 비교할 수 없는 위치는 검출되지 않는다. 따라서 가장자리 검출이 필요한 필터에는 `Must` 계열 대신 `Count`를 쓰고 `DiffCount`로 조건을 준다. `Must` 계열과 3-4절의 Edge 파라미터를 함께 설정하면 검사 시작 시 경고 로그가 남는다.

### 3-4. 가장자리(Edge) 파라미터

`EdgeDiffCount`를 -1이 아닌 값으로 두면, **주변 블록 일부가 영상이나 마스크 밖에 있는 위치**에서만 다음 규칙이 적용된다.

1. 유효한 주변 블록 수가 `EdgeDiffCount` 이상이면: `DiffCount`를 `EdgeDiffCount`로, Factor를 `Factor + EdgeOffset`으로 바꿔 판정한다.
2. 유효한 주변 블록 수가 `EdgeDiffCount`보다 적고 `EdgeOffset2`가 0 이상이면: `DiffCount`를 유효 블록 수로, Factor를 `Factor + EdgeOffset2`로 바꿔 판정한다.
3. 유효한 주변 블록 수가 `EdgeDiffCount`보다 적고 `EdgeOffset2`가 -1이면: 1과 같이 처리한다. 유효 블록 수가 `EdgeDiffCount`보다 적으므로 사실상 검출되지 않는다.

전형적인 설정:

- 가장자리를 더 엄격하게: `EdgeDiffCount = 2`, `EdgeOffset = 3` (가장자리에서는 3레벨 더 차이 나야 함).
- 가장자리에서도 꼭 잡아야 할 때: `EdgeDiffCount = 2`, `EdgeOffset = 0`, `EdgeOffset2 = 2` (모서리에서 1방향만 남아도 2레벨 더 엄격하게 판정).
- 가장자리 검출을 사실상 끄기: `EdgeDiffCount`를 주변 블록 수(Circle4면 4)로 둔다.

`EdgeUseDefectLevel`은 비교할 수 없는 주변 블록을 어떻게 볼지 정한다.

- true(기본): 없는 블록을 중심값과 같다고 가정한다. 주변 평균이 중심에 가까워져 `DefectLevel`이 보수적으로 낮게 나온다.
- false: 있는 블록만으로 평균을 낸다. 레벨은 정직하지만 블록 한두 개로 결정되어 흔들릴 수 있다.

Edge 파라미터(`EdgeDiffCount`, `EdgeOffset`, `EdgeOffset2`, `EdgeUseDefectLevel`)는 `Must` 계열 비교 옵션과 함께 쓰지 않는다. `Must` 블록은 가장자리에서 판정되지 않으므로 Edge 파라미터로 가장자리 기준을 조정해도 효과가 없다.

### 3-5. 검사 실행 옵션

`InspectEx`의 인자:

| 인자 | 기본값 | 의미 |
| --- | --- | --- |
| `mat` | - | 검사 영상. 8비트/16비트 그레이 또는 컬러(자동으로 그레이 변환) |
| `config` | - | `MuraConfig` 목록 |
| `maxDefect` | 10000 | 필터 하나가 반환할 최대 단위 검출 수. 넘치면 잘린다 |
| `mask` | null | 검사 영역 마스크(8비트, 255가 검사 영역). null이면 전체 |
| `threadType` | Multi | `Single`, `Multi`, `Gpu` |
| `maxThreadCount` | 4 | Multi일 때 동시에 처리할 설정 수 |
| `muraOptions` | null | 병합/후처리 옵션 |
| `log` | null | 로거 |

`MuraInspectOptions<T>`에서 사용자가 조정할 만한 값:

| 속성 | 기본값 | 의미 |
| --- | --- | --- |
| `UnitDefectRotate` | false | 각도 검사 시 단위 검출 좌표까지 원본 좌표로 회전할지 여부. 단위 검출을 그릴 때만 켠다 |
| `FilterFunc` | null | 필터별 결과에 적용할 사용자 후처리 콜백 |

`MaxCountForFastMerge`는 더 이상 사용되지 않는다(Obsolete). 설정해도 영향이 없다.

## 4. 파라미터 설정과 튜닝 방법

### 4-1. 튜닝 절차

1. **기준 영상을 확보한다.** 양품 영상 여러 장, 등급별 불량 영상 여러 장을 준비한다. 조명과 노출이 실제 장비와 같아야 한다. 양품 영상이 없으면 오검출 기준을 잡을 수 없다.
2. **필터 크기를 정한다.** 찾을 얼룩의 짧은 변 길이의 절반~같은 크기로 시작한다. 지름 100픽셀 얼룩이면 32~64. 얼룩 크기가 다양하면 32, 128, 512처럼 2~4배 간격으로 여러 필터를 둔다.
3. **필터 종류를 정한다.** 둥근 얼룩은 `Circle4`, 라인성은 `Horizontal`/`Vertical`, 레벨 안정성이 필요하면 `Circle8`.
4. **Factor를 정한다.** 처음에는 양품 영상에서 아무것도 검출되지 않는 가장 작은 값을 찾는다. `DiffCount = 2`로 두고 Factor를 3부터 올리며 양품 검출이 0이 되는 값을 찾은 뒤, 1~2레벨 여유를 더한다. 불량 영상에서 그 값으로 얼룩이 잡히는지 확인한다.
5. **DiffCount를 조정한다.** 양품에서 패턴 경계나 조명 구배 때문에 한쪽만 다른 위치가 잡히면 3 이상으로 올린다.
6. **MinArea를 정한다.** 결과의 `UnitDefectCount` 분포를 보고, 노이즈성 검출이 몰린 작은 값을 잘라낸다.
7. **가장자리를 본다.** 가장자리 오검출이 있으면 마스크로 제외하거나 3-4절의 Edge 파라미터를 쓴다.
8. **그룹과 병합을 정리한다.** 필터별 `MergeGroup`을 정하고, 겹치는 결과가 많으면 `MergeRectGroup`을 적용한다.
9. **전체 영상으로 검증한다.** 양품 오검출률과 불량 검출률을 함께 본다. 한쪽만 보고 조정하면 다른 쪽이 무너진다.

### 4-2. 증상별 조정

| 증상 | 조정 |
| --- | --- |
| 양품에서 검출이 많다(과검출) | Factor 올리기 → DiffCount 올리기 → MinArea 올리기 순으로 시도한다. 작은 필터의 Factor를 큰 필터보다 높게 둔다 |
| 뚜렷한 얼룩을 놓친다(미검출) | Factor 내리기. 얼룩이 필터보다 훨씬 크면 필터 크기를 키운다 |
| 얼룩 가운데는 잡히는데 경계만 잡힌다 | 필터가 얼룩보다 작다. 필터 크기를 키우거나 큰 필터를 추가한다 |
| 얼룩 하나가 여러 조각으로 나온다 | Step을 기본값(5)으로 되돌린다. 같은 그룹에 `MergeRectGroup`을 적용한다 |
| 조명 구배(한쪽이 점점 어두움)가 잡힌다 | DiffCount를 올린다. 구배는 한 방향으로만 차이가 나므로 양방향 조건에서 걸러진다 |
| 패턴 경계(블랙 매트릭스, 배선)가 잡힌다 | 필터 크기를 패턴 주기의 정수배로 맞추거나, 마스크로 제외한다 |
| 가장자리에서만 오검출된다 | 마스크로 제외하거나 Edge 파라미터로 가장자리 기준을 올린다 |
| 라인성 얼룩이 끊겨 나온다 | `Horizontal`/`Vertical` 필터의 긴 변을 더 길게 한다(예: 256 → 512) |
| 비스듬한 라인이 안 잡힌다 | `Angle`을 라인 각도에 맞춘다. 각도를 모르면 15도 간격으로 여러 설정을 둔다 |
| 검사 시간이 길다 | 필터 수를 줄인다. 각도 검사를 줄인다. `maxThreadCount`를 CPU 코어 수에 맞춘다. 대형 영상은 GPU를 쓴다 |

### 4-3. 레벨 값의 활용

`DefectLevel`은 등급 판정의 기준이 된다. Factor는 "이 값 미만은 보지 않겠다"는 하한이므로, Factor를 낮게 잡고 결과를 모두 받은 뒤 `DefectLevel`로 등급을 나누는 방식이 흔하다. 예를 들어 Factor 3으로 검사하고, 레벨 3~5는 경미, 5~8은 보통, 8 이상은 심각으로 분류한다.

이 방식은 Factor를 등급마다 바꿔 여러 번 검사하는 것보다 빠르고, 경계값 근처의 흔들림을 사후에 조정할 수 있다.

### 4-4. 여러 필터 구성 예

패널 전면 얼룩 검사에서 자주 쓰는 구성이다. 작은 필터일수록 노이즈에 민감하므로 Factor를 높게 둔다.

```csharp
static MuraConfig Named(MuraConfig c, string name, string group, int minArea = 1)
{
    c.FilterName = name;
    c.MergeGroup = group;
    c.MinArea = minArea;
    return c;
}

var configs = new List<MuraConfig>
{
    Named(MuraConfig.MakeCir4Config(32, 32, blackFactor: 8, whiteFactor: 8, diffCount: 2), "Spot32", "Small"),
    Named(MuraConfig.MakeCir4Config(128, 128, blackFactor: 4, whiteFactor: 4, diffCount: 2), "Mura128", "Middle"),
    Named(MuraConfig.MakeCir4Config(512, 512, blackFactor: 2, whiteFactor: 2, diffCount: 3), "Mura512", "Large", minArea: 4),
    Named(MuraConfig.MakeHoriConfig(512, 16, blackFactor: 3, whiteFactor: 3, diffCount: 2), "HLine", "Line"),
    Named(MuraConfig.MakeVertConfig(16, 512, blackFactor: 3, whiteFactor: 3, diffCount: 2), "VLine", "Line"),
};
```

### 4-5. 필터 편집 창으로 튜닝하기

`RatelSoft.Vision.Wpf`의 `MuraConfigEditWindow`는 설정 하나를 화면에서 편집하고 비교 블록 배치를 그림으로 확인하는 창이다. 필터 종류 버튼(Circle4/Circle8/Horizontal/Vertical)으로 배치를 만들고, 각 주변 블록의 `CompareOption`과 `Percent`를 표에서 바꿀 수 있다.

```csharp
using MuraConfigEditWindow = RatelSoft.Vision.Wpf.Mura.MuraConfigEditWindow;

var win = new MuraConfigEditWindow { MuraConfig = config.Clone() };
if (win.ShowDialog() == true)
{
    config.CopyFrom(win.MuraConfig);
}
```

### 4-6. 설정 저장

`MuraConfig`는 MessagePack과 JSON(System.Text.Json) 직렬화를 지원한다. 레시피 파일에는 설정 목록 전체를 저장하고, 불러온 뒤 `Use`로 개별 필터를 켜고 끄는 방식이 관리하기 쉽다.

## 5. 사용 방법

### 5-1. 설치

```bash
dotnet add package RatelSoft.Vision
```

`RatelSoft.Types`는 함께 설치된다. 편집 창과 뷰어가 필요하면 `RatelSoft.Vision.Wpf`를 추가한다. 패키지 피드 설정은 `README.md`를 참고한다.

요구 사항: .NET 8.0, Windows 10 이상, x64.

### 5-2. 기본 검사

```csharp
using OpenCvSharp;
using RatelSoft.Vision.Mura;

var mat = new Mat("panel.png", ImreadModes.Grayscale);

var config = MuraConfig.MakeCir4Config(64, 64, blackFactor: 5, whiteFactor: 5, diffCount: 2);
config.FilterName = "Mura64";

var groups = MuraEx.InspectEx<MuraDefect>(mat, new[] { config }, threadType: ThreadType.Multi);
var defects = groups.GetDefectList();

foreach (var d in defects)
{
    Console.WriteLine($"{d.FilterName} {d.WhiteBlack} rect={d.Rectangle} level={d.DefectLevel:F1} area={d.UnitDefectCount}");
}
```

`InspectEx`는 `MergeGroup` 이름을 키로 하는 딕셔너리를 반환한다. `GetDefectList()`는 모든 그룹을 한 리스트로 합친다. 그룹별로 보려면 `groups["Small"]`처럼 접근한다.

### 5-3. 결과 그리기

```csharp
var canvas = mat.CvtColor(ColorConversionCodes.GRAY2BGR);
foreach (var d in defects)
{
    var color = d.WhiteBlack == WB.White ? Scalar.Red : Scalar.Blue;
    var r = d.Rectangle;
    canvas.Rectangle(new Rect(r.X, r.Y, r.Width, r.Height), color, 2);
}
canvas.SaveImage("result.png");
```

각도 검사 결과는 `Rectangle`이 회전된 사각형의 외접 사각형이다. 실제 네 꼭짓점은 `Pts`에 있다.

### 5-4. 마스크 사용

```csharp
var mask = new Mat(mat.Size(), MatType.CV_8UC1, Scalar.All(0));
mask.Rectangle(new Rect(100, 100, mat.Width - 200, mat.Height - 200), Scalar.All(255), -1);

var groups = MuraEx.InspectEx<MuraDefect>(mat, configs, mask: mask);
```

### 5-5. 같은 그룹 결과 병합

필터 크기가 다른 설정을 같은 그룹에 넣으면 같은 얼룩이 두 번 나올 수 있다. 겹치는 사각형을 하나로 합치려면 `MergeRectGroup`을 쓴다. 합쳐진 결과는 가장 레벨이 높은 검출의 속성을 가진다.

```csharp
var merged = MuraEx.InspectEx<MuraDefect>(mat, configs).MergeRectGroup();
```

병합은 **입력 사각형끼리 겹치는지**로만 판단하고, 합쳐진 결과 사각형끼리는 다시 비교하지 않는다. 그래서 L자 얼룩의 외접 사각형 안에 독립된 작은 얼룩이 있는 경우처럼, 결과에 서로 겹치는 사각형이 남을 수 있다. 이것은 독립 결함을 따로 세기 위한 의도된 동작이다. 겹침 없는 결과가 필요하면 결과가 더 줄지 않을 때까지 `MergeRectGroup`을 반복 호출한다.

### 5-6. GPU 사용

GPU 실행에는 NVIDIA GPU, CUDA 런타임, CudaMura 런타임 DLL이 필요하다. 런타임 설치 프로그램(`RatelSoft_CudaMura_Runtime_<version>.exe`)은 별도로 제공된다.

```csharp
using RatelSoft.Vision.GPU;

bool gpuReady = Cuda.InitGPU();   // 프로그램 시작 시 한 번
var groups = MuraEx.InspectEx<MuraDefect>(mat, configs, threadType: ThreadType.Gpu);
```

`Cuda.InitGPU()`가 실패했거나 호출하지 않았으면 `ThreadType.Gpu`는 경고 로그를 남기고 자동으로 CPU 멀티스레드로 검사한다. GPU로 실제 검사되는지는 로그의 `Cuda.InitGPU loaded(...)` 줄로 확인한다.

런타임 DLL 위치는 환경 변수 `CUDA_MURA_DLL_PATH`로 바꿀 수 있다. 설정하지 않으면 `C:\Program Files\RatelSoft\CudaMura\current\CudaMura.dll`, 그다음 실행 파일 폴더 순으로 찾는다.

여러 스레드에서 동시에 GPU 검사를 호출하는 구조라면 `MuraEx.SetGpuExecutionOptions(useLock, usePreprocessStream)`로 직렬화 여부를 정한다. 기본값 `(true, false)`가 가장 안전하다. 다른 조합은 장비에서 검증한 뒤 쓴다.

### 5-7. 스크래치 검사

가늘고 긴 스크래치는 얼룩 필터보다 전용 스크래치 검사가 적합하다. `ScratchParams`로 두께(`Thickness`), 최소 길이(`MinLength`), 임계 레벨(`Level`)을 정하고 여러 각도를 한 번에 검사한다.

```csharp
var param = new ScratchParams { Thickness = 16, MinLength = 200, Level = 6, MinCount = 8 };
var scratches = ScratchAlgo.InspectScratch<MuraDefect>(mat, mask, param);   // 기본 각도: 0, 90, ±30, ±60
```

| 속성 | 기본값 | 의미 |
| --- | --- | --- |
| `Thickness` | 32 | 찾을 스크래치의 최대 두께(픽셀) |
| `MinLength` | 100 | 찾을 스크래치의 최소 길이(픽셀) |
| `Level` | 5 | 스크래치가 주변보다 밝아야 하는 레벨 |
| `MinCount` | 4 | 스크래치로 인정할 최소 연결 조각 수 |
| `HStep`, `VStep` | 4 | 길이/두께 방향 분해능. 작을수록 정밀하고 느리다 |

스크래치 검사는 밝은 스크래치를 찾는다. 어두운 스크래치는 영상을 반전해서 검사한다.

## 6. 문제 해결

| 문제 | 확인 |
| --- | --- |
| 예외: "Mat is null or empty" | 영상 로드 실패. 경로와 포맷을 확인한다 |
| 예외: "Mat의 채널이 1이 아님" | 2채널 등 지원하지 않는 영상. 그레이 또는 BGR/BGRA로 변환한다 |
| 예외: `RotateSizeException` | 각도 검사에서 회전 후 크기가 30000을 넘는다. 영상을 나누거나 축소한다 |
| 결과가 `maxDefect`에서 잘린다 | Factor가 너무 낮다. Factor를 올리거나 `maxDefect`를 키운다 |
| GPU 검사가 CPU로 실행된다 | `Cuda.InitGPU()` 반환값과 로그를 확인한다. 런타임 DLL, CUDA 버전, 환경 변수 순으로 본다 |
| 같은 영상인데 CPU와 GPU 결과가 조금 다르다 | 부동소수 누적 순서 차이로 경계값 근처 검출이 달라질 수 있다. 등급 경계에 여유를 둔다 |
| 16비트 영상에서 아무것도 안 잡힌다 | Factor는 8비트 기준이다. 16비트 값은 1/256로 환산된다. 16비트 값을 그대로 Factor에 쓰지 않는다 |
| 로그에 "Must* 비교 옵션과 Edge 옵션이 함께 설정됨" 경고가 나온다 | 해당 필터의 `Must` 계열 블록을 `Count`로 바꾸거나 Edge 파라미터를 기본값으로 되돌린다(3-3, 3-4절) |

## 7. 관련 문서

- `README.md`: 패키지 피드와 설치
- `README_PRACTICAL.md`: 소비자 프로젝트 구성 템플릿
- `RatelSoft_CODEBOOK_RatelViewer.md`: 결과 표시용 뷰어
