# 前言

EventBus 的使用非常简单，但背后的逻辑又是如何的呢 ？这里简单起步，从 **register** 开始向内部进行窥探学习，本文 EventBus 版本为 3.2.0 。

# 1. register 注册

```java
public void register(Object subscriber) {
    // 获取 subscriber 这个订阅者的 Class 然后传入 findSubscriberMethods 方法，
    // 根据 findSubscriberMethods 方法名可以知道这个方法就是为了查找订阅者通过
    // @Subscribe 注解标记的订阅方法。
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    
    // 这里主要将查找到的订阅方法进行订阅相关的处理。
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

这个方法主要逻辑分为了两块，分别是查找订阅方法和处理找到的订阅方法。从 `findSubscriberMethods()` 进去看看里面是怎么工作的。

## 1.1 findSubscriberMethods

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    // 首先通过 METHOD_CACHE 找一下有没有缓存的订阅方法信息，如果有就将找到的返回。
    List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
    if (subscriberMethods != null) {
        return subscriberMethods;
    }
    // 提供了两种查找方式，如果不自定义配置，ignoreGeneratedIndex 默认为 false
    if (ignoreGeneratedIndex) {
        // 通过反射进行查找
        subscriberMethods = findUsingReflection(subscriberClass);
    } else {
        // 通过 EventBus 索引进行查找
        subscriberMethods = findUsingInfo(subscriberClass);
    }
    // 这里说明了单单只为订阅者注册是不行的，如果不提供一个合格的 @Subscribe 方法
    // 那么在运行时会直接抛异常中止程序执行。
    if (subscriberMethods.isEmpty()) {
        throw new EventBusException("Subscriber " + subscriberClass
                + " and its super classes have no public methods with the @Subscribe annotation");
    } else {
        // 查找到目标订阅方法，放入缓存加快下次使用然后返回。
        METHOD_CACHE.put(subscriberClass, subscriberMethods);
        return subscriberMethods;
    }
}
```

从上面可以得知 EventBus 提供了反射和索引两种查找方式，重点说下索引查找，它是通过编译时生成所有订阅者信息为一个类，然后找的时候就可以不用到反射，直接从索引里面拿，直接提高了查找效率。虽然 **ignoreGeneratedIndex** 这个标记是默认为 false ，也就是会使用索引查找，但我们还需要额外配置一下才能真正使用这个功能。

```groovy
android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
            }
        }
    }
}
 
dependencies {
    def eventbus_version = '3.2.0'
    implementation "org.greenrobot:eventbus:$eventbus_version"
    annotationProcessor "org.greenrobot:eventbus-annotation-processor:$eventbus_version"
}
```

可以看到这里使用了注解处理器，就是由它来生成索引，然后上面指定了索引的生成类名，接着在对 EventBus 进行初始化。

```java
public class App extends Application {

	@Override
	public void onCreate() {
		EventBus bus = EventBus.builder()
			.addIndex(new MyEventBusIndex())
			.build()
	}
}
```

接下来使用我们自己构建的 EventBus 替换默认的 EventBus 就可以使用索引功能了。

上面我们说到了了 EventBus 提供了两种查找方式，这里先从 `findUsingReflection()` 进去看看。

