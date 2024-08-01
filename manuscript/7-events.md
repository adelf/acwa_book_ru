# События

Действие Слоя Приложения всегда содержит главную часть, где выполняется основная логика действия, а также некоторые дополнительные действия.
Регистрация пользователя состоит из, собственно, создания сущности пользователя, а также посылки соответствующего письма.
Обновление текста статьи содержит обновление значения **$post->text** с сохранением сущности, а также, например вызов **Cache::forget** для инвалидации кеша.
Псевдо-реальный пример: сайт с опросами.
Создание сущности опроса:

```php
final class SurveyService
{
    public function __construct(
        private ProfanityFilter $profanityFilter
        /*, ... другие зависимости */
    ) {}
    
    public function create(SurveyCreateDto $request)
    {
        $survey = new Survey();
        $survey->question = $this->profanityFilter->filter(
            $request->getQuestion());
        //...
        $survey->save();
        
        foreach($request->getOptions() as $optionText)
        {
            $option = new SurveyOption();
            $option->survey_id = $survey->id;
            $option->text = 
                    $this->profanityFilter->filter($optionText);
            $option->save();
        }
        
        // Вызов генерации sitemap
        
        // Оповестить внешнее API о новом опросе
    }
}
```
Это обычное создание опроса с вариантами ответа, с фильтрацией мата во всех текстах и некоторыми пост-действиями.
Объект опроса непростой.
Он абсолютно бесполезен без вариантов ответа.
Мы должны позаботиться о его консистентности.
Этот маленький промежуток времени, когда мы уже создали объект опроса и добавили его в базу данных, но еще не создали его варианты ответа, очень опасен!

## Database transactions

Первая проблема - база данных.
В это время может произойти некоторая ошибка (с соединением с базой данных, да даже просто в функции проверки на мат) и объект опроса будет в базе данных, но без вариантов ответа.
Все движки баз данных, которые созданы для хранения важных данных, имеют механизм транзакций.
Они гарантируют консистентность внутри транзакции.
Все запросы, обернутые в транзакцию, либо будут выполнены полностью, либо, при возникновении ошибки во время выполнения, ни один из них.
Выглядит как решение:

```php
final class SurveyService
{
    public function __construct(
        ..., private DatabaseConnection $connection
    ) {}
 
    public function create(SurveyCreateDto $request)
    {
        $this->connection->transaction(function() use ($request) {
            $survey = new Survey();
            $survey->question = $this->profanityFilter->filter(
                    $request->getQuestion());
            //...
            $survey->save();
            
            foreach($request->getOptions() as $optionText) {
                $option = new SurveyOption();
                $option->survey_id = $survey->id;
                $option->text = 
                    $this->profanityFilter->filter($optionText);
                $option->save();
            }
            
            // Вызов генерации sitemap
        
            // Оповестить внешнее API о новом опросе          
        });
    }
}
```

Отлично, наши данные теперь консистентны, но эта магия транзакций даётся базе данных не бесплатно.
Когда мы выполняем запросы в транзакции, база данных вынуждена хранить две копии данных: для успешного или неуспешного результатов.
Для проектов с нагрузкой, которые могут выполнять сотни одновременных транзакций, длящихся долго, это может сильно сказаться на производительности.
Для проектов с небольшой нагрузкой это не настолько важно, но все равно стоит приобрести привычку выполнять транзакции как можно быстрее.
Проверка на мат может требовать запроса на специальное API, которое может занять страшно много времени.
Попробуем вынести из транзакции все, что возможно:

```php
final class SurveyService
{
public function create(SurveyCreateDto $request)
{
    $filteredRequest = $this->filterRequest($request);
    
    $this->connection->transaction(
        function() use ($filteredRequest) {
            $survey = new Survey();
            $survey->question = $filteredRequest->getQuestion();
            //...
            $survey->save();
            
            foreach($filteredRequest->getOptions() 
                                as $optionText) {
                $option = new SurveyOption();
                $option->survey_id = $survey->id;
                $option->text = $optionText;
                $option->save();
            }
        });
    
    // Вызов генерации sitemap
            
    // Оповестить внешнее API о новом опросе
}

private function filterRequest(
    SurveyCreateDto $request): SurveyCreateDto
{
    // фильтрует тексты в запросе
    // и возвращает такое же DTO но с "чистыми" данными
}
}
```

Теперь внутри транзакции только легкие действия и она выполняется моментально. Так и должно быть.

## Очереди

