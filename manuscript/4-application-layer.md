# Слой Приложения

Продолжаем наш пример.
Приложение растёт и в форму регистрации добавились новые поля: дата рождения и опция согласия получения email-рассылки.

```php
public function store(
    Request $request, 
    ImageUploader $imageUploader) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    $avatarFileName = ...;    
    $imageUploader->upload(
        $avatarFileName, $request->file('avatar'));
        
    $user = new User();
    $user->email = $request['email'];
    $user->name = $request['name'];
    $user->avatarUrl = $avatarFileName;
    $user->subscribed = $request->has('subscribed');
    $user->birthDate = new DateTime($request['birthDate']);
    
    if(!$user->save()) {
        return redirect()->back()->withMessage('...');
    }
    
    \Email::send($user->email, 'Hi email');
        
    return redirect()->route('users');
}
```
Потом у приложения появляется API для мобильного приложения и регистрация пользователей должна быть реализована и там.
Давайте ещё нафантазируем некую консольную artisan-команду, которая импортирует пользователей и она тоже хочет их регистрировать.
И telegram-бот! 
В итоге, у приложения появилось несколько интерфейсов, кроме стандартного HTML и везде необходимо использовать такие действия как регистрация пользователя или публикация статьи.
Самое естественное решение здесь выделить общую логику работы с сущностью (User в данном примере) в отдельный класс.
Такие классы часто называют сервисными классами:

```php
final class UserService
{
    public function getById(...): User;
    public function getByEmail(...): User;

    public function create(...);
    public function ban(...);
    ...
}
```
Но множество интерфейсов приложения (API, Web, и т.д.) не являются единственной причиной создания сервисных классов.
Методы контроллеров начинают расти и обычно содержат две большие части:

```php
public function doSomething(Request $request, $id)
{
    $entity = Entity::find($id);
    
    if(!$entity) {
        abort(404);
    }
    
    if(count($request['options']) < 2) {
        return redirect()->back()->withMessage('...');
    }
    
    if($entity->something) {
        return redirect()->back()->withMessage('...');
    }
    
    \Db::transaction(function () use ($request, $entity) {
        $entity->someProperty = $request['someProperty'];
        
        foreach($request['options'] as $option) {
            //...
        }
        
        $entity->save();
    });
    
    return redirect()->...
}
```
Этот метод реализует как минимум две ответственности: логику работы с HTTP запросом/ответом и бизнес-логику.
Каждый раз когда разработчик меняет http-логику, он вынужден читать много кода бизнес-логики и наоборот.
Такой код сложнее дебажить и рефакторить, поэтому вынесение логики в сервис-классы тоже может быть хорошой идеей для этого проекта.

## Передача данных запроса

Начнем создавать класс **UserService**.
Первой проблемой будет передача данных запроса туда.
Некоторым методам не нужно много данных.
Например, для удаления статьи нужно только её id.
Однако, для таких действий, как регистрация пользователя, данных может понадобиться много.
Мы не можем использовать класс **Request**, поскольку он доступен только для web.
Попробуем простые массивы:

```php
final class UserService
{
    /** @var ImageUploader */
    private $imageUploader;
    
    /** @var EmailSender */
    private $emailSender;
    
    public function __construct(
        ImageUploader $imageUploader, EmailSender $emailSender) 
    {
        $this->imageUploader = $imageUploader;
        $this->emailSender = $emailSender;
    }
    
    public function create(array $request)
    {
        $avatarFileName = ...;
        $this->imageUploader->upload(
            $avatarFileName, $request['avatar']);

        $user = new User();
        $user->email = $request['email'];
        $user->name = $request['name'];
        $user->avatarUrl = $avatarFileName;
        $user->subscribed = isset($request['subscribed']);
        $user->birthDate = new DateTime($request['birthDate']);
        
        if(!$user->save()) {
            return false;
        }
        
        $this->emailSender->send($user->email, 'Hi email');
        
        return true;
    }
}

public function store(Request $request, UserService $userService) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    if(!$userService->create($request->all())) {
        return redirect()->back()->withMessage('...');
    }
    
    return redirect()->route('users');
}
```
Я просто вынес логику без каких-либо изменений и вижу проблему.
Когда мы попытаемся зарегистрировать пользователя из консоли, то код будет выглядеть примерно так:

```php
$data = [
    'email' => $email,
    'name' => $name,
    'avatar' => $avatarFile,
    'birthDate' => $birthDate->format('Y-m-d'),
];

if($subscribed) {
    $data['subscribed'] = true;
}

$userService->create($data);
```
Выглядит чересчур витиевато.
Просто вместе с кодом создания пользователя в сервис переехала и HTML-специфика передаваемых данных.
Булевы значения проверяются присутствием нужного ключа в массиве.
Значения даты получаются преобразованием из строк.
Использование такого формата данных весьма неудобно, если эти данные пришли не из HTML-формы.
Нам нужен новый формат данных. 
Более естественный, более удобный.
Шаблон **Data Transfer Object**(DTO) предлагает просто создавать объекты с нужными полями.

