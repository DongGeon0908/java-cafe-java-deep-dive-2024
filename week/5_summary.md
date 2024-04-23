# 5장 Summary

**스레드 부족 데드락**
- 스레드 풀의 크기가 크더라도 실행되는 모든 스레드가 큐에 쌓여 아직 실행되지 않은 작업의 결과를 받으려고 대기중리라면 동일한 상황이 발생할 수 있다.
- 이런 현상을 스레드 부족 데드락 thread starvation deadlock 이라고 하며 특정 자원을 확보하고자 계속해서 대기하거나 풀 내부의 다른 작업이 실행돼야 알 수 있는 조건이 만족하기를 기다리는것처럼 끝없이 계속 대기할 가능성이 있는 가ㅣ능을 사용하는 작업이 풀에 등록 된 경우에는 언제든지 발생 할 수 있다.
- 필요한 작업을 데드락 없이 실행 시킬수 잇을만큼 풀의 크기가 충분히 크다면 물론 문제가 없을 수도 있다.
- **완전히 독립적이지 않은 작업을 Executor에 등록할때 항상 스레드 부족 데드락이 발생할 수 있다는 사실을 염두에 둬야하며, 작업을 구현한 코드나 Executor를 설정하는 설정파일등에 항상 스레드 풀의 크기나 설정에 대한 내용을 설명해야 한다.**

**오래 실행 되는 작업**
- 특정 작업이 예상보다 긴 시간동안 종료되지 않고 실행된다면 스레드 풀의 응답속도에 문제점이 생긴다.
- 오래 실행될 것이라고 예상되는 작업이 대략 몇개인지를 알고 있을때 그 개수에 비해 스레드 풀의 크기가 상당히 작은 수준이라면 시간이 지나면서 스레드풀에 속한 스레드 가운데 상당수가 오래 실행되는 작업에 잡혀있을 가능성이 크다. 이러한 상황에 다다르면 스레드풀의 응답 속도가 크게 느려진다.

**스레드 풀 크기 조정**
- CPU 을 많이 사용하는 작업의 경우 N개의 CPU 을 탑재하고 있는 하드웨어에서 스레드풀을 사용할때는 스레드의 개수를 N+1로 맞추면 최적의 성능을 발휘한다고 알려져있다.
- I/O 작업이 많거나 기타 다른 블로킹 작업을 해야하는 경우라면 어느 순간에 모든 스레드가 대기 상태에 들어가 전체적인 진행이 멈출 수 있기 때문에 스레드풀의 크기를 훨씬 크게 작아야할 필요가 있다.
- Nthread = Ncpu * Ucpu * (1+W/C) (스레드 풀 사이즈 공식)
- cpu 의 개수는 Runtime availableProcessors 메소드로 다음과 같이 알아 낼 수 있다.

**집중 대응 정책**
- 크기가 제한된 큐에 작업이 가득차면 집중대응 staturation policy 가 동작한다
- ThreadPoolExecutor의 집중대응 정책은 setRejectedExecutionHandler 메소드를 사용해 원하는 정책으로 변경 할 수 잇다.
- 여러가지 종류의 RejectedExecutionHandler 를 사용해 다양한 집중 대응 정책을 적용 할 수 있다.
- AbortPolicy, CallRunsPolicy, DiscardPolicy, DiscardOldestPolicy 등이 있다.
- 중단 abort 정책 기본적으로 사용
  - execute 메소드에서 RuntimeException 을 상속받은 RejectedExecutionException 을 던진다.
  - RejectedExecutionException을 잡아서 더이상 추가 할 수 없는ㅅ 상황에 직접 대응해야한다
- 제거 discard 정책
  - 큐에 작업을 더이상 쌇을수 없다면 방금 추가시키려고 했던 정책을 아무 반응없이 제거해버린다.
  - 이와 비슷한 오래된 항목 제거 discard oldest 정책은 큐에 쌇은 항목중 가장 오래되어 다음에 실행될 예정이던 작업을 제거하고, 추가하고자 했던 작업을 큐에 다시 추가해본다.
- 호출자 실행 caller runs 정책
  -  작업을 제거해 버리거나 예외를 던지지 않으면서 큐의 크기를 초과하는 작업을 프로듀서에게 거꾸로 넘겨 작업 추가 속도를 늦출 수 있도록 일종의 속도 조절 방법으로 사용한다
  -  새로 등록하려고 했던 작업을 스레드 풀의 작업 스레드로 실행하지 않고 execute 메소드를 호출해 작업을 등록하려 했던 스레드에서 실행시킨다.
  -  ex) 웹서버 여러단계를 거쳐 부하가 전달 되기 때문에 부하가 걸린 상태에서도 점진적으로 어플리케이션 성능이 떨어지도록 조절가능