Вторая проблема - время выполнения запроса.
Приложение должно отвечать как можно быстрее.
Создание опроса содержит тяжелые действия, такие как генерация sitemap или вызовы внешних API.
Обычное решение - отложить эти действия с помощью механизма очередей.
Вместо того чтобы выполнять тяжелое действие в обработчике web-запроса, приложение может создать задачу для выполнения этого действия и положить его в очередь.
Дальше из очереди его возьмет служба, расположенная на том же сервере. Или она будет на других серверах, созданных специально для обработки задач из очередей. Такие сервера называют воркер-серверами.
Очередью может быть таблица в базе данных, список в Redis или специальный софт для очередей, например Kafka или RabbitMQ.

Laravel предоставляет несколько путей работы с очередями. Один из них: jobs.
Как я говорил ранее, действие состоит из главной части и несколько второстепенных действий.
Главное действие при создании сущности опроса не может быть выполнено без фильтрации мата, но все пост-действия можно отложить.
Вообще, в некоторых ситуациях, когда действие занимает слишком много времени, можно его полностью отложить, сказав пользователю "Скоро выполним", но это пока не наш случай.

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

final class SurveyService
{
    public function __construct(..., 
        private Dispatcher $dispatcher
    ) {}
        
    public function create(SurveyCreateDto $request)
    {
        $filteredRequest = $this->filterRequest($request);
        
        $survey = new Survey();
        $this->connection->transaction(...);
        
        $this->dispatcher->dispatch(
            new SitemapGenerationJob());
        $this->dispatcher->dispatch(
            new NotifyExternalApiJob($survey->id));
    }
}
```

Если класс содержит интерфейс **ShouldQueue**, то выполнение этой задачи будет отложено в очередь.
Этот код выполняется достаточно быстро, но мне он все ещё не нравится.
Таких пост-действий может быть очень много и сервисный класс начинает знать слишком много.
Он выполняет каждое пост-действие, но с высокоуровневой точки зрения, действие создания опроса не должно знать про генерацию sitemap или взаимодействие с внешними API.
В проектах с огромным количеством пост-действий, контролировать их становится очень непросто.

## События

Вместо того чтобы напрямую вызывать каждое пост-действие, сервисный класс может просто информировать приложение о том, что что-то произошло.
Приложение может реагировать на эти события, выполняя нужные пост-действия.
В Laravel есть поддержка механизма событий:

```php
final class SurveyCreated
{
    public function __construct(
        public readonly int $surveyId
    ) {}
}

use Illuminate\Contracts\Events\Dispatcher;

final class SurveyService
{
    public function __construct(..., 
        private Dispatcher $dispatcher
    ) {}

    public function create(SurveyCreateDto $request)
    {
        // ...
        
        $survey = new Survey();
        $this->connection->transaction(
            function() use ($filteredRequest, $survey) {
                // ...
            });
        
        $this->dispatcher->dispatch(new SurveyCreated($survey->id));
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
        SurveyCreated::class => [
            SitemapGenerationListener::class,
            NotifyExternalApiListener::class,
        ],
    ];
}
```

Теперь Слой Приложения просто сообщает, что был создан новый опрос (событие **SurveyCreated**).
Приложение имеет конфигурацию для реагирования на события.
Классы слушателей событий (**Listener**) содержат реакцию на события.
Интерфейс **ShouldQueue** работает точно также, сообщая о том, когда должно быть запущено выполнение этого слушателя: сразу же или отложено.
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

Событие **UserSaved** будет генерироваться каждый раз когда сущность пользователя будет сохранена в базу данных.
Сохранена, означает любой update или insert запрос.
Использование этих событий имеет множество недостатков.

**UserSaved** не самое удачное название для этого события.
**UsersTableRowInsertedOrUpdated** более подходящее.
Но и оно не всегда верное.
Это событие не будет сгенерировано при массовых операциях со строками базы данных.
Событие "Deleted" не будет вызвано, если строка в базе данных будет удалена с помощью механизма каскадного удаления в базе данных.
Главная же проблема это то, что это события уровня инфраструктуры, события строчек базы данных, но они используются как бизнес-события, или события доменной области.
Разницу легко осознать в примере с созданием сущности опроса:

```php
final class SurveyService
{
public function create(SurveyCreateDto $request)
{
    //...
    $this->connection->transaction(function() use (...) {
        $survey = new Survey();
        $survey->question = $filteredRequest->getQuestion();
        //...
        $survey->save();
        
        foreach($filteredRequest->getOptions() as $optionText){
            $option = new SurveyOption();
            $option->survey_id = $survey->id;
            $option->text = $optionText;
            $option->save();
        }
    });
    //...
}
}
```

Вызов **$survey->save();** сгенерирует событие 'saved' для этой сущности.
Первая проблема в том, что объект опроса еще не готов и неконсистентен.
Он все еще не имеет вариантов ответа.
В слушателе, который захочет отправить email с этим опросом, явно хочется иметь этот объект полностью, со всеми вариантами ответа.
Для отложенных слушателей это не проблема, но на машинах разработчиков часто значение **QUEUE_DRIVER** - 'sync', поэтому все отложенные слушатели и задачи будут выполняться сразу и поведение будет разным.
Я сильно рекомендую избегать кода, который правильно работает "иногда, в некоторой особой ситуации", а иногда может преподнести неприятный сюрприз.

Вторая проблема - эти события вызываются прямо внутри транзакции.
Выполнение слушателей сразу или даже отправка их в очередь делают транзакции более долгими и хрупкими.
А самое страшное, что событие вроде **SurveyCreated**, может быть вызвано, но дальше в транзакции будет ошибка и вся она будет откачена назад.
Письмо же пользователю об опросе, который даже не создался, все равно будет отправлено.
Я нашел пару пакетов для Laravel, которые ловят все эти события, хранят их временно, и выполняют только после того, как транзакция будет успешно завершена (гуглите "Laravel transactional events").
Да, они решают множество из этих проблем, но всё это выглядит так неестественно!
Простая идея генерировать нормальное бизнес-событие **SurveyCreated** после успешной транзакции намного лучше.

## Сущности как поля классов-событий
Я часто вижу как Eloquent-сущности используются напрямую в полях событий:

```php
final class SurveyCreated
{
    public function __construct(
        public readonly Survey $survey
    ) {}
}

