# EventBus 学习
EventBus...

 * simplifies the communication between components
    * decouples event senders and receivers
    * performs well with Activities, Fragments, and background threads
    * avoids complex and error-prone dependencies and life cycle issues
 * makes your code simpler
 * is fast
 * is tiny (~50k jar)
 * is proven in practice by apps with 100,000,000+ installs
 * has advanced features like delivery threads, subscriber priorities, etc.

* [项目地址](https://github.com/greenrobot/EventBus)

## 使用

EventBus in 3 steps<br>

1.Define events:
```java  
    public static class MessageEvent { /* Additional fields if needed */ }
```
2.Prepare subscribers:Declare and annotate your subscribing method, optionally specify a thread mode 
```java
    @Subscribe(threadMode = ThreadMode.MAIN)  
    public void onMessageEvent(MessageEvent event) {/* Do something */};
```
Register and unregister your subscriber. For example on Android, activities and fragments should usually register according to their life cycle:
```java
    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }
 
    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
```
3.Post events:
```java
    EventBus.getDefault().post(new MessageEvent());
```

# 分析过程 1-7 订阅 8、9 取消订阅 10-13 post 7、14 postSticky

## 重要变量
```java
    // subscriptionsByEventType是用EventType（即订阅方法的参数的class类型）作为key，存放相应的Subscription的列表，注意这个Subscription列表中存放的是不同订阅对象的Subscription，只是它们的EventType都和key一样，当然同一个订阅对象也可以在同一个key下有多个Subscription
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    // typesBySubscriber是用订阅者对象作为key，存放的是该对象所有订阅方法的EventType（即订阅方法的参数的class类型）组成的列表
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    // 与粘性事件相关
    private final Map<Class<?>, Object> stickyEvents;
```

## 1.EventBus -> getDefault()，获取EventBus的单例
```java
    /** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }
```

## 2.EventBus -> register
```java
    /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        // 跳转3 查询对象的类中需要订阅的方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                // 跳转7 订阅者的相应方法订阅
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

## 3.SubscriberMethodFinder -> findSubscriberMethods
```java
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        // METHOD_CACHE实际是ConcurrentHashMap，cache中有了方法的List就直接返回
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        // ignoreGeneratedIndex默认为false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            // 跳转4
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            // 缓存方法List
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

## 4.SubscriberMethodFinder -> findUsingInfo
```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        // prepareFindState获取一个FindState的对象，和Handler的obtainMessage方法差不多的重用机制，避免创建过多的对象
        FindState findState = prepareFindState();
        // initForSubscriber,重置FindState的各个属性
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            // 跳转5 getSubscriberInfo一般来说返回null
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    // FindState -> checkAdd检查方法是否已经添加过
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                // 跳转6 查找subscriberClass的方法保存到findState中
                findUsingReflectionInSingleClass(findState);
            }
            // 循环检查父类，但是不会检查java、javax、android包下的类
            findState.moveToSuperclass();
        }
        // getMethodsAndRelease返回SubscriberMethod对象的ArrayList，释放findState，有一个FIND_STATE_POOL缓存findState，避免产生过多的对象
        return getMethodsAndRelease(findState);
    }
```

## 5.SubscriberMethodFinder -> getSubscriberInfo
```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        // subscriberInfoIndexes是从EventBusBuilder传过来的，默认为null，所以该方法一般来说返回null
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

## 6.SubscriberMethodFinder -> findUsingReflectionInSingleClass
注意getDeclaredMethods与getMethods的区别：<br>
getMethods返回某个类的所有公用（public）方法包括其继承类的公用方法，当然也包括它所实现接口的方法。<br>
getDeclaredMethods()返回某个类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但<font color=#ff0000 size=3>不包括继承的方法</font>，当然也包括它所实现接口的方法<br>
```java
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        // 循环查询订阅的方法，包装为SubscriberMethod对象再添加到FindState的subscriberMethods中
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
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

## 7.EventBus -> subscribe
```java
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        // eventType是订阅方法的参数类型，订阅的方法只支持一个参数
        Class<?> eventType = subscriberMethod.eventType;
        // 用Subscription类包装订阅者和一个订阅的方法
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

        // 把newSubscription插入到subscriptions的相应位置
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        // 粘性事件的处理,粘性事件有一些特点：
        // 1.在发送的时候用postSticky(event) 方法。
        // 2.每种类型的事件只存储一个，多个事件会被覆盖成最后一个。
        // 3.新注册的订阅者，在注册之后就会马上收到之前发过的最后一个事件。
        // 4.接受事件的订阅者必须标明为sticky = true
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    // isAssignableFrom 判断两个类是否相同
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
```

## 8.EventBus -> unregister
```java
    public synchronized void unregister(Object subscriber) {
        // subscribedTypes是subscriber所有订阅方法参数类型的列表
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

## 9.EventBus -> unsubscribeByEventType
```java
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        // subscriptions是相应eventType下所有subscription列表
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                // 删除相应订阅者的subscription
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

## 10.EventBus -> unsubscribeByEventType
```java
    public void post(Object event) {
        // currentPostingThreadState是ThreadLocal对象，把event加入currentPostingThreadState的eventQueue中
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                // 循环执行所有事件
                while (!eventQueue.isEmpty()) {
                    // 跳转11 
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

## 11.EventBus -> postSingleEvent
```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            /** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            // 跳转12 调用postSingleEventForEventType
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                Log.d(TAG, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

## 12.EventBus -> postSingleEventForEventType
```java
     private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            // 循环执行postToSubscription
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
                    // 跳转13 postToSubscription
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

## 13.EventBus -> postToSubscription 
```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        // 根据不同的model选择执行的线程
        switch (subscription.subscriberMethod.threadMode) {
            // 当前线程调用
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            // 主线程调用
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    // mainThreadPoster是HandlerPoster对象，本质为一个Handler
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            // EventBus线程池按顺序调用
            case BACKGROUND:
                if (isMainThread) {
                    // backgroundPoster是BackgroundPoster对象，本质是一个Runnable
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            // EventBus线程池直接调用
            case ASYNC:
                //  asyncPoster是AsyncPoster对象，本质为一个Runnable
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

## 14.EventBus -> postSticky
```java
    public void postSticky(Object event) {
        // 与post的区别是将事件放入stickyEvents中，处理见7
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```