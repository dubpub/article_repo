#Laravel 4.1 Http-middleware

С выходом L4.1 мы уже увидели не мало приятных плюшек – расширенные методы для работы со связями в Eloquent, полностью переделанный и ускоренный роутинг, улучшенный Tinker, отличные нововведения по поводу SSH – но это не всё. Теперь разработчикам Laravel доступено управление промежуточным Http слоем(Middleware). Для этого в фрэймворк был интегрирована уже многим известная реализация StackPHP из Symphony HttpKernel. Что же это нам даёт?

<ul>
    <li>Обработка/внедрение сессий</li>
    <li>Преждевременная обработка строк запроса</li>
    <li>Введение ограничение количества запросов по временному интервалу</li>
    <li>Отлавливание ботов</li>
    <li>Расширенные возможности логирования</li>
    <li>Преждевременная обработка json</li>
    <li>И всё что угодно, что связанно с циклом жизни request|response</li>
</ul>    

Перед тем как говорить о своей реализации, стоит заметить, что Middleware использует паттерн декоратор – знаете ли вы этот паттерн или нет, в данном контексте это говорит о том, что тот middleware декоратор, что мы напишем должен реализовывать интерфейс HttpKernelInterface. Для внедрения своих декораторов L4.1 как всегда предоставляет удобные средства для внедрения

    /**
     * Add a HttpKernel middleware onto the stack.
     *
     * @param  string  $class
     * @param  array  $parameters
     * @return \Illuminate\Foundation\Application
     */
    public function middleware($class, array $parameters = array())
    {
        $this->middlewares[] = compact('class', 'parameters');
        return $this;
    }

###И так как же это всё выглядит?

Напишем простой ограничитель количества request с одного ip-адреса по времени. Сразу хотелось бы оговорить – код писался на скорую руку, дабы продемонстрировать возможности новой прослойки.
Для начала опишем наш “rate-limiting” класс, реализующий HttpKernelInterface, который собственно и будет являться нашим middleware.

    <?php 
        namespace Fideloper\Http;
        
        use Symfony\Component\HttpKernel\HttpKernelInterface;
        use Symfony\Component\HttpFoundation\Request as SymfonyRequest;
        
        class RateLimiter implements HttpKernelInterface {
        
            /**
             * The wrapped kernel implementation.
             *
             * @var \Symfony\Component\HttpKernel\HttpKernelInterface
             */
            protected $app;
        
            /**
             * Create a new RateLimiter instance.
             *
             * @param  \Symfony\Component\HttpKernel\HttpKernelInterface  $app
             * @return void
             */
            public function __construct(HttpKernelInterface $app)
            {
                $this->app = $app;
            }
        
            /**
             * Handle the given request and get the response.
             *
             * @implements HttpKernelInterface::handle
             *
             * @param  \Symfony\Component\HttpFoundation\Request  $request
             * @param  int   $type
             * @param  bool  $catch
             * @return \Symfony\Component\HttpFoundation\Response
             */
            public function handle(SymfonyRequest $request, $type = HttpKernelInterface::MASTER_REQUEST, $catch = true)
            {
                // Handle on passed down request
                $response = $this->app->handle($request, $type, $catch);
        
                $requestsPerHour = 60;
        
                // Rate limit by IP address
                $key = sprintf('api:%s', $request->getClientIp());
        
                // Add if doesn't exist
                // Remember for 1 hour
                \Cache::add($key, 0, 60);
        
                // Add to count
                $count = \Cache::increment($key);
        
                if( $count > $requestsPerHour )
                {
                    // Short-circuit response - we're ignoring
                    $response->setContent('Rate limit exceeded');
                    $response->setStatusCode(403);
                }
        
                $response->headers->set('X-Ratelimit-Limit', $requestsPerHour, false);
                $response->headers->set('X-Ratelimit-Remaining', $requestsPerHour-(int)$count, false);
        
                return $response;
            }
        
        }

Реализуя интерфейс HttpKernelInterface нам необходимо описать метод handle. Наш метод handle первым делом вызовет метод handle() предыдущего middleware, что бы получить объект ```$response```. Чтобы разобраться подробнее зачем мы обращаемся к предыдущему middleware ```$response``` настоятельно советую изучить паттерн Observer. После вкратце описывается наша логика по работе с rate-limit. 
Мой rate-limit создаёт уникальный ключ в кэше, основанный на IP-адресе клиетна и содержит подсчитанное количество request, которые он совершил за час с момента создания ключа. Время жизни ключа выставлено на час и максимальное количество request 60 после чего на клиент прилетит статус-код 403. В противном случае, так как наверняка у нашего приложения есть rest-сервисы мы отдадим предупредительные заголовки X-Ratelimit-Limit, содержащий информацию о максимальном количестве попыток, которое допускает наше приложение, и X-Ratelimit-Limit в котором мы укажем количество оставшихся у клиента попыток. Так что только 60 реквестов и час спустя наши предполагаемые злоумышленники смогут продолжить свою работу. 

###И как же мы подтянем это в наш проект?

Всё как всегда просто и красиво – мы по старинке напишем напишем сервис-провайдер.

    <?php 
        namespace Fideloper\Http;
        
        use Illuminate\Support\ServiceProvider;
        
        class HttpServiceProvider extends ServiceProvider {
        
            /**
             * Register the binding
             *
             * @return void
             */
            public function register()
            {
                $this->app->middleware( new RateLimiter($this->app) );
            }
        
        }
    
