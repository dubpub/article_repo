Доброго времени суток! В этой статье я хотел бы рассмотреть многие вопросы - модные контроллеры, ViewModel, http-ресолвинг, аннотации и пакеты. Хотя она началась всего лишь с желания описать работу с подходом ViewModel, руки так и чесались поделиться чем-то большим. В статье много слов, много кода - кому интересно, добро пожаловать.

##Цель.
--
Итак нашей целья является написать гибкий контроллер, который поддерживал бы расширяемые аннотации и занимался бы ресолвингом вью-моделей на лету. И начнём мы с последнего - поговорим о моделях.

Что для нас модели в Laravel? Документация нас сразу отправляет смотреть, грызть и читать про Eloquent. Там мы видим как всё красиво - хочешь связи? объяви метод! хочешь ограничить результат "видимости" запроса? объяви метод! хочешь контсруктор запросов? без проблемм - он всё там же. И в итоге дочитав доконца о магии, которую предоставляет нам  Eloquent в колову приходит одна пугаящая мысль - всё прекрасно и легко, не будь это ActiveRecord. Я знаю, что многие разработчики считают, что, ActiveRecord - это априори антипаттерн, но это с какой стороны посмотреть. Все прекрасно понимают, что любой паттерн можно запросто превратить в антипаттерн при неправильном его использовании, и посмотрев у различных людей примеры кода с Eloquent это убеждение подтверждается. 

Первые костыли мы всегда наблюдаем при попытках задать модели значения по умолчанию - для этого у нас есть публичное свойство - ассоциативный массив $attributes. Можно частенько наблюдать такой код:

    class Entity extends \Eloquent
    {
        public $table = 'sometable';
        
        public function __construct(array $attributes = array())
        {
            $defaults = array('default' => 'attributes');
            $this->setRawAttributes($defaults, true);
            parent::__construct($attributes);
        }
    }
    
Ужасно не так ли? Причём ужасна не только инициализация аттрибутов, но и сам факт того, что, как ни крути, у вашей Eloquent-модели занят уже конструктор - вы всегда обязанны помнить о родителе в силу специфики изначальной реализации. Теперь если мы создадим наследованную от Entity модель без конструктора, в ней уже всегда будут аттрибуты по умолчанию, что указанны в родительском конструкторе. Я уверен, что "матёрый" читатель уже сказал, что можно вынести $defaults в protected свойство и получить модель вида:

    class Entity extends \Eloquent
    {
        public $table = 'sometable';
        protected $defaults = array('default' => 'attributes');
        
        public function __construct(array $attributes = array())
        {
            $this->setRawAttributes($this->defaults, true);
            parent::__construct($attributes);
        }
    }

Да, в этом куске кода мы вроде как ослабили связь значений по умолчанию, но теперь вы всегда должны будете помнить, что у вас в моделе родителя в $defaults может что-то уже сидеть итд итп. 

Что бы мы ещё могли ракритивать в Eloquent-моделях? Да опять всё те же аттрибуты - они инкапсулированны и будут пополняться/обновляться через __set() всякий раз, когда вы присваиваете свойству модели какое-либо значение. То есть нас абсолютно не устраивает инкапсуляция аттрибутов - она требует излишный контроль какими-то средствами: будь то $fillable или $guarded, будь то Observer, который перед сохранением модели будет проверять что сохранить, а что нет, будь то что угодно, что не попадает под значения "удобное" или "несвязанное". К тому же инкапсуляция аттрибутов выводит многих из себя, лишая людей тайпхинтинга. Если кто-то скажет "поставь себе ide-helper и сгенерируй описание модели через artisan", то я буду в растерянности - пологаться на docHere модели, которую бог-весть когда описали(ведь возможно описали всего один раз) - не самый лучший метод, да и при надобности с моделью всё равно не сможешь "порефлексировать".

