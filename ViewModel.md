Доброго времени суток! В этой статье я хотел бы рассмотреть подход к сбору и агрегации данных из request, или такое явление как ViewModel. 

##Цель.

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
    
####Примеры

Допустим у нас есть модель Account, указанная в конфиге auth.php, которая реализует какой-то ISecurityUser.

    interface ISecurityUser {}
    class Account extends Eloquent implements ISecurityUser {/*...*/} 
    
И вся эта красота будет у нас находится в IoC.

    App::bind('ISecurityUser', function ()
    {
        return Auth::user();
    });


##### I.	Resolve переданного closure


    /**
    * наш closure
    */
    $myCallback = function ($passedParameter, ISecurityUser $injectedUser = null)
    {
        return [$passedParameter, $injectedUser];
    };

    $resolver = new Resolver();
    $result = $resolver->closure($myCallback, [5]); //передаём closure и предопределённый массив параметров
    var_dump($result); // на выходе наш массив из подставленного $passedParameter и инъектированного ISecutiryUser


##### II.	Создание resolvable closure


    /**
    * наш closure
    */
    $myCallback = function ($passedParameter, ISecurityUser $injectedUser = null)
    {
        return [$passedParameter, $injectedUser];
    };
    
    $resolver = new Resolver();
    $callback = $resolver->makeClosure($myCallback); //передаём closure
    var_dump($callback); // види closure
    $result = $callback(5); //передаём в closure параметр 5
    var_dump($result);  // на выходе наш массив из подставленного $passedParameter и инъектированного ISecutiryUser


##### III.	Resolve метода


    class SomeClass
    {
        public function someMethod($passedParameter, ISecurityUser $injectedUser = null)
        {
            return [$passedParameter, $injectedUser];
        }
    }
    
    $resolver = new Resolver();
    $result = $resolver->method('SomeClass', 'someMethod', [5]); 
        // или $resolver->method(new SomeClass, 'someMethod', [5]) 
        // или $resolver->method($someClass, 'someMethod', [5])
    var_dump($result); // на выходе наш массив из подставленного $passedParameter и инъектированного ISecutiryUser


##### IV.	Превращение метода в Resolvable closure


    class SomeClass
    {
        public function someMethod($passedParameter, ISecurityUser $injectedUser = null)
        {
            return [$passedParameter, $injectedUser];
        }
    }
    
    $resolver = new Resolver();
    $callback = $resolver->methodToClosure('SomeClass', 'someMethod'); 
        // или $resolver->method(new SomeClass, 'someMethod') 
        // или $resolver->method($someClass, 'someMethod')
    var_dump($callback); // closure
    $result = $callback(5);
    var_dump($result); // на выходе наш массив из подставленного $passedParameter и инъектированного ISecutiryUser
