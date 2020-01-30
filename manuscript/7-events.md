# События

A> Правило прокрастинотора: если что-то можно отложить, это надо отложить.

Действие Слоя Приложения всегда содержит главную часть, некоторые дополнительные действия.
Регистрация пользователя состоит из собственно создания сущности пользователя, а также посылка соответсвующего письма.
Обновление текста статьи содержит обновление значения **$post->text** с сохранением сущности, а также, например вызов **Cache::forget** для инвалидации кеша.
Псевдо-реальный пример: сайт с опросами.
Создание сущности опроса:

```php
final class PollService
{
    /** @var BadWordsFilter */
    private $badWordsFilter;
    
    public function __construct(BadWordsFilter $badWordsFilter
        /*, ... другие зависимости */)
    {
        $this->badWordsFilter = $badWordsFilter;
    }
    
    public function create(PollCreateDto $request)
    {
        $poll = new Poll();
        $poll->question = $this->badWordsFilter->filter(
            $request->getQuestion());
        //...
        $poll->save();
        
        foreach($request->getOptionTexts() as $optionText)
        {
            $pollOption = new PollOption();
            $pollOption->poll_id = $poll->id;
            $pollOption->text = 
                    $this->badWordsFilter->filter($optionText);
            $pollOption->save();
        }
        
        // Вызов генерации sitemap
        
        // Оповестить внешнее API о новом опросе
    }
}
```
Это обычное создание опроса с опциями ответа, с фильтрацией мата во всех текстах и некоторыми пост-действиями.
Объект опроса непростой.
Он абсолютно бесполезен без опций ответа.
Мы должны позаботиться о его консистентности.
Этот маленький промежутор времени, когда мы уже создали объект опроса и добавили его в базу данных, но еще не создали его опции ответа, очень опасен!

## Database transactions

Первая проблема - база данных.
В это время может произойти некоторая ошибка (с соединением с базой данных, да даже просто в функции проверки на мат) и объект опроса будет в базе данных, но без опций ответа.
Все движки баз данных, которые созданы для хранения важных данных, имеют механизм транзакций.
Они гарантируют консистентность внутри транзакции.
Все запросы, обернутые в транзакцию, либо будут выполнены полностью, либо, при возникновении ошибки во время выполнения, ни один из них.
Выглядит как решение:

```php
final class PollService
{
    /** @var DatabaseConnection */
    private $connection;
    
    public function __construct(..., DatabaseConnection $connection) 
    {
        ...
        $this->connection = $connection;
    }
        
    public function create(PollCreateDto $request)
    {
        $this->connection->transaction(function() use ($request) {
            $poll = new Poll();
            $poll->question = $this->badWordsFilter->filter(
                    $request->getQuestion());
            //...
            $poll->save();
            
            foreach($request->getOptionTexts() as $optionText) {
                $pollOption = new PollOption();
                $pollOption->poll_id = $poll->id;
                $pollOption->text = 
                    $this->badWordsFilter->filter($optionText);
                $pollOption->save();
            }
            
            // Вызов генерации sitemap
        
            // Оповестить внешнее API о новом опросе          
        });
    }
}
```

Отлично, наши данные теперь консистентны, но эта магия транзакций даётся базе данных небесплатно.
Когда мы выполняем запросы в транзакции, база данных вынуждена хранить две копии данных: для успешного или неуспешного результатов.
Для проектов с нагрузкой, которые могут выполнять сотни одновременных транзакций, которые длятся долго, это может сильно сказаться на производительности.
Для проектов с небольшой нагрузкой это не настолько важно, но все равно стоит приобрести привычку выполнять транзакции как можно быстрее.
Проверка на мат может требовать запроса на специальное API, которое может занять страшно много времени.
Попробуем вынести из транзакции все, что возможно:

```php
final class PollService
{
public function create(PollCreateDto $request)
{
    $filteredRequest = $this->filterRequest($request);
    
    $this->connection->transaction(
        function() use ($filteredRequest) {
            $poll = new Poll();
            $poll->question = $filteredRequest->getQuestion();
            //...
            $poll->save();
            
            foreach($filteredRequest->getOptionTexts() 
                                as $optionText) {
                $pollOption = new PollOption();
                $pollOption->poll_id = $poll->id;
                $pollOption->text = $optionText;
                $pollOption->save();
            }
        });
    
    // Вызов генерации sitemap
            
    // Оповестить внешнее API о новом опросе
}

private function filterRequest(
    PollCreateDto $request): PollCreateDto
{
    // фильтрует тексты в запросе
    // и возвращает такое же DTO но с "чистыми" данными
}
}
```

## Очереди

Вторая проблема - время выполнения запроса.
Приложение должно отвечать как можно быстрее.
Создание опроса содержит тяжелые действия, такие как генерация sitemap или вызовы внешних API.
Обычное решение - отложить эти действия с помощью механизма очередей.
Вместо того, чтобы выполнять тяжелое действие в обработчике web-запроса, приложение может создать задачу для выполнения этого действия и положить его в очередь.
Очередью может быть таблица в базе данных, список в Redis или специальный софт для очередей, например RabbitMQ.

