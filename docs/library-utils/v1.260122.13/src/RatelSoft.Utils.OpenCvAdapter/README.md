# RatelSoft.Utils.OpenCvAdapter

OpenCvSharp 의존 기능을 Opt-in으로 제공하는 어댑터 패키지입니다.

## 포함 모듈
- MessagePack: Mat formatter 등
- System.Text.Json: Mat JSON converter

## Mat JSON Converter 등록

```csharp
using System.Text.Json;
using RatelSoft.Utils.OpenCvAdapter.Json;

var options = new JsonSerializerOptions();
options.AddOpenCvMatConverter(ext: ".png");
```

이 `options`를 `DataStore<T>` 생성자의 `jsonOptions`에 전달하면 Mat 직렬화가 가능합니다.
