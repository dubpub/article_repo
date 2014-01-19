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

##Реализация

Для начала, было бы неплохо разбить задачу на несколько частей - нам понядобяться интерфейс ViewModel, его частичная абстрактная реализация, некий Resolver, которй бы справлялся с инъекцией моделей в методы, ну и прослойка, отвечающая за наполнение свойствами ViewModel, реализация которых имеет место быть во множественных вариациях.

Перед тем как преступить в ViewModel, было бы неплохо описать интерфейс IViewModel, потому что у наши вариации ViewModel могут базироваться в зависимости от целей, поэтому планирование минимального интерфейса просто необходимо. Мне очень нравиться так же использование интерфесоф ArrayableInterface и JsonableInterface, поэтому мы обяжем наш IViewModel их реализовывать. Помимо было бы неплохо реализовать следующие возможности: 

<ul>
    <li>
        у нашей модели должна быть возможность привести себя к базовой Eloquent-сущности. За эту возможность будет отвечать toModel();
    </li>
    <li>
        так как наша ViewModel будет ориентирована на контекстное использование и будет отвечать за входящие данные - было бы неплохо добавить метод isValid($isNew = true). При валидации ;
    </li>
</ul>

    namespace Application\Utils\ViewModel\Interfaces;
    
    use Illuminate\Support\Contracts\ArrayableInterface;
    use Illuminate\Support\Contracts\JsonableInterface;

    interface IViewModel implements ArrayableInterface, JsonableInterface
    {
        public function getBaseModel($attributes = [], $isNew = true);
        public function getValidationObject($input, $isNew = true);
        public function getValidator();
        public function isValid($isNew = true);
        public function toModel($isNew = true);
    }
