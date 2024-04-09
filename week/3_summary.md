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

### 세마포어
- 특정 자원이나 특정 연산을 동시에 사용하거나 호출할 수 있는 스레드의 수를 제한하고자 할 때 사용
- 가상의 퍼밋(permit)을 만들어 내부 상태를 관리하며, 세마포어를 생성할 때 생성자에 최초로 생성할 퍼밋 수를 넘김
- 외부 스레드는 퍼밋을 요청해 확보하거나, 이전에 확보한 퍼밋을 반납할 수 있음
- 현재 사용할 수 있는 퍼밋이 없는 경우, acquire 메소드는 남는 퍼밋이 생시거나, 인터럽트가 걸리거나, 타임아웃이 걸리기 전까지 대기함

```
public class BoundedHashSet <T> {
    private final Set<T> set;
    private final Semaphore sem;

    public BoundedHashSet(int bound) {
        this.set = Collections.synchronizedSet(new HashSet<T>());
        sem = new Semaphore(bound);
    }

    public boolean add(T o) throws InterruptedException {
        sem.acquire();
        
        boolean wasAdded = false;
        try {
            wasAdded = set.add(o);
            return wasAdded;
        } finally {
            if (!wasAdded)
                sem.release();
        }
    }

    public boolean remove(Object o) {
        boolean wasRemoved = set.remove(o);
        if (wasRemoved)
            sem.release();
        return wasRemoved;
    }
}
```

- 여기서 조금 재미있던건 세마포어도 내부 구현을 봤을 때, AbstractQueuedSynchronizer를 사용하고 있습니다.

### 배리어

- 배리어(barrier)는 특정 이벤트가 발생할 때까지 여러 개의 스레드를 대기 상태로 잡아둘 수 있다는 측면에서 래치와 비슷함
- 래치와의 차이점은 모든 스레드가 배리어 위치에 동시에 이르러야 관문이 열리고 계속해서 실행 할 수 있다는 점
- 배치는 "이벤트"를 기다리기 위한 동기화 클래스이고, 배리어는 "다른 스레드"를 기다리기 위한 동기화 클래스
- "모두들 오후 6시 출판사 앞에서 만나자. 약속 장소에 도착하면 모두 도착할 때까지 대기하고, 모두 도착하면 모두가 모인 후 어디로 이동할지 생각해보자"

```
public class CellularAutomata {
    private final Board mainBoard;
    private final CyclicBarrier barrier;
    private final Worker[] workers;

    public CellularAutomata(Board board) {
        this.mainBoard = board;
        int count = Runtime.getRuntime().availableProcessors();
        this.barrier = new CyclicBarrier(count,
                new Runnable() {
                    public void run() {
                        mainBoard.commitNewValues();
                    }});
        this.workers = new Worker[count];
        for (int i = 0; i < count; i++)
            workers[i] = new Worker(mainBoard.getSubBoard(count, i));
    }

    private class Worker implements Runnable {
        private final Board board;

        public Worker(Board board) { this.board = board; }
        public void run() {
            while (!board.hasConverged()) {
                for (int x = 0; x < board.getMaxX(); x++)
                    for (int y = 0; y < board.getMaxY(); y++)
                        board.setNewValue(x, y, computeValue(x, y));
                try {
                    barrier.await();
                } catch (InterruptedException ex) {
                    return;
                } catch (BrokenBarrierException ex) {
                    return;
                }
            }
        }
    }

    public void start() {
        for (int i = 0; i < workers.length; i++)
            new Thread(workers[i]).start();
        mainBoard.waitForConvergence();
    }
}
```
- 카운트랜치와는 조금 다르게 동작, 카운트랜치 사용법이 동시에 여러 이벤트를 같은 타이밍에 실행시키고 대기하는 거였더라면,
- 배리어는, 일단 실행시키고 모두 끝날때 까지 대기
- 결과적으로 비슷한 기능을 만들 수 있지만, 살짝의 용도가 차이가 있을 것 같음


### Cache 로직 개선

