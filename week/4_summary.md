# Week 4

### 목차

- 7장 중단 및 종료
  - 7장 중단 및 종료
  - 7.1 작업중단
  - 7.2 스레드 기반 서비스 중단
  - 7.3 비정상적인 스레드 종료 상황 처리
  - 7.4 JVM 종료

<br>
<hr>
<br>

**7장 Summary**

- 작업을 취소하고 인터럽트를 거는 부분에 대한 개념 설명
- 작업이나 서비스 취소 요청에도 잘반응 하도록 프로그램하는 방법을 살펴볼 예정


<br>
<hr>
<br>

## 7.1 작업중단

실행중인 작업을 취소하고자 하는 요구사항은 여러가지 경우에 나타남

- 사용자가 취소하기를 요구한경우
  - GUI 화면에서 취소 버튼 클릭
  - JMX(java management Extension) 등의 관리 인터페이스를 통해 작업을 취소하도록 요청한경우
- 시간이 제한된 작업
  - 제한된 시간이 지나면 그시점에 계속 동작중이던 작업은 모두 취소
- 어플리케이션 이벤트
  -  특정 작업 결과로 원하던 값을 얻으면 나머지 실행중이던 작업은 취소되는 경우
- 오류
  - 웹 크롤러가 진행하던 특정 작업에서 오류가 발생하면 다른 작업도 모두 취소시켜야된다.
- 종료 
  - 급하게 종료해야 하는 경우라면 실행중이던 작업을 모두 취소 시켜야 할 수도 있음

- 자바언어에서는 특정 스레드를 명확하게 종료시킬수 있는 방법은 없으며, 특정작업을 임의로 종료 시킬 수 있는 방법이 없다는말
  - 작업을 실행하는 스레드와 작업을 취소했으면 한다고 요청하는 스레드가 함께 작업을 멈추는 협력적인 방법을 사용해야함


- 취소 요청이 들어왔다는 플래그를 설정하고 실행중인 작업은 취소 요청 플래그를 주기적으로 확인하는 방법

~~~java
@ThreadSafe
public class PrimeGenerator implements Runnable {
    @GuardedBy("this")

    private final List<BigInteger> primes = new ArrayList<>();
    private volatile boolean cancelled;

    @Override
    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }

    public void cancel() {cancelled = true;}

    public synchronized List<BigInteger> get() {
        return new ArrayList<>(primes);
    }
}
~~~

- volatile 변수를 사용해 취소 상태를 확인
- 취소 요청이 들어오기 전까지 계속해서 소수를 찾아내는 작업을 진행
- 취소 요청 플래그를 사용해 작업을 멈추도록 되어있음
- 이런 방법이 안정적으로 동작하도록 하려면 cancelled 변수를 반드시 volatile 형식으로 선언해야한다.
- 가장 기본적인 취소 정책
- 외부 프로그램에서는 cancel 메소드를 호출해 취소 요청을 보낼 수 있고 PrimeGenerator 는 소수를 찾을 때마다 취소 요청이 들어왔는지 확인할것이며 취소요청이 들어왔을 경우 즉시 작업을 멈춤

~~~java
    List<BigInteger> aSecondOfPrimes() throws InterruptedException {
        PrimeGenerator generator = new PrimeGenerator();
        new Thread(generator).start();
        try {
            SECONDS.sleep(1);
        } finally {
            generator.cancel();
        }
        return generator.get();
    }
~~~

- 1초간 소수를 계산하는 프로그램
- 정확하게 1초후에 멈춰 있으리라는 보장 X
- 취소를 실행하는 시점 이후에 run 메소드 내부의 반복문이 cancelled 플래그를 확인할때까지 최소한의 시간이 흘러야 하기 떄문이다.
- cancel 메소드는 finally 구문에서 호출하도록 되어있기 때문에 sleep 메소드를 호출해 대기하던 도중에 인터럽트가 걸린다해도 소수계산작업은 반드시 멈출 수 있다.
- 작업을 쉽게 취소 시킬 수 있도록 만들려면 작업을 취소할 때 어떻게 언제 어떤일을 해야하는지 이른바 취소 정책을 명확히 정의해야한다.
  - ex) 수표에 대한 지급을 중단하는 것