```php
final class UserCreateDto
{
    /** @var string */
    private $email;
    
    /** @var DateTime */
    private $birthDate;
    
    /** @var bool */
    private $subscribed;
    
    public function __construct(
        string $email, DateTime $birthDate, bool $subscribed) 
    {
        $this->email = $email;
        $this->birthDate = $birthDate;
        $this->subscribed = $subscribed;
    }
    
    public function getEmail(): string
    {
        return $this->email;
    }
        
    public function getBirthDate(): DateTime 
    {
        return $this->birthDate;
    }
    
    public function isSubscribed(): bool
    {
        return $this->subscribed;
    }
}
```

Частенько я слышу возражения вроде "Я не хочу создавать целый класс только для того, чтобы передать данные. Массивы справляются не хуже."
Это верно отчасти и для какого-то уровня сложности приложений создавать классы DTO, вероятно, не стоит.
Дальше в книге я приведу пару аргументов в пользу DTO, а здесь я лишь напишу, что в современных IDE создать такой класс - это написать его имя в диалоге создания класса, горячая клавиша создания конструктора (Alt-Ins > Constructor...), перечислить поля с типами в аргументах конструктора, горячая клавиша по которой PhpStorm создаст все нужные поля класса (Alt-Enter на параметрах конструктора > Initialize fields) и горячая клавиша создания методов-геттеров (Alt-Ins в теле класса > Getters...).
Меньше 30 секунд и класс DTO с полной типизацией создан. Заглядывать в этот класс разработчики будут крайне редко, поэтому какой-то сложности в поддержке приложения он не добавляет.

```php
final class UserService
{
    //...
    
    public function create(UserCreateDto $request)
    {
        $avatarFileName = ...;
        $this->imageUploader->upload(
            $avatarFileName, $request->getAvatarFile());

        $user = new User();
        $user->email = $request->getEmail();
        $user->name = $request->getName();
        $user->avatarUrl = $avatarFileName;
        $user->subscribed = $request->isSubscribed();
        $user->birthDate = $request->getBirthDate();
        
        if(!$user->save()) {
            return false;
        }
        
        $this->emailSender->send($user->email, 'Hi email');
        
        return true;
    }
}

public function store(Request $request, UserService $userService) 
{
    $this->validate($request, [
        'email' => 'required|email',
        'name' => 'required',
        'avatar' => 'required|image',
        'birthDate' => 'required|date',
    ]);
    
    $dto = new UserCreateDto(
        $request['email'], 
        new DateTime($request['birthDate']), 
        $request->has('subscribed'));
    
    if(!$userService->create($dto)) {
        return redirect()->back()->withMessage('...');
    }
    
    return redirect()->route('users');
}
```
Теперь это выглядит канонично.
Сервисный класс получает чистую DTO и выполняет действие.
Правда, метод контроллера теперь довольно большой. Конструктор DTO класса может быть весьма длинным.
Можно попробовать вынести логику работы с данными запроса оттуда.
В Laravel есть удобные классы для этого - **Form Requests**.

```php
final class UserCreateRequest extends FormRequest
{
    public function rules()
    {
        return [
            'email' => 'required|email',
            'name' => 'required',
            'avatar' => 'required|image',
            'birthDate' => 'required|date',
        ];
    }
    
    public function authorize()
    {
        return true;
    }
    
    public function getDto(): UserCreateDto
    {
        return new UserCreateDto(
            $this->get('email'), 
            new DateTime($this->get('birthDate')), 
            $this->has('subscribed'));
    }
}

final class UserController extends Controller
{
    public function store(
        UserCreateRequest $request, UserService $userService) 
    {        
        if(!$userService->create($request->getDto())) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('users');
    }
}
```
Если какой-либо класс просит класс, наследованный от **FormRequest** как зависимость, Laravel создаёт его и выполняет валидацию автоматически. 
В случае неверных данных метод **store** не будет выполняться, поэтому можно всегда быть уверенными в валидности данных в **UserCreateRequest**. 

Классы **Form request** содержат методы **rules** для валидации данных и **authorize** для авторизации действия.
**Form request** явно не лучшее место для реализации авторизации.
Авторизация часто требует контекста, например какой юзер, с какой сущностью, какое действие.
Это все можно делать в методе **authorize**, но в сервисном классе это делать намного проще, поскольку там уже будут все нужные данные да и не придется повторять тоже самое для API и других интерфейсов.
В моих проектах всегда есть базовый **FormRequest** класс, в котором метод **authorize** с одним оператором `return true;`, который позволяет забыть про авторизацию на этом уровне.  

## Работа с базой данных

Простой пример:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        $post = Post::find($id);
        
        if(!$post) {
            abort(404);
        }
        
        if(!$postService->publish($post)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}

