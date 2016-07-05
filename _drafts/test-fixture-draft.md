Одна из самых больших проблем, с который я сталкиваюсь при написании unit-тестов - инициализация тестируемого объекта ([sut][sut-ploeh]). Конечно, мы знаем [принцип явных зависимостей][explicit-dependencies-principle] и тестируемый класс всегда принимает зависимости через конструктор, например, так:
```csharp
public MyClass(IMyInterface dependency)
```
Отлично, так в чем же, собственно, проблема?
1. Набор зависимостей может измениться. Например, добавится еще одна, в этом случае придется обновлять все тесты для данного класса. Получаем ту самую хрупкость тестов, которую нужно бояться как огня.
2. Каждому тестовому методу нужно создать *все* зависимости тестируемого класса. Причем некоторые, а то и большинство, в тестируемом методе не используются и их создание только усложняет код теста. Рассмотрим пример: (часть кода примера я взял из [статьи Марка Симана][sut-double-ploeh])
```csharp
public class ReservationsController : ApiController
{
    public ReservationsController(IReservationsRepository repository, IConfiguration configuration)
    {
        if (repository == null)
            throw new ArgumentNullException(nameof(repository));
		if (configuration == null)
            throw new ArgumentNullException(nameof(configuration));

        this.Repository = repository;
		this.Configuration = configuration;
    }
 
    public IReservationsRepository Repository { get; }
	public IConfiguration Configuration { get; }
 
    public IHttpActionResult Post(ReservationDto reservationDto)
    {
        DateTime requestedDate;
        if (!DateTime.TryParse(reservationDto.Date, out requestedDate))
            return this.BadRequest("Invalid date.");
 
        var reservedSeats =
            this.Repository.ReadReservedSeats(requestedDate);
        if (this.Configuration.Capacity < reservationDto.Quantity + reservedSeats)
            return this.StatusCode(HttpStatusCode.Forbidden);
 
        this.Repository.SaveReservation(requestedDate, reservationDto);
 
        return this.Ok();
    }
	
	public IHttpActionResult Delete(long reservationId)
	{
		this.Repository.DeleteReservation(reservationId);
		
		return this.Ok();
	}
}
```
Для того, чтобы протестировать метод Delete, нам придётся передать в ReservationsController обе его зависимости: IReservationsRepository и IConfiguration. При этом, вторая использоваться не будет.

[sut-ploeh]: https://blogs.msdn.microsoft.com/ploeh/2008/10/06/naming-sut-test-variables/
[explicit-dependencies-principle]: http://deviq.com/explicit-dependencies-principle/
[sut-double-ploeh]: http://blog.ploeh.dk/2016/06/15/sut-double/

[about-development-testing]: http://sergeyteplyakov.blogspot.ru/2014/04/about-development-testing.html