## 1.2 findUsingReflection

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
    // FindState 是一个查找辅助类，这里通过 prepareFindState 方法获取一个，
    // 它是可以被 EventBus 缓存的，接着传入 subscriberClass 让 FindState
    // 针对这个订阅者进行初始化。
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    // 遍历查找订阅类，findState.clazz 指的就是被查找的订阅类，使用遍历时因为
    // 订阅类可以不局限于当前 subscriber ，也可以是该 subscriber 的父类。
    while (findState.clazz != null) {
        // 通过方法名可以知道这里面才是真正使用反射进行方法查找。
        findUsingReflectionInSingleClass(findState);
        // 查找完后切换到下一个待查找类再进行查找
        findState.moveToSuperclass();
    }
    // 当所有和 subscriber 相关的订阅类查找完后，拿到订阅方法信息返回。
    return getMethodsAndRelease(findState);
}
```

从第一段开始看，这里说到了 FindState 这个类，进去 `prepareFindState()` 看看。

```java
private FindState prepareFindState() {
    // private static final int POOL_SIZE = 4;
    // private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            FindState state = FIND_STATE_POOL[i];
            if (state != null) {
                FIND_STATE_POOL[i] = null;
                return state;
            }
        }
    }
    return new FindState();
}
```

从这里可以知道，EventBus 最多可以缓存 4 个 FindState ，毕竟这是会被频繁使用的类，如果不进行缓存会造成不必要的内存开销，如果没找到可以用的缓存，会创建一个新的对象进行返回。

接着重点看循环内的查找，先看看 `findState.moveToSuperClass()` 这个方法，从方法名得知它就是将当前循环的订阅类切换为父类。

```java
void moveToSuperclass() {
    // skipSuperClasses 默认为 false ，在 findState.initForSubscriber() 中对其赋值
    // 当它为 true 时即会跳出外部的循环，结束该订阅者的查找。
    if (skipSuperClasses) {
        clazz = null;
    } else {
        // 获取当前订阅者的父类
        clazz = clazz.getSuperclass();
        String clazzName = clazz.getName();
        /** Skip system classes, this just degrades performance. */
        if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
            clazz = null;
        }
    }
}
```

可以看到确实是遍历查找订阅者自身和它的父类，这里还做了额外的处理，即跳过系统类。

回到外部方法，重点来了，我们将进去 `findUsingReflectionInSingleClass()` 了解真正的查找逻辑。 

### 1.2.1 findUsingReflectionInSingleClass

```java
private void findUsingReflectionInSingleClass(FindState findState) {
    Method[] methods;
    try {
        // 获取当前订阅类所有声明方法
        methods = findState.clazz.getDeclaredMethods();
    } catch (Throwable th) {
        // 出现异常时改用 getMethods() 并跳过父类查找，也就是外面的循环只会循环这一次。
        methods = findState.clazz.getMethods();
        findState.skipSuperClasses = true;
    }
    // 检查每一个方法，看是不是符合要求的订阅方法。
    for (Method method : methods) {
        int modifiers = method.getModifiers();
        // 检查订阅方法是否 public 、非 absctract 、非 static 、只有一个形参
        // 并且有注解 @Subscribe 标注。
        if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 1) {
                Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                if (subscribeAnnotation != null) {
                    Class<?> eventType = parameterTypes[0];
                    if (findState.checkAdd(method, eventType)) {
                        // 当 method 可以被放入 findState 后，分别将 threadMode 、 priority
                        // 和 method 打包成 SubscribeMethod 放入 findState ，这个 			                    // SubscribeMethod 很熟悉，在我们刚进入 register 时就能看到，
                        // 它其实就是包含了订阅方法本身和它附带的 threadMode 和 priority 。
                        ThreadMode threadMode = subscribeAnnotation.threadMode();
                        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                    }
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException("@Subscribe method " + methodName +
                        "must have exactly 1 parameter but has " + parameterTypes.length);
            }
        } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
            String methodName = method.getDeclaringClass().getName() + "." + method.getName();
            throw new EventBusException(methodName +
                    " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
        }
    }
}
```

这个方法有点长，首先反射获取所有方法（`getDeclaredMethods()` 和 `getMethods()` 的区别是前者只会获取这个类的所有声明方法，包含 private ，而后者只会获取 public 方法，但包含所有从超类和接口继承实现的方法），接着就是对反射到的方法进行校验。回到 `findUsingReflection()` ，当所有方法遍历完后，会执行 `getMethodAndRelease()` ，进去看看。

```java
private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
    List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
    findState.recycle();
    synchronized (FIND_STATE_POOL) {
        for (int i = 0; i < POOL_SIZE; i++) {
            if (FIND_STATE_POOL[i] == null) {
                FIND_STATE_POOL[i] = findState;
                break;
            }
        }
    }
    return subscriberMethods;
}
```

可以看到，创建了一个列表来装载刚才找到的所有 SubscribeMethod ，然后释放 FindState ，并尝试将 FInsState 加入缓存。

反射查找方式到这里就结束了，接下来分析使用索引进行查找的方式。

## 1.3 findUsingInfo

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}

private SubscriberInfo getSubscriberInfo(FindState findState) {
    // 这里主要区分了继承关系间的索引，当 SubscriberInfo 存在 superSubscriberInfo
    // 且当前被查找的订阅类与 superSubscriberClass 相同时才返回 Super SubscriberInfo 。
    if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
        SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
        if (findState.clazz == superclassInfo.getSubscriberClass()) {
            return superclassInfo;
        }
    }
    // 上面的检查没有获取到合适的 SubscriberInfo ，因此这里遍历 subscriberInfoIndexes
    // 进行查找，当找到与当前被查找订阅类相匹配的 SubscriberInfo 时返回。
    if (subscriberInfoIndexes != null) {
        for (SubscriberInfoIndex index : subscriberInfoIndexes) {
            SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
            if (info != null) {
                return info;
            }
        }
    }
    // 找不到索引，返回 null ，意味着外面需要用反射了。
    return null;
}
```