<br>
<br>

### 인터럽트

~~~java
public class BrokenPrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;
    private volatile boolean cancelled = false;

    BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run(){
        try {
            BigInteger p = BigInteger.ONE;
            while (!cancelled){
                queue.put(p = p.nextProbablePrime());
            }
        }catch (InterruptedException consumed){

        }
    }


    public void cancel() {cancelled = true;}

    void consumPrimes() throws InterruptedException{
        BlockingQueue<BigInteger> primes =...;
        BrokenPrimeProducer producer = new BrokenPrimeProducer(primes);
        producer.start();
        try{
            while(needMorePrimes())
                consume(primes.take());
        }finally {
            producer.cancel();
        }
    }

}
~~~

- 프로듀서가 대기중인 상태로 계속 멈춰있을 가능성이 있는 안전하지 않은 취소방법의 예
- 이런코드는 금물!!
- 프로듀서는 put 메소드에서 멈춰잇고 put 메소드에서 멈춘작업을 풀어줘야 할 컨슈머가 더이상 작업을 처리지 못하기 때문에 cancelled 변수를 확인할 수 없다.

- **실제 상황에서는 작업을 중단하고자 하는 부분이 아닌 다른부분에 인터럽트를 사용한다면 오류가 발생하기 쉬울 수 밖에 없으며, 애플리케이션 규모가 커질수록 관리하기도 어려워진다.**

~~~java
public class Thread{
    public void interrupt(){...}
    public boolean isInterrupted(){...}
    public static boolean interrupted() {...}
    ...
}
~~~

- Thread 클래스와 인터럽트 관련 메소드
- interrupt 메소드는 해당하는 스레드에 인터럽트를 거는 역할
- isInterrupted메소드는 해당 스레드에 인터럽트가 결려 있는지 알려준다.
- 스태틱으로 선언된 interrupted 메소드를 호출하면 현재 스레드의 인터럽트 상태를 해제하고, 해제하기 이전의 값이 무엇이었는지 알려준다.
  - interrupted 메소드는 인터럽트 상태를 해제할 수 있는 유일한 방법
- Thread.sleep, Object.wait 메소드와 같은 블로킹 메소드는 인터럽트 상태를 확인하고 있다가 인터럽트가 걸리면 즉시 리턴된다.
- 대기하던 중에 인터럽트가 갈리면 인터럽트 상태를 해제하면서 InterruptedException 을 던진다. 
- InterruptedException은 인터럽트가 발생해 대기중이던 상태가 예상보다 빨리 끝났다는것을 뜻한다.
- 인터럽트가 결렸을때 인터럽트가 결렷다는 사실을 얼마나 빠르게 확인하는지는 JVM 보장하지 X
- **특정 스레드의 interrupt 메소드를 호출한다해도 해당 스레드가 처리하던 작업을 멈추지 않는다 단지 해당스레드에게 인터럽트 요청이 있었다는 메세지만 전달할 뿐이다.**
- wait, sleep, join 메소드는 인터럽트 요청을 굉장히 심각하게 처리
  - 실제로 인터럽트 요청을 받거나 실행할때 인터럽트 상태라고 지정했던 시점이 되는 순간 예외를 던진다.
- static interrupted 메소드는 현재 스레드의 인터럽트 상태를 초기화하기때문에 사용할때 상당히 주의가 필요함

~~~java
public class PrimeProducer extends Thread{
    private final BlockingQueue<BigInteger> queue;

    PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    public void run(){
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()){
                queue.put(p = p.nextProbablePrime());
            }
        }catch (InterruptedException consumed){
            /*스레드를 종료한다.*/
        }
    }


    public void cancel() { interrupt();}
}
~~~