**ThreadPoolExecutor 상속**
- ThreadPoolExecutor 는 애초부터 상속받아 기능을 추가 할 수 있도록 만들어짐
- 상속받은 하위 클래스가 오버라이드해 사용 할 수 잇도록 beforeExecute, afterExecute, terminated 와 같은 여러 가지 hook도 제공하고 있으며, 이런 훅을 사용하면 훨씬 다양한 기능을 구사 할 수 있다.
- beforeExecute, afterExecute 메소드는 작업을 싱행할 스레드의 내부에서 호출하도록 되어 있으며, 로그 메세지를 남기거나 작업 실행시점이 언제인지 기록해두거나 실행 상태를 모니터링하거나 기타 다양한 통계값을 뽑는등의 작업을 하기에 적당하다
- afterExecute 훅 메소드는 run 메소드가 정상적으로 종료되거나 아니면 예외가 발생해 Execption을 던지고 종료되는 등의 어떤 상황에서도 항상 호출된다.(Error 일때는 호출 안됨) 
- beforeExecute 메소드에서 RuntimeException 이 발생한다면 해당 작업도 실행되지 않을 뿐더러 afterExecute 메소드 역시 실행되지 않으니 주의
- 스레드 풀이 종료절차를 마무리한 후 모든작업과 모든 스레드가 종료되고나면 terminated 훅 메소드 호출
- terminated 메소드에서는 Executor 가 동작하는 과정에서 사용했던 각종 자원을 반납하는 등 일을 처리하거나 여러가지 알람이나 로그출력 다양한 통계값을 확보하는 등의 작업을 진행하기에 적당한 메소드이다.

### 질문 논의 사항
1. ThreadPoolExecutor 를 보통 어떤 식으로 사용하나?
본인의 경우, 다음과 같이 Factory Class를 활요하여 구성한다.

```
@Slf4j
@RequiredArgsConstructor
public class ExecutorGenerator {
    private final int corePoolSize;
    private final int maxPoolSize;
    private final int queueCapacity;
    private final String threadNamePrefix;

    public ThreadPoolTaskExecutor generate() {
        var threadPoolTaskExecutor = new ThreadPoolTaskExecutor();
        threadPoolTaskExecutor.setCorePoolSize(this.corePoolSize);
        threadPoolTaskExecutor.setMaxPoolSize(this.maxPoolSize);
        threadPoolTaskExecutor.setQueueCapacity(this.queueCapacity);
        threadPoolTaskExecutor.setThreadNamePrefix(this.threadNamePrefix + "-");
        threadPoolTaskExecutor.initialize();
        log.info("generate ThreadPoolTaskExecutor : " + this.threadNamePrefix);
        return threadPoolTaskExecutor;
    }
}

@Slf4j
@Configuration
@EnableAsync
public class AsyncConfig extends AsyncConfigurerSupport {
    @Bean(name = "taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        var executor = new ExecutorGenerator(10, 15, 15, "taskExecutor");

        return executor.generate();
    }
}
```

2. **데이터 공유 모델, 분할 데이터 모델**

```
데이터 공유 모델, 분할 데이터 모델이라는 내용이 9장에 나왔는데,
api 서버로 보면 각각의 레이어가 서로 다른 모델 정보를 다뤄야 한다?라고도 생각해 볼 수 있을 것 같다.

그렇다면, 우리는 api-server를 만들때, DTO, Model을 각각의 Layer마다 서로 다르게 설정하나?
혹은 다들 사용하는 방식은?

exampe) controller, service, repository 모두 다른 모델 (인스턴스)를 사용.
controller에서 requestA를 사용, service에서는 requestB를 사용, repostory에서는 별도의 모델C를 사용
```

3. Thread의 수 혹은 메모리 등의 다양한 지표에 대한 모니터링을 하는가? 한다면 목적은?
- Thread와 Memory의 수가 적정한지 파악을 해얄하는 이유는?

4. @Async를 통해 비동기 처리를 많이 하는데, @Async(valie=) 설정에 pool을 지정하지 않았다면?

- 1. TaskExecutor 관련 빈이 있는지 확인, 없다면 SimpleAsyncTaskExecutor 사용
```
이 구현은 컨텍스트에서 고유한 org.springframework.core.task.TaskExecutor 빈을 검색하거나 그렇지 않으면 "taskExecutor"라는 이름의 Executor 빈을 검색합니다. 둘 중 어느 것도 해결 가능하지 않은 경우(예: BeanFactory 전혀 구성되지 않은 경우) 이 구현은 기본값을 찾을 수 없는 경우 로컬 사용을 위해 새로 생성된 SimpleAsyncTaskExecutor 인스턴스로 대체됩니다.

	/**
	 * This implementation searches for a unique {@link org.springframework.core.task.TaskExecutor}
	 * bean in the context, or for an {@link Executor} bean named "taskExecutor" otherwise.
	 * If neither of the two is resolvable (e.g. if no {@code BeanFactory} was configured at all),
	 * this implementation falls back to a newly created {@link SimpleAsyncTaskExecutor} instance
	 * for local use if no default could be found.
	 * @see #DEFAULT_TASK_EXECUTOR_BEAN_NAME
	 */
	@Override
	@Nullable
	protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
		Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
		return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
	}
```

