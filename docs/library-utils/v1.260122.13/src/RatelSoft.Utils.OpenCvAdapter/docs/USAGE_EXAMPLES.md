# OpenCvAdapter 실사용 예제

이 문서는 `RatelSoft.Utils.OpenCvAdapter`에서 제공하는 JSON/MessagePack 컨버터를 **바로 적용할 수 있는 형태**로 정리합니다.

## 1) System.Text.Json (Mat + 자주 쓰는 OpenCv 타입)

```csharp
using System.Text.Json;
using OpenCvSharp;
using RatelSoft.Utils.OpenCvAdapter.Json;

public sealed class SampleModel
{
    public Mat? Image { get; set; }
    public Rect Roi { get; set; }
    public Point Origin { get; set; }
    public Size Size { get; set; }
    public Scalar Color { get; set; }
}

var mat = new Mat(new Size(64, 48), MatType.CV_8UC3, new Scalar(0, 255, 0));

var model = new SampleModel
{
    Image = mat,
    Roi = new Rect(10, 10, 20, 15),
    Origin = new Point(1, 2),
    Size = new Size(64, 48),
    Color = new Scalar(0, 255, 0, 255),
};

var options = new JsonSerializerOptions();
options.AddOpenCvConverters(matExt: ".png");

string json = JsonSerializer.Serialize(model, options);
var back = JsonSerializer.Deserialize<SampleModel>(json, options);
```

## 2) MessagePack (Mat + 자주 쓰는 OpenCv 타입)

서버/전송 포맷에서 MessagePack을 쓰는 경우, 아래처럼 OpenCv formatter를 resolver에 포함시키면 됩니다.

```csharp
using MessagePack;
using MessagePack.Resolvers;
using OpenCvSharp;
using RatelSoft.Utils.OpenCvAdapter.Formatters;

// 1) 전역 옵션(레거시 방식):
Formatter.Init();

// 2) 권장: 옵션을 명시적으로 만들어 사용
var options = Formatter.CreateDefaultOptions();

public sealed class Packet
{
    public Mat? Image { get; set; }
    public Rect Roi { get; set; }
    public Point Origin { get; set; }
}

var packet = new Packet
{
    Image = new Mat(new Size(32, 32), MatType.CV_8UC1, new Scalar(128)),
    Roi = new Rect(0, 0, 10, 10),
    Origin = new Point(5, 6),
};

byte[] bytes = MessagePackSerializer.Serialize(packet, options);
var back = MessagePackSerializer.Deserialize<Packet>(bytes, options);
```

## 3) TestLibApp에서 확인

`TestLibApp`는 네트워크 창에서 Protobuf/MessagePack 직렬화 샘플이 포함되어 있습니다.
- Protobuf: `StringValue` 직렬화/역직렬화
- MessagePack: DTO 직렬화/역직렬화

실행:

```bash
dotnet run --project library-utils/apps/TestLibApp/TestLibApp.csproj
```
