# 写在前面

内容借鉴了郭霖的 Hilt 文章：[Jetpack新成员，一篇文章带你玩转Hilt和依赖注入](https://blog.csdn.net/guolin_blog/article/details/109787732)

# 预览

## 1. Application DSL

`KoinApplication` 配置 Koin 注入的入口，有一下 API 可使用：

1. `koinApplication { }` ：创建一个 Koin 容器。
2. `startKoin { }` ：创建一个 Koin 容器并将之注册到 `GlobalContext` 中，如此可以使用 `GlobalContext` 的 API 。
3. `logger( )` ：选择使用什么 Log 方式，Koin 提供了 3 种默认 Logger ，分别是 **AndroidLogger** 、**PrintLogger** 、**EmptyLogger** ，它们都继承自抽象类 **Logger** ，如不配置默认使用 EmptyLogger ，即不打印。
4. `modules( )` ：配置 Koin 模块并将之注入到 Koin 容器。
5. `properties()` ：使用 HashMap 注入属性，供全局查询修改。
6. `fileProperties( )` ：使用给定 properties 文件注入属性，文件需要放在 **src/main/resources** 目录下。
7. `environmentProperties( )` ：注入系统、环境属性，通过 `java.lang.System` 注入。

## 2. Module DSL

1. `module { // module content }` ：创建一个 Koin 模块。
2. `factory { //definition }` ：创建注入类的实例。
3. `single { //definition }` ：与 `factory` 功能一致，只不过创建的是单例。
4. `get()` ：通过解析组件依赖，注入类实例。
5. `inject()` ：与 `get()` 功能一致，都是提供类注入，但 `inject` 是懒加载。
6. `bind()` ：为注入类增加类型绑定，因为默认注入类只能对应一个类型。
7. `binds()` ：功能与上述 `bind()` 一致，一次性提供多个类型绑定。
8. `scope { // scope group }` ：为下述 `scoped` 定义一个合理的组，作用是控制注入类的生命周期。
9. `scoped { //definition }` ：与上述 `scope` 配合使用，定义内放注入类的实例，表示只在 `scope` 的范围内存在。
10. `named()` ：如果遇到相同类型需要两个或以上的注入，可以通过这个函数给注入进行命名，然后在注入处只要指定好名称即可获取正确的注入。

# 场景

## 1. 简单注入

- 首先封装一下常用的 API ，接下来都会用到这些封装 API ：

  ```kotlin
  inline fun <reified T : Any> get(
      qualifier: Qualifier? = null,
      noinline parameters: ParametersDefinition? = null
  ): T {
      return GlobalContext.get().get<T>(qualifier, parameters)
  }
  
  inline fun <reified T : Any> inject(
      qualifier: Qualifier? = null,
      mode: LazyThreadSafetyMode = KoinPlatformTools.defaultLazyMode(),
      noinline parameters: ParametersDefinition? = null
  ): Lazy<T> {
      return GlobalContext.get().inject(qualifier, mode, parameters)
  }
  ```

  这里主要针对 `GlobalContext.get()` 进行了封装，否则在业务代码中免不了需要写多一些样板代码。

- 接下来配置一下注入信息，例如需要一辆车需要引擎：

  ```kotlin
  class Car
  
  val module = modules {
      factory { Car() }
  }
  
  class App : Application() {
      
      override fun onCreate() {
          super.onCreate()
          startKoin {
              modules(module)
          }
      }
  }
  ```

- 然后就可以在调用处进行注入了：

  ```kotlin
  private val car = get<Car>()
  // or
  private val car by inject<Car>()
  ```

## 2. 带参数注入

- 给 `Car` 注入一个司机：

  ```kotlin
  class Driver
  
  class Car(val driver: Driver)
  
  val module = module {
      factory { params -> Car(params.get()) }
      // or
      factory { (driver: Driver) -> Car(driver) }
  }
  ```

  在配置时我们使用到了`Definition<T>` ，这是 `factory()` 和 `single()` 函数里定义的，它的原型是 `Scope.(ParametersHolder) -> T` ，我们可以通过 **ParametersHolder** 取出我们需要的参数进行创建对象，如果使用解构声明则参数最多只接受 **5** 个，超出无法使用解构声明，看代码：

  ```kotlin
  val module = module {
      factory { (p1: Byte, p2: Short, p3: Int, p4: Long, p5: String) -> 
      	Obj(p1, p2, p3, p4, p5)
      }
      // 超出 5 个后手动调用 params 获取
      factory { params -> 
      	Obj(params[0], params[1], params[2], params[3], params[4], params[5]) 
      }
  }
  
  private val obj = get<Obj> {
      parametersOf(
      	1, 2, 3, 4L, "", //...
      )
  }
  ```

  注入时使用到了 `ParametersDefinition` ，这是 `get()` 和 `inject()` 函数里定义的，它的原型是 `() -> ParametersHolder` ，意味着需要返回一个 `ParametersHolder` 给 Koin ，通过 `parametersOf()` 函数传入参数，支持可变参数。

## 3. 接口注入

- 给车注入引擎：

  ```kotlin
  interface Engine
  
  class GasEngine
  
  class ElectricEngine
  
  val module = module {
      // 绑定为 GasEngine 类型
      factory { GasEngine() }
      // 绑定为 ElectricEngine 类型
      factory { ElectricEngine() }
  }
  
  class Car {
      
      private val gasEngine = get<GasEngine>()
      private val electricEngine = get<ElectricEngine>()
  }
  ```

  这样可以完成接口的注入，但存在问题，默认情况下配置注入时只能绑定一个类型，即上述代码各自绑定了 `GasEngine` 和 `ElectricEngine` ，因此在注入时只能明确使用配置时绑定的类型，如果使用 `get<Engine>()` 会报找不到定义异常。

  那如果想要通过 `Engine` 类型来获取注入该怎么做呢？

  ```kotlin
  // 第一种方法是在配置时明确类型：
  factory<Engine> { GasEngine() }
  // or
  factory { GasEngine as Engine }
  
  // 第二种方法是使用 `bind()` 中缀函数：
  factory { GasEngine() } bind Engine::class
  ```

  `bind()` 函数需要传入想同时绑定的 KClass 类型，如果有多个类型需要同时绑定可以使用 `binds()` 函数传入一个类型 List 。

  当配置时使用了多类型绑定后，在定义注入时就可以根据想要的类型来进行注入了，但仍需要注意，一个类型最多只能有一个配置，新的注入配置会覆盖旧配置（V2 版本中 override 默认关闭需要手动开启，V3 版本默认开启，可手动关闭），下面用例子来说明：

  ```kotlin
  val module = module {
      factory { GasEngine() } bind Engine::class
  	factory { ElectricEngine() } bind Engine::class
  }
  
  private val gasEngine = get<Engine>()
  private val electricEngine = get<ElectricEngine>()
  ```

  运行后可以发现，无论是 `gasEngine` 还是 `electrictEngine` ，都会注入了 `ElectricEngine` ，因为在第二行配置时同时绑定了 `Engine` 类型，导致上一行的配置被覆盖，如果去除第二行配置的 `bind Engine::class` 即恢复正常。那如果确实需要注入多个同类型该怎么办呢？下面会说到。

## 4. 相同类型注入

直接上代码：

```kotlin
val module = module {
    factory(named("gas")) { GasEngine() } bind Engine::class
    factory(named("electric")) { ElectricEngine() } bind Engine::class
}

private var gasEngine = get<Engine>(named("gas"))
private var electricEngine = get<Engine>(named("electric"))
```

在配置与注入时均使用到了 `Qualifier` 限定符，`factory()` 和 `single()` 函数都有提供形参，通过 `named()` 函数获取到 `Qualifier` 并传入即可获取需要的效果。

`named()` 支持 3 种使用方式：

1. `named<KClass>()` ：通过泛型进行限定
2. `named(String)` ：通过字符串进行限定
3. `named(Enum)` ：通过枚举进行限定

与 `named()` 作用相同的还有 `qualifier()` 、`_q()` ，它们只是名字的区别，所以我也搞不明白为啥提供两个多余 API 。

## 5. 第三方类注入

用法较前面没有差别，略~

## 6. 泛型的处理

如果业务中不想重复创建容器类，需要创建后持续使用，可以这么写：

```kotlin
val module = module {
    single(named("Ints")) { ArrayList<Int>() }
    single(named("Strings")) { ArrayList<String>() }
}
```

必须使用 `Qualifier` 进行限定。

## 7. Scope

作用域，可以赋予我们使用的注入一段生命周期，用完手动关闭。

```kotlin
// 定义作用域内的配置
val module = module {
    scope<String>{
        scoped { Obj() }
        // ...
    }
}

class Test {
    
    private val scope = GlobalContext.get().getOrCreateScope<String>("")
    
    private val obj = scope.get<Obj>()
    
    fun recycle() {
        scope.close()
    }
}
```

配置 **Scope** 时需要指定 **scopeId** 、**scopeName（可使用 Qualifier 或泛型）** ，使用注入时首先创建出 scope ，有一下函数可供获取：

1. `createScope()` ：创建作用域
2. `getScope` ：获取作用域，如不存在会报异常
3. `getOrCreateScope` ：获取作用域，如不存在会创建并返回

## 8. Properties

可通过 Koin 增删改查临时键值对，有一下 API ：

1. `getProperty` ：根据 key 获取 value

2. `setProperty` ：设置键值对
3. `deleteProperty` ：删除键值对

## 9. 其他有用的 API

1. `loadModules` ：可加载配置模块
2. `unloadModules` ：可卸载配置模块
3. `KoinComponent` ：实现这个接口获得 Koin KTX 功能
4. `KoinScopeComponent` ：`KoinComponent` 的子接口，实现此接口可使用 Koin Scope KTX 功能