- 2. SimpleAsyncTaskExecutor()에 대해
 
```
/**
 * {@link TaskExecutor} implementation that fires up a new Thread for each task,
 * executing it asynchronously. Provides a virtual thread option on JDK 21.
 *
 * <p>Supports a graceful shutdown through {@link #setTaskTerminationTimeout},
 * at the expense of task tracking overhead per execution thread at runtime.
 * Supports limiting concurrent threads through {@link #setConcurrencyLimit}.
 * By default, the number of concurrent task executions is unlimited.
 *
 * <p><b>NOTE: This implementation does not reuse threads!</b> Consider a
 * thread-pooling TaskExecutor implementation instead, in particular for
 * executing a large number of short-lived tasks. Alternatively, on JDK 21,
 * consider setting {@link #setVirtualThreads} to {@code true}.
 *
 * @author Juergen Hoeller
 * @since 2.0
 * @see #setVirtualThreads
 * @see #setTaskTerminationTimeout
 * @see #setConcurrencyLimit
 * @see org.springframework.scheduling.concurrent.SimpleAsyncTaskScheduler
 * @see org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor
 */
@SuppressWarnings({"serial", "deprecation"})
public class SimpleAsyncTaskExecutor extends CustomizableThreadCreator
		implements AsyncListenableTaskExecutor, Serializable, AutoCloseable {

	/**
	 * Permit any number of concurrent invocations: that is, don't throttle concurrency.
	 * @see ConcurrencyThrottleSupport#UNBOUNDED_CONCURRENCY
	 */
	public static final int UNBOUNDED_CONCURRENCY = ConcurrencyThrottleSupport.UNBOUNDED_CONCURRENCY;

	/**
	 * Switch concurrency 'off': that is, don't allow any concurrent invocations.
	 * @see ConcurrencyThrottleSupport#NO_CONCURRENCY
	 */
	public static final int NO_CONCURRENCY = ConcurrencyThrottleSupport.NO_CONCURRENCY;


	/** Internal concurrency throttle used by this executor. */
	private final ConcurrencyThrottleAdapter concurrencyThrottle = new ConcurrencyThrottleAdapter();

	@Nullable
	private VirtualThreadDelegate virtualThreadDelegate;

	@Nullable
	private ThreadFactory threadFactory;

	@Nullable
	private TaskDecorator taskDecorator;

	private long taskTerminationTimeout;

	@Nullable
	private Set<Thread> activeThreads;

	private volatile boolean active = true;
```

- 3. 그런데, 여기서 @EnableAsync를 사용하는 경우, 조금 다르게 동작한다.
- [Spring](https://docs.spring.io/spring-boot/docs/2.1.13.RELEASE/reference/html/boot-features-task-execution-scheduling.html)

```
다음과 같이 진행
/**
 * {@link EnableAutoConfiguration Auto-configuration} for {@link TaskExecutor}.
 *
 * @author Stephane Nicoll
 * @author Camille Vienot
 * @author Moritz Halbritter
 * @since 2.1.0
 */
@ConditionalOnClass(ThreadPoolTaskExecutor.class)
@AutoConfiguration
@EnableConfigurationProperties(TaskExecutionProperties.class)
@Import({ TaskExecutorConfigurations.ThreadPoolTaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.TaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.SimpleAsyncTaskExecutorBuilderConfiguration.class,
		TaskExecutorConfigurations.TaskExecutorConfiguration.class })
public class TaskExecutionAutoConfiguration {

	/**
	 * Bean name of the application {@link TaskExecutor}.
	 */
	public static final String APPLICATION_TASK_EXECUTOR_BEAN_NAME = "applicationTaskExecutor";

}

		/**
		 * Queue capacity. An unbounded capacity does not increase the pool and therefore
		 * ignores the "max-size" property.
		 */
		private int queueCapacity = Integer.MAX_VALUE;

		/**
		 * Core number of threads.
		 */
		private int coreSize = 8;

		/**
		 * Maximum allowed number of threads. If tasks are filling up the queue, the pool
		 * can expand up to that size to accommodate the load. Ignored if the queue is
		 * unbounded.
		 */
		private int maxSize = Integer.MAX_VALUE;

		/**
		 * Whether core threads are allowed to time out. This enables dynamic growing and
		 * shrinking of the pool.
		 */
		private boolean allowCoreThreadTimeout = true;

		/**
		 * Time limit for which threads may remain idle before being terminated.
		 */
		private Duration keepAlive = Duration.ofSeconds(60);

```