Laravel предоставляет несколько путей работы с очередями. Один из них: jobs. Можно перевести как Работы или Задачи.
Как я говорил ранее, действие состоит из главной части и несколько второстепенных действий.
Главное действие при создании сущности опроса не может быть выполнено без фильтрации мата, но все пост-действия можно отложить.
Вообще, в некоторых ситуациях, которые занимают слишком много времени, можно действие полностью отложить, сказав пользователю "Скоро выполним", но это пока не наш случай.

```php
final class SitemapGenerationJob implements ShouldQueue
{
    public function handle()
    {
        // Вызов генератора sitemap
    }
}

final class NotifyExternalApiJob implements ShouldQueue {}

use Illuminate\Contracts\Bus\Dispatcher;

final class PollService
{
    /** @var Dispatcher */
    private $dispatcher;
    
    public function __construct(..., Dispatcher $dispatcher) 
    {
        ...
        $this->dispatcher = $dispatcher;
    }
        
    public function create(PollCreateDto $request)
    {
        $filteredRequest = $this->filterRequest($request);
        
        $poll = new Poll();
        $this->connection->transaction(...);
        
        $this->dispatcher->dispatch(
            new SitemapGenerationJob());
        $this->dispatcher->dispatch(
            new NotifyExternalApiJob($poll->id));
    }
}
```

Если класс содержит интерфейс **ShouldQueue**, то выполнение этой задачи будет отложено в очередь.
Этот код выполняется достаточно быстро, но мне он все ещё не нравится.
Таких пост-действий может быть очень много и сервисный класс начинает знать слишком много.
Он выполняет каждое пост-действие, но с высокоуровневой точки зрения, действие создания опроса не должно знать про генерацию sitemap или взаимодействие с внешними API.
В проектах с огромным количеством пост-действий, контролировать их становится очень непросто.

## События

Вместо того, чтобы напрямую вызывать каждое пост-действие, сервисный класс может просто информировать приложение о том, что что-то произошло.
Приложение может реагировать на эти события, выполняя нужные пост-действия.
В Laravel есть поддержка механизма событий:

```php
final class PollCreated
{
    /** @var int */
    private $pollId;
    
    public function __construct(int $pollId)
    {
        $this->pollId = $pollId;
    }
    
    public function getPollId(): int
    {
        return $this->pollId;
    }
}

use Illuminate\Contracts\Events\Dispatcher;

final class PollService
{
    /** @var Dispatcher */
    private $dispatcher;
    
    public function __construct(..., Dispatcher $dispatcher) 
    {
        ...
        $this->dispatcher = $dispatcher;
    }
        
    public function create(PollCreateDto $request)
    {
        // ...
        
        $poll = new Poll();
        $this->connection->transaction(
            function() use ($filteredRequest, $poll) {
                // ...
            });
        
        $this->dispatcher->dispatch(new PollCreated($poll->id));
    }
}

final class SitemapGenerationListener implements ShouldQueue
{
    public function handle($event)
    {
        // Call sitemap generator
    }
}

final class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        PollCreated::class => [
            SitemapGenerationListener::class,
            NotifyExternalApiListener::class,
        ],
    ];
}
```

Теперь Слой Приложения просто сообщает, что был создан новый опрос (событие **PollCreated**).
Приложение имеет конфигурацию для реагирования на события.
Классы слушателей событий (**Listener**) содержат реакцию на события.
Интерфейс **ShouldQueue** работает точно также, сообщая о том, когда должно быть запущено выполнение этого слушателя: сразу же или в отложенно.
События - очень мощная вещь, но здесь тоже есть ловушки.

## Использование событий Eloquent

Laravel генерирует кучу событий.
События системы кеширования: **CacheHit**, **CacheMissed**, и т.д..
События оповещений: **NotificationSent**, **NotificationFailed**, и т.д..
Eloquent тоже генерирует свои события.
Пример из документации:

```php

class User extends Authenticatable
{
    /**
     * The event map for the model.
     *
     * @var array
     */
    protected $dispatchesEvents = [
        'saved' => UserSaved::class,
        'deleted' => UserDeleted::class,
    ];
}
```

Событие **UserSaved** будет генерироваться каждый раз когда сущность пользователя будет сохранено в базу данных.
Сохранено, означает любой update или insert запрос.
Использование этих событий имеет множество недостатков.

**UserSaved** не самое удачное название для этого события.
**UsersTableRowInsertedOrUpdated** более подходящее.
Но и оно не всегда верное.
Это событие не будет сгенерировано при массовых операциях со строками базы данных.
Событие "Deleted" не будет вызвано, если строка в базе данных будет удалена с помощью механизма каскадного удаления в базе данных.
Главная же проблема это то, что это события уровня инфраструктуры, события строчек базы данных, но они используются как бизнес-события, или события доменной области.
Разницу легко осознать в примере с созданием сущности опроса:

