---
title: "Rxjava源码阅读"
date: 2019-09-20T21:34:06+08:00
draft: false
tags: ["Android", "Rxjava", "源码", "开源框架"]
categories: ["工作"]
---

<div align="center">
<img src=https://imgoldjii.oss-cn-beijing.aliyuncs.com/rxjava0022.png width=80%/>
</div>

本文不对Rxjava的基本使用进行讲解，仅对源码做分析，如果你对Rxjava的基本使用还有不清楚的，建议学习官方文档之后再阅读本文

[ReactiveX文档中文翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/Operators.html)<br/>
[Rxjava](https://github.com/ReactiveX/RxJava)<br/>
[给Android开发者的RxJava详解](http://gank.io/post/560e15be2dca930e00da1083)

本文会逐一解析Rxjava的create()、subscribe()、操作符、subscribeOn()、obsweveOn()的源码，模式是先给出一段模版代码，然后逐渐深入分析

<!--more-->

# 正文

## Create()方法
这里给出一个最简单的Rxjava的实例
```java
Observable.create(new ObservableOnSubscribe<String>() {
			@Override
			public void subscribe(ObservableEmitter<String> e) throws Exception {
				e.onNext("next");
				e.onComplete();
			}
		}).subscribe(new Observer<String>() {
			@Override
			public void onSubscribe(Disposable d) {
				Log.d(TAG, "onSubscribe: " + d);
			}
			@Override
			public void onNext(String value) {
				Log.d(TAG, "onNext: " + value);
			}
			@Override
			public void onError(Throwable e) {
				Log.d(TAG, "onError: " + e);
			}
			@Override
			public void onComplete() {
				Log.d(TAG, "onComplete: ");
			}
		});
```
直接看create()方法主体
```java
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
		ObjectHelper.requireNonNull(source, "source is null");
		return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
	}
```
调用对象和返回对象都为Observable，而传入参数为ObservableOnSubscribe
```java
public interface ObservableOnSubscribe<T> {
		void subscribe(@NonNull ObservableEmitter<T> e) throws Exception;
	}
```
这是一个接口，仅包含一个方法，就是上面我们在new ObservableOnSubscribe时候需要重写的那个方法。
再看subscribe()的形参类型ObservableEmitter
```java
public interface ObservableEmitter<T> extends Emitter<T> {
		void setDisposable(@Nullable Disposable d);
		void setCancellable(@Nullable Cancellable c);
		boolean isDisposed();

		@NonNull
		ObservableEmitter<T> serialize();
		@Experimental
		boolean tryOnError(@NonNull Throwable t);
	}
```
发现这也是一个接口，不需要太过关注，值得关注的是他的上层，由接口特性我们知道Emitter肯定也是一个接口，我们来看下它定义了什么方法
```java
public interface Emitter<T> {
		void onNext(@NonNull T value);

		void onError(@NonNull Throwable error);

		void onComplete();
	}
```
看到这三个熟悉的方法，你就知道为什么我们实例化的ObservableEmitter对象e可以调用onNext()、onError()、onComplete()这三个方法了

**create()的参数已经看完了，下面看下create()的内容**

第一句`ObjectHelper.requireNonNull(source, "source is null");`是判空代码。
返回值是`RxJavaPlugins.onAssembly(new ObservableCreate<T>(source))`，我看看下这个方法。
```java
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
		...
		return source;
	}
```
具体的内容我们不需要去解读，我们只看他的返回值和传入参数，经过观察发现都是Observable类型，乍看好像没什么问题，但是看上面，源码中传入的是一个ObservableCreate类型，所以这里ObservableCreate有适配器的作用，将ObservableOnSubscribe适配为Observable类型。
下面我们就看看这个适配器ObservableCreate

```java
public final class ObservableCreate<T> extends Observable<T> {
		final ObservableOnSubscribe<T> source;
		
		public ObservableCreate(ObservableOnSubscribe<T> source) {
			this.source = source;
		}
		
		@Override
		protected void subscribeActual(Observer<? super T> observer) {
			
			CreateEmitter<T> parent = new CreateEmitter<T>(observer);
			
			observer.onSubscribe(parent);
			
			source.subscribe(parent);
			
			...
		}
	}
```
成员变量、构造方法略过，我们先看看这里频频出现的观察者observer
```java
public interface Observer<T> {
    void onSubscribe(@NonNull Disposable d);
    void onNext(@NonNull T t);
    void onError(@NonNull Throwable e);
    void onComplete();
}
```
同样的也是一个接口，定义的这个四个方法就是我们在订阅时，观察者需要重写的四个方法，注意与上面的Emitter接口及其三个方法进行区分。
看这行`observer.onSubscribe(parent);`，由上面我们知道observer.onSubscribe()是接受Disposable类型，而这里的parent是CreateEmitter类型，你可能已经猜出来了，没错，这里的CreateEmitter也是一个适配器，前面的ObservableCreate对被观察者进行了适配，CreateEmitter则对观察者进行了适配，将observer类型转化为Disposable类型，下面看下他的源码
```java
static final class CreateEmitter<T> extends AtomicReference<Disposable> implements ObservableEmitter<T>, Disposable {

		...
		
		@Override
		public void onNext(T t) {
			if (t == null) {
				onError(new NullPointerException("..."));
				return;
			}
			if (!isDisposed()) {
				observer.onNext(t);
			}
		}

		@Override
		public void onError(Throwable t) {
			if (!tryOnError(t)) {
				
				RxJavaPlugins.onError(t);
			}
		}

		@Override
		public void onComplete() {
			if (!isDisposed()) {
				try {
					observer.onComplete();
				} finally {
					dispose();
				}
			}
		}

		@Override
		public void dispose() {
			DisposableHelper.dispose(this);
		}
		
		...
	}
```
主要就是重写了那四个方法，定义了规则，比如：
- onComplete()与onError()互斥，切CreateEmitter在回调他们两中任意一个后，都会自动dispose()
- Observable和Observer的关系没有被dispose，才会回调Observer的onXXXX()方法

> 并且，到这里你对onCreate()中的数据流动也一定有了一定的理解：<br/>
--> `e.onNext("next")` e是ObservableEmitter，是一个接口 <br/>
--> `ObservableCreate.CreateEmitter.onNext("next")` ObservableCreate是Observable装饰类，CreateEmitter使其内部类也是Observer的装饰类并实现了上面的这个接口 <br/>
--> `Observer.onNext("next")` <br/>
--> `Log.d(TAG, "onNext: "+value)`

我们再回到ObservableCreate的subscribeActual()中。

`source.subscribe(parent);`，最重要的是这一行，调用者是被观察者，传入的参数为观察者，基本可以猜出来了，这里是订阅的作用，真正将被观察者与观察者联系起来的地方

## subscribe()方法
```java
public final void subscribe(Observer<? super T> observer) {
		ObjectHelper.requireNonNull(observer, "observer is null");

		observer = RxJavaPlugins.onSubscribe(this, observer);

		subscribeActual(observer);
		...
	}
```
第一句的作用同样是判空，接下来先获取了传入的observer并进行了相关配置，然后调用`subscribeActual(observer);`，细心的同学可能注意到了，subscribeActual()正是在上面ObservableCreate中被重写的方法，而具有“**订阅**”意义的那行代码也包含其中，结合subscribe()的本意，这行代码的作用也很明朗了

如果你只是想对Rxjava基本的数据传输流程、订阅的原理感兴趣，那么就不用看下去了，下面的内容主要是**Rxjava操作符**、**线程调度**、**背包**的源码分析

---
## 操作符（Map）
开始分析Rxjava的操作符部分

我们以Map操作符为例展开分析，首先，还是给出一个最简单的实例
```java
Observable.create(new ObservableOnSubscribe<String>() {
			@Override
			public void subscribe(ObservableEmitter<String> e) throws Exception {
				e.onNext("next");
				e.onComplete();
			}
		}).map(new Function<String, Integer>() {
			@Override
			public Integer apply(String s) throws Exception {
				return Integer.parseInt(s);
			}
		}).subscribe(new Observer<Integer>() {
			@Override
			public void onSubscribe(Disposable d) {
				Log.d(TAG, "onSubscribe: " + d);
			}
			@Override
			public void onNext(Integer value) {
				Log.d(TAG, "onNext: " + value);
			}
			@Override
			public void onError(Throwable e) {
				Log.d(TAG, "onError: " + e);
			}
			@Override
			public void onComplete() {
				Log.d(TAG, "onComplete: ");
			}
		});
```
先看map()方法整体
```java
public final <R > Observable < R > map(Function < ? super T, ? extends R > mapper){
			ObjectHelper.requireNonNull(mapper, "mapper is null");
			return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
		}
```
返回值肯定是Observable，参数是一个泛型接口，我们看下这个接口
```java
public interface Function<T, R> {
    R apply(@NonNull T t) throws Exception;
}
```
传入T，返回R，符合Map操作符传入两个数据类型进行转换的效果，在意料之中。
继续看map()的方法内容，第一行按照惯例是判空语句，我们发现map()的return语句与create()极为相似，都是调用了RxJavaPlugins.onAssembly()，仅是传入的参数不同，其实不只是Map操作符，大多操作符都是这样的，他们的不同仅仅是传入参数的不同，也就是适配器的不同，这说明，操作符的具体实现（比如Map的类型转换）都是在各自的适配器中做的。
> 小结：create以及对大多数操作符的retun语句都是RxJavaPlugins.onAssembly()，仅是传入参数不同

进入ObservableMap的部分
```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
			final Function<? super T, ? extends U> function;

			public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
				super(source);
				this.function = function;
			}

			@Override
			public void subscribeActual(Observer<? super U> t) {
				source.subscribe(new MapObserver<T, U>(t, function));
			}
		}
```
我们发现，ObservableMap做的事情很少，就三件事，第一：在构造方法中，将传入的Observable也就是本身抛给父类（ObservableSource是Observable的父类，所以可以接受）；第二：对转换逻辑funtion进行保存；第三：重写subscribeActual()方法并在其中实现**订阅**，这里与ObservableCreate是一样的，只是传递的参数不同
> 小结：create以及对大多数操作符的第一层适配器中都会重写subscribeActual()并实现**订阅**逻辑

我们并没有在ObservableMap的代码中发现进行类型转换的代码，不要心急，有的同学估计已经发现了，这里的进行订阅操作的`source.subscribe()`传入的参数类型改变了 ，之前是CreateEmitter，现在变为了一个叫MapObserver的类，我们知道CreateEmitter中实现了那四个常用的方法并制定了相关规则，所以你推测MapObserver中做了同样的操作，其实不是的，但也差不了太多，除onNext()之外的三个方法是在它的父类BasicFuseableObserver中重写的，MapObserver中只对onNext()进行的重写，而且在其中**进行了数据类型转换**的工作，我们看一下源码(这里我们只看onNext()部分就可以了)
```java
static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
			@Override
			public void onNext(T t) {
				if (done) {
					return;
				}
				if (sourceMode != NONE) {
					actual.onNext(null);
					return;
				}
				U v;
				try {
					v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
				} catch (Throwable ex) {
					fail(ex);
					return;
				}
				actual.onNext(v);
			}
			...
		}
```
可以看到再代码中利用ObjectHelper将上游传过来的T，转换成了下游需要的U

> 到这里你对.map()下的数据流动也一定有了一定的理解：<br/>
--> `e.onNext("next")`<br/>
--> `ObservableMap.MapObserver.onNext ("next")`<br/>
--> `Observer.onNext("next")` <br/>
--> `Log.d(TAG, "onNext: "+value)`<br/>
> 订阅的发送顺序：<br/>
--> `.subscribe(observer)`<br/>
--> `ObservableMap.subscribeActual(observer)`<br/>
--> `ObservableCreate.subscribeActual(new MapObserver(observer,function))`

---
下面进入**线程调度**源码分析的阶段，先看subscribeOn()。

## 线程调度-subscribeOn()
老规矩，先来一个参考代码
```java
Observable.create(new ObservableOnSubscribe<String>() {
			@Override
			public void subscribe(ObservableEmitter<String> e) throws Exception {
				e.onNext("next");
				e.onComplete();
			}
		}).subscribeOn(Schedulers.io())
				.subscribe(new Observer<String>() {
					@Override
					public void onSubscribe(Disposable d) {
						Log.d(TAG, "onSubscribe: " + d);
					}
					@Override
					public void onNext(String value) {
						Log.d(TAG, "onNext: " + value);
					}
					@Override
					public void onError(Throwable e) {
						Log.d(TAG, "onError: " + e);
					}
					@Override
					public void onComplete() {
						Log.d(TAG, "onComplete");
					}
				});
```
还是一样，直接看SubscribeOn()
```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
			ObjectHelper.requireNonNull(scheduler, "scheduler is null");
			return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
		}
```
返回值Observable情理之中，return返回RxJavaPlugins.onAssembly()也是一样，两点不同：
- 装饰类（也就是上文说的适配器）是ObservableSubscribeOn
- 传入参数为一个Scheduler

进入ObservableSubscribeOn
```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
	final Scheduler scheduler;

	public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
		super(source);
		this.scheduler = scheduler;
	}

	@Override
	public void subscribeActual(final Observer<? super T> s) {
		final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);
		s.onSubscribe(parent);
		parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
	}
}
```
根据经验，构造中进行调用父类、存值一些操作，没什么可看的，直接看订阅的实现subscribeActual()方法，可以看见，这次对下游观察者进行封装的适配器是SubscribeOnObserver类，根据CreateEmitter、MapObserver的经验，我们可以猜测出它或它的父类肯定实现了那四个方法，下面我们看一下
```java
static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

	private static final long serialVersionUID = 8094547886072529208L;

	final Observer<? super T> actual;

	final AtomicReference<Disposable> s;

	SubscribeOnObserver(Observer<? super T> actual) {
		this.actual = actual;
		this.s = new AtomicReference<Disposable>();
	}

	@Override
	public void onSubscribe(Disposable s) {
		DisposableHelper.setOnce(this.s, s);
	}

	@Override
	public void onNext(T t) {
		actual.onNext(t);
	}

	@Override
	public void onError(Throwable t) {
		actual.onError(t);
	}

	@Override
	public void onComplete() {
		actual.onComplete();
	}

	@Override
	public void dispose() {
		DisposableHelper.dispose(s);
		DisposableHelper.dispose(this);
	}

	@Override
	public boolean isDisposed() {
		return DisposableHelper.isDisposed(get());
	}

	void setDisposable(Disposable d) {
		DisposableHelper.setOnce(this, d);
	}
}
```
除去构造、四个方法、基本的存储语句就剩下一个setDisposable()方法了，如果你对Scheduler有研究，你就知道在Scheduler中真正处理线程调用逻辑的是Worker类，这里setDisposable()的作用就是将你传入的Scheduler返回的worker加入管理。

目光回到subscribeActual()中，调用观察者的onSubscribe()之后，马上调用了parent.setDisposable()，这里停一下，你可以翻上去观察一下其他方法的subscribeActual()部分，都是在这时候执行**订阅**操作，但是我们在这里并没有发现，订阅操作不可能没有发生，那么是不是发生在了parent.setDisposable()这个方法里面呢？我们之前只关注了这个方法的内容，对于传入的参数还没有解析，我们现在看一下，希望有新的发现。

传入的参数是`scheduler.scheduleDirect(new SubscribeTask(parent))`。
先看SubscribeTask这个类
```java
final class SubscribeTask implements Runnable {
	private final SubscribeOnObserver<T> parent;

	SubscribeTask(SubscribeOnObserver<T> parent) {
		this.parent = parent;
	}

	@Override
	public void run() {
		source.subscribe(parent);
	}
}
```
这个类继承Runnable，所以实现了一个子线程，在run()中执行操作，没错，`source.subscribe(parent);`，熟悉的语句，这就证明了这里的**订阅**操作发生在了Scheduler的线程中。

我们继续看scheduleDirect()这个方法
```java
public Disposable scheduleDirect(@NonNull Runnable run) {
	return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}
```
继续
```java
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
	final Worker w = createWorker();

	final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

	DisposeTask task = new DisposeTask(decoratedRun, w);

	w.schedule(task, delay, unit);

	return task;
}
```
我们可以发现，传入的子线程被包装配置之后，开始在Worker也就是Scheduler线程中执行
我们继续看DisposeTask这个类，具体的订阅子线程的启动就在这里
```java
static final class DisposeTask implements Runnable, Disposable {
	final Runnable decoratedRun;
	final Worker w;

	Thread runner;

	DisposeTask(Runnable decoratedRun, Worker w) {
		this.decoratedRun = decoratedRun;
		this.w = w;
	}

	@Override
	public void run() {
		runner = Thread.currentThread();
		try {
			decoratedRun.run();
		} finally {
			dispose();
			runner = null;
		}
	}

	@Override
	public void dispose() {
		if (runner == Thread.currentThread() && w instanceof NewThreadWorker) {
			((NewThreadWorker) w).shutdown();
		} else {
			w.dispose();
		}
	}

	@Override
	public boolean isDisposed() {
		return w.isDisposed();
	}
}
```
可以看到run()中调用了`ecoratedRun.run();`来启动线程，注意**这里是使用的run()而不是start()**，而且整个rxjava流程走完后会自己调用`dispose();`关闭线程。
> 到这里，你应该明白了subscribeOn()线程调度的过程，正如它的效果描述一样：将观察者的操作运行在Scheduler.io()线程中<br/>
--> subscribeOn(Schedulers.io())<br/>
--> 返回一个ObservableSubscribeOn的包装类<br/>
--> 当上游的被观察者被订阅之后，回调ObservableSubscribeOn包装类中的subscribeActual()<br/>
--> 线程切换至Schedulers.io()，并进行**订阅操作**`source.subscribe(parent)`

> 理顺思路之后我们发现，这里订阅，模式与之前相同，还是下游观察者对上游被观察者进行订阅，依旧是自下向上的，但是我们通过之前的源码分析知道，上游发送数据时调用的那个四个方法实际是调用下游观察者对应重写的四个方法，所以这边满足了线程调度的目的：将观察者所做的操作置与Schedulers.io()线程中

> 并且，我们这里还可以解释一个问题
> **为什么subscribeOn(Schedulers.xxx())切换线程N次，总是以第一次为准**？
> 我们知道使用subscribeOn()进行线程调度时订阅的顺序是从下往上，所以有多个subscribeOn()时，从最后一个开始执行，一直执行到第一个，最后的结果还是以第一个为准

---
然后看obsweveOn()，有了上面subscribeOn()的经验，分析obsweveOn()就快了

## 线程调度-observeOn()
实例
```java
Observable.create(new ObservableOnSubscribe<String>() {
			@Override
			public void subscribe(ObservableEmitter<String> e) throws Exception {
				e.onNext("next");
				e.onComplete();
			}
		}).subscribeOn(Schedulers.io())
				.observeOn(AndroidSchedulers.mainThread())
				.subscribe(new Observer<String>() {
					@Override
					public void onSubscribe(Disposable d) {
						Log.d(TAG, "onSubscribe: " + d);
					}

					@Override
					public void onNext(String value) {
						Log.d(TAG, "onNext: " + value);
					}

					@Override
					public void onError(Throwable e) {
						Log.d(TAG, "onError: " + e);
					}

					@Override
					public void onComplete() {
						Log.d(TAG, "onComplete: ");
					}
				});
```
observeOn()
```java
public final Observable<T> observeOn (Scheduler scheduler){
	return observeOn(scheduler, false, bufferSize());
}
```
没看见RxJavaPlugins.onAssembly()，担心不一样？不存在的，被包了一层而已
```java
public final Observable<T> observeOn (Scheduler scheduler,boolean delayError, int bufferSize){
	ObjectHelper.requireNonNull(scheduler, "scheduler is null");
	ObjectHelper.verifyPositive(bufferSize, "bufferSize");
	return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
还是那个顺序，ObservableObserveOn
```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
	final Scheduler scheduler;
	final boolean delayError;
	final int bufferSize;

	public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
		super(source);
		this.scheduler = scheduler;
		this.delayError = delayError;
		this.bufferSize = bufferSize;
	}

	@Override
	protected void subscribeActual(Observer<? super T> observer) {
		if (scheduler instanceof TrampolineScheduler) {
			source.subscribe(observer);
		} else {
			Scheduler.Worker w = scheduler.createWorker();
			source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
		}
	}
}
```
看subscribeActual()，很好理解，就是先判断是不是在主线程，是的话，直接**订阅**完事，不是的话跳到主线程去，在**订阅**，切换线程依旧是使用的Worker那一套，与subscribeOn()中类似，先创建一个主线程的Worker，然后把Worker放进观察者的包装类ObserveOnObserver中，不用多说，里面肯定有对那四个方法的实现，我这里简化一下他的代码
```java
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T> implements Observer<T>, Runnable {

	@Override
	public void onNext(T t) {
		if (done) {
			return;
		}
		if (sourceMode != QueueDisposable.ASYNC) {
			queue.offer(t);
		}
		schedule();
	}
	
	...
	
	void schedule() {
		if (getAndIncrement() == 0) {
			worker.schedule(this);
		}
	}
}
```
其他那三个方法与onNext()大致相同，只看这一个就可以了，`schedule();`这行代码上面都是取数据的操作，并没有对数据进行发送，所以说即使使用线程调用将被观察者的操作放在主线程，他的数据准备阶段仍然是在原线程执行的，当`schedule();`执行后，进入上面传入Workder线程，也就是主线程，然后才将queue中的T取出，继而发送给下游的观察者。其他方法也是一样的流程，比如onError()、onComplete()，都是将错误或完成的信息先保存，等待切换线程后在执行发送操作。

> 由此，我们可知ObserverOn()是向下作用的，每次调用都对下游的代码产生作用，所以多次调用ObserverOn()，是最后一次生效的

---

好了，Rxjava的源码分析到这里结束了，文中有很多没有讲到的地方，日后有时间的会继续讲解剩余部分。

# 总结

我们来整理一下文中出现的各个装饰者
```xml
Observable.create()
	- ObservableCreate
	- CreateEmitter
Observable.map()
	- ObservableMap
	- MapObserver
Observable.subscribeOn()
	- ObservableSubscribeOn
	- SubscribeOnObserver
Observable.observeOn()：
	- ObservableObserveOn
	- ObserveOnObserver

-------------
第一层装饰者的作用：
	- 对被观察者进行适配
	- 根据自己的需求实现subscribeActual()
第二层装饰者的作用：
	- 对观察者进行适配
	- 根据自己的需求实现onNext()、onError()、onComplete()...等上层定义的方法
```


本文中，我们对create()、subscribe()、map()、subscribeOn()、observeOn()的源码进行的阅读，想你已经可以从源码的角度回答以下问题：
- 被观察者如何发送数据？
- 观察者如何接受数据？
- 操作符的实现原理是什么？
- Map关键字是如何实现类型转换的？
- 线程调度是如何实现的？
- 为什么多次调用subscribeOn()，只有第一次生效？
- 为什么多次调用observeOn()，只有最后一次生效？