可以看到，索引查找方式在代码逻辑上与反射方式相差不大，并且当这种方式找不到后会再次通过反射进行查找，这里看下 `getSubscriberInfo()` 这个方法，针对被查找的订阅类进行 SubscriberInfo 的获取，SubscriberInfo 包含了订阅类相关的信息，由注解处理器在编译器为我们生成。

索引查找方式到这里也结束了，接下来可以回归主线了，回到 `register()` 中。

## 1.4 subscribe

前面梳理了查找 SubscriberMethod 的流程，那么现在找到了就需要对这些 SubscriberMethod 进行订阅处理。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 这部分代码主要对 subscriber 和 subscriberMethod 进行了包装，生成
    // 统一的 Subscription ，然后使用列表缓存起来，这里会判断是否已经存在，
    // 如果已经存在就抛异常，即不允许同一个订阅类多次注册，这里的 Subscription
    // 重写了 equals ，内部使用 subscriber 和 scubscriberMethod 进行比较。
    Class<?> eventType = subscriberMethod.eventType;
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }
    // 这里进行了优先级处理，优先级高的排在前面。
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    // 将订阅类和相关事件进行缓存，方便判断是否已经注册与后期 unregister 。
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    
    // ... 处理粘性事件部分，放到后面分析。
}
```

这部分处理订阅的代码相对简单清晰，register 分析到这里就告一段落了，下面简单总结一下。

总结：注册流程通过订阅者去查找相关订阅信息，有缓存则直接使用，否则通过反射或索引的方式进行查找，默认使用索引的方式，这种方式如果找不到就用反射找到，最后再遍历找到的 SubscriberMethod 进行订阅处理，缓存订阅者与事件以方便判断注册状态与后期解注册。

# 2. unregister 解注册

```java
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}

private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                // 将订阅激活状态取消，如果还有事件将无法接收。
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```

通过 subscriber 在 typesBySubscriber 中查找，找到后对相关事件进行解注册，然后移除相关缓存。

# 3. post 发送普通事件

```java
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

currentPostingThreadState 是一个 ThreadLocal ，用于获取当前线程私有的 PostingThreadState ，这个类封装了 事件队列、订阅信息和分发控制，将事件加入队列后判断 PostingThreadState 当前是否正在分发事件，如果不是则开始分发，通过遍历事件队列并调用 `postSingleEvent()` 进行单个事件的发送。