- 인터럽트를 사용해 작업을 취소
- **작업 취소 기능을 구현하고자 할대는 인터럽트가 가장 적절한 방법이라고 볼 수 있다**
- 인터럽트를 사용하면 작업 취소 기능을 훨씬 쉽고 간결하게 구현
- 인터럽트 상태를 확인하면 소수를 계산하는것처럼 시간이 오래 걸리는 작업을 시작조차 하지 않도록 할 수 있기 때문에 PrimeProducer 클래스가 취소되는 시점에 CPU 등의 자원을 덜 사용하게되어 그 의미를 충분히 찾을 수 있음
- 반복문의 조건 확인 부분에서 인터럽트 여부를 확인하는 방법으로 응답 속도를 개선 할 수 있다.


<br>
<br>

### 인터럽트 정책

- 인터럽트 요청이 들어 왔을때 해당 스레드가 인터럽트를 어떻게 처리해야 하는지 대한 지침
  - ex) 어디서 무슨일을하고 인터럽트에 대비해 단일 연산으로 보호 할 수 있는 범위가 어디까지인지도 확인 인터럽트가 발생했을떄 해당하는 인터럽트에 어떻게 재빠르게 대응할지의 지침
- 가장 범용적인 정책은 스레드 수준이나 서비스 수준에서 작업 중단 기능을 제공하는 것
- 작업task과 스레드가 인터럽트 상황에서 서로 어떻게 동작해야하는지 명확하게 구분해야 할 필요가 있다.
  - ex) 스레드 풀에서 작업을 실행하는 스레드에 인터럽트를 거는것은 현재작업을 중단하라라는 의미 일 수도있고 작업 스레드를 중단시켜라는 뜻일 수 도 있다
  - 스레드 풀에서 작업을 실행하는 스레드의 인터럽트 상태를 그대로 유지해 스레드를 소유하는 프로그램이 인터럽트 상태에 직업 대응 할 수 있도록 해야한다. 
- 작업은 실행중인 인터럽트 상태를 그대로 유지시켜줘야한다. 일반적인 방법은 InterruptedException 을 던지는 것
- Thread.currentThread().interrupt(); 
- **각 스레드는 각자의 인터럽트 정책을 갖고 있다. 따라서 해당 스레드에서 인터럽트 요청을 받았을때 어떻게 동작할지 정확하게 알고 있지 않은 경우에는 함수로 인터럽트를 걸어서는 안된다.**
- 자바에서 제공하는 인터럽트 기능관련 비판적인 의견이있음 선점형 preemptive 인터럽트 기능 제공 X 
- 개발자가 직업 InterruptedException 처리하도록 강요
- 인터럽트에 대한 실제적인 중단 시점을 개발자가 임의로 늦출 수도 있도록 하는것은 프로그램의 요구사항에 지정되어 있는 응답성과 안정성을 능동적으로 관리 할 수 있는 기회를 제공하는 셈이다.


<br>
<br>

### 인터럽트에 대한 대응

- 발생한 예외를 호출 스택의 상위 메소드로 전달한다 
  - 이방법을 사용하는 메소드는 역시 인터럽트를 걸수 있는 블로킹 메소드가 된다.
- 호출 스택 상단에 위치한 메소드가 직접처리 할 수 있도록 인터럽트 상태를 유지한다.

~~~java
BlockingQueue<Task> queue;
...
public Task getNextTask() throws InterruptedExcetpion {
    return queue.take();
}
~~~

- InterruptedException 을 상위 메소드로 전달
- InterruptedException 상위 메소드로 전달 할 수 없는 경우(Runnable 인터페이스를 구현해 작업을 정의한 경우)
  - interrupt 메소드를 다시 한번 호출
- InterruptedException 잡아낸다음에 아무런 행동을 취하지 않고 말그대로 예외를 먹어버리는 일은 하지말아야됨

- **스레드의 인터럽트 처리 정책을 정확하게 구현하는 작업만이 인터럽트 요청을 삼켜버릴수 있다. 일반적인 용도로 작성된 작업이나 라이브러리 메소드는 인터럽트 요청을 그냥 삼켜버려서는 안된다.**

