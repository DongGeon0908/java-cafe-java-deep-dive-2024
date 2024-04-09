# 3주차 Summary

### 동기화 클래스
> 자바에는 동기화를 지원하는 여러 로직이 존재한다.

### 래치 (CountDownLatch)

- 특점 시점까지 스레드를 종료시키지 않고, 대기시킨다.
- 내부적으로 Queue로 구현되어 있음
- 어떤 동시 작업을 진행할때, 해당 작업들이 모두 동시에 시작하게 진행하거나, 완료되어 join하는 시점을 통일시킴

```java
public class CountDownLatch {
    private final Sync sync;

    public CountDownLatch(int count) {
        if (count < 0) {
            throw new IllegalArgumentException("count < 0");
        } else {
            this.sync = new Sync(count);
        }
    }

    public void await() throws InterruptedException {
        this.sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
        return this.sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void countDown() {
        this.sync.releaseShared(1);
    }

    public long getCount() {
        return (long)this.sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + this.sync.getCount() + "]";
    }

    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            this.setState(count);
        }

        int getCount() {
            return this.getState();
        }

        protected int tryAcquireShared(int acquires) {
            return this.getState() == 0 ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            int c;
            int nextc;
            do {
                c = this.getState();
                if (c == 0) {
                    return false;
                }

                nextc = c - 1;
            } while(!this.compareAndSetState(c, nextc));

            return nextc == 0;
        }
    }
}

```

### FutureTask

- Executor 프레임웍에서 비동기적인 작업을 실행하고자할 때 사용하며, 시간이 많이 필요한 작업이 있을 때 실제 결과가 필요한 시점 이전에 미리 작업을 실행시켜두는 용도로 사용함
- 미리 연산을 시켜두는 용도
```java
public class Preloader {
    private final FutureTask<ProductInfo> future =
            new FutureTask<ProductInfo>(new Callable<ProductInfo>() {
                public ProductInfo call() throws DataLoadException {
                    return loadProductInfo();
                }
            });
    private final Thread thread = new Thread(future);

    public void start() {
        thread.start();
    }

    public ProductInfo get() throws DataLoadException, InterruptedException {
        try {
            return future.get();
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof DataLoadException)
                throw (DataLoadException) cause;
            else
                throw launderThrowable(cause);
        }
    }
}
```



