Одна из самых больших проблем, с который я сталкиваюсь при написании unit-тестов - это инициализация тестируемого объекта ([sut][sut-ploeh]). Конечно, мы знаем [принцип явных зависимостей][explicit-dependencies-principle] и тестируемый класс всегда принимает зависимости через конструктор. Давайте рассмотрим пример:
```csharp
public interface IGoodsProvider
{
    IEnumerable<Good> GetGoods();
}

public class AmazonGoodsProvider : IGoodsProvider
{
    // Получаем товары от Amazon
    public IEnumerable<Good> GetGoods()
    {
        // ...
    }
}

public interface IConfiguration
{
    decimal GetDiscountPercent();
}

public class WebConfigWrapper : IConfiguration
{
    // Получаем настройки из Web.config
    public decimal GetDiscountPercent()
    {
        // ...
    }
}

public class PurchaseLogic
{
    private readonly IGoodsProvider _goodsProvider;
    private readonly IConfiguration _configuration;

    public PurchaseLogic(IGoodsProvider goodsProvider, IConfiguration configuration)
    {
        if (goodsProvider == null)
            throw new ArgumentNullException(nameof(goodsProvider));
        if (configuration == null)
            throw new ArgumentNullException(nameof(configuration));
        _goodsProvider = goodsProvider;
        _configuration = configuration;
    }

    // Возвращаем товары, применив скидку к каждому из них
    public IList<Good> GetGoods()
    {
        var goods = _goodsProvider.GetGoods();
        var factor = (100 - _configuration.GetDiscountPercent()) / 100;
        foreach (var good in goods)
        {
            good.Price = good.Price * factor;
        }
        return goods;
    }
}
```
Здесь у класса *PurchaseLogic* есть две зависимости: *IGoodsProvider* для получения списка товаров и *IConfiguration* для получения размера скидки. 
## Наивный подход к написанию юнит-тестов
Написать тест для метода *GetGoods* довольно просто. Для этого достаточно замокировать *IGoodsProvider* и *IConfiguration* так, чтобы они возвращали продукт с определенной ценой и размер скидки соответственно. И после этого убедиться, что скидка была применена правильно. Решение "в лоб" будет выглядеть так:
```csharp
[Test]
public void GetGoods_WhenGoodProvidedAndDiscountConfiguredToBeGreaterThenZero_AppliesDiscount()
{
    // Arrange
    Mock<IGoodsProvider> mockedGoodsProvider = new Mock<IGoodsProvider>();
    mockedGoodsProvider.Setup(p => p.GetGoods())
        .Returns(new List<Good> {new Good {Price = 100}});
    Mock<IConfiguration> mockedConfiguration = new Mock<IConfiguration>();
    mockedConfiguration.Setup(p => p.GetDiscountPercent())
        .Returns(5);

    var sut = new PurchaseLogic(mockedGoodsProvider.Object, mockedConfiguration.Object);

    // Act
    var goodsWithDiscount = sut.GetGoods();

    // Assert
    Assert.That(goodsWithDiscount.Count, Is.EqualTo(1));
    Good goodWithDiscount = goodsWithDiscount.Single();
    Assert.That(goodWithDiscount.Price, Is.EqualTo(100 * 0.95));
}
```
У этого тестового метода есть несколько проблем:
1. Для простого класса с двумя зависимостями и простым поведением количество вспомогательного кода кажется весьма значительным. В реальном мире код будет сложнее и читабельность тестов будет еще ниже. 
2. Каждый тестовый метод должен передать в конструктор *sut* все зависимости, не смотря на то, используются они в тестируемом методе или нет. Например, если бы в классе *PurchaseLogic* появился метод *CancelPurchase*, который использовал только зависимость *IGoodsProvider* - в тесте все равно пришлось бы иметь дело с *IConfiguration*.
3. Количество зависимостей у класса *PurchaseLogic* может измениться и в этом случае нам придётся изменять код **всех** тестов.
## Использование метода инициализации теста
Большинство тестовых фреймворков предоставляют возможность выполнить код инициализации перед вызовом каждого тестового метода. В NUnit - это метод, помеченный атрибутом *SetUp*:
```csharp
private PurchaseLogic _sut;

[SetUp]
public void SetUp()
{
    Mock<IGoodsProvider> mockedGoodsProvider = new Mock<IGoodsProvider>();
    mockedGoodsProvider.Setup(p => p.GetGoods())
        .Returns(new List<Good> { new Good { Price = 100 } });
    Mock<IConfiguration> mockedConfiguration = new Mock<IConfiguration>();
    mockedConfiguration.Setup(p => p.GetDiscountPercent())
        .Returns(5);

    _sut = new PurchaseLogic(mockedGoodsProvider.Object, mockedConfiguration.Object);
}

[Test]
public void GetGoods_WhenGoodProvidedAndDiscountConfiguredToBeGreaterThenZero_AppliesDiscount()
{
    // Act
    var goodsWithDiscount = _sut.GetGoods();

    // Assert
    Assert.That(goodsWithDiscount.Count, Is.EqualTo(1));
    Good goodWithDiscount = goodsWithDiscount.Single();
    Assert.That(goodWithDiscount.Price, Is.EqualTo(100 * 0.95));
}
```
Метод инициализации понижает хрупкость тестов, ведь теперь при изменении конструктора *PurchaseLogic* не придется менять все тесты. Однако, у этого подхода есть и недостатки. Во-первых, этапы Arrange, Act и Assert обычно связаны между собой и при чтении теста возникает "переключение контекста". Во-вторых, создание тестируемого класса вполне может отличаться от теста к тесту, в этом случае следует выделить логику создания тестируемого класса в отдельный фабричный метод. Методы *SetUp* следует использовать для инициализации контекста, одинакового для всех тестов.
## Использование фабричного метода в тестах
Предположим, что тестам класса *PurchaseLogic* не важно устанавливать размер скидки, но важно настраивать *IGoodsProvider*. Тогда использовать фабричный метод можно так:
```csharp
public static PurchaseLogic CreatePurchaseLogic(IGoodsProvider goodsProvider)
{
  Mock<IConfiguration> mockedConfiguration = new Mock<IConfiguration>();
    mockedConfiguration.Setup(p => p.GetDiscountPercent())
        .Returns(5);

    return new PurchaseLogic(goodsProvider, mockedConfiguration.Object);
}

[Test]
public void GetGoods_WhenGoodProvidedAndDiscountConfiguredToBeGreaterThenZero_AppliesDiscount()
{
    // Arrange
    Mock<IGoodsProvider> mockedGoodsProvider = new Mock<IGoodsProvider>();
    mockedGoodsProvider.Setup(p => p.GetGoods())
        .Returns(new List<Good> { new Good { Price = 100 } });
    var sut = CreatePurchaseLogic(mockedGoodsProvider.Object);

    // Act
    var goodsWithDiscount = sut.GetGoods();

    // Assert
    Assert.That(goodsWithDiscount.Count, Is.EqualTo(1));
    Good goodWithDiscount = goodsWithDiscount.Single();
    Assert.That(goodWithDiscount.Price, Is.EqualTo(100 * 0.95));
}
```
Разница кажется небольшой, но теперь вся *нужная* информация для анализа логики теста находится в одном месте. Фабричный метод скрывает детали создания тестируемого класса, предоставляя возможность каждому тесту настроить его под себя. Но только до определенной степени. На моей практике, для классов с большим количеством зависимостей, выделить удачный фабричный метод бывает трудно, поскольку каждому тесту требуется настраивать *sut* совершенно по-разному. Но на днях в книге Сергея Теплякова [Паттерны проектирования на платформе .NET][patters-book], я прочитал про паттерн Test Fixture, который изящно решает эту проблему.
## Паттерн Test Fixture
Предлагается скрыть все аспекты создания тестового объекта от самих тестов, при этом предоставив им максимум контроля над этим процессом. Для этого процесс создания тестируемого объекта выделяем в отдельный класс, который принято называть с *Fixture* на конце:
```
internal class PurchaseLogicFixture
{
    private IConfiguration _configuration;
    private IGoodsProvider _goodsProvider;
    private readonly Lazy<PurchaseLogic> _lazyCut;

    public PurchaseLogicFixture()
    {
        _lazyCut = new Lazy<PurchaseLogic>(
            () => new PurchaseLogic(_goodsProvider, _configuration));
    }

    public PurchaseLogicFixture WithGoodsProvider(IGoodsProvider goodsProvider)
    {
        _goodsProvider = goodsProvider;
        return this;
    }

    public PurchaseLogicFixture WithConfiguration(IConfiguration configuration)
    {
        _configuration = configuration;
        return this;
    }

    public PurchaseLogic PurchaseLogic
    {
        get { return _lazyCut.Value; }
    }
}
```

[sut-ploeh]: https://blogs.msdn.microsoft.com/ploeh/2008/10/06/naming-sut-test-variables/
[explicit-dependencies-principle]: http://deviq.com/explicit-dependencies-principle/
[sut-double-ploeh]: http://blog.ploeh.dk/2016/06/15/sut-double/
[patters-book]: https://www.ozon.ru/context/detail/id/31789305/
[about-development-testing]: http://sergeyteplyakov.blogspot.ru/2014/04/about-development-testing.html