Так что же модели в Laravel? Из всего вышесказанного я могу сделать вывод, что Eloquent отличная вещь для моего непринуждённого общения с бд, но не для ведения сущностей в проекте - высока цена (в рефлексии свыше 120 искоробочных методов и не факт что они понадобяться), неудобство для контекстного положения ну и конечно же жёсткая привязанность моделей данных к транспортному слою, ведь модель сохраняющая себе хороша, но только в очень маленьких проектах.

Сделав пару выводов я пришёл к тому, что слой моделей надо реализовавать отдельно от Eloquent, сделав так называемый ViewModel подход и написав "умный контроллер".

##Чего нужно достичь.


--

#oldtext

Первым делом хотелось бы рассмотреть гипотетический метод postCreate() типичного контроллера NewsController, который будет у нас отвечать за создание простой сущности News. Наш контроллер вооружён до зубов - его контруктор ждёт инъекцию INewsService, который отвечает за выборку и поставку данных, INewsStrategy - стратегия для CRUD операций над сущностью и сервиса валидаций. В этом методе мы наблюдаем обычную картину:

    public function postCreate()
    {
        $news = Input::get(); // или ::only(), или ::except()
        $validation = $this->validationService->news($news);
        
        $onFail = Redirect::to(URL::current())->withInput();
        
        if (!$validation->isValid()) {
            return $onFail->withErrors($validation->errors());
        }
        
        if (!$this->strategy->create($news)) {
            return $onFail->with('error', 'Failed to save');
        }
        
        return Redirect::route("news.index");
    }
    
Это неплохо написанный код и моей целью не является критика такого подхода, а скорее внесение небольшого синтаксического сахара. Хотя если честно, мне не нравиться использовать фасад Input без конца и на то есть свои причины - вызов <code>Input::get()</code> без параметров заставляет излишне извращаться со свойствоми $guarded, $hidden, $fillable и подобными в eloquent моделях, которое может варироваться от контекста, а использование <code>Input::except()</code> или <code>Input::only()</code> не выглядит гибким и комфортным.

И так, как же я вижу этот метод:

    public function postCreate(NewsViewModel $model)
    {
        $failed = Redirect::to(URL::current())->withInput();
        
        if (!$model->isValid()) {
            return $failed->withErrors($model->getValidator());
        }
        
        if (!$this->strategy->create($model->toModel())) {
            return $failed->with('error', 'Failed to save');
        }
        
        return Redirect::route("news.index");
    }

Во-первых, как вы видите - хочетелось бы магической инъекции ViewModel и сборка её свойств на лету. Во-вторых выше вы можете наблюдать полноценную контекстно-ориентированную модель никак не связанную со слоем транспорта данных. То есть мы получаем полноценный профит.

##Реализация.

Для начала, было бы неплохо разбить задачу на несколько частей - нам понядобяться интерфейс ViewModel, его частичная абстрактная реализация, некий Resolver, которй бы справлялся с инъекцией моделей в методы, ну и прослойка, отвечающая за наполнение свойствами ViewModel, реализация которых имеет место быть во множественных вариациях.

###ViewModel.

Перед тем как преступить в ViewModel, было бы неплохо описать интерфейс IViewModel, потому что у наши вариации ViewModel могут базироваться в зависимости от целей, поэтому планирование минимального интерфейса просто необходимо. Мне очень нравиться так же использование интерфесоф ArrayableInterface и JsonableInterface, поэтому мы обяжем наш IViewModel их реализовывать. Помимо было бы неплохо реализовать следующие возможности: 

<ul>
    <li>
        у нашей модели должна быть возможность привести себя к базовой Eloquent-сущности. За эту возможность будет отвечать <code>toModel()</code>;
    </li>
    <li>
        так как наша ViewModel будет ориентирована на контекстное использование и будет отвечать за входящие данные - было бы неплохо добавить метод <code>isValid($isNew = true)</code>. Параметр <code>$isNew</code> будет необязательным, но создаст возможность в  контексте сказать, что валидируемая модель не новая и использовать другие правила;
    </li>
    <li>
        у нашей модели должен быть метод <code>fill($attributes = array())</code> - для удобства заполнения модели. 
    </li>
