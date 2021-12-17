# Обработка ошибок

Язык С, который дал основу синтаксиса для многих современных языков, имеет простую конвенцию для ошибок.
Если функция что-то возвращает, но не может вернуть из-за ошибки, она возвращает null.
Если функция выполняет какую-то задачу, не возвращая никакого результата, то в случае успеха она возвращает 0, а в случае ошибки -1 или какой-нибудь код ошибки.
Много PHP разработчиков полюбили такую простоту и используют те же принципы.
Код может выглядеть так:

```php
final class ChangeUserPasswordDto
{
    private $userId;
    private $oldPassword;
    private $newPassword;
    
    public function __construct(
        int $userId, string $oldPassword, string $newPassword) 
    {
        $this->userId = $userId; 
        $this->oldPassword = $oldPassword;
        $this->newPassword = $newPassword;
    }
    
    public function getUserId(): int
    {
        return $this->userId;
    }
    
    public function getOldPassword(): string
    {
        return $this->oldPassword;
    }
    
    public function getNewPassword(): string
    {
        return $this->newPassword;
    }
}

final class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $command): bool
    {
        $user = User::find($command->getUserId());
        if($user === null) {
            return false; // пользователь не найден
        }
        
        if(!password_verify($command->getOldPassword(), 
                            $user->password)) {
            return false; // старый пароль неверный
        }
        
        $user->password = password_hash($command->getNewPassword());
        return $user->save();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        if($service->changePassword($request->getDto())) {
            // возвращаем успешный ответ
        } else {
            // возвращаем ответ с ошибкой
        }
    }
}
```
Ну, по крайней мере, это работает.
Но что, если пользователь хочет узнать в чем причина ошибки?
Комментарии рядом с `return false` бесполезны во время выполнения кода.
Можно попробовать коды ошибки, но часто кроме кода нужна и дополнительная информация для пользователя.
Попробуем создать специальный класс результата функции:

```php
final class FunctionResult
{
    /** @var bool */
    public $success;
    
    /** @var mixed */
    public $returnValue;
    
    /** @var string */
    public $errorMessage;
    
    private function __construct() {}
    
    public static function success(
        $returnValue = null): FunctionResult
    {
        $result = new self();
        $result->success = true;
        $result->returnValue = $returnValue;
        
        return $result;
    }
    
    public static function error(
        string $errorMessage): FunctionResult
    {
        $result = new self();
        $result->success = false;
        $result->errorMessage = $errorMessage;
        
        return $result;
    }
}
```
Конструктор этого класса приватный, поэтому все объекты могут быть созданы только с помощью статических методов **FunctionResult::success** и **FunctionResult::error**. 
Этот простенький трюк называется "именованные конструкторы".

```php
return FunctionResult::error("Something is wrong");
```
Выглядит намного проще и информативнее, чем

```php
return new FunctionResult(false, null, "Something is wrong");
```
Код будет выглядеть так:

```php
class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $command): FunctionResult
    {
        $user = User::find($command->getUserId());
        if($user === null) {
            return FunctionResult::error("User was not found");
        }
        
        if(!password_verify($command->getOldPassword(), 
                            $user->password)) {
            return FunctionResult::error("Old password isn't valid");
        }
        
        $user->password = password_hash($command->getNewPassword());
        
        $databaseSaveResult = $user->save();
                
        if(!$databaseSaveResult->success) {
            return FunctionResult::error("Database error");
        }
    
        return FunctionResult::success();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                     ChangeUserPasswordRequest $request)
    {
        $result = $service->changePassword($request->getDto());
        
        if($result->success) {
            // возвращаем успешный ответ
        } else {
            // возвращаем ответ с ошибкой
            // с текстом $result->errorMessage
        }
    }
}
```
Каждый метод (даже save() Eloquent модели в этом воображаемом мире) возвращает объект **FunctionResult** с полной информацией о том, как завершилось выполнение функции.
Когда я показывал этот пример на одном семинаре один слушатель сказал: "Зачем так делать? Есть же исключения!"
Да, исключения (exceptions) есть, но дайте лишь показать пример из языка Go:

```go
f, err := os.Open("filename.ext")
if err != nil {
    log.Fatal(err)
}
// do something with the open *File f
```

Обработка ошибок там реализована примерно так же. Без исключений. Популярность языка растёт, поэтому без исключений вполне можно жить.
Однако, чтобы продолжать использовать класс **FunctionResult** придётся реализовать стек вызовов функций, необходимый для отлавливания ошибок в будущем и корректное логирование каждой ошибки.
Все приложение будет состоять из проверок **if($result->success)**.
Не очень похоже на код моей мечты... 
Мне нравится код, который просто описывает действия, не проверяя состояние ошибки на каждом шагу. 
Попробуем использовать исключения.

