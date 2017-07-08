# objc-0926
更高效的同步锁-GCD 同步锁
转自http://www.jianshu.com/p/306481753216
本文整理自《Effective Objective-C 2.0》，通过分析比较不同的同步锁的优缺点，使用GCD方法一步步找到更高效的同步锁。
在Objective-C中，如果有多个线程要执行同一份代码，那么这时就会出现线程安全问题。首先，我们看下什么时候线程安全问题。

线程安全
如果一段代码所在的进程中有多个线程在同时运行，那么这些线程就有可能会同时运行这段代码。假如多个线程每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

由于可读写的全局变量及静态变量（在 Objective-C 中还包括属性和实例变量）可以在不同线程修改，所以这两者也通常是引起线程安全问题的所在。

Objective-C中的同步锁
在 Objective-C 中，如果有多个线程执行同一份代码，那么有可能会出现线程安全问题。这种情况下，就需要使用所来实现某种同步机制。

在 GCD出现之前，有两种方法，一种采用的是内置的“同步块”(synchronization block)，另一种方法是使用锁对象。

同步块(synchronization block)
- (void)synchronizedMethod {
    @synchronized (self) {
        //Safe
    }
}
这种写法会根据给定的对象，自动创建一个锁，并等待块中的代码执行完毕。执行到这段代码结尾处，锁就释放了。

该同步方法的优点就是我们不需要在代码中显式的创建锁对象，便可以实现锁的机制。然而，滥用@synchronized (self)则会降低代码效率，因为公用同一个锁的那些同步块，都必须按顺序执行。若是在self对象上频繁加锁，程序可能要等另一段与此无关的代码执行完毕，才能继续执行当前代码，这样效率就低了。
注：因为@synchronized (self)方法针对self只有一个锁，相当于对于self的所有用到同步块的地方都是公用同一个锁，所以如果有多个同步块，则其他的同步块都要等待当前同步块执行完毕才能继续执行。

```objc
- (void)synchronizedAMethod {
    @synchronized (self) {
        //Safe
    }
}

- (void)synchronizedBMethod {
    @synchronized (self) {
        //Safe
    }
}

- (void)synchronizedCMethod {
    @synchronized (self) {
        //Safe
    }
}
```
以上代码，如果当前synchronizedAMethod方法正在执行，则synchronizedBMethod和synchronizedCMethod方法需要等待synchronizedAMethod完毕后才能执行，不能达到并发的效果。

锁对象
```objc
@property (nonatomic,strong) NSLock *lock;

_lock = [[NSLock alloc] init];

- (void)synchronizedMethod {
    [_lock lock];
    //Safe
    [_lock unlock];
}
```
以上是简单锁对象的实现方式，但是如果锁使用不当，会出现死锁现象，这时可以使用NSRecursiveLock这种“递归锁”（recursive lock）。

除了以上锁对象，还有NSConditionLock 条件锁 、NSDistributedLock 分布式锁 ，这些适用于不同的场景，这里就不展开说了。

以上这些锁，使用的时候还是有缺陷的。在极端情况下，同步块会导致死锁，另外，效率也不见得高，而如果直接使用锁对象的话，一旦遇到死锁，就会非常麻烦。

GCD锁
在开始说GCD锁之前，我们先了解一下GCD的中的任务派发和队列。

任务派发
任务派发方式	说明
dispatch_sync()	同步执行，完成了它预定的任务后才返回，阻塞当前线程
dispatch_async()	异步执行，会立即返回，预定的任务会完成但不会等它完成，不阻塞当前线程
队列种类
队列种类	说明
串行队列	每次只能执行一个任务，并且必须等待前一个执行任务完成
并发队列	一次可以并发执行多个任务，不必等待执行中的任务完成
GCD队列种类
GCD队列种类	获取方法	队列类型	说明
主队列	dispatch_get_main_queue	串行队列	主线中执行
全局队列	dispatch_get_global_queue	并发队列	子线程中执行
用户队列	dispatch_queue_create	串并都可以	子线程中执行
以前同步锁的实现方式
在Objective-C中，属性就是开发者经常需要同步的地方。通常开发者想省事的话（以前我也是这样觉得），会这样写：


```objc
    - (NSString *)someString {
        @synchronized (self) {
            return _someString;
        }
    }

    - (void)setSomeString:(NSString *)someString {
        @synchronized (self) {
            _someString = someString;
        }
    }
```
以上代码除了上文提到的效率低以外，还有一个问题，就是该方法并不能保证访问该对象时绝对是线程安全的。虽然，这种方法在访问属性时，确实是“原子”的，也必定能从中获取到有效值，然而在同一线程上多次调用getter方法，每次获取到的结果未必相同。在两次访问操作之间，其他线程可能会写入新的属性值。此时，只能保证读写操作是“原子”的，而多个线程的执行顺序，我们没有办法控制。

