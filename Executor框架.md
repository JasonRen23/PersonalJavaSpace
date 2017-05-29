Executor框架是指java 5中引入的一系列并发库中与executor相关的一些功能类，其中包括线程池，Executor，Executors，ExecutorService，CompletionService，Future，Callable等。他们的关系为：

并发编程的一种编程方式是把任务拆分为一些列的小任务，即Runnable，然后在提交给一个Executor执行，Executor.execute(Runnalbe) 。Executor在执行时使用内部的线程池完成操作。

一、创建线程池

Executors类，提供了一系列工厂方法用于创先线程池，返回的线程池都实现了ExecutorService接口。

public static ExecutorService newFixedThreadPool(int nThreads)

创建固定数目线程的线程池。

public static ExecutorService newCachedThreadPool()

创建一个可缓存的线程池，调用execute 将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。

public static ExecutorService newSingleThreadExecutor()

创建一个单线程化的Executor。

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)

创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。

```Java
    Executor executor = Executors.newFixedThreadPool(10);  
    Runnable task = new Runnable() {  
        @Override  
        public void run() {  
            System.out.println("task over");  
        }  
    };  
    executor.execute(task);  
      
    executor = Executors.newScheduledThreadPool(10);  
    ScheduledExecutorService scheduler = (ScheduledExecutorService) executor;  
    scheduler.scheduleAtFixedRate(task, 10, 10, TimeUnit.SECONDS);  
```
 二、ExecutorService与生命周期

ExecutorService扩展了Executor并添加了一些生命周期管理的方法。一个Executor的生命周期有三种状态，运行 ，关闭 ，终止 。Executor创建时处于运行状态。当调用ExecutorService.shutdown()后，处于关闭状态，isShutdown()方法返回true。这时，不应该再想Executor中添加任务，所有已添加的任务执行完毕后，Executor处于终止状态，isTerminated()返回true。

如果Executor处于关闭状态，往Executor提交任务会抛出unchecked exception RejectedExecutionException。

```Java
    ExecutorService executorService = (ExecutorService) executor;  
    while (!executorService.isShutdown()) {  
        try {  
            executorService.execute(task);  
        } catch (RejectedExecutionException ignored) {  
              
        }  
    }  
    executorService.shutdown();  
  ```
 三、使用Callable，Future返回结果

Future<V>代表一个异步执行的操作，通过get()方法可以获得操作的结果，如果异步操作还没有完成，则，get()会使当前线程阻塞。FutureTask<V>实现了Future<V>和Runable<V>。Callable代表一个有返回值得操作。


    Callable<Integer> func = new Callable<Integer>(){  
        public Integer call() throws Exception {  
            System.out.println("inside callable");  
            Thread.sleep(1000);  
            return new Integer(8);  
        }         
    };        
    FutureTask<Integer> futureTask  = new FutureTask<Integer>(func);  
    Thread newThread = new Thread(futureTask);  
    newThread.start();  
      
    try {  
        System.out.println("blocking here");  
        Integer result = futureTask.get();  
        System.out.println(result);  
    } catch (InterruptedException ignored) {  
    } catch (ExecutionException ignored) {  
    }  

 ExecutoreService提供了submit()方法，传递一个Callable，或Runnable，返回Future。如果Executor后台线程池还没有完成Callable的计算，这调用返回Future对象的get()方法，会阻塞直到计算完成。

例子：并行计算数组的和。