- 해당 부분에서 필자가 의도한건 메이비, 캐싱에 동기를 사용하지 말고, 비동기로 작업이 동작하도록 유도하는 것으로 파악.
```
@ThreadSafe
public class Factorizer implements Servlet {
    private final Computable<BigInteger, BigInteger[]> c =
            new Computable<BigInteger, BigInteger[]>() {
                public BigInteger[] compute(BigInteger arg) {
                    return factor(arg);
                }
            };
    private final Computable<BigInteger, BigInteger[]> cache = new Memoizer<BigInteger, BigInteger[]>(c);

    public void service(ServletRequest req, ServletResponse resp) {
        try {
            BigInteger i = extractFromRequest(req);
            encodeIntoResponse(resp, cache.compute(i));
        } catch (InterruptedException e) {
            encodeError(resp, "factorization interrupted");
        }
    }
```

### 스레드를 많이 생성할 때의 문제점 (이번 스터디의 핵심 아닐까?)

- 작업마다 스레드를 생성하는 정책은 상용서비스에서 사용하기는 무리
- 특정 상황에서 엄청나게 많은 대량의 스레드가 생성할 수 있는데 단점이 발생함

**스레드 라이프 사이클 문제 **

- 스레드를 생성하고 제거하는 작업에도 자원이 소모
- 스레드를 생성하는 과정에는 일정량의 시간이 필요 따라서 클라이언트 요청을 처리할때 딜레이가 발생
- 클라이언트 요청이 자주 발생하는 유형이면 요청이 들어올때마다 새로운 스레드를 생성하는 일이 상대적으로 전체작업에서 많은 부분을 차지 할 수 있다.

**자원 낭비**

- 실행중인 스레드는 시스템자원 특히 메모리 소모
- 하드웨어 실제로 장착되어있는 프로세서보다 많은 수의 스레드가 만들어져 동작중이라면, 실제로는 대부분의 스레드가 대기 상태에 머무른다.
- 대기상태에 머무르는 스레드가 많아지면 많아질수록 많은 메모리 필요로하며 JVM 가비지 콜렉터에 가해지는 부하가 늘어날 뿐만 아니라 CPU를 사용하기 위해 여러스레드가 경쟁하여 메모리 이외에도 많은자원 소모
- cpu 개수의 해당하는 스레드가 동작중이라면, 스레드를 더 만들어낸다해도 성능이 직접적으로 개선 되지 않을수 있으며, 악영향을 미칠 가능성도 있음

**안전성 문제**

- 모든 시스템에는 생성할 수 잇는 스레드의 개수가 제한
- 제한되어있는 개수 플랫폼과 운영체제마다 다름, jvm을 실행할떄 지정하는 인자나  Thread 클래스에 필요한 스택의 크기에 따라서 달라짐
- 만약 제한된 양을 모두 사용하고나면 OutOfMemoryError 가 발생 
  - 오류 바로잡을 방법 별로없으며 가능하다고해도 안정적으로 처리하기 어려움
  - 제한된 값 안에서 동작하도록 생성해 OutOfMemoryError 가 발생하지 않도록 방지하는 방법이 쉬움
- 32비트 시스템이라면 큰 제약요소는 스레드 스택에 적용되는 주소공간
  - 자바 코드와 네이티브 코드를 실행할수 있도록 모든 스레드는 두개의 스택 

- 스레드 추가로 만들어서 사용해서 성능상의 이점을 얻을 수 있지만 특정 수준을 넘어간다면 성능이 떨어짐
  - 계속 생성한다면 어플리케이션 다운됨
- 어플리케이션이 만들어 낼 수 있는 스레드 제한을 두는것이 현명한 방법
- 제한을 행을때 제한된 수의 스레드만으로 동작할때 용량이 넘치는 요청이  들어오는 상황에서도 자원이 고갈되어 멈추는경우가 발생하지 않는지 세심하게 테스트해야함
- 실제 개발하는 과정에서 문제 없을 수 있음 서버가 다운되고 이러한 문제는 엄청난 부하가 가해졌을때만 발생 
- 이런 문제가 발생한다면 성능이 점진적으로 떨어지도록 만들어야 할 서버 어플리케이션에서 치명적인 오류이다.

**본인이 생각하는 스레드를 많이 생성할 때 문제점**
- Thread 1개는 대략 1mgb의 용량을 가짐
- Thread를 계속 생성하게 되면 oom 발생여지가 있음
- Thread의 Context-Switch 비용이 많이 발생 -> 로직 느려짐
- 물론, 돈이 많은 구조라면, Thread를 많이 생성하는 것도 좋은 방식이라고 생각함 (Thread에 대한 유후시간 처리가 복잡하니까)
