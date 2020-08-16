# RateLimiter源码阅读
基于guava-23.0版本

## RateLimiter概述
RateLimiter是一个基于令牌桶算法实现::限流器::，常用于控制网站的QPS。与Semaphore不同，Semaphore控制的是某个时间点的访问量，而RateLimiter控制的是某一时间间隔内的访问量。
## RateLimiter的使用方法
以每秒2次的速度执行任务
```java
final RateLimiter rateLimiter = RateLimiter.create(2.0); // rate is "2 permits per second"
void submitTasks(List<Runnable> tasks, Executor executor) {
	for (Runnable task : tasks) {
		rateLimiter.acquire(); // may wait
		executor.execute(task);
}
}
```
以每秒5000字节的速度发送数据
```java
final RateLimiter rateLimiter = RateLimiter.create(5000.0); // rate = 5000 permits per second
void submitPacket(byte[] packet) {
	rateLimiter.acquire(packet.length);
	networkService.send(packet);
}
```
## RateLimiter的类结构
![](https://github.com/617076674/Study-The-Code/blob/master/pictures/RateLimiter/RateLimiter%E7%9A%84%E7%B1%BB%E7%BB%93%E6%9E%84.png)
RateLimiter是一个抽象类，SmoothRateLimiter是继承了RateLimiter的一个抽象类。SmoothWarmingUp和SmoothBursty是SmoothRateLimiter的两个静态内部类，是RateLimiter的两种实现。
## 抽象类RateLimiter
### 属性
RateLimiter中只有两个属性
```java
private final SleepingStopwatch stopwatch;
private volatile Object mutexDoNotUseDirectly;
```
我们可以看到，stopwatch的类型是SleepStopWatch，这是定义在RateLimiter中的一个静态抽象类，在介绍这个类之前，先讲一下StopWatch这个类。
#### StopWatch
StopWatch是一个以纳秒为单位的计时器。我们知道，在System类中有一个方法也是以纳秒为单位进行计时的：
```java
public static native long nanoTime();
```
nanoTime这个方法是一个native方法，其返回的绝对值是没有什么使用价值的，两个nanoTime方法返回的差值才是有使用价值的，可以用于统计某段程序的执行时间，使用方法如下：
```java
long startTime = System.nanoTime();
// ... the code being measured ...
long estimatedTime = System.nanoTime() - startTime;
```
StopWatch是对nanoTime方法的一层抽象封装，只提供计算时间相对值的功能，用法如下：
```java
Stopwatch stopwatch = Stopwatch.createStarted();
doSomething();
stopwatch.stop(); // optional
Duration duration = stopwatch.elapsed();
log.info("time: " + stopwatch); // formatted string like "12.3 ms"
```
#### SleepingStopwatch
SleepStopWatch包含了两个抽象方法：
```java
protected abstract long readMicros();
protected abstract void sleepMicrosUninterruptibly(long micros);
```
readMicros方法返回一个相对时间戳，该时间戳是一个以豪秒为单位的，相对于SleepStopWatch被创建时刻的时间戳。

sleepMicrosUninterruptibly方法用于使当前线程sleep一段时间，和Thread.sleep方法不同的是，在这段时间内不会响应InterruptedException异常。

回到RateLimiter里的两个属性：
```java
private final SleepingStopwatch stopwatch;
private volatile Object mutexDoNotUseDirectly;
```
我们可以推测，stopWatch是一个计时工具，并能提供sleep功能；mutexDoNotUseDirectly是一个用于同步的锁对象。
### 静态工厂方法
#### 生成SmoothBursty对象的静态工厂方法
```java
public static RateLimiter create(double permitsPerSecond) {
	return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}

@VisibleForTesting
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
  RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
  rateLimiter.setRate(permitsPerSecond);
  return rateLimiter;
}
```
SmoothBursty会预先缓存1s能产生的令牌数量，在该令牌被消耗时，以固定的速率产生新的令牌，但令牌池中的总数目不会超过其1s能产生的令牌数量。

举个例子：

假设一个SmoothBursty产生令牌的速度是每秒1个，有4个线程分别在以下4个时间点用acquire方法获取1个令牌：
（1）线程T0在0s时刻尝试获取1个令牌。
（2）线程T1在1.05s时刻尝试获取1个令牌。
（3）线程T2在2s时刻尝试获取1个令牌。
（4）线程T3在3s时刻尝试获取1个令牌。

在2s时刻，距离上一次令牌被消耗的时刻1.05s只有0.95s，线程T2还不能立即获取到令牌，在2.05s时才能获取成功。同理，线程T3在3.05s时刻才能获取成功。
#### 生成SmoothWarmingUp对象的静态工厂方法
```java
public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) {
  checkArgument(warmupPeriod >= 0, "warmupPeriod must not be negative: %s", warmupPeriod);
  return create(
      permitsPerSecond, warmupPeriod, unit, 3.0, SleepingStopwatch.createFromSystemTimer());
}

@VisibleForTesting
static RateLimiter create(
    double permitsPerSecond,
    long warmupPeriod,
    TimeUnit unit,
    double coldFactor,
    SleepingStopwatch stopwatch) {
  RateLimiter rateLimiter = new SmoothWarmingUp(stopwatch, warmupPeriod, unit, coldFactor);
  rateLimiter.setRate(permitsPerSecond);
  return rateLimiter;
}
```
SmoothWarmingUp适用于资源需要预热的场景。假设业务在稳定状态下，可以承受的最大QPS是1000。如果线程池是冷的，让系统立即达到1000QPS会拖垮系统，需要有一个预热升温的过程。表现在SmoothWarmingUp中，从令牌池中获取令牌是需要等待时间的，该等待时间随着越来越多的令牌被消耗会逐渐缩短，直至一个稳定的等待时间。
### acquire方法
```java
@CanIgnoreReturnValue
public double acquire() {
  return acquire(1);
}

@CanIgnoreReturnValue
public double acquire(int permits) {
  // reserve方法的返回值表示何时能获取令牌
  long microsToWait = reserve(permits);
  // sleep一段时间，直到能够获取令牌，因此如果不能获取到令牌，acquire方法会阻塞当前线程
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}

final long reserve(int permits) {
  checkPermits(permits);
  synchronized (mutex()) {
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}

final long reserveAndGetWaitLength(int permits, long nowMicros) {
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
	// 如果当前时间已经大于等于了能获取到令牌的时间，需要等待的时间为0
  return max(momentAvailable - nowMicros, 0);
}

/**
 * 这是一个抽象方法，在 SmoothRateLimiter 中实现，用于获取能获得 permits 个令牌的时间戳
 */
abstract long reserveEarliestAvailable(int permits, long nowMicros);
```
### tryAcquire方法
```java
public boolean tryAcquire() {
  // 默认传入的超时时间是 0
  return tryAcquire(1, 0, MICROSECONDS);
}

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits);
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    // 由于传入的超时时间 timeoutMicros 是 0，所以不会阻塞
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}

/**
 * 这是一个抽象方法，在 SmoothRateLimiter 中实现，用于获取能获得 1 个令牌的时间戳
 */
abstract long queryEarliestAvailable(long nowMicros);
```
## 抽象类SmoothRateLimiter
抽象类SmoothRateLimiter继承自RateLimiter，其包含有4个属性：
### 属性
```java
// 当前令牌池中缓存的令牌数量
double storedPermits;
// 令牌池中能够缓存的最大令牌数量
double maxPermits;
// 产生一个令牌的时间
double stableIntervalMicros;
// 只有当前时间戳大于等于 nextFreeTicketMicros 时，才能从令牌池中获取令牌
private long nextFreeTicketMicros = 0L;
```
### doSetRate方法
```java
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
  resync(nowMicros);
  double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
  this.stableIntervalMicros = stableIntervalMicros;
  doSetRate(permitsPerSecond, stableIntervalMicros);
}

/**
 * 根据当前时间戳 nowMicros 更新 storedPermits 和 nextFreeTicketMicros
 */
void resync(long nowMicros) {
  // if nextFreeTicket is in the past, resync to now
  if (nowMicros > nextFreeTicketMicros) {
    // 对于 SmoothBursty 而言，coolDownIntervalMicros 方法的返回结果是 0.0，因此 newPermits 是无穷大。
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}

/**
 * 在子类 SmoothWarmingUp 和 SmoothBursty 中实现，返回结果是在 cool down 阶段，获取一个令牌所需等待的时间间隔。cool down 阶段是 SmoothWarmingUp 中的一个阶段，对于 SmoothBursty 而言，返回结果是 stableIntervalMicros。
 */
abstract double coolDownIntervalMicros();

/**
 * 在子类 SmoothWarmingUp 和 SmoothBursty 中实现。
 */
abstract void doSetRate(double permitsPerSecond, double stableIntervalMicros);
```
### reserveEarliestAvailable方法
```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  resync(nowMicros);
  // 返回值是 nextFreeTicketMicros
  long returnValue = nextFreeTicketMicros;
	// 从令牌缓存池中获取到的令牌数量
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  // 除了令牌缓存池中的令牌外，还需额外生产的令牌数量
  double freshPermits = requiredPermits - storedPermitsToSpend;
  // waitMicros = 从令牌缓存池中获取 storedPermitsToSpend 个令牌所需花费的时间 + 生产 freshPermits 个新令牌所需的时间
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
          + (long) (freshPermits * stableIntervalMicros);

  // 更新 nextFreeTicketMicros
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  // 更新 storedPermits
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}

/**
 * 在子类 SmoothWarmingUp 和 SmoothBursty 中实现。特别地，对于 SmoothBursty，从令牌池中获取令牌不需要等待时间，因此返回值是 0。
 */
abstract long storedPermitsToWaitTime(double storedPermits, double permitsToTake);
```
## 具体实现类SmoothBursty
```java
static final class SmoothBursty extends SmoothRateLimiter {
  /** The work (permits) of how many seconds can be saved up if this RateLimiter is unused? */
  // 令牌缓存池中能够预先缓存 maxBurstSeconds 时间内产生的令牌数量
  final double maxBurstSeconds;

  SmoothBursty(SleepingStopwatch stopwatch, double maxBurstSeconds) {
    super(stopwatch);
    this.maxBurstSeconds = maxBurstSeconds;
  }

  @Override
  void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = this.maxPermits;
    maxPermits = maxBurstSeconds * permitsPerSecond;
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
      // if we don't special-case this, we would get storedPermits == NaN, below
      storedPermits = maxPermits;
    } else {
      // 由于产生令牌的速率发生了改变导致了令牌缓存池中能够缓存的最大令牌数量发生了变化，进而导致令牌缓存池中已缓存的令牌数量进行等比例的缩放
      storedPermits =
          (oldMaxPermits == 0.0)
              ? 0.0 // initial state
              : storedPermits * maxPermits / oldMaxPermits;
    }
  }

/**
 * 对于 SmoothBursty，从令牌池中获取令牌不需要等待时间，因此返回值是 0。
 */
  @Override
  long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
    return 0L;
  }

/**
 * 生产令牌的时间间隔。
 */
  @Override
  double coolDownIntervalMicros() {
    return stableIntervalMicros;
  }
}
```
## 具体实现类SmoothWarmingUp
在分析具体实现类之前，先来分析一张图：
![](https://github.com/617076674/Study-The-Code/blob/master/pictures/RateLimiter/SmoothWarmingUp.png)
由上图可知，随着令牌池中令牌数量的减少，每产生一个令牌的时间会越来越少。令牌池中令牌数量在thresholdPermits ~ maxPermits时为预热阶段。

根据SmoothWaringUp的实现来看，cool down阶段所消耗的总时间是 warm up阶段所消耗的总时间的一半，即矩形面积是梯形面积的一半。
```java
static final class SmoothWarmingUp extends SmoothRateLimiter {
  private final long warmupPeriodMicros;
  /**
   * The slope of the line from the stable interval (when permits == 0), to the cold interval
   * (when permits == maxPermits)
   */
  private double slope;
  private double thresholdPermits;
  private double coldFactor;

  SmoothWarmingUp(
      SleepingStopwatch stopwatch, long warmupPeriod, TimeUnit timeUnit, double coldFactor) {
    super(stopwatch);
    this.warmupPeriodMicros = timeUnit.toMicros(warmupPeriod);
    // 事实上，在 SmoothWaringUp 的实现中，coldFactor 写死为 3
    this.coldFactor = coldFactor;
  }

  @Override
  void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = maxPermits;
    double coldIntervalMicros = stableIntervalMicros * coldFactor;
	  // 矩形面积公式
    thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;
    // 梯形面积公式
    maxPermits =
        thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);
    // 计算斜率
    slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
      // if we don't special-case this, we would get storedPermits == NaN, below
      storedPermits = 0.0;
    } else {
      // 由于产生令牌的速率发生了改变导致了令牌缓存池中能够缓存的最大令牌数量发生了变化，进而导致令牌缓存池中已缓存的令牌数量进行等比例的缩放
      storedPermits =
          (oldMaxPermits == 0.0)
              ? maxPermits // initial state is cold
              : storedPermits * maxPermits / oldMaxPermits;
    }
  }

  @Override
  long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
    double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
    long micros = 0;
    // measuring the integral on the right part of the function (the climbing line)
    if (availablePermitsAboveThreshold > 0.0) {
      // 在梯形中获得的令牌数量
      double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
      // TODO(cpovirk): Figure out a good name for this variable.
      // length = 上底 + 下底
      double length = permitsToTime(availablePermitsAboveThreshold)
              + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
      micros = (long) (permitsAboveThresholdToTake * length / 2.0);
      // permitsAboveThresholdToTake 个令牌数已经在梯形区域获取
      permitsToTake -= permitsAboveThresholdToTake;
    }
    // measuring the integral on the left part of the function (the horizontal line)
    // 加上矩形中的面积
    micros += (stableIntervalMicros * permitsToTake);
    return micros;
  }
  
  // 对于梯形，根据令牌池中的令牌数 permits 计算获取一个令牌所需的时间
  private double permitsToTime(double permits) {
    return stableIntervalMicros + permits * slope;
  }

  /**
   * warmupPeriodMicros / maxPermits = stableIntervalMicros，即生产令牌的时间间隔
   */
  @Override
  double coolDownIntervalMicros() {
    return warmupPeriodMicros / maxPermits;
  }
}
```