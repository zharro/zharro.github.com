---
layout:     post
title:      Как вернуть ничего?
categories: design-patterns
summary:    null не предлагать!
date:       2016-08-14
---

## А чем null не нравится?

Инкапсуляция призывает нас к созданию "честных" интерфейсов классов. В этом случае, клиенты знают, чего от них ожидать и воспользуются ими правильно. Марк Симан [уточняет] [seemann-course], что если возвращаемое значение метода - string, то вернуть null будет не по-пацански. Более того, создатель null reference, Tony Hoare [называет][null-ref] ее ошибкой на миллиард долларов. В качестве примера давайте рассмотрим метод, который читает текст из файла, путь до которого формируется по некоторому правилу:

```csharp
public string Read(int id)
{
	var path = CustructFileName(id);
	var msg = File.ReadAllText(path);
	return msg;
}
```
Для простоты предположим, что `CustructFileName` всегда возвращает строку. Никаких null и исключений. Наша задача добиться того же для метода `Read`. Марк в своем курсе предлагает 3 варианта.
## Tester/Doer

[seemann-course]: https://app.pluralsight.com/player?course=encapsulation-solid&author=mark-seemann&name=encapsulation-solid-m2-encapsulation&clip=15&mode=live
[null-ref]: https://www.lucidchart.com/techblog/2015/08/31/the-worst-mistake-of-computer-science/