## Исключения (Exceptions)

Когда пользователь просит приложение выполнить действие (зарегистрироваться или отменить заказ), приложение может выполнить его или нет.
Во втором случае, причин может быть множество.
Одним из лучших иллюстраций этого является список кодов HTTP-ответа.
Там есть коды 2хх и 3хх для успешных ответов, таких как 200 Ok или 302 Found.
Коды 4xx и 5xx нужны для неуспешных ответов, но они разные.

* 4xx для ошибок клиента: 400 Bad Request, 401 Unauthorized, 403 Forbidden, и т.д.
* 5xx для ошибок сервера: 500 Internal Server Error, 503 Service Unavailable, и т.д.

Соответственно, все ошибки валидации, авторизации, не найденные сущности и попытки изменить пароль с неверным старым паролем - это ошибки клиента.
Недоступность стороннего API, ошибка хранилища файлов или проблемы со связью с базой данных - это ошибки сервера.

Есть две противоборствующие школы обработок ошибок:

1. Девизом школы Аскетов Исключения является "Исключения только для исключительных ситуаций". Любое исключение считают вещью весьма необычной, способной произойти только из-за событий непреодолимой силы (отказ бд или файловой системы) и почти все исключения превращаются в 500-тые ответы сервера. Для ситуаций с неверно введённым email или неправильным паролем они используют что-то вроде объекта **FunctionResult**.
2. Адепты же школы Единого Верного Пути считают любую негативную ситуацию, т.е. ситуацию, которая не даёт выполнить действие пользователя, исключением.

Код аскетов, как и их девиз, выглядит более логично, но ошибки клиента придётся постоянно протаскивать наверх, как в примерах выше, из функций к тем, кто их вызвал, из Слоя приложения в контроллеры и т.д.
Код же их противников имеет унифицированный алгоритм работы с любой ошибкой (просто выбросить исключение) и более чистый код, поскольку не надо проверять результаты методов на ошибочность.
Есть только один вариант выполнения запроса, который приводит к успеху: приложение получило валидные данные, сущность пользователя загружена из базы данных, старый пароль совпал, поменяли пароль на новый и сохранили все в базе.
Любой шаг в сторону от этого единого пути должен вызывать исключение.
Юзер ввёл невалидные данные - исключение, этому пользователю нельзя выполнить это действие - исключение, упал сервер с базой данных - разумеется, тоже исключение.
Проблемой Единого Верного Пути является то, что где-то нужно будет отделить ошибки клиента от ошибок сервера, поскольку ответы мы должны сгенерировать разные (помните про 400-ые и 500-ые коды ответов?) да и логироваться такие ошибки должны по-разному.

Сложно сказать какой из путей предпочтительнее.
Когда приложение только-только обзавелось отдельным слоем Приложения, второй путь кажется более приятным.
Код чище, в любом приватном методе сервисного класса если что-то не понравилось можно просто выбросить исключение и оно сразу дойдёт до адресата.
Однако если приложение будет расти дальше, например будет создан еще и Доменный слой, то это увлечение исключениями может оказаться вредным.
Некоторые из них, будучи выброшенными, но не пойманными на нужном уровне могут быть проинтерпретированы неверно на более высоком уровне. 
Количество try-catch блоков начнёт расти и код уже не будет таким чистым.

Laravel выбрасывает исключения для 404 ошибки, для ошибки доступа (код 403) да и вообще имеет класс **HttpException** в котором можно указать HTTP-код ошибки.
Поэтому, в этой книге я тоже выберу второй вариант и буду генерировать исключения при любых проблемах.

Пишем код с исключениями:

```php
class UserService
{
    public function changePassword(
        ChangeUserPasswordDto $command): void
    {
        $user = User::findOrFail($command->getUserId());
        
        if(!password_verify($command->getOldPassword(), 
                            $user->password)) {
            throw new \Exception("Old password is not valid");
        }
        
        $user->password = password_hash($command->getNewPassword());
        
        $user->saveOrFail();
    }
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        try {
            $service->changePassword($request->getDto());
        } catch(\Throwable $e) {
            // log error
            // return failure web response with $e->getMessage();
        }
        
        // return success web response
    }
}
```
Даже на таком простом примере видно, что код Метода **UserService::changePassword** стал намного чище.
Любой шаг в сторону от основной ветки выполнения вызывает исключение, которое ловится в контроллере.
Eloquent тоже имеет методы для работы в таком стиле: **findOrFail()**, **firstOrFail()** и кое-какие другие ***OrFail()** методы.
Правда этот код все ещё не без проблем:

1. **Exception::getMessage()** не самое лучшее сообщение для того, чтобы показывать пользователю. 
Сообщение "Old password is not valid" ещё неплохо, но, например, "Server Has Gone Away (error 2006)" точно нет.
2. Любые серверные ошибки должны быть записаны в лог.
Мелкие приложения используют лог-файлы.
Когда приложение становится популярным, исключения могут происходить каждую секунду.
Некоторые исключения сигнализируют о проблемах в коде и должны быть исправлены немедленно.
Некоторые исключения являются нормой: интернет не идеален, запросы в самые стабильные API один раз из миллиона могут заканчиваться неудачей.
Однако если частота таких ошибок резко возрастает, то разработчики должны среагировать тоже.
В таких случаях, когда контроль за ошибками требует много внимания, лучше использовать специализированные сервисы, которые позволят группировать исключения и работать с ними намного удобнее.
Если интересно, можете просто погуглить "error monitoring services" и найдёте несколько таких сервисов.
Большие компании строят свои специализированные решения для записи и анализа логов со всех своих серверов (часто на основе популярного на момент написания книги стэка ELK: Elastic, LogStash, Kibana).
Некоторые компании не логируют ошибки клиента. 
Некоторые логируют, но в отдельных хранилищах.
В любом случае, для любого приложения необходимо четко разделять ошибки сервера и клиента.

## Базовый класс исключения

Первый шаг - создать базовый класс для всех исключений бизнес-логики, таких как "Старый пароль неверен".
В PHP есть класс **\DomainException**, который мог бы быть использован с этой целью, но он уже используется в других местах, например в сторонних библиотеках и это может привести к путанице.
Проще создать свой класс, скажем **BusinessException**.

```php
class BusinessException extends \Exception
{
    /**
    * @var string 
    */
    private $userMessage;
    
    public function __construct(string $userMessage)
    {
        $this->userMessage = $userMessage;
        parent::__construct("Business exception");
    }
    
    public function getUserMessage(): string
    {
        return $this->userMessage; 
    }
}

// Теперь ошибка верификации старого пароля вызовет исключение 

if(!password_verify($command->getOldPassword(), $user->password)) {
    throw new BusinessException("Old password is not valid");
}

final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        try {
            $service->changePassword($request->getDto());
        } catch(BusinessException $e) {
            // вернуть ошибочный ответ
            // с одним из 400-ых кодов
            // с $e->getUserMessage();
        } catch(\Throwable $e) {
            // залогировать ошибку
            
            // вернуть ошибочный ответ (с кодом 500)
            // с текстом "Houston, we have a problem"
            // Не возвращая реальный текст ошибки
        }
        
        // вернуть успешный ответ
    }
}
```
Этот код ловит **BusinessException** и показывает его сообщение пользователю.
Другие исключения покажут некое "Внутренняя ошибка, мы работаем над этим" и исключение будет отправлено в лог.
Код работает корректно, но секция **catch** будет повторена один в один в каждом методе каждого контроллера.
Стоит вынести логику обработки исключений на более высокий уровень.

## Глобальный обработчик

В Laravel (как и почти во всех фреймворках) есть глобальный обработчик исключений и, как ни странно, здесь весьма удобно обрабатывать почти все исключения нашего приложения.
В Laravel класс **app/Exceptions/Handler.php** реализует две очень близкие ответственности: логирование исключений и сообщение пользователям о них.

```php

namespace App\Exceptions;

class Handler extends ExceptionHandler
{
    protected $dontReport = [
        // Это означает, что BusinessException
        // не будет логироваться
        // но будет показан пользователю
        BusinessException::class,
    ];

    public function report(Exception $e)
    {
        if ($this->shouldReport($e))
        {
            // Это отличное место для
            // интеграции сторонних сервисов
            // для мониторинга ошибок
        }

        // это залогирует исключение 
        // по умолчанию в файл laravel.log
        parent::report($e);
    }

    public function render($request, Exception $e)
    {
        if ($e instanceof BusinessException)
        {
            if($request->ajax())
            {
                $json = [
                    'success' => false,
                    'error' => $e->getUserMessage(),
                ];

                return response()->json($json, 400);
            }
            else
            {
                return redirect()->back()
                       ->withInput()
                       ->withErrors([
                           'error' => trans($e->getUserMessage())]);
            }
        }

        // Стандартный показ ошибки
        // такой как страница 404
        // или страница "Oops" для 500 ошибок
        return parent::render($request, $e);
    }
}
```
Простой пример глобального обработчика.
Метод **report** может быть использован для дополнительного логирования.
Вся **catch** секция из контроллера переехала в метод **render**.
Здесь все ошибки логики будут отловлены и будут сгенерированы правильные сообщения для пользователя.
Для ошибок консоли есть незадокументированный метод **renderForConsole**(**$output**, **Exception** **$e**), который тоже можно перекрыть в этом классе.
Посмотрите на контроллер:

```php
final class UserController
{
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        $service->changePassword($request->getDto());
                
        // возвращаем успешный ответ
    }
}
```

Ну не красота?

## Проверяемые и непроверяемые исключения

Закройте глаза. Сейчас я буду вещать о высоких материях, которые в конце концов окажутся бесполезными. Представьте себе берег моря и метод **UserService::changePassword**.
Подумайте какие ошибки там могут возникнуть?

* **Illuminate\Database\Eloquent\ModelNotFoundException** если пользователя с таким **id** не существует
* **Illuminate\Database\QueryException** если запрос в базу данных не может быть выполнен
* **App\Exceptions\BusinessException** если старый пароль неверен
* **\TypeError** если где-то глубоко внутри кода функция **foo**(**SomeClass** **$x**) получит параметр **$x** с другим типом
* **\Error** если **$var->method()** будет вызван, когда переменная **$var** == **null**
* еще много других исключений

С точки зрения вызывающего этот метод, некоторые из этих ошибок, такие как **Error**, **TypeError**, **QueryException**, абсолютно вне контекста.
Какой-нибудь HTTP-контроллер вообще не знает, что с этими ошибками делать.
Единственное, что он может сделать - показать пользователю "Произошло что-то плохое и я не знаю, что с этим делать".
Но некоторые из них имеют смысл для него.
**BusinessException** говорит о том, что что-то не так с логикой и там есть сообщения прямо для пользователя и контроллер точно знает, что с этим исключением делать.
То же самое можно сказать про **ModelNotFoundException**. Контроллер может показать 404 ошибку на это.
Да, мы вынесли все это из контроллеров в глобальный обработчик, но это не важно.
Итак, два типа ошибок: 

1. Ошибки, которые понятны вызывающему коду и могут быть эффективно обработаны там
2. Другие ошибки

Первые ошибки хорошо бы обработать там, где этот метод вызывается, а вторые можно и пробросить выше.
Запомним это и взглянем на язык Java.

```java
public class Foo
{
    public void bar()
    {
        throw new Exception("test");
    }
}
```
Этот код даже не скомпилируется.
Сообщение компилятора: "Error:(5, 9) java: unreported exception java.lang.Exception; must be caught or declared to be thrown"
Есть два способа исправить это. Поймать его:

```java
public class Foo
{
    public void bar()
    {
        try {
            throw new Exception("test");
        } catch(Exception e) {
            // do something
        }
    }
}
```
Или описать исключение в сигнатуре метода:

```java
public class Foo
{
    public void bar() throws Exception
    {
        throw new Exception("test");
    }
}
```
В этом случае, каждый код вызывающий метод **bar** будет вынужден также что-то делать с этим исключением:

```java
public class FooCaller
{
    public void caller() throws Exception
    {
        (new Foo)->bar();
    }
    
    public void caller2()
    {
        try {
            (new Foo)->bar();
        } catch(Exception e) {
            // do something
        }
    }
}
```
Разумеется, работать так с **каждым** исключение будет той еще пыткой.
В Java есть **проверяемые** исключения, которые обязаны быть пойманы или объявлены в сигнатуре, и **непроверяемые**, которые могут быть выброшены без всяких дополнительных условий.
Взглянем на корень дерева классов исключений в Java (PHP, начиная с седьмой версии, имеет абсолютно такое же):

```
          Throwable(checked)
         /         \
Error(unchecked)  Exception(checked)
                        \
                      RuntimeException(unchecked)
```
**Throwable**, **Exception** и все их наследники - проверяемые исключения.
Кроме **Error**, **RuntimeException** и всех их наследников. Их можно выбросить везде и ничего за это не будет.

```java
public class File 
{
    public String getCanonicalPath() throws IOException {
        //...
    }
}
```
Что сигнатура метода **getCanonicalPath** говорит разработчику?
Там нет никаких параметров, возвращает строку, может выбросить исключение **IOException**, а также любое непроверяемое исключение.
Возвращаясь к двум типам ошибок:

1. Ошибки, которые понятны вызывающему коду и могут быть эффективно обработаны там
2. Другие ошибки

Проверяемые исключения созданы для ошибок первого типа.
Непроверяемые - для второго.
Вызывающий код может что-то сделать с проверяемыми исключениями и эта строгость обязывает его сделать это.
Все это приводит к более корректной обработке ошибок.

Хорошо, в Java это есть, в PHP - нет.
Почему я все ещё об этом говорю?
IDE, которое я использую, PhpStorm, имитирует поведение Java.

```php
class Foo
{
    public function bar()
    {
        throw new Exception();
    }
}
```
PhpStorm подсветит 'throw new Exception();' с предупреждением: 'Unhandled Exception'.
И есть два пути избавиться от этого:

1. Поймать исключение
2. Описать его в тэге @throws phpDoc-комментария метода: 

```php
class Foo
{
    /**
     * @throws Exception
     */
    public function bar()
    {
        throw new Exception();
    }
}
```

Список непроверяемых классов конфигурируется. 
По умолчанию он выглядит так: **\Error**, **\RuntimeException** и **\LogicException**.
Их можно выбрасывать не опасаясь предупреждений.

Со всей этой информацией можно попробовать спроектировать структуру классов исключения для приложения.
Я бы хотел информировать код, вызывающий **UserService::changePassword** про ошибки:

1. **ModelNotFoundException**, когда пользователь с таким **id** не найден
2. **BusinessException**, эта ошибка содержит сообщение, предназначенное для пользователя и может быть обработано сразу.
Все остальные ошибки могут быть обработаны позже.
Итак, в идеальном мире:

```php
class ModelNotFoundException extends \Exception
{...}

class BusinessException extends \Exception
{...}

final class UserService
{
    /**
     * @param ChangeUserPasswordDto $command
     * @throws ModelNotFoundException
     * @throws BusinessException
     */
    public function changePassword(
        ChangeUserPasswordDto $command): void
    {...}
}
```
Но мы уже вынесли всю логику обработки ошибок в глобальный обработчик, поэтому придется копировать все эти **@throws** тэги в методе контроллера:

```php
final class UserController
{
    /**
     * @param UserService $service
     * @param Request $request
     * @throws ModelNotFoundException
     * @throws BusinessException
     */
    public function changePassword(UserService $service, 
                        ChangeUserPasswordRequest $request)
    {
        $service->changePassword($request->getDto());
                
        // возвращаем успешный ответ
    }
}
```
Не очень удобно. Даже если учесть, что PhpStorm умеет генерировать все эти тэги автоматически.
Возвращаясь к нашему неидеальному миру: Класс **ModelNotFoundException** в Laravel уже отнаследован от **\RuntimeException**. 
Соответственно, он непроверяемый по умолчанию.
Это имеет смысл, поскольку глубоко внутри собственного обработчика ошибок Laravel обрабатывает эти исключения сам.
Поэтому, в нашем текущем положении, стоит тоже пойти на такую сделку с совестью:

```php
class BusinessException extends \RuntimeException
{...}
```

и забыть про тэги **@throws** держа в голове то, что все исключения **BusinessException** будут обработаны в глобальном обработчике.

Это одна из главных причин почему новые языки не имеют такую фичу с проверяемыми исключениями и большинство Java-разработчиков не любят их.
Другая причина: некоторые библиотеки просто пишут "throws Exception" в своих методах.
"throws Exception" вообще не дает никакой полезной информации.
Это просто заставляет клиентский код повторять этот бесполезный "throws Exception" в своей сигнатуре.

Я вернусь к исключениям в главе про Доменный слой, когда этот подход с непроверяемыми исключениями станет не очень удобным.

## Пара слов в конце главы

Функция или метод, возвращающие более одного типа, могущие вернуть **null** или возвращающие булевое значение (хорошо все прошло или нет), делают вызывающий код грязным.
Возвращенное значение нужно будет проверять сразу после вызова.
Код с исключениями выглядит намного чище:

```php
// Без исключений
$user = User::find($command->getUserId());
if($user === null) {
    // обрабатываем ошибку
}

$user->doSomething();


// С исключением
$user = User::findOrFail($command->getUserId());
$user->doSomething();
```

С другой стороны, использование объектов как **FunctionResult** даёт разработчикам больший контроль над исполнением.
Например, **findOrFail** вызванное в неправильном месте в неправильное время заставит приложение показать пользователю 404ю ошибку вместо корректного сообщения об ошибке.
С исключениями надо всегда быть настороже.