</ul>

    namespace Application\Utils\ViewModel\Interfaces;
    
    use Illuminate\Support\Contracts\ArrayableInterface;
    use Illuminate\Support\Contracts\JsonableInterface;

    interface IViewModel implements ArrayableInterface, JsonableInterface
    {
        public function fill($attributes = array());
        public function isValid($isNew = true);
        public function toModel($isNew = true);
    }

Для приёма и обработки файлов можно будет сделать отдельный интерфейс IFileViewModel, который будет содержать метод только save() и реализовывать IViewModel:

    namespace Application\Utils\ViewModel\Interfaces;

    interface IFileViewModel implements IViewModel
    {
        public function save();
    }

Итак план составлен - займёмся абстракцией нашей ViewModel и подготовим часть механизма, необходимого для работы нашей затеи. 

    namespace Application\Utils\ViewModel;
    
    use Application\Utils\ViewModel\Interfaces\IViewModel;
    use ReflectionClass,
        ReflectionProperty;
    
    abstract class ViewModel implements IViewModel
    {
        protected $_validator = null;
        protected $attributes = array();
        
        abstract protected function getBaseModel($attributes = array(), $isNew = true);
        abstract protected function getValidationObject($input, $isNew = true);

        protected function extractAttributes()
        {
            if (!count($this->attributes)) {
                $reflection = new ReflectionClass($this);
    
                foreach($reflection->getProperties(ReflectionProperty::IS_PUBLIC) as $property)
                {
                    $this->attributes[] = $property->name;
                }
            }
            
            return $this->attributes;
        }
        
        protected function extractValidatior($isNew = true)
        {
            if (is_null($this->_validator)) {
                $this->_validator = $this->getValidationObject($this->toArray(), $isNew);
            }
            
            return $this->_validator;
        }

        public function toModel($isNew = true)
        {
            $model = $this->getBaseModel($this->toArray(), $isNew);
            return $model;
        }

        public function toArray()
        {
            $array = array();

            foreach($this->extractAttributes() as $attribute)
            {
                $array[$attribute] = $this->{$attribute};
            }

            return $array;
        }

        public function toJson($options = 0)
        {
            return json_encode($this->toArray(), $options);
        }

        public function isValid($isNew = true)
        {
            return !$this->extractValidator($isNew)->fails();
        }
        
        public function fill($attributes = array()) {
            foreach($this->extractAttributes() as $key) {
                if (isset($attributes[$key])) { 
                    $this->{$key} = $attributes[$key];
                }
            }
        }

        public function getValidator($isNew = true)
        {
            return $this->extractValidator($isNew);
        }
    }

Пройдёмся в кратце по первой части вышеизложенного механизма.

<ul>
    <li>
        Защищённый метод <code>extractAttributes()</code> будет извлекать из нашей модели public свойства ViewModel, так как подозреается, что они и есть наши аттрибуты;
    </li>
    <li>
        Метод <code>toModel($isNew = true)</code> будет вызывать абстрактный метод <code>getBaseModel($attributes = array(), $isNew = true)</code>, передавая в него текущие значения модели в виде ассоциативного массива и флаг, показывающий на то новали модель, или нет;
    </li>
    <li>
        Метод <code>toArray()</code> будет реализацией метода из ArrayableInterface. Он будет возвращать ассоциативный массив свойств-значений извлекая ключи модели из метода <code>extractAttributes()</code> и сохраняя их для избежания повторного вызова метода. 
    </li>
    <li>
        Метод <code>toJson($options = 0)</code> будет реализацией метода из JsonableInterface. Он будет приводить результат метода toArray() в json-объект. 
    </li>
</ul>

Теперь посмотрим на часть, отвечающую за валидацию:

<ul>
    <li>
        Защищённый метод <code>extractValidatior($isNew = true)</code> будет вызывать абстрактный метод <code>getValidationObject($attributes = array(), $isNew = true)</code>, передавая в него текущие значения модели в виде ассоциативного массива и флаг, показывающий на то новали модель, или нет и моментально сохранять результат. Предполагается, что результатом метода getValidationObject() будет объект вализации из компонентов Illuminate, чтобы в дальнейшем в случае невалидности нашей модели можно было бы вернуть MessageBag;
    </li>
    <li>
        Метод <code>isValid($isNew)</code> извлекать собранный образец объекта валидации и возращать t/f значения в зависимости от вализности объекта;
    </li>
    <li>
        Метод <code>getValidator()</code> будет работать как публичный getter для получения объекта валидации.
    </li>
</ul>

Во что же у нас вытекает этот код и с чем его едят? Давайте посмотрим как будет выглядеть ViewModel для сущности Post:
    
    namespace Data\ViewModels;
    
    use Application\DAL\Eloquent\Post;
    use Application\Utils\ViewModel\Interfaces\IViewModel;
    use Application\Utils\ViewModel\ViewModel;
    
    class PostViewModel extends ViewModel implements IViewModel
    {
        public $name;
        public $description;
        public $content;
        
        protected $validationService;
        
        public function __construct(IValidationService $validationService)
        {
            $this->validationService = $validationService;
        }
        
        protected function getBaseModel($attributes = array(), $isNew = true)
        {
            $model = new Post($attributes);
            $model->exists = !$isNew;
            return $model;
        }
        
        protected function getValidationObject($input = array(), $isNew = true)
        {
            $validation = $this->validationService->post($input);
            
            //if ($isNew) {
                //$validation->sometimes(...);
            //}
            
            return $validation;
        }
    }

Как вы видите в этом классе уютно помещается контекстная валидация. Если кто ещё не понял, то класс Post, возвращаемый из метода <code>getBaseModel</code> это образец или наследник модели eloquent. 
Как вы могли заметить я использовал в конструкции интерфейс <code>IValidationService</code>, который будет прилетать к нам из IoC-контейнера, что плавно переносит нас ко второй части реализации нашего механизма.

Написания одной ViewModel маловато для того, чтобы она прилетала в метод контроллера, поэтому сейчас я покажу вам как реализовать инъекцию не прибегая к route binding, указанному в документации.


###Resolver

Мне не нравится как в Laravel устроен model binding - слишком много придётся описывать для большого колличества моделей в проекте, поэтому хотелось бы обзавестись механизмом как в ASP.NET MVC - просто указывать класс в методе и принимать модель без излишних описаний в роутере, который и подавно не должена знать, ничего о слое данных. Так же хотелось бы исправить большой недостаток IoC-контейнера Laravel - обзавестись механизмом разрешения зависимостей не только в конструкторах классов, но и в методах и даже в closure. Это решение мы назовём Resolver.

С появлением Reflection Api php преобразился, но в силу того, что php относится к duck-typing языкам, видимо поэтому разработчики решили не добавлять методы по извлечению типов аргументов, которые ждёт метод, или функция, поэтому наш Resolver должен уметь извлекать типы аргументов.

    namespace Application\Utils\Resolver;
    
    use ReflectionMethod;
    use ReflectionFunction;
    
    class Resolver
    {
        protected function resolve($reflectionParams, $params = array())
        {
            for($i = 0; $i < count($reflectionParams); $i++)
            {
                $exp = '/\[\s\<\w+?>\s([\w\\\\]+)/s';
                $match = preg_match($exp, $reflectionParams[$i]->__toString(), $matches);
    
                if (isset($params[$i])) continue;
    
                if (isset($matches[1]) && !$reflectionParams[$i]->isArray() && is_string($matches[1]) && (class_exists($matches[1]) || \App::bound($matches[1])))
                {
                    $params[$i] = \App ::make($matches[1]);
                }
    
            }
    
            return $params;
        }
    
        public function method($class, $method, $params = array())
        {
            $reflection = new ReflectionMethod($class, $method);
            $resolved = self::resolve($reflection->getParameters(), $params);
            return call_user_func_array([$class, $method], $resolved);
        }
    
        public function methodToClosure($class, $method)
        {
            return function () use ($class, $method)
            {
                return $this->method($class, $method, func_get_args());
            };
        }
    
        public function closure(Closure $closure, $params = array())
        {
            $reflection = new \ReflectionFunction($closure);
            $resolved = self::resolve($reflection->getParameters(), $params);
            return call_user_func_array($closure, $resolved);
        }
    
        public function makeClosure(Closure $closure)
        {
            return function () use ($closure)
            {
                return $this->closure($closure, func_get_args());
            };
        }
    }