final class PostService
{
    public function publish(Post $post)
    {
        $post->published = true;
        return $post->save();
    }
}
```
Публикация статьи - это пример простейшего не-CRUD действия и я буду использовать его часто.
Я буду переводить Post как Статья, чтобы не было "публикация публикации".
В примере все выглядит неплохо, но если попробовать вызвать метод сервиса из консоли, то нужно будет опять доставать сущность **Post** из базы данных.

```php
public function handle(PostService $postService)
{
    $post = Post::find(...);
    
    if(!$post) {
        $this->error(...);
        return;
    }
    
    if(!$postService->publish($post)) {
        $this->error(...);
    } else {
        $this->info(...);
    }
}
```
Это пример нарушения Принципа единственной ответственности (SRP) и высокой связанности (coupling).
Каждая часть приложения (в том числе и веб-контроллеры и консольные команды) работает с базой данных.
Каждое изменение, связанное с работой с базой данных, может повлечь изменения во всем приложении.
Иногда я вижу странные решения в виде метода **getById** в сервисах:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        $post = $postService->getById($id);
        
        if(!$postService->publish($post)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}
```
Этот код просто получает сущность и передаёт её в другой метод сервиса.
Метод контроллера абсолютно не нуждается в сущности **Post**.
Он может просто попросить сервис опубликовать статью.
Наиболее логичное и простое решение:

```php
class PostController
{
    public function publish($id, PostService $postService)
    {
        if(!$postService->publish($id)) {
            return redirect()->back()->withMessage('...');
        }
        
        return redirect()->route('posts');
    }
}

final class PostService
{
    public function publish(int $id)
    {
        $post = Post::find($id);
            
        if(!$post) {
            return false;
        }
        
        $post->published = true;
        
        return $post->save();
    }
}
```
Одно из главных преимуществ создания сервисных классов - это консолидация всей работы с бизнес-логикой и инфраструктурой, включая хранилища данных, такие как базы данных и файлы, в одном месте, оставляя Web, API, Console и другим интерфейсам работу исключительно со своими обязанностями.
Часть для работы с Web должна просто готовить данные для сервис классов и показывать результаты пользователю.
Тоже самое про другие интерфейсы.
Это Принцип единственной ответственности для слоёв.
Слоёв? Да. Все сервис-классы, которые прячут логику приложения внутри себя и предоставляют удобные методы для Web, API и других частей приложения, формируют структуру, которая имеет множество имён:

* **Сервисный слой** (Service layer), из-за **сервисных** классов.
* **Слой приложения** (Application layer), потому что он содержит всю логику приложения, исключая интерфейсы к нему.
* В GRASP этот слой называется **Слой контроллеров** (Controllers layer), поскольку сервисные классы там называются контроллерами.
* наверняка есть и другие названия.

В этой книге я буду называть его **Слоем приложения**. Потому что могу.

## Сервисные классы или классы команд?

Когда сущность большая, с кучей различных действий, сервисный класс для нее тоже будет большим.
Разные действия, такие как отредактировать статью или опубликовать её, требуют разных зависимостей.
Всем нужна база данных, но одни хотят посылать письма, другие - залить файл в хранилище, третьи - дёрнуть какое-нибудь API.
Количество параметров конструктора сервисного класса растёт довольно быстро, хотя каждое из них может использоваться в одном-двух методах.
Довольно быстро становится понятно, что класс занимается слишком многими делами.
Поэтому часто разработчики начинают создавать один класс на каждое действие с сущностями.

Насколько мне известно, стандарта на название таких классов тоже нет.
Я видел суффикс **UseCase** (например **PublishPostUseCase**), суффикс **Action** (**PublishPostAction**), но предпочитаю суффикс **Command**: **CreatePostCommand**, **PublishPostCommand**, **DeletePostCommand**.

```php
final class PublishPostCommand
{
    public function execute($id)
    {
        //...
    }
}
```
В шаблоне **Command Bus** суффикс**Command** используется для DTO-классов, а классы испольняющие команды называются **CommandHandler**.

```php
final class ChangeUserPasswordCommand
{
    //...
}

final class ChangeUserPasswordCommandHandler
{
    public function handle(
        ChangeUserPasswordCommand $command)
    {
        //...
    }
}

// или если один класс исполняет много команд

final class UserCommandHandler
{
    public function handleChangePassword(
        ChangeUserPasswordCommand $command)
    {
        //...
    }
}
```

В книге я, для простоты, буду использовать **Service**-классы.

Небольшая ремарка про длинные имена классов.
"**ChangeUserPasswordCommandHandler** - ого! Не слишком ли длинное имя? Не хочу писать его каждый раз!"
Полностью его написать придется всего один раз - при создании.
Дальше IDE будет полностью подсказывать его.
Хорошее, полностью описывающее класс, название намного важнее.
Каждый разработчик может сразу сказать что примерно делает класс **ChangeUserPasswordCommandHandler** не заглядывая внутрь него.