```Java
    package executorservice;  
      
    import java.util.ArrayList;  
    import java.util.List;  
    import java.util.concurrent.Callable;  
    import java.util.concurrent.ExecutionException;  
    import java.util.concurrent.ExecutorService;  
    import java.util.concurrent.Executors;  
    import java.util.concurrent.Future;  
    import java.util.concurrent.FutureTask;  
      
    public class ConcurrentCalculator {  
      
        private ExecutorService exec;  
        private int cpuCoreNumber;  
        private List<Future<Long>> tasks = new ArrayList<Future<Long>>();  
      
        // 内部类  
        class SumCalculator implements Callable<Long> {  
            private int[] numbers;  
            private int start;  
            private int end;  
      
            public SumCalculator(final int[] numbers, int start, int end) {  
                this.numbers = numbers;  
                this.start = start;  
                this.end = end;  
            }  
      
            public Long call() throws Exception {  
                Long sum = 0l;  
                for (int i = start; i < end; i++) {  
                    sum += numbers[i];  
                }  
                return sum;  
            }  
        }  
      
        public ConcurrentCalculator() {  
            cpuCoreNumber = Runtime.getRuntime().availableProcessors();  
            exec = Executors.newFixedThreadPool(cpuCoreNumber);  
        }  
      
        public Long sum(final int[] numbers) {  
            // 根据CPU核心个数拆分任务，创建FutureTask并提交到Executor  
            for (int i = 0; i < cpuCoreNumber; i++) {  
                int increment = numbers.length / cpuCoreNumber + 1;  
                int start = increment * i;  
                int end = increment * i + increment;  
                if (end > numbers.length)  
                    end = numbers.length;  
                SumCalculator subCalc = new SumCalculator(numbers, start, end);  
                FutureTask<Long> task = new FutureTask<Long>(subCalc);  
                tasks.add(task);  
                if (!exec.isShutdown()) {  
                    exec.submit(task);  
                }  
            }  
            return getResult();  
        }  
      
        /** 
         * 迭代每个只任务，获得部分和，相加返回 
         *  
         * @return 
         */  
        public Long getResult() {  
            Long result = 0l;  
            for (Future<Long> task : tasks) {  
                try {  
                    // 如果计算未完成则阻塞  
                    Long subSum = task.get();  
                    result += subSum;  
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                } catch (ExecutionException e) {  
                    e.printStackTrace();  
                }  
            }  
            return result;  
        }  
      
        public void close() {  
            exec.shutdown();  
        }  
    }  
```
 Main

```Java
    int[] numbers = new int[] { 1, 2, 3, 4, 5, 6, 7, 8, 10, 11 };  
    ConcurrentCalculator calc = new ConcurrentCalculator();  
    Long sum = calc.sum(numbers);  
    System.out.println(sum);  
    calc.close();  
```
 四、CompletionService

在刚在的例子中，getResult()方法的实现过程中，迭代了FutureTask的数组，如果任务还没有完成则当前线程会阻塞，如果我们希望任意字任务完成后就把其结果加到result中，而不用依次等待每个任务完成，可以使CompletionService。生产者submit()执行的任务。使用者take()已完成的任务，并按照完成这些任务的顺序处理它们的结果 。也就是调用CompletionService的take方法是，会返回按完成顺序放回任务的结果，CompletionService内部维护了一个阻塞队列BlockingQueue，如果没有任务完成，take()方法也会阻塞。修改刚才的例子使用CompletionService：

```Java
    public class ConcurrentCalculator2 {  
      
        private ExecutorService exec;  
        private CompletionService<Long> completionService;  
      
      
        private int cpuCoreNumber;  
      
        // 内部类  
        class SumCalculator implements Callable<Long> {  
            ......  
        }  
      
        public ConcurrentCalculator2() {  
            cpuCoreNumber = Runtime.getRuntime().availableProcessors();  
            exec = Executors.newFixedThreadPool(cpuCoreNumber);  
            completionService = new ExecutorCompletionService<Long>(exec);  
      
      
        }  
      
        public Long sum(final int[] numbers) {  
            // 根据CPU核心个数拆分任务，创建FutureTask并提交到Executor  
            for (int i = 0; i < cpuCoreNumber; i++) {  
                int increment = numbers.length / cpuCoreNumber + 1;  
                int start = increment * i;  
                int end = increment * i + increment;  
                if (end > numbers.length)  
                    end = numbers.length;  
                SumCalculator subCalc = new SumCalculator(numbers, start, end);   
                if (!exec.isShutdown()) {  
                    completionService.submit(subCalc);  
      
      
                }  
                  
            }  
            return getResult();  
        }  
      
        /** 
         * 迭代每个只任务，获得部分和，相加返回 
         *  
         * @return 
         */  
        public Long getResult() {  
            Long result = 0l;  
            for (int i = 0; i < cpuCoreNumber; i++) {              
                try {  
                    Long subSum = completionService.take().get();  
                    result += subSum;             
                } catch (InterruptedException e) {  
                    e.printStackTrace();  
                } catch (ExecutionException e) {  
                    e.printStackTrace();  
                }  
            }  
            return result;  
        }  
      
        public void close() {  
            exec.shutdown();  
        }  
    }  
   ```