Что же это такое? Resolver смотрит reflectionParametrs метода или функции, и пропускает их через IoC-контейнер Laravel в том случае, если они являются классом, или если это интерфейс, под который контейнере зарегистрированна реализация.

Рассмотрим примеры его работы, так как помимо реализации ViewModel подхода это полезный механизм в повседневной жизни. Допустим у нас есть модель безопасности/учётной записи SecurityAccount, указанная в конфиге auth.php, которая реализует какой-то ISecurityUser.

    interface ISecurityUser 
    {
        pubic function getUserName();
        pubic function isAdmin();
    }
    
    class SecurityAccount extends EloquentUserModel implements ISecurityUser 
    {
        pubic function getUserName()
        {
            return "{$this->firstname} {$this->lastname}";
        }
        
        public function isAdmin()
        {
            return $this->access()->can(Access::Admin);
        }
    } 
    
И вся эта красота будет у нас находится в IoC-контейнере фрэймворка.

    App::bind('ISecurityUser', function ()
    {
        return Auth::user();
    });


Рассмотрим как Resolver ведёт себя с closure. Для примера я напишу простой callable, который ждёт обязательный параметр, который нам известен заранее и необходим в использовании, и параметр, который нам хотелось бы получить из контейнера зависимостей. Этот callable будет складывать в строку и возвращать подставленное преветствие и юзернэйм:

    /**
    * наш callable - простой вызов предполагает $callback('Hi', Auth::user())
    */
    $callback = function ($string, ISecurityUser $injectedUser = null)
    {
        return "{$string}, {$injectedUser->getUserName()}!";
    };
    
    /**
    * Простое разрешение зависимостей
    */
    $resolver = new Application\Utils\Resolver\Resolver();
    $result = $resolver->closure($myCallback, array('Hi!')); //передаём closure и предопределённый массив параметров
    var_dump($result); // на выходе 'Hello, xxx yyy!'
    
    /**
    * Создние отложенного разрешения зависимостей
    */
    $resolver = new Resolver();
    $callback = $resolver->makeClosure($myCallback); //передаём closure
    var_dump($callback); // видим closure
    $result = $callback('Hi!'); //передаём в closure параметр строку 'Hi!'
    var_dump($result);  // на выходе 'Hello, xxx yyy!'
    
Теперь взглянем как будет вести себя Resolver с методами классов. Для примера это будет класс SomeClass и метод someMethod, который в точности похож на наш $callback из предыдущего примера:

    class SomeClass
    {
        public function someMethod ($string, ISecurityUser $injectedUser = null)
        {
            return "{$string}, {$injectedUser->getUserName()}!";
        }
    }
    
    /**
    *   Простой вызов с разрешением зависимостей
    */
    $resolver = new Resolver();
    $result = $resolver->method('SomeClass', 'someMethod', array('Hi!')); 
            // или $resolver->method(new SomeClass(), 'someMethod', array('Hi!'));
    var_dump($result); // на выходе 'Hello, xxx yyy!'

    /**
    *   Простой вызов с разрешением зависимостей
    */
    $resolver = new Resolver();
    $callback = $resolver->methodToClosure('SomeClass', 'someMethod'); 
                // или $resolver->methodToClosure(new SomeClass, 'someMethod');
    var_dump($callback); // closure
    $result = $callback(5);
    var_dump($result); // на выходе 'Hello, xxx yyy!'
    
