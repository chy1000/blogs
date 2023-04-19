# Laravel的依赖注入和服务容器

## 依赖注入（DI）

依赖注入这概念初听起来很高级，但实现原理很简单：

**在构造函数或者方法的形式参数将实例传入，而不是在当前类中实例对象，并且形式参数要使用抽像类或接口。**

这样做好处是解除了类与类之间的强耦合、易于扩展和单元测试。



## 服务容器（ `IoC` ）

从上面我们知道，可以通过参数的方式将对象注入。如果只是手动在类的外面创建依赖对象并注入，并没有真正解决耦合的问题，只不过将问题从一个地方转移到另一个地方。

`Laravel`框架的依赖注入可以自动注入，其实现原理是基于`PHP`的反射机制。具体来说，`Laravel`的容器通过`PHP`反射类来解析类的构造函数或方法的参数，从而确定要注入的依赖项。当应用程序请求一个被容器管理的对象时，`Laravel`的容器会使用反射类解析该类的构造函数的参数，并查找容器中是否有这些参数所需的实例。如果容器中找到了依赖项，容器将实例化它并将其传递给构造函数或方法。

我觉得服务容器很重要的一点是**可配置化**，当变化时，可以在服务提供者哪里直接更改。

下面的代码摘自《`laravel`框架关键技术解析》，这段代码很好的还原了`laravel`的服务容器的核心思想，代码可以直接运行调试，这样更易于理解。

```php
// 设计容器类，容器类装实例或提供实例的回调函数
class Container {
    // 容器绑定数组
    // 用于装提供实例的回调函数，真正的容器还会装实例等其他内容，从而实现单例等高级功能
    protected $bindings = [];

    /* 绑定接口和生成生成相应实例的回调函数
     * $abstract：这是要绑定的抽象类或接口。它通常是一个字符串，用于标识抽象类或接口的名称
     * $concrete：这是具体的实现类或回调函数。它可以是一个类名字符串，表示具体实现的类名，或者是一个回调函数（Closure），用于在需要时创建具体实现的对象。如果 $concrete 参数没有提供回调函数，该函数会使用 getClosure() 方法生成默认的回调函数，该回调函数会根据 $abstract 参数查找默认的实现类。
     * $shared：这是一个布尔值，用于指示是否共享绑定。如果设置为 true，则表示将使用同一个实例来解析每次对该抽象类或接口的依赖。如果设置为 false，则表示每次解析都会创建一个新的实例
     */
    public function bind($abstract, $concrete = null, $shared = false) {
        if (!$concrete instanceof Closure) {
            // 如果提供的参数不是回调参数，则产生默认的回调函数
            $concrete = $this->getClosure($abstract, $concrete);
        }

        $this->bindings[$abstract] = compact('concrete', 'shared');
    }

    // 默认生成实例的回调函数
    public function getClosure($abstract, $concrete) {
        // 生成实例的回调函数，$c 一般为 IoC 容器对象，在调用回调生成实例时提供
        // 即 build 函数中的 $concrete($this)
        return function ($c) use ($abstract, $concrete) {
        	//var_dump($abstract == $concrete);
            $method = ($abstract == $concrete) ? 'build' : 'make';
            // 调用的是容器的 build 或 make 方法生成实例
            return $c->$method($concrete);
        };
    }

    // 生成实例对象，首先解决接口和要实例化类之间的依赖关系
    public function make($abstract) {
        $concrete = $this->getConcrete($abstract);
        if ($this->isBuildable($concrete, $abstract)) {
            $object = $this->build($concrete);
        } else {
            $object = $this->make($concrete);
        }

        return $object;
    }

    protected function isBuildable($concrete, $abstract) {
        return $concrete === $abstract || $concrete instanceof Closure;
    }

    // 获取绑定的回调函数
    protected function getConcrete($abstract) {
        if (!isset($this->bindings[$abstract])) {
            return $abstract;
        }
        return $this->bindings[$abstract]['concrete'];
    }

    // 实例化对象
    public function build($concrete) {
        if ($concrete instanceof Closure) {
            return $concrete($this);
        }
        // ReflectionClass 类报告了一个类的有关信息
        $reflector = new ReflectionClass($concrete);
        // 检查类是否可实例化 return bool|false
        if (!$reflector->isInstantiable()) {
            echo $message = "Target [$concrete] is not instantiable.";
        }
        // 获取类的构造函数
        $constructor = $reflector->getConstructor();
        if (is_null($constructor)) {
            return new $concrete;
        }

        // 获取类构造函数的参数
        $dependencies = $constructor->getParameters();
        $instances = $this->getDependencies($dependencies);
        return $reflector->newInstanceArgs($instances);
    }

    protected function getDependencies($parameters) {
        $dependencies = [];
        foreach ($parameters as $parameter) {
            $dependency = $parameter->getClass();
            if (is_null($dependency)) {
                $dependencies[] = null;
            } else {
                $dependencies[] = $this->resolveClass($parameter);
            }
        }
        return (array)$dependencies;
    }

    protected function resolveClass(ReflectionParameter $parameter) {
        return $this->make($parameter->getClass()->name);
    }
}

// 设计公共接口
interface Visit {
    public function go();
}

// 实现不同交通工具类
// 走路
class Leg implements Visit {
    public function go(){
        echo 'walk to Tibet！！！';
    }
}

// 开车
class Car implements Visit {
    public function go(){
        echo 'drive car to Tibet！！！';
    }
}

// 列车
class Train implements Visit {
    public function go() {
        echo 'go to Tibet！！！';
    }
}

// 设计旅游者类，该类在实现游西藏的功能时要依赖交通工具类
class Traveller {
    private $trafficTool;

    public function __construct( Visit $trafficTool) {
        $this->trafficTool = $trafficTool;
    }

    public function visitTibet() {
        $this->trafficTool->go();
    }
}

// 实例化 IoC 容器
$app = new Container();
// 提前将类的实例方法和容器绑定，绑定关系如下面数组显示
$app->bind("traveller", "Traveller");
$app->bind("Visit", "Train");

// 通过容器实现依赖注入，完成类的实例化
$tra = $app->make("traveller");
$tra->visitTibet();

$tra = $app->make("Visit");
$tra->go();
```

下面是我输出的容器内部的绑定数组

```php
Array(
    [traveller] => Array
        (
            [concrete] => [匿名函数：IoC->make('Traveller')]
            [shared] => 
        )

    [Visit] => Array
        (
            [concrete] => [匿名函数：IoC->make('Train')]
            [shared] => 
        )
)
```

上面的代码只是还原了`laravel`框架的服务容器的核心思想，实现上`laravel`框架的服务容器会比这里得复杂得多。还有一点需要注意的是`laravel`框架不单可以在构造函数依赖注入，像一些路由使用到的方法也可以依赖注入。



---

### 参考：

[Laravel IoC Container: Why we need it and How it works](https://medium.com/@NahidulHasan/laravel-ioc-container-why-we-need-it-and-how-it-works-a603d4cef10f)

[Laravel源码解析 — 服务容器](https://www.cnblogs.com/houdabao/p/14646902.html)

[Laravel 之依赖注入浅析](https://learnku.com/articles/4559/analysis-of-laravel-dependency-injection)

[控制反转（IoC）与依赖注入（DI）](https://www.jianshu.com/p/07af9dbbbc4b)
