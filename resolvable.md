    class Resolver
    {
        protected function resolve($reflectionParams, $params = [])
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

        public function method($class, $method, $params = [])
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

        public function closure(Closure $closure, $params = [])
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


###Примеры

Допустим у нас есть модель Account, указанная в конфиге auth.php, которая реализует какой-то ISecurityUser.

    interface ISecurityUser {}
    class Account extends Eloquent implements ISecurityUser {/*...*/} 
    
И вся эта красота будет у нас находится в IoC.

    App::bind('ISecurityUser', function ()
    {
        return Auth::user();
    });


### I.	Resolve переданного closure


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


### II.	Создание resolvable closure


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


### III.	Resolve метода


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


###IV.	Превращение метода в Resolvable closure


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