~~~java
    public Task getNextTask(BlockingQueue<Task> queue){
        boolean interrunted = false;
        try{
            while (true){
                try{
                    return queue.take();
                }catch (InterruptedException e){
                    interrunted = true;
                    // 그냥넘어가고 재시도
                }
            }
        }finally {
            if(interrunted)
                Thread.currentThread().interrupt();
        }
    }
~~~

- 인터럽트 상태를 종료 직전에 복구시키는 중단 불가능한 작업
- 작업 중단 기능을 지원하지 않으면서 인터럽트를 걸 수 잇는 블로킹 메소드를 호출하는 작업은 인터럽트가 걸렸을때 블로킹 메소드의 기능을 자동으로 재시도하도록 반복문 내부에서 블로킹 메소드를 호출하도록 구성하는것이 좋다.
- 인터럽트 상태를 내부적으로 보관하고 잇다가 메소드가 리턴되기 직전에 인터럽트 상태를 원래대로 복구하고 리턴되도록 해야한다.
- 인터럽트 걸수 잇는 블로킹 메소드는 대부분 실행되자마자 가장 먼저 인터럽트 상태를 확인하며 인터럽트가 걸린 상태라면 즉시 InterruptedException 을 던지는 경우가 많기 때문에 인터럽트 상태를 너무 일찍 지정하면 반복문이 무한 반복에 빠질수 있다.
- 현재 스레드의 인터럽트 상태를 확인해준다면 인터럽트에 대한 응답속도를 크게 높일 수 있다.
- 인터럽트 상태를 얼마만에 한번씩 확인해야 할 것인지 주기를 결정할떄에는 응답 속도와 효율성 측면에서 적절한 타협점을 찾아야한다.
- 만약 응답 속도가 굉장히 중요한 어플리케이션을 만들고 있다면 인터럽트에 재빠르게 응답하지 않으면서 오래 실행되는 메소드를 호출하지 않는것이 좋은데, 이런 요구사항을 충족시키려다보면 자바라이브러리 메소드 가운데에서도 호출하지 않아야 할 것들이 눈에 띌수 있다.
- 작업 중단 기능은 인터럽트 상태 뿐만아니라 다른 여러가지 다른 상태와 관련이 있을 수  있다.
- 인터럽트는 해당 스레드의 주의를 끄는 정도로만 사용하고, 인터럽트 요청하는 스레드가 갖고 있는 다른 상태 값을 사용해 인터럽트가 걸린 스레드가 어떻게 동작해야 하는지를 지정하는 경우도 있다(물론 자신의 상태값을 다른 스레드가 읽어가도록 열어 둘때에는 해당 상태값의 동기화에 신경써야한다.)
- 예를 들어 ThreadPoolExecutor 내부의 풀에ㅔ 등록되어 있는 스레드에 인터럽트가 걸렸다면, 인터럽트가 걸린 스레드는 전체 스레드 풀이 종료되는 상태인지 먼저 확인
- 스레드 풀 자체가 종료되는 상태였다면 스레드를 종료하기전에 스레드 풀을 정리하는 작업을 실행하고, 스레드풀이 종료되는 상태가 아니라면 스레드 풀에서 동작하는 스레드의 수를 그대로 유지시킬 수있또록 새로운 스레드를 하나 생성해 풀에 등록시킨다.

### 이번 파트에서 중요한 내용은 다음과 같다.

1. Thread를 강제로 종료시킬 수 없다.
2. 대신, interrupt를 통해 종료해줘! 라는 시그널을 보낼 수 있다.
3. 시그널을 보낸 것 뿐이지, 실제 종료된 것이 아니다.
4. 실제로 종료할려면, 종료할 스레드가 동작하는 로직에 대비책이 있어야 한다. (Interrup시 어떤 행동을 할 것인지)

### 우리가 고민해 볼 필요

1. 스레드를 강제 중지, 혹은 어떠한 로직에 대한 강제 중지 로직을 보통 어떻게 구성하는가?