```php
final class PollService
{
public function create(PollCreateDto $request)
{
    //...
    $this->connection->transaction(function() use (...) {
        $poll = new Poll();
        $poll->question = $filteredRequest->getQuestion();
        //...
        $poll->save();
        
        foreach($filteredRequest->getOptionTexts() as $optionText){
            $pollOption = new PollOption();
            $pollOption->poll_id = $poll->id;
            $pollOption->text = $optionText;
            $pollOption->save();
        }
    });
    //...
}
}
```

Вызов **$poll->save();** сгенерирует событие 'saved' для этой сущности.
Первая проблема в том, что объект опроса еще не готов и неконсистентен.
Он все еще не имеет опций ответа.
В слушателе, который захочет отправить email с этим опросом, явно хочется иметь этот объект полностью, со всеми вариантами ответа.
Для отложенных слушателей это не проблема, но на машинах разработчиков часто значение **QUEUE_DRIVER** - 'sync', поэтому все отложенные слушатели и задачи будут выполняться сразу и поведение будет разным.
Я сильно рекомендую избегать кода, который правильно работает "иногда, в некоторой особой ситуации", а иногда может преподнести неприятный сюрприз.

Вторая проблема - эти события вызываются прямо внутри транзакции.
Выполнение слушателей сразу или даже отправка их в очередь делают транзакции более долгими и хрупкими.
А самое страшное, что событие вроде **PollCreated**, может быть вызвано, но дальше в транзакции будет ошибка и вся она будет откачена назад.
Письмо же пользователю об опросе, который даже не создался, все равно будет отправлено.
Я нашел пару пакетов для Laravel, которые ловят все эти события, хранят их временно, и выполняют только после того, как транзакция будет успешно завершена (гуглите "Laravel transactional events").
Да, они решают множество из этих проблем, но всё это выглядит так нестественно!
Простая идея генерировать нормальное бизнес-событие **PollCreated** после успешной транзакции намного лучше.

## Сущности как поля классов-событий
Я часто вижу как Eloquent-сущности используются напрямую в поля событий:

```php
final class PollCreated
{
    /** @var Poll */
    private $poll;
    
    public function __construct(Poll $poll)
    {
        $this->poll = $poll;
    }
    
    public function getPoll(): Poll
    {
        return $this->poll;
    }
}

final class PollService
{
    public function create(PollCreateDto $request)
    {
        // ...
        $poll = new Poll();
        // ...
        $this->dispatcher->dispatch(new PollCreated($poll));
    }
}

final class SendPollCreatedEmailListener implements ShouldQueue
{
    public function handle(PollCreated $event)
    {
        // ...
        foreach($event->getPoll()->options as $option)
        {...}
    }
}
```
Это просто пример слушателя, который использует значения **HasMany**-отношения.
Этот код работает. Когда выполняется код **$event->getPoll()->options** Eloquent делает запрос в базу данных и получает все опции ответа.
Другой пример:

```php
final class PollOptionAdded
{
    /** @var Poll */
    private $poll;
    
    public function __construct(Poll $poll)
    {
        $this->poll = $poll;
    }
    
    public function getPoll(): Poll
    {
        return $this->poll;
    }
}

final class PollService
{
public function addOption(PollAddOptionDto $request)
{
    $poll = Poll::findOrFail($request->getPollId());
    
    if($poll->options->count() >= Poll::MAX_POSSIBLE_OPTIONS) {
        throw new BusinessException('Max options amount exceeded');
    }

    $poll->options()->create(...);
    
    $this->dispatcher->dispatch(new PollOptionAdded($poll));
}
}

final class SomeListener implements ShouldQueue
{
    public function handle(PollOptionAdded $event)
    {
        // ...
        foreach($event->getPoll()->options as $option)
        {...}
    }
}
```

А вот тут уже не все хорошо.
Когда сервисный класс проверяет количество опций ответа, он получается свежую коллекцию текущих опций ответа данного опроса.
Потом он добавляет новую опцию, вызвав **$poll->options()->create(...)**;
Дальше, слушатель, выполняя **$event->getPoll()->options** получает старую версию опций ответа, без новосозданной.
Это поведение Eloquent, который имеет два механизма работы с отношениями. Метод **options()** и псевдо-поле **options**, которое вроде бы и соответсвует этмоу методу, но хранит свою версию данных.
Поэтому, передавая сущность в события, разработчик должен озаботиться консистентностью значений в отношениях, например вызвав:

```php
$poll->load('options');
```

до передачи объекта в событие.
Это все делает код приложения весьма хрупким. Его легко сломать неосторожной передачей объекта в событие.
Намного проще просто передавать **id** сущности.
Каждый слушатель всегда может получить свежую версию сущности, запросив её из базы данных по этому **id**.