final class SurveyService
{
    public function create(SurveyCreateDto $request)
    {
        // ...
        $survey = new Survey();
        // ...
        $this->dispatcher->dispatch(new SurveyCreated($survey));
    }
}

final class SendSurveyCreatedEmailListener implements ShouldQueue
{
    public function handle(SurveyCreated $event)
    {
        // ...
        foreach($event->survey->options as $option)
        {...}
    }
}
```
Это простой пример слушателя, который использует значения **HasMany**-отношения.
Этот код работает. Когда выполняется код **$event->survey->options** Eloquent делает запрос в базу данных и получает все варианты ответа.
Другой пример:

```php
final class SurveyOptionAdded
{
    public function __construct(
        public readonly Survey $survey
    ) {}
}

final class SurveyService
{
public function addOption(SurveyAddOptionDto $request)
{
    $survey = Survey::findOrFail($request->getSurveyId());
    
    if($survey->options->count() >= Survey::MAX_POSSIBLE_OPTIONS) {
        throw new BusinessException('Max options amount exceeded');
    }

    $survey->options()->create(...);
    
    $this->dispatcher->dispatch(new SurveyOptionAdded($survey));
}
}

final class SomeListener implements ShouldQueue
{
    public function handle(SurveyOptionAdded $event)
    {
        // ...
        foreach($event->survey->options as $option)
        {...}
    }
}
```

А вот тут уже не все хорошо.
Когда сервисный класс проверяет количество вариантов ответа, он получает свежую коллекцию текущих вариантов ответа данного опроса.
Потом он добавляет новый вариант ответа, вызвав **$survey->options()->create(...)**;
Дальше, слушатель, выполняя **$event->survey->options** получает старую версию вариантов ответа, без новосозданной.
Это поведение Eloquent, который имеет два механизма работы с отношениями. Метод **options()** и псевдо-поле **options**, которое вроде бы и соответствует этому методу, но хранит свою версию данных.
Поэтому, передавая сущность в события, разработчик должен озаботиться консистентностью значений в отношениях, например вызвав:

```php
$survey->load('options');
```

до передачи объекта в событие.
Это все делает код приложения весьма хрупким. Его легко сломать неосторожной передачей объекта в событие.
Намного проще просто передавать **id** сущности.
Каждый слушатель всегда может получить свежую версию сущности, запросив её из базы данных по этому **id**.

## Пара слов в конце главы

С развитием приложения, а особенно с ростом нагрузки на сервера, появляется естественное желание сократить время ответа сервера и время выполнения транзакций базы данных.

Все тяжелые действия лучше выполнить в другом потоке. Как правило, с помощью очередей. В этом нет какой-то серьезной сложности. Вполне достаточно организованно и скрупулезно подойти к данному вопросу. В сложном кейсе, когда количество действий и пост-действий становится большим, события и настройка их слушателей сильно помогает организовать все не превращая код приложения в хаос.

С транзакциями также нет какой-либо сложности, но неявная и скрытая логика, такая как события **Eloquent**-моделей, могут внести неразбериху в этот процесс. Явные события, вызываемые в контролируемых местах, намного более стабильная и управляемая стратегия. Событие **UserBanned** явно выражает, что произошло, в отличие от сильно более общего **UserSaved**. 