使用GCD串行队列来实现同步锁
有种简单而高效的方法可以替代同步块或锁对象，那就是使用“串行同步队列”。将读取操作以及写入操作都安排在同一个队列里，即可保证数据同步。
用法如下：
```objc
        @property (nonatomic,strong) dispatch_queue_t syncQueue;

        _syncQueue = dispatch_queue_create("com.effetiveobjectivec.syncQueue", NULL);

        - (NSString *)someString {
            __block NSString *localSomeString;
            dispatch_sync(_syncQueue, ^{
                localSomeString = _someString;
            });
            return _someString;
        }

        - (void)setSomeString:(NSString *)someString {
            dispatch_sync(_syncQueue, ^{
                _someString = someString;
            });
        }
   ```
此模式的思路是：把设置操作与获取操作都安排在序列化的队列里执行，这样的话，所有针对属性的访问操作就都同步了。
注：getter方法中，用一个临时变量来保存值，是因为在block中return的话，只是return到block中了，没有真正返回到对应的getter方法中，而__block是为了可以在block中改变改临时变量而用。

虽然问题解决了，但是我们还可以进一步优化。设置方法不一定非得是同步的。设置实例变量所用的块，并不需要向设置方法返回什么值。那代码可以改成：

```objc
- (void)setSomeString:(NSString *)someString {
    dispatch_async(_syncQueue, ^{
        _someString = someString;
    });
}
```
这次把同步改成了异步，也许看来，这样改动，性能是会有提升的，但是你测一下程序的性能，可能会发现这种写法比原来慢。因为执行异步派发时，是需要拷贝块。若拷贝块所用的时间明显超过执行块所需的时间，则这种做法将比原来的更慢。
注：本例子代码比较简单，若是要执行的块代码逻辑比较复杂的话，那么该写法可能还是比原来的块些

使用GCD并发队列来实现同步锁
对于属性的读写，我们希望多个获取方法可以并发执行，而获取方法与设置方法之间不能并发执行，利用这个特点，还能写出更快一些的代码来。此时正可体现出GCD的好处。而用同步锁或锁对象，是无法轻易实现下面这种方案的。这次我们使用并发队列：
```objc
@property (nonatomic,strong) dispatch_queue_t syncQueue;

_syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

- (NSString *)someString {
    __block NSString *localSomeString;
    dispatch_sync(_syncQueue, ^{
        localSomeString = _someString;
    });
    return localSomeString;
}

- (void)setSomeString:(NSString *)someString {
    dispatch_async(_syncQueue, ^{
        _someString = someString;
    });
}
```
以上代码，还无法正确实现同步。因为所有读写操作都会在同一个队列上执行，而该队列是并发队列，所有读取和写入操作都可以随时执行，没有达到同步效果。此问题我们可以通过一个简单的GCD功能解决--栅栏（barrier）。下列函数可以向队列中派发块，将其作为栅栏使用：

void dispatch_barrier_async(dispatch_queue_t queue, dispatch_block_t block);
void dispatch_barrier_sync(dispatch_queue_t queue, dispatch_block_t block);
注：dispatch_barrier_async如果传入自己创建的并行队列时，会阻塞当前队列执行，而不阻塞当前线程。
dispatch_barrier_sync如果传入自己创建的并行队列时，阻塞当前队列的同时也会阻塞当前线程，请注意

并发队列如果发现接下来要处理的块是个栅栏块，那么就一直要等当前所有并发块都执行完毕，才会单独执行这个栅栏块。这待栅栏块执行完毕，在按正常方式继续向下处理。这样就解决了并发队列的同步问题。

GCD并发队列中加入栅栏
本例中，可以用栅栏块来实现属性的设置方法。在设置方法中使用了栅栏块之后，对属性的读取操作依然可以并发执行，但写入操作却必须单独执行了。在下图中演示的这个队列中，有多个读取操作，而且还有一个写入操作。


在这个并发队列中，读取操作是用普通的块来实现的，而写入操作则是用栅栏块来实现的
读取操作可以并行，但写入操作必须单独执行，因为它是栅栏块
实现代码很简单：
```objc

    @property (nonatomic,strong) dispatch_queue_t syncQueue;

    //_syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    /* 这里应该使用自己创建的并发队列，因为苹果文档中指出，如果使用的是全局队列或者创建的不是并发队列，
    则dispatch_barrier_async实际上就相当于dispatch_async，就达不到我们想要的效果了 */
    _syncQueue = dispatch_queue_create("com.effetiveobjectivec.syncQueue", DISPATCH_QUEUE_CONCURRENT);

    - (NSString *)someString {
        __block NSString *localSomeString;
        dispatch_sync(_syncQueue, ^{
            localSomeString = _someString;
        });
        return localSomeString;
    }

    - (void)setSomeString:(NSString *)someString {
        dispatch_barrier_async(_syncQueue, ^{
            _someString = someString;
        });
    }
```
测试一下性能，你就会发现，这种做法肯定比使用串行队列要快。其中，设置函数也可以改用同步栅栏块来实现，那样做可能会更高效，其原因之前已经解释过了——这里就要权衡拷贝块的时间和块执行时间了