## 3.1 postSingleEvent

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    if (eventInheritance) {
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```

这个方法主要处理 2 种情况，如果允许事件有继承关系（默认为 true ，可关）则查询出该事件的所有父类、接口，然后遍历这些事件相关类进行发送事件；如果不允许继承则直接开始事件发送，方法的返回值表示有没有找到订阅者，如果没找到默认打印 Log 并发送 NoSubscriberEvent ；接下来看看 `postSingleEventForEventType()` 。

## 3.2 postSingleEventForEventType

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

首先取出前期存下来的订阅信息列表，如果为空直接返回 false ，不为空后遍历取出订阅信息开始真正的事件发送，这里的 **aborted** 表示中断事件分发，当一个事件被接收并取消后会直接取消这一系列 eventType 事件。接着看 `postToSubscription` 。

## 3.3 postToSubscription

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```

在这里看到了针对线程分发的处理，当没有显式指定 threadMode 时默认就是 POSTING ，即在当前线程发送，接下来分别分析以下几种线程模式。

- **POSTING**

  进来后直接调用 `invokeSubscriber()`  ，进去看看。

  ```java
  void invokeSubscriber(Subscription subscription, Object event) {
      try {
          subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
      } catch (InvocationTargetException e) {
          handleSubscriberException(subscription, event, e.getCause());
      } catch (IllegalAccessException e) {
          throw new IllegalStateException("Unexpected exception", e);
      }
  }
  
  private void handleSubscriberException(Subscription subscription, Object event, Throwable cause) {
      if (event instanceof SubscriberExceptionEvent) {
          if (logSubscriberExceptions) {
              logger.log(Level.SEVERE, "SubscriberExceptionEvent subscriber " + subscription.subscriber.getClass()
                      + " threw an exception", cause);
              SubscriberExceptionEvent exEvent = (SubscriberExceptionEvent) event;
              logger.log(Level.SEVERE, "Initial event " + exEvent.causingEvent + " caused exception in "
                      + exEvent.causingSubscriber, exEvent.throwable);
          }
      } else {
          if (throwSubscriberException) {
              throw new EventBusException("Invoking subscriber failed", cause);
          }
          if (logSubscriberExceptions) {
              logger.log(Level.SEVERE, "Could not dispatch event: " + event.getClass() + " to subscribing class "
                      + subscription.subscriber.getClass(), cause);
          }
          if (sendSubscriberExceptionEvent) {
              SubscriberExceptionEvent exEvent = new SubscriberExceptionEvent(this, cause, event,
                      subscription.subscriber);
              post(exEvent);
          }
      }
  }
  ```

  可以看到直接通过反射调用订阅方法，然后处理异常，`handleSubscriberException()` 默认会发出 SubscriberExceptionEvent 。

- **MAIN**

  ```java
  public class HandlerPoster extends Handler implements Poster {
  
      private final PendingPostQueue queue;
      private final int maxMillisInsideHandleMessage;
      private final EventBus eventBus;
      private boolean handlerActive;
  
      protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
          super(looper);
          this.eventBus = eventBus;
          this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
          queue = new PendingPostQueue();
      }
  
      public void enqueue(Subscription subscription, Object event) {
          PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
          synchronized (this) {
              queue.enqueue(pendingPost);
              if (!handlerActive) {
                  handlerActive = true;
                  if (!sendMessage(obtainMessage())) {
                      throw new EventBusException("Could not send handler message");
                  }
              }
          }
      }
  
      @Override
      public void handleMessage(Message msg) {
          boolean rescheduled = false;
          try {
              long started = SystemClock.uptimeMillis();
              while (true) {
                  PendingPost pendingPost = queue.poll();
                  if (pendingPost == null) {
                      synchronized (this) {
                          pendingPost = queue.poll();
                          if (pendingPost == null) {
                              handlerActive = false;
                              return;
                          }
                      }
                  }
                  eventBus.invokeSubscriber(pendingPost);
                  long timeInMethod = SystemClock.uptimeMillis() - started;
                  if (timeInMethod >= maxMillisInsideHandleMessage) {
                      if (!sendMessage(obtainMessage())) {
                          throw new EventBusException("Could not send handler message");
                      }
                      rescheduled = true;
                      return;
                  }
              }
          } finally {
              handlerActive = rescheduled;
          }
      }
  }
  
  // PendingPost
  static PendingPost obtainPendingPost(Subscription subscription, Object event) {
      synchronized (pendingPostPool) {
          int size = pendingPostPool.size();
          if (size > 0) {
              PendingPost pendingPost = pendingPostPool.remove(size - 1);
              pendingPost.event = event;
              pendingPost.subscription = subscription;
              pendingPost.next = null;
              return pendingPost;
          }
      }
      return new PendingPost(event, subscription);
  }
  ```

  如果当前是主线程则直接发送，否则事件入队，在 `HandlePoster.enqueue()` 中使用 PendingPost 装载待发送的事件信息，接着将 PendingPost 入队并在 Handler 空闲情况下发送空 Message 激活 Handler 处理。

  在 `handleMessage()` 中将 PendingPost 出队并发出事件，接着判断 **maxMillisInsideHandleMessage** 时间还能不能再执行一次事件发送，不然就结束发送了，**maxMillisInsideHandleMessage** 默认 10 毫秒。

  可以看到 `obtainPendingPost()` 中也是使用了缓存池，以节约内存，这种思想在 EventBus 中得到大量的应用。

  另外由于 **MAIN** 这个模式如果分发线程是在主线程，那么默认是不会入队处理的，意味着订阅方法需要快速返回，否则会阻塞事件的分发，说远一点会影响到外部 `post()` 的调用者。

- **MAIN_ORDER**

  上面的 MAIN 在 `postToSubscription()` 中默认是阻塞的（主线程情况下），即上一次事件没处理完是不会发送下一个事件的，而 **MAIN_ORDER** 默认都是将事件入队到 mainPoster ，这种模式确保了分发线程不会被阻塞。

- **BACKGROUND**

  ```java
  final class BackgroundPoster implements Runnable, Poster {
  
      private final PendingPostQueue queue;
      private final EventBus eventBus;
  
      private volatile boolean executorRunning;
  
      BackgroundPoster(EventBus eventBus) {
          this.eventBus = eventBus;
          queue = new PendingPostQueue();
      }
  
      public void enqueue(Subscription subscription, Object event) {
          PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
          synchronized (this) {
              queue.enqueue(pendingPost);
              if (!executorRunning) {
                  executorRunning = true;
                  eventBus.getExecutorService().execute(this);
              }
          }
      }
  
      @Override
      public void run() {
          try {
              try {
                  while (true) {
                      PendingPost pendingPost = queue.poll(1000);
                      if (pendingPost == null) {
                          synchronized (this) {
                              // Check again, this time in synchronized
                              pendingPost = queue.poll();
                              if (pendingPost == null) {
                                  executorRunning = false;
                                  return;
                              }
                          }
                      }
                      eventBus.invokeSubscriber(pendingPost);
                  }
              } catch (InterruptedException e) {
                  eventBus.getLogger().log(Level.WARNING, Thread.currentThread().getName() + " was interruppted", e);
              }
          } finally {
              executorRunning = false;
          }
      }
  }
  
  // PendingPostQueue
  synchronized PendingPost poll() {
      PendingPost pendingPost = head;
      if (head != null) {
          head = head.next;
          if (head == null) {
              tail = null;
          }
      }
      return pendingPost;
  }
  
  synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
      if (head == null) {
          wait(maxMillisToWait);
      }
      return poll();
  }
  ```

  如果当前线程是主线程则入队，否则直接发送，入队后由线程池处理，线程池是 **`Executors.newCachedThreadPool()`** 。从 **BACKGROUND** 线程模式的注释中可以得知，在这个模式下事件只会在一个单独的线程中进行有序分发，但内部使用的确实这种可以拥有几乎无限多线程的线程池，是如何做到的呢？

  直接看 `run()` 方法，可以看到 while 循环内使用了同步代码块，并且最关键的一点 `queue.poll(1000)` 这个方法，可以看到实际是调用了 `wait()` 使当前线程休眠，这是为了防止当没有及时获取到 PendingPost 后线程直接执行完毕（如果是这种情况那么就无法保证使系列事件都在一个线程里处理了），然后线程醒过来并取得执行权后判断如果取出为 null ，则使用同步代码块包住再调用普通 `queue.poll()` 取一遍，如果这里还没有 PendingPost 那么表示没有事件需要处理了，线程被 return 也就执行结束，否则反射调用订阅方法。

  看到这里也明白了为啥使用 `Executors.newCachedThreadPool()` 确还能保证在单一线程中进行事件分发，原因也就是在循环体中利用休眠机制，等一等事件的入队，然后遍历处理。

  需要注意的是，这个模式下不一定每次都会切线程，如果刚好不在主线程进行分发则会直接反射调用订阅方法，因此这个模式也是会阻塞的，应尽量使订阅方法快速返回。

- **ASYNC**

  ```java
  class AsyncPoster implements Runnable, Poster {
  
      private final PendingPostQueue queue;
      private final EventBus eventBus;
  
      AsyncPoster(EventBus eventBus) {
          this.eventBus = eventBus;
          queue = new PendingPostQueue();
      }
  
      public void enqueue(Subscription subscription, Object event) {
          PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
          queue.enqueue(pendingPost);
          eventBus.getExecutorService().execute(this);
      }
  
      @Override
      public void run() {
          PendingPost pendingPost = queue.poll();
          if(pendingPost == null) {
              throw new IllegalStateException("No pending post available");
          }
          eventBus.invokeSubscriber(pendingPost);
      }
  }
  ```

  

  这个模式与 BACKGROUND 模式类似，不同的是这个永远入队线程池处理，所以每次都在不同线程处理，它保证了每一次都与分发线程错开，这种模式下不会阻塞分发线程。

  到这里普通事件的发送流程结束。

# 4. postSticky 发送粘性事件

```java
public void postSticky(Object event) {
    synchronized (stickyEvents) {
        stickyEvents.put(event.getClass(), event);
    }
    post(event);
}
```

这里主要将粘性事件存起来，然后发送一次普通事件。前面我们知道了粘性事件的处理是在注册流程中的，回去看看。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // ... 省略无关代码
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}

private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
    if (stickyEvent != null) {
        postToSubscription(newSubscription, stickyEvent, isMainThread());
    }
}
```

根据事件可继承性进行处理，如果可继承则需要遍历所有粘性事件找出派生的事件进行发送，否则直接可以发送，接着 `checkPostStickyEventToSubscription()` 仅仅是判断事件不为空然后就进入 `postToSubscription()` ，这在前面已经分析过了。

到这里粘性事件的流程也结束了。

**Ending**