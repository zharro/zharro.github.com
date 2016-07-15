---
layout: post
title:  "4 способа инициализировать тестируемый класс"
category: design-patterns
tags : [design-patterns, unit-tests]
---
## Что случилось?
Стало понятно, что писать и поддерживать unit-тесты для классов с большим количеством зависимостей очень сложно.
## Это почему? Дайте пример!
В качестве примера рассмотрим сервис, который возвращает набор продуктов, перед этим применяя скидку к каждому из них. Зависимости будем внедрять через конструктор, как учит [принцип явных зависимостей][explicit-dependencies-principle].
```csharp
public interface IGoodsProvider
{
    IList<Good> GetGoods();
}

public class AmazonGoodsProvider : IGoodsProvider
{
    // Получаем товары от Amazon
    public IList<Good> GetGoods()
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
## Ну и чего сложного? Берешь и пишешь тест.
Действительно, написать тест "в лоб" для для метода *GetGoods* довольно просто. Для этого достаточно создать такие "фейковые" реализации *IGoodsProvider* и *IConfiguration*, чтобы они возвращали продукт с определенной ценой и некоторый размер скидки соответственно. А после этого убедиться, что скидка была применена правильно:
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
1. Каждый тестовый метод должен передать в конструктор *sut* все зависимости, не смотря на то, используются они в тестируемом методе или нет.
2. Количество зависимостей у класса *PurchaseLogic* может измениться и в этом случае нам придётся изменять код **всех** тестов.
## Ну ладно, тогда можно создавать sut в методе SetUp
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
Метод инициализации понижает хрупкость тестов, ведь теперь при изменении конструктора *PurchaseLogic* не придется менять все тесты. Однако, у этого подхода есть и недостатки: 
1. Этапы Arrange, Act и Assert обычно связаны между собой и при чтении теста возникает "переключение контекста". 
2. Создание тестируемого класса вполне может отличаться от теста к тесту. В этом случае следует выделить логику создания тестируемого класса в отдельный фабричный метод.
## Ну уж фабрика точно решит все проблемы?
Не совсем. Реализуем фабричный метод так, чтобы он принимал все зависимости:
```csharp
public static PurchaseLogic CreatePurchaseLogic(IGoodsProvider goodsProvider, IConfiguration configuration)
{
    return new PurchaseLogic(goodsProvider, configuration);
}

[Test]
public void GetGoods_WhenGoodProvidedAndDiscountConfiguredToBeGreaterThenZero_AppliesDiscount()
{
    // Arrange
    Mock<IGoodsProvider> mockedGoodsProvider = new Mock<IGoodsProvider>();
    mockedGoodsProvider.Setup(p => p.GetGoods())
        .Returns(new List<Good> { new Good { Price = 100 } });
    Mock<IConfiguration> mockedConfiguration = new Mock<IConfiguration>();
    mockedConfiguration.Setup(p => p.GetDiscountPercent())
        .Returns(5);
    var sut = CreatePurchaseLogic(mockedGoodsProvider.Object, mockedConfiguration.Object);

    // Act
    var goodsWithDiscount = sut.GetGoods();

    // Assert
    Assert.That(goodsWithDiscount.Count, Is.EqualTo(1));
    Good goodWithDiscount = goodsWithDiscount.Single();
    Assert.That(goodWithDiscount.Price, Is.EqualTo(100 * 0.95));
}
```
Теперь вся *нужная* информация для анализа логики теста находится в одном месте. Фабричный метод скрывает детали создания тестируемого класса, предоставляя возможность тестам настроить его под себя. Но только до определенной степени. На моей практике, для классов с большим количеством зависимостей, выделить удачный фабричный метод бывает трудно, поскольку каждому тесту требуется настраивать *sut* совершенно по-разному. Но на днях в книге Сергея Теплякова [Паттерны проектирования на платформе .NET][patters-book], я прочитал про паттерн Test Fixture, который изящно решает эту проблему.
## Паттерн Test Fixture
Предлагается скрыть все аспекты создания тестируемого объекта от самих тестов, при этом предоставив им максимум контроля над этим процессом. Для этого будем использовать паттерн [Builder][builder-wiki], выделяя процесс создания тестируемого объекта в отдельный класс, который принято называть с *Fixture* на конце:
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
Теперь, когда вся подготовительная работа завершена, тесты могут заниматься тем, чем должны: проверять, что функциональность тестируемого кода работает так, как ожидается:
```
[Test]
public void GetGoods_WhenGoodProvidedAndDiscountConfiguredToBeGreaterThenZero_AppliesDiscount()
{
    // Arrange
    Mock<IGoodsProvider> mockedGoodsProvider = new Mock<IGoodsProvider>();
    mockedGoodsProvider.Setup(p => p.GetGoods())
        .Returns(new List<Good> { new Good { Price = 100 } });
    Mock<IConfiguration> mockedConfiguration = new Mock<IConfiguration>();
    mockedConfiguration.Setup(p => p.GetDiscountPercent())
        .Returns(5);
    var sut = new PurchaseLogicFixture()
        .WithGoodsProvider(mockedGoodsProvider.Object)
        .WithConfiguration(mockedConfiguration.Object)
        .PurchaseLogic;

    // Act
    var goodsWithDiscount = sut.GetGoods();

    // Assert
    Assert.That(goodsWithDiscount.Count, Is.EqualTo(1));
    Good goodWithDiscount = goodsWithDiscount.Single();
    Assert.That(goodWithDiscount.Price, Is.EqualTo(100 * 0.95));
}
```
## Отлично, может, теперь использовать Test Fixture во всех тестах?
Пожалуй, не стоит. Test Fixture хорошо решает задачу инициализации тестируемого объекта, имеющего много зависимостей, которые нужно настраивать по-разному от теста к тесту. Для более простых случаев вполне подойдет фабричный метод.

[sut-ploeh]: https://blogs.msdn.microsoft.com/ploeh/2008/10/06/naming-sut-test-variables/
[explicit-dependencies-principle]: http://deviq.com/explicit-dependencies-principle/
[patters-book]: https://www.ozon.ru/context/detail/id/31789305/
[builder-wiki]: https://en.wikipedia.org/wiki/Builder_pattern