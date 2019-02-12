# Java-9 Concurrency Cookbook_-_Second Edition

## Thread Management

> In the computer world, when we talk about concurrency, we refer to a series of independent and unrelated tasks that run simultaneously on a computer.
>
> Concurrent tasks that run inside a process are called **threads**. 



### Creating, running, and setting the characteristics of a thread

创建线程的两种方法

- 继承`java.lang.Thread` 并重写`java.lang.Thread#run`方法

- 实现`java.lang.Runnable`接口,创建`java.lang.Thread`类,传递Runnable对象 

  (this is the preferred approach and it gives you more flexibility)

线程属性(attributes)

- ID 
- Name 
- Priority:保存线程对象的优先级 1->10 1最低优先级;10 最高优先级 (It's only a hint to the underlying operating system and it doesn't guarantee anything, but it's a possibility that you can use if you want)
- Status:保存线程的状态
  - NEW:被创建但是未开始
  - RUNNABLE:在JVM中被执行
  - BLOCKED:等待一个监视器(monitor lock)被释放
  - WAITING:当前线程正在等待其他线程
  - TIMED_WAITING:当前线程以一个指定时间等待其他线程
  - TERMINATED:执行结束

example

```java
public class Calculator implements Runnable {
    @Override
    public void run() {
        long current = 1L;
        long max = 20000L;
        long numPrimes = 0L;
        System.out.printf("Thread '%s': START\n", Thread.currentThread().getName());
        while (current < max) {
            if (isPrime(current)) {
                numPrimes++;
            }
            current++;
        }

        System.out.printf("Thread '%s':END. Number of Primes: %d\n", Thread.currentThread().getName(), numPrimes);
    }
}
```

```java
public class Main01 {

    private static final int size = 10;

    public static void main(String[] args) {
        System.out.printf("Minimum Priority: %s\n", Thread.MIN_PRIORITY);
        System.out.printf("Normal Priority: %s\n", Thread.NORM_PRIORITY);
        System.out.printf("Maximum Priority: %s\n", Thread.MAX_PRIORITY);


        //create 10 Thread Objects
        Thread[] threads = new Thread[size];
        Thread.State[] status = new Thread.State[size];

        for (int i = 0; i < size; i++) {
            threads[i] = new Thread(new Calculator());
            if (i % 2 == 0) {
                //偶数为最大优先级
                threads[i].setPriority(Thread.MAX_PRIORITY);
            } else {
                //偶数为最低优先级
                threads[i].setPriority(Thread.MIN_PRIORITY);
            }
            threads[i].setName("Calculator Thread [" + i + "]");
        }

        //write information in a text file
        try (FileWriter file = new FileWriter("d:\\data\\log.txt");
             PrintWriter printWriter = new PrintWriter(file)) {
            for (int i = 0; i < size; i++) {
                printWriter.println("Main: Status of Thread" + i + " : " + threads[i].getState());
                status[i] = threads[i].getState();
            }

            //run
            for (int i = 0; i < size; i++) {
                threads[i].start();
            }

            //wait for the finalization of the threads
            boolean finish = false;
            while (!finish) {
                for (int i = 0; i < size; i++) {
                    if (threads[i].getState() != status[i]) {
                        writeThreadInfo(printWriter, threads[i], status[i]);
                        status[i] = threads[i].getState();
                    }
                }

                finish = true;
                for (int i = 0; i < 10; i++) {
                    finish = finish && (threads[i].getState() == Thread.State.TERMINATED);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     *  write information about the status of a thread in the file
     * @param printWriter PrintWriter
     * @param thread 线程
     * @param status 线程状态
     */
    private static void writeThreadInfo(PrintWriter printWriter, Thread thread, Thread.State status) {
        printWriter.printf("Main : Id %d - %s \n",thread.getId(),thread.getName());
        printWriter.printf("Main : Priority: %d \n",thread.getPriority());
        printWriter.printf("Main : Old State: %s \n",status);
        printWriter.printf("Main : New State: %s \n",thread.getState());
        printWriter.printf("Main : *************************************\n");
    }

}
```

```
Minimum Priority: 1
Normal Priority: 5
Maximum Priority: 10
Thread 'Calculator Thread [0]': START
Thread 'Calculator Thread [6]': START
Thread 'Calculator Thread [5]': START
Thread 'Calculator Thread [1]': START
Thread 'Calculator Thread [2]': START
Thread 'Calculator Thread [4]': START
Thread 'Calculator Thread [7]': START
Thread 'Calculator Thread [3]': START
Thread 'Calculator Thread [9]': START
Thread 'Calculator Thread [8]': START
Thread 'Calculator Thread [4]':END. Number of Primes: 2263
Thread 'Calculator Thread [1]':END. Number of Primes: 2263
Thread 'Calculator Thread [2]':END. Number of Primes: 2263
Thread 'Calculator Thread [0]':END. Number of Primes: 2263
Thread 'Calculator Thread [3]':END. Number of Primes: 2263
Thread 'Calculator Thread [5]':END. Number of Primes: 2263
Thread 'Calculator Thread [6]':END. Number of Primes: 2263
Thread 'Calculator Thread [9]':END. Number of Primes: 2263
Thread 'Calculator Thread [8]':END. Number of Primes: 2263
Thread 'Calculator Thread [7]':END. Number of Primes: 2263
```



- When you run the program, JVM runs the execution thread that calls the main() method of the program
- When we call the start() method of a Thread object, we are creating another execution thread.
- You can't modify the ID or status of a thread.
- A Java program ends when all its threads finish(when all its non-daemon threads finish)
- If one of the threads uses the System.exit() instruction to end the execution of the program, all the threads will end their respective execution.
- Creating an object of the `java.lang.Thread` class Only when you call the `java.lang.Thread#start` method, a new execution thread is created.



### Interrupting a thread

example

```java
public class PrimeGenerator extends Thread {
    @Override
    public void run() {
        long number = 1L;
        while (true) {
            if (isPrime(number)) {
                System.out.printf("Number %d is Prime\n", number);

            }
            if (isInterrupted()) {
                System.out.print("The Prime Generator has bean Interrupted\n");
                return;
            }
            number++;
        }
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        Thread task = new PrimeGenerator();
        task.start();

        try {
            Thread.sleep(5000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        task.interrupt();


//        try {
//            Thread.sleep(1000L);
//        } catch (InterruptedException e) {
//            e.printStackTrace();
//        }
        System.out.printf("Main: Status of the Thread: %s\n", task.getState());
        System.out.printf("Main: isInterrupted: %s\n", task.isInterrupted());
        System.out.printf("Main: isAlive: %s\n", task.isAlive());

    }
}
```

```
Number 129469 is Prime
Number 129491 is Prime
Main: Status of the Thread: RUNNABLE
Number 129497 is Prime
Main: isInterrupted: true
The Prime Generator has bean Interrupted
Main: isAlive: true
```

- The `java.lang.Thread` class has an attribute that stores a boolean value indicating whether the thread has been interrupted or not.
- There is an important difference between the isInterrupted() and interrupted() methods. The
  first one doesn't change the value of the interrupted attribute, but the second one sets it to false .



### Controlling the interrupting of a thread

use the` java.lang.InterruptedException` exception to control the interruption of a thread.

example

```java
public class FileSearch implements Runnable {

    private String initPath;
    private String fileName;

    public FileSearch(String initPath, String fileName) {
        this.initPath = initPath;
        this.fileName = fileName;
    }

    @Override
    public void run() {
        File file = new File(initPath);
        if (file.isDirectory()) {
            try {
                directoryProcess(file);
            } catch (InterruptedException e) {
                System.out.printf("%s : The search has been interrupted", Thread.currentThread().getName());
            }
        }
    }

    private void directoryProcess(File file) throws InterruptedException {
        File[] files = file.listFiles();
        if (null != files) {
            for (int i = 0; i < files.length; i++) {
                if(files[i].isDirectory()){
                    directoryProcess(files[i]);
                }else {
                    fileProcess(files[i]);
                }
            }
        }

        if(Thread.interrupted()){
            throw new InterruptedException();
        }
    }

    private void fileProcess(File file) throws InterruptedException {
        if(file.getName().equals(fileName)){
            System.out.printf("%s : %s\n",Thread.currentThread().getName(),file.getAbsolutePath());
        }

        if(Thread.interrupted()){
            throw new InterruptedException();
        }
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        FileSearch fileSearch = new FileSearch("c:/Windows","explorer.exe");
        Thread thread = new Thread(fileSearch);
        thread.start();

        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        thread.interrupt();
    }
}
```

```
Thread-0 : c:\Windows\explorer.exe
Thread-0 : c:\Windows\SysWOW64\explorer.exe
Thread-0 : c:\Windows\WinSxS\amd64_microsoft-windows-explorer_31bf3856ad364e35_10.0.17134.165_none_3349560dcfdf676c\explorer.exe
Thread-0 : c:\Windows\WinSxS\amd64_microsoft-windows-explorer_31bf3856ad364e35_10.0.17134.1_none_37353369e2e6d4a9\explorer.exe
Thread-0 : The search has been interrupted
```



### Sleeping and resuming a thread

use the `java.lang.Thread#sleep(long)` method of the Thread class for pausing the execution

makes a thread object leave the CPU

example

```java
public class ConsoleClock implements Runnable {

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.printf("%s \n",new Date());
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                System.out.print("TheFileClock has been interrupt");
            }
        }
    }
}
```

```java
public class Main04 {

    public static void main(String[] args) {
        ConsoleClock clock = new ConsoleClock();
        Thread thread = new Thread(clock);
        thread.start();

        try{
            TimeUnit.SECONDS.sleep(5);
        }catch (InterruptedException e){
            e.printStackTrace();
        }

        thread.interrupt();
    }
}
```

```
Thu Dec 27 08:47:41 CST 2018 
Thu Dec 27 08:47:42 CST 2018 
Thu Dec 27 08:47:43 CST 2018 
Thu Dec 27 08:47:44 CST 2018 
Thu Dec 27 08:47:45 CST 2018 
TheFileClock has been interruptThu Dec 27 08:47:46 CST 2018 
Thu Dec 27 08:47:47 CST 2018 
Thu Dec 27 08:47:48 CST 2018 
Thu Dec 27 08:47:49 CST 2018 
Thu Dec 27 08:47:50 CST 2018 
```



### Waiting for the finalization of a thread

example

```java
public class DataSourcesLoader implements Runnable{
    @Override
    public void run() {
        System.out.printf("Beginning data sources loading: %s\n", LocalDateTime.now());

        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Data sources loading has finished: %s\n",LocalDateTime.now());

    }
}
```

```java
public class NetworkConnectionsLoader implements Runnable{
    @Override
    public void run() {
        System.out.printf("Beginning Network connection loading: %s\n", LocalDateTime.now());

        try {
            TimeUnit.SECONDS.sleep(6);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Network connection loading has finished: %s\n",LocalDateTime.now());
        
    }
}
```

```java
public class Main05 {

    public static void main(String[] args) {
        DataSourcesLoader dataSourcesLoader = new DataSourcesLoader();
        Thread thread1 = new Thread(dataSourcesLoader);

        NetworkConnectionsLoader networkConnectionsLoader = new NetworkConnectionsLoader();
        Thread thread2 = new Thread(networkConnectionsLoader);

        thread1.start();
        thread2.start();

        try {
            thread1.join();
            thread2.join();
        }catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.printf("Main: Configuration has been loaded : %s\n", LocalDateTime.now());
    }
}
```

```
Beginning data sources loading: 2018-12-27T09:05:59.226423100
Beginning Network connection loading: 2018-12-27T09:05:59.226423100
Data sources loading has finished: 2018-12-27T09:06:03.243795
Network connection loading has finished: 2018-12-27T09:06:05.243382600
Main: Configuration has been loaded : 2018-12-27T09:06:05.243382600
```



### Creating and running a daemon thread

Java has a special kind of thread called daemon thread. When daemon threads  are the only threads running in a program, the JVM ends the program after finishing these threads.

example

```java
public class WriterTask implements Runnable {

    private Deque<Event> deque;

    public WriterTask(Deque<Event> deque) {
        this.deque = deque;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            Event event = new Event();
            event.setDate(LocalDateTime.now());
            event.setEvent(String.format("The thread %s has generated as event",Thread.currentThread().getId()));

            deque.addFirst(event);

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class CleanerTask extends Thread {
    private Deque<Event> deque;

    public CleanerTask(Deque<Event> deque) {
        this.deque = deque;
        setDaemon(true);
    }

    @Override
    public void run() {

        while (true) {
            LocalDateTime date = LocalDateTime.now();
            clean(date);
        }
    }

    private void clean(LocalDateTime date) {
        long difference;
        boolean delete;
        if (deque.isEmpty()) {
            return;
        }
        delete = false;
        do {
            Event event = deque.getLast();
            difference = date.toInstant(ZoneOffset.of("+08:00")).toEpochMilli() - event.getDate().toInstant(ZoneOffset.of("+08:00")).toEpochMilli();
            if (difference > 10000) {
                System.out.printf("Cleaner: %s\n", event.getEvent());
                deque.removeLast();
                delete = true;
            }
        } while (difference > 10000);

        if (delete) {
            System.out.printf("Cleaner: Size of the queue : %d\n", deque.size());
        }
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        Deque<Event> deque = new ConcurrentLinkedDeque<>();

        WriterTask writer = new WriterTask(deque);
        for(int i=0; i< 10; i++){
            Thread thread = new Thread(writer);
            thread.start();
        }

        CleanerTask cleaner = new CleanerTask(deque);

        cleaner.start();
    }
}
```

```
Cleaner: Size of the queue : 90
Cleaner: The thread 22 has generated as event
Cleaner: The thread 14 has generated as event
Cleaner: The thread 17 has generated as event
Cleaner: The thread 20 has generated as event
Cleaner: The thread 16 has generated as event
Cleaner: The thread 19 has generated as event
Cleaner: The thread 23 has generated as event
Cleaner: The thread 21 has generated as event
Cleaner: The thread 15 has generated as event
Cleaner: The thread 18 has generated as event
Cleaner: Size of the queue : 90
```

- `java.lang.Thread#setDaemon` call the `setDaemon()` method before you call the `start()` method
- `java.lang.Thread#isDaemon` whether a thread is a daemon thread



### Processing uncontrolled exceptions in a thread

- **Checked Exception**: These must be specified in the throws clause of a method or caught inside them
- **Unchecked Exception**: These don't need to be specified or caught

> When an unchecked exception is thrown inside the `run()` method of a thread object, the default behavior is to write the stack trace in the console and exit the program.



example

```java
public class ExceptionHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.print("An Exception has been captured\n");
        System.out.printf("Thread: %s \n", t.getId());
        System.out.printf("Exception: %s : %s\n", e.getClass(), e.getMessage());
        System.out.print("Stack Trace:\n");
        e.printStackTrace();
        System.out.printf("Thread Status: %s\n",t.getState());
    }
}
```

```java
public class Task implements Runnable{
    @Override
    public void run() {
        int number = Integer.parseInt("TTT");
    }
}
```

```java
public class Main07 {
    public static void main(String[] args) {
        Task task = new Task();
        Thread thread = new Thread(task);
        thread.setUncaughtExceptionHandler(new ExceptionHandler());
        thread.start();
    }
}
```

```
An Exception has been captured
Thread: 14 
Exception: class java.lang.NumberFormatException : For input string: "TTT"
Stack Trace:
java.lang.NumberFormatException: For input string: "TTT"
	at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.base/java.lang.Integer.parseInt(Integer.java:652)
	at java.base/java.lang.Integer.parseInt(Integer.java:770)
	at org.zucc.concurrency.chapter01.task.Task.run(Task.java:12)
	at java.base/java.lang.Thread.run(Thread.java:834)
Thread Status: RUNNABLE

```

- If the thread doesn't have an uncaught exception handler, the JVM prints the stack trace in the console and ends the execution of the thread that had thrown the exception.
- `java.lang.Thread#setDefaultUncaughtExceptionHandler`
- Three possible handlers:
  1. looks for the uncaught exception handler of the thread objects
  2. the JVM looks for the uncaught exception handler of `java.lang.ThreadGroup` as explained in the Grouping threads and processing uncontrolled exceptions in a group of threads recipe
  3. the JVM looks for the default uncaught exception handler



### Using thread local variables

 **thread-local variables** have some disadvantages as well. They retain their value while the thread is alive. This can be problematic in situations where threads are reused.

example

```java
public class UnsafeTask implements Runnable {
    private Date startDate;

    @Override
    public void run() {
        startDate = new Date();
        System.out.printf("Starting Thread: %s :%s\n", Thread.currentThread().getId(), startDate);

        try {
            TimeUnit.SECONDS.sleep((int) Math.rint(Math.random() * 10));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Thread Finished: %s : %s\n", Thread.currentThread().getId(), startDate);
    }
}
```

```java
public class Main08_1 {
    public static void main(String[] args) {
        UnsafeTask task = new UnsafeTask();
        for(int i = 0; i < 10; i++){
            Thread thread = new Thread(task);
            thread.start();

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java

public class SafeTask implements Runnable{

    private static ThreadLocal<Date> startDate = new ThreadLocal<Date>(){

        @Override
        protected Date initialValue(){
            return new Date();
        }
    };

    @Override
    public void run() {
        System.out.printf("Starting Thread: %s :%s\n", Thread.currentThread().getId(), startDate.get());

        try {
            TimeUnit.SECONDS.sleep((int) Math.rint(Math.random() * 10));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Thread Finished: %s : %s\n", Thread.currentThread().getId(), startDate.get());

    }
}
```

```java
public class Main08_2 {
    public static void main(String[] args) {
        SafeTask task = new SafeTask();
        for(int i = 0; i < 10; i++){
            Thread thread = new Thread(task);
            thread.start();

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```
Starting Thread: 14 :Thu Dec 27 10:55:27 CST 2018
Thread Finished: 14 : Thu Dec 27 10:55:27 CST 2018
Starting Thread: 15 :Thu Dec 27 10:55:29 CST 2018
Starting Thread: 16 :Thu Dec 27 10:55:31 CST 2018
Thread Finished: 15 : Thu Dec 27 10:55:29 CST 2018
Starting Thread: 17 :Thu Dec 27 10:55:33 CST 2018
Thread Finished: 16 : Thu Dec 27 10:55:31 CST 2018
Starting Thread: 18 :Thu Dec 27 10:55:35 CST 2018
Thread Finished: 17 : Thu Dec 27 10:55:33 CST 2018
Starting Thread: 19 :Thu Dec 27 10:55:37 CST 2018
Thread Finished: 18 : Thu Dec 27 10:55:35 CST 2018
Starting Thread: 20 :Thu Dec 27 10:55:39 CST 2018
```

- The thread-local variables mechanism stores a value of an attribute for each thread that uses one of these variables. 
- `java.lang.ThreadLocal`
  - `java.lang.ThreadLocal#get`
  - `java.lang.ThreadLocal#set`
  - `java.lang.ThreadLocal#remove`

- `java.lang.InheritableThreadLocal`
  - `java.lang.InheritableThreadLocal#childValue`



### Grouping threads and processing uncontrolled exceptions in a group of threads

the `java.lang.ThreadGroup` class to work with a groups of threads

example

```java
public class MyThreadGroup extends ThreadGroup {
    public MyThreadGroup(String name) {
        super(name);
    }

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.printf("The thread %s has thrown an Exception\n",t.getId());
        e.printStackTrace();
        System.out.print("Terminating the rest of the Threads\n");
        interrupt();
    }
}
```

```java
public class Task implements Runnable {
    @Override
    public void run() {
        int result;
        Random random = new Random(Thread.currentThread().getId());
        while(true){
            result = 1000/((int)(random.nextDouble()*1000000000));
            if(Thread.currentThread().isInterrupted()){
                System.out.printf("%d : Interrupted\n",Thread.currentThread().getId());
                return;
            }
        }
    }
}
```

```java
public class Main09 {
    public static void main(String[] args) {
        int numberOfThreads = 2 * Runtime.getRuntime().availableProcessors();
        MyThreadGroup threadGroup = new MyThreadGroup("myThreadGroup");
        Task task = new Task();

        for (int i = 0; i < numberOfThreads; i++) {
            Thread t = new Thread(threadGroup, task);
            t.start();
        }

        System.out.printf("Number of Threads: %d\n", threadGroup.activeCount());
        System.out.print("Information about the Thread Group\n");
        threadGroup.list();

        Thread[] threads = new Thread[threadGroup.activeCount()];
        threadGroup.enumerate(threads);

        for (int i = 0; i < threadGroup.activeCount(); i++) {
            System.out.printf("Thread %s: %s\n",threads[i].getName(),threads[i].getState());
        }
    }
}
```

```
Number of Threads: 16
Information about the Thread Group
org.zucc.concurrency.chapter01.group.MyThreadGroup[name=myThreadGroup,maxpri=10]
    Thread[Thread-0,5,myThreadGroup]
    Thread[Thread-1,5,myThreadGroup]
    Thread[Thread-2,5,myThreadGroup]
    Thread[Thread-3,5,myThreadGroup]
    Thread[Thread-4,5,myThreadGroup]
    
    
The thread 25 has thrown an Exception
java.lang.ArithmeticException: / by zero
	at org.zucc.concurrency.chapter01.task.Task.run(Task.java:17)
	at java.base/java.lang.Thread.run(Thread.java:834)
Terminating the rest of the Threads
27 : Interrupted
21 : Interrupted
17 : Interrupted
```

- `java.lang.ThreadGroup#list()`
- possible handlers for exception 

### Creating threads through a factory

factory pattern

- It's easy to change the class of the objects created or the way you'd create them.
- It's easy to limit the creation of objects for limited resources.
- It's easy to generate statistical data about the creation of objects.

`java.util.concurrent.ThreadFactory` interface

example

```java
public class MyThreadFactory implements ThreadFactory {

    private int counter;
    private String name;
    private List<String> stats;

    public MyThreadFactory(String name) {
        this.counter = 0;
        this.name = name;
        this.stats = new ArrayList<>();
    }

    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r, name + "Thread_" + counter);
        counter++;
        stats.add(String.format("Created thread %d with name %s on %s\n",
                thread.getId(), thread.getName(), new Date()));

        return thread;
    }

    public String getStats(){
        StringBuffer buffer = new StringBuffer();
        stats.forEach(stat->{
            buffer.append(stat);
            buffer.append("\n");
        });
        return buffer.toString();
    }
}
```

```java
public class DoNothingTask implements Runnable {
    @Override
    public void run() {
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Main10 {
    public static void main(String[] args) {
        MyThreadFactory factory = new MyThreadFactory("myThreadFactory");
        DoNothingTask task = new DoNothingTask();

        Thread thread;
        for (int i = 0; i < 10; i++) {
            thread = factory.newThread(task);
            thread.start();
        }

        System.out.print("Factory stats:\n");
        System.out.printf("%s\n", factory.getStats());
    }
}
```

```
Factory stats:
Created thread 14 with name myThreadFactoryThread_0 on Thu Dec 27 13:52:29 CST 2018

Created thread 15 with name myThreadFactoryThread_1 on Thu Dec 27 13:52:29 CST 2018

Created thread 16 with name myThreadFactoryThread_2 on Thu Dec 27 13:52:29 CST 2018

Created thread 17 with name myThreadFactoryThread_3 on Thu Dec 27 13:52:29 CST 2018

Created thread 18 with name myThreadFactoryThread_4 on Thu Dec 27 13:52:29 CST 2018

Created thread 19 with name myThreadFactoryThread_5 on Thu Dec 27 13:52:29 CST 2018

Created thread 20 with name myThreadFactoryThread_6 on Thu Dec 27 13:52:29 CST 2018

Created thread 21 with name myThreadFactoryThread_7 on Thu Dec 27 13:52:29 CST 2018

Created thread 22 with name myThreadFactoryThread_8 on Thu Dec 27 13:52:29 CST 2018

Created thread 23 with name myThreadFactoryThread_9 on Thu Dec 27 13:52:29 CST 2018
```



- `java.util.concurrent.ThreadFactory#newThread`   `java.lang.Runnable`->`java.lang.Thread`


## Basic Thread Synchronization

> One of the most common situations in concurrent programming occurs when more than one execution thread shares a resource.
>
> **race conditions** they occur when different threads have access to the same shared resource at the same time.
>
> also have problems with change visibility. So if a thread changes the value of a shared variable, the changes would only be written in the local cache of that thread; other threads will not have access to the change (they will only be able to see the old value).
>
> the concept of **critical section**.A critical section is a block of code that accesses a shared resource and can't be executed by more than one thread at the same time.
>
> Java offers synchronization mechanisms.
>
> two basic synchronization mechanisms 
>
> - The `synchronized` keyword
> - The `java.util.concurrent.locks.Lock` interface and its implementations



### Synchronizing a method

the `synchronized` keyword to control concurrent access to a method or a block of code.

When you use the `synchronized` keyword with a method, the object reference is implicit.

every method declared with the synchronized keyword is a critical section.

When you use the `synchronized` keyword to protect a block of code, you must pass an object reference as a parameter.

You should keep the objects used for synchronization private.

example

```java
public class ParkingCash {
    private static final int cost = 2;
    private long cash;

    public ParkingCash() {
        this.cash = 0;
    }

    public void vehiclePay() {
        cash += cost;
    }
    public void close(){
        System.out.print("Closing accounting");
        long totalAmount;
        totalAmount = cash;
        cash = 0 ;
        System.out.printf("The total amount is : %d",totalAmount);
    }
}
```

```java
public class ParkingStats {
    private long numberCars;
    private long numberMotorcycles;
    private ParkingCash cash;

    public ParkingStats(ParkingCash cash){
        this.numberCars = 0;
        this.numberMotorcycles = 0;
        this.cash = cash;
    }

    public void carComeIn(){
        numberCars++;
    }

    public void carGoOut(){
        numberCars--;

        cash.vehiclePay();
    }

    public void motoComeIn(){
        numberMotorcycles++;
    }

    public void motoGoOut(){
        numberMotorcycles--;

        cash.vehiclePay();
    }

    public long getNumberCars() {
        return numberCars;
    }

    public long getNumberMotorcycles() {
        return numberMotorcycles;
    }
}
```

```java
public class Sensor implements Runnable {

    private ParkingStats parkingStats;

    public Sensor(ParkingStats stats) {
        this.parkingStats = stats;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            parkingStats.carComeIn();
            parkingStats.carComeIn();

            try {
                TimeUnit.MILLISECONDS.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            parkingStats.motoComeIn();

            try {
                TimeUnit.MILLISECONDS.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            parkingStats.motoGoOut();

            parkingStats.carGoOut();
            parkingStats.carGoOut();
        }
    }
}
```

```java
public class Main_Problem {
    public static void main(String[] args) {
        ParkingCash cash = new ParkingCash();
        ParkingStats stats = new ParkingStats(cash);
        System.out.print("Parking Simulator\n");

        int numberSensors = 2 * Runtime.getRuntime().availableProcessors();

        Thread[] threads = new Thread[numberSensors];

        for (int i = 0; i < numberSensors; i++) {
            Sensor sensor = new Sensor(stats);
            Thread thread = new Thread(sensor);

            thread.start();
            threads[i] = thread;
        }

        for (int i = 0; i < numberSensors; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Number of cars: %d\n", stats.getNumberCars());
        System.out.printf("Number of motorcycles: %d\n", stats.getNumberMotorcycles());
        cash.close();
    }
}
```

```
Parking Simulator
Number of cars: 4
Number of motorcycles: 8
Closing accountingThe total amount is : 768

```

solution

```java
public class ParkingCash {
    private static final int cost = 2;
    private long cash;

    public ParkingCash() {
        this.cash = 0;
    }

    public synchronized void vehiclePay() {
        cash += cost;
    }

    public void close(){
        System.out.print("Closing accounting");
        long totalAmount;
        synchronized (this){
            totalAmount = cash;
            cash = 0 ;
        }
        System.out.printf("The total amount is : %d",totalAmount);
    }
}
```

```java
public class ParkingStats {
    private long numberCars;
    private long numberMotorcycles;
    private ParkingCash cash;

    private final Object controlCars;
    private final Object controlMotorcycles;

    public ParkingStats(ParkingCash cash){
        this.numberCars = 0;
        this.numberMotorcycles = 0;
        this.cash = cash;
        controlCars = new Object();
        controlMotorcycles = new Object();
    }

    public void carComeIn(){
        synchronized (controlCars){
            numberCars++;
        }
    }

    public void carGoOut(){
        synchronized (controlCars) {
            numberCars--;
        }

        cash.vehiclePay();
    }

    public void motoComeIn(){
        synchronized(controlMotorcycles){
            numberMotorcycles++;
        }
    }

    public void motoGoOut(){

        synchronized(controlMotorcycles){
            numberMotorcycles--;
        }
        cash.vehiclePay();
    }

    public synchronized long getNumberCars() {
        return numberCars;
    }

    public synchronized long getNumberMotorcycles() {
        return numberMotorcycles;
    }
}
```

```java
public class Sensor implements Runnable {

    private ParkingStats parkingStats;

    public Sensor(ParkingStats stats) {
        this.parkingStats = stats;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            parkingStats.carComeIn();
            parkingStats.carComeIn();

            try {
                TimeUnit.MILLISECONDS.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            parkingStats.motoComeIn();

            try {
                TimeUnit.MILLISECONDS.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            parkingStats.motoGoOut();

            parkingStats.carGoOut();
            parkingStats.carGoOut();
        }
    }
}
```

```java
public class Main_Solution {
    public static void main(String[] args) {
        ParkingCash cash = new ParkingCash();
        ParkingStats stats = new ParkingStats(cash);
        System.out.print("Parking Simulator\n");

        int numberSensors = 2 * Runtime.getRuntime().availableProcessors();

        Thread[] threads = new Thread[numberSensors];

        for (int i = 0; i < numberSensors; i++) {
            Sensor sensor = new Sensor(stats);
            Thread thread = new Thread(sensor);

            thread.start();
            threads[i] = thread;
        }

        for (int i = 0; i < numberSensors; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Number of cars: %d\n", stats.getNumberCars());
        System.out.printf("Number of motorcycles: %d\n", stats.getNumberMotorcycles());
        cash.close();
    }
}
```

```
Parking Simulator
Number of cars: 0
Number of motorcycles: 0
Closing accountingThe total amount is : 960
```

- If thread (A) is executing a synchronized method and thread (B) wants to execute another synchronized method of the same object, it will be blocked until thread (A) is finished. But if thread (B) has access to different objects of the same class, none of them will be blocked.
-  When you use the synchronized keyword to protect a block of code, you use an object as a parameter. JVM guarantees that only one thread can have access to all the blocks of code protected with this object.
- You can use recursive calls with synchronized methods. As the thread has access to the synchronized methods of an object, you can call other synchronized methods of that object, including the method that is being executed. It won't have to get access to the synchronized methods again.
-  avoid calling blocking operations inside a critical section



### Using conditions in synchronized code

A classic problem in concurrent programming is the producer-consumer problem.

- `java.lang.Object#wait()`
- `java.lang.Object#notify`
- `java.lang.Object#notifyAll`

example

```java
public class EventStorage {
    private int maxSize;
    private Queue<Date> storage;

    public EventStorage() {
        this.maxSize = 10;
        this.storage = new LinkedList<>();
    }

    public synchronized void set() {
        while (storage.size() == maxSize) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        storage.offer(new Date());
        System.out.printf("Set: %d\n", storage.size());
        notify();
    }

    public synchronized void get() {
        while (storage.size() == 0) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        String element = storage.poll().toString();
        System.out.printf("Get: %d : %s\n", storage.size(), element);
        notify();
    }
}
```

```java
public class Consumer implements Runnable {
    private EventStorage storage;

    public Consumer(EventStorage storage) {
        this.storage = storage;
    }

    @Override
    public void run() {
        for(int i = 0; i < 100; i++){
            storage.get();
        }
    }
}
```

```java
public class Producer implements Runnable {
    private EventStorage storage;

    public Producer(EventStorage storage) {
        this.storage = storage;
    }

    @Override
    public void run() {
        for(int i=0;i<100;i++){
            storage.set();
        }
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        EventStorage storage = new EventStorage();

        Producer producer = new Producer(storage);
        Consumer consumer = new Consumer(storage);
        Thread t1 = new Thread(producer);
        Thread t2 = new Thread(consumer);

        t1.start();
        t2.start();
    }
}
```



### Synchronizing a block of code with a lock

`java.util.concurrent.Lock`

 advantages:

- It allows you to structure synchronized blocks in a more flexible way.
- The Lock interface provides additional functionalities over the synchronized keyword.
  - `java.util.concurrent.locks.Lock#tryLock()`
- The ReadWriteLock interface allows a separation of read and write operations with multiple readers and only one modifier
- `java.util.concurrent.locks.ReadWriteLock`
- The `java.util.concurrent.Lock` interface offers better performance than the `synchronized` keyword

`java.util.concurrent.locks.ReentrantLock`

**non-fair mode**

In this mode,if some threads are waiting for a lock and the lock has to select one of these threads to get access to the critical section, it **randomly** selects anyone of them. 

**fair mode**

In this mode, ifsome threads are waiting for a lock and the lock has to select one to get access to a critical section, it selects the thread that has been **waiting for the longest period of time**. 

As the `java.util.concurrent.locks.Lock#tryLock()` method doesn't put the thread to sleep if the Lock interface is used, the fair attribute doesn't affect its functionality

examle

```java
public class PrintQueue {

    private Lock queueLock;

    public PrintQueue(boolean fairMode) {
        queueLock = new ReentrantLock(fairMode);
    }

    public void printJob(Object document) {
        doSomething();

        doSomething();
    }

    private void doSomething() {
        queueLock.lock();

        try {
            long duration = (long) (Math.random() * 10000);
            System.out.println(Thread.currentThread().getName() +
                    "PrintQueue: Printing a Job during" +
                    duration / 1000 + "seconds");
            Thread.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            queueLock.unlock();
        }
    }
}
```

```java
public class Job implements Runnable {
    private PrintQueue printQueue;

    public Job(PrintQueue printQueue){
        this.printQueue = printQueue;
    }
    @Override
    public void run() {
        System.out.printf("%s:Going to print a document\n",Thread.currentThread().getName());
        printQueue.printJob(new Object());
        System.out.printf("%s:The document has been printed\n",Thread.currentThread().getName());
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        System.out.print("Running example with fair-mode = false\n");
        testPrintQueue(false);
        System.out.print("Running example with fair-mode = true\n");
        testPrintQueue(true);
    }

    private static void testPrintQueue(boolean failMode) {
        PrintQueue printQueue = new PrintQueue(failMode);
        Thread[] threads = new Thread[10];

        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(new Job(printQueue), "Thread_" + i);
        }

        for (int i = 0; i < 10; i++) {
            threads[i].start();
        }

        for (int i = 0; i < 10; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

- At the beginning of the critical section, we have to get control of the lock using the `java.util.concurrent.locks.Lock#lock` method. When thread (A) calls this method, if no other thread has control of the lock, it gives thread (A) control of the lock and returns immediately to allow the thread to execute the critical section. Otherwise, if there is another, say thread (B), executing the critical section controlled by this lock, the `java.util.concurrent.locks.Lock#lock` method puts thread (A) to sleep until thread (B) finishes the execution of the critical section.

- `java.util.concurrent.locks.Lock#tryLock()`

  if the thread that uses it can't get control of the Lock interface, returns immediately and **doesn't put the thread to sleep**.

- You should call the unlock() method as many times as you called the lock() method in your code.

- **Avoiding deadlocks**



### Synchronizing data access with read/write locks

`java.util.concurrent.locks.ReadWriteLock`

-> `java.util.concurrent.locks.ReentrantReadWriteLock`

This class has two locks: one for read operations and one for write operations. 

There can be more than one thread using read operations simultaneously, but only one thread can use write operations.

example

```java
public class PricesInfo {
    private double price1;
    private double price2;

    private ReadWriteLock lock;

    public PricesInfo(){
        this.price1 = 1.0;
        this.price2 = 2.0;
        this.lock = new ReentrantReadWriteLock();
    }

    public double getPrice1(){
        lock.readLock().lock();
        double value = price1;
        lock.readLock().unlock();
        return value;
    }

    public double getPrice2(){
        lock.readLock().lock();
        double value = price2;
        lock.readLock().unlock();
        return value;
    }

    public void setPrices(double price1, double price2){
        lock.writeLock().lock();
        System.out.printf("%s: PricesInfo: Write Lock Acquired.\n",new Date());

        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        this.price1 = price1;
        this.price2 = price2;

        System.out.printf("%s: PricesInfo: Write Lock Released.\n",new Date());
        lock.writeLock().unlock();
    }
}
```

```java
public class Reader implements Runnable {
    private PricesInfo pricesInfo;

    public Reader(PricesInfo pricesInfo) {
        this.pricesInfo = pricesInfo;
    }

    @Override
    public void run() {
        for (int i = 0; i < 20; i++) {
            System.out.printf("%s: %s Price 1: %f\n",
                    new Date(), Thread.currentThread().getName(), pricesInfo.getPrice1());

            System.out.printf("%s: %s Price 2: %f\n",
                    new Date(), Thread.currentThread().getName(), pricesInfo.getPrice2());
        }
    }
}
```

```java
public class Writer implements Runnable {

    private PricesInfo pricesInfo;

    public Writer(PricesInfo pricesInfo) {
        this.pricesInfo = pricesInfo;
    }

    @Override
    public void run() {
        for (int i = 0; i < 3; i++) {
            System.out.printf("%s: Writer Attempt to modify the prices.\n", new Date());

            pricesInfo.setPrices(Math.random() * 10, Math.random() * 8);

            System.out.printf("%s: Writer: Prices have bean modified.\n", new Date());

            try {
                Thread.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        PricesInfo pricesInfo = new PricesInfo();

        Reader[] readers = new Reader[5];
        Thread[] readerThreads = new Thread[5];

        for (int i = 0; i < 5; i++) {
            readers[i] = new Reader(pricesInfo);
            readerThreads[i] = new Thread(readers[i]);
        }

        Writer writer = new Writer(pricesInfo);
        Thread writerThead = new Thread(writer);

        for (int i = 0; i < 5; i++) {
            readerThreads[i].start();
        }
        writerThead.start();

    }
}
```



### Using multiple conditions in a lock

A classic problem in concurrent programming is the **producer-consumer problem**.

`java.util.concurrent.locks.Condition`

The `java.util.concurrent.locks.Condition` interface provides the mechanisms to suspend a thread and wake up a suspended thread.

example

```java
public class FileMock {

    private String[] content;
    private int index;

    public FileMock(int size, int length) {
        content = new String[size];
        for (int i = 0; i < size; i++) {
            StringBuilder buffer = new StringBuilder(length);
            for (int j = 0; j < length; j++) {
                int randomCharacter = (int) (Math.random() * 255);
                buffer.append((char) randomCharacter);
            }
            content[i] = buffer.toString();
        }
        index = 0;
    }

    public boolean hasMoreLines() {
        return index < content.length;
    }

    public String getLine() {
        if (hasMoreLines()) {
            System.out.println("Mock:" + (content.length - index));
            return content[index++];
        }
        return null;
    }
}
```

```java
public class Buffer {
    private final LinkedList<String> buffer;
    private final int maxSize;
    private final ReentrantLock lock;
    private final Condition lines;
    private final Condition space;
    private boolean pendingLines;

    public Buffer(int maxSize) {
        this.maxSize = maxSize;
        this.buffer = new LinkedList<>();
        this.lock = new ReentrantLock();
        this.lines = lock.newCondition();
        this.space = lock.newCondition();
        this.pendingLines = true;
    }

    public void insert(String line) {
        lock.lock();
        try {
            while (buffer.size() == maxSize) {
                space.await();
            }
            buffer.offer(line);
            System.out.printf("%s: Inserted Line: %d\n",
                    Thread.currentThread().getName(), buffer.size());

            lines.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public String get() {
        String line = null;
        lock.lock();
        try {
            while (buffer.size() == 0 && hasPendingLines()) {
                lines.await();
            }

            if (hasPendingLines()) {
                line = buffer.poll();
                System.out.printf("%s: Line Readed: %d\n",
                        Thread.currentThread().getName(), buffer.size());

                space.signalAll();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return line;
    }

    public synchronized void setPendingLines(boolean pendingLines) {
        this.pendingLines = pendingLines;
    }

    public synchronized boolean hasPendingLines() {
        return pendingLines || buffer.size() > 0;
    }
}
```

```java
public class BufferConsumer implements Runnable{
    private Buffer buffer;

    public BufferConsumer(Buffer buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        while(buffer.hasPendingLines()){
            String line = buffer.get();
            processLine(line);
        }
    }

    private void processLine(String line) {
        try {
            Random random = new Random();
            Thread.sleep(random.nextInt(100));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class BufferProducer implements Runnable {
    private FileMock mock;
    private Buffer buffer;

    public BufferProducer(FileMock mock, Buffer buffer) {
        this.mock = mock;
        this.buffer = buffer;
    }

    @Override
    public void run() {
        buffer.setPendingLines(true);
        while (mock.hasMoreLines()) {
            String line = mock.getLine();
            buffer.insert(line);
        }
        buffer.setPendingLines(false);
    }
}
```

```java
public class Main05 {

    public static void main(String[] args) {
        FileMock mock = new FileMock(100, 10);
        Buffer buffer = new Buffer(20);

        BufferProducer producer = new BufferProducer(mock, buffer);
        Thread producerThread = new Thread(producer);

        BufferConsumer[] consumers = new BufferConsumer[3];
        Thread[] consumerThreads = new Thread[3];
        for (int i = 0; i < 3; i++) {
            consumers[i] = new BufferConsumer(buffer);
            consumerThreads[i] = new Thread(consumers[i]);
        }

        producerThread.start();
        for (int i = 0; i < 3; i++) {
            consumerThreads[i].start();
        }
    }
}
```

- `java.util.concurrent.locks.Lock#newCondition`

  return `java.util.concurrent.locks.Condition` instance that is bound to this `java.util.concurrent.locks.Lock` instance

- Before we can do any operation with a condition, you have to have control of the lock associated with the condition. 

- You must be careful with the use of await() and signal() . If you call the await() method in a condition and never call the signal() method in the same condition, the **thread will sleep forever**.

- `java.util.concurrent.locks.Condition#await(long, java.util.concurrent.TimeUnit)`

  the thread will sleep until:

  - It's interrupted
  - Another thread calls the signal() or signalAll() methods in the condition
  - The specified time passes
  - The TimeUnit class is an enumeration with the following constants: DAYS , HOURS , MICROSECONDS ,MILLISECONDS , MINUTES , NANOSECONDS , and SECONDS

- `java.util.concurrent.locks.Condition#awaitUninterruptibly`

  The thread will sleep until another thread calls the signal() or signalAll() methods, which can't be interrupted

- `java.util.concurrent.locks.Condition#awaitUntil`

  The thread will sleep until:

  - It's interrupted
  - Another thread calls the signal() or signalAll() methods in the condition
  - The specified date arrives

### Advanced locking with the StampedLock class

`java.util.concurrent.locks.StampedLock`

its main purpose is to be a helper class to implement thread-safe components

most important features of `StampedLock`

- You can obtain control of the lock in three different modes
  - **Write**

  - **Read**

  - **Optimistic Read**

    the thread doesn't have control over the block.Other threads can get control of the lock in write mode. When you get a lock in the Optimistic Read mode and you want to access the shared data protected by it, you will have to check whether you can access them or not using the `java.util.concurrent.locks.StampedLock#validate` method

-  provides methods

  - `java.util.concurrent.locks.StampedLock#readLock`
  - `java.util.concurrent.locks.StampedLock#writeLock`
  - `java.util.concurrent.locks.StampedLock#readLockInterruptibly`
  - `java.util.concurrent.locks.StampedLock#tryReadLock()`
  - `java.util.concurrent.locks.StampedLock#tryWriteLock()`
  - `java.util.concurrent.locks.StampedLock#tryOptimisticRead`
  - Convert one mode into another
    - `java.util.concurrent.locks.StampedLock#asReadLock`
    - `java.util.concurrent.locks.StampedLock#asWriteLock`
    - `java.util.concurrent.locks.StampedLock#asReadWriteLock`

  - Release the lock

- All these methods return a long value called stamp that we need to use to work with the lock. If a method returns zero, it means it tried to get a lock but it couldn't

- A `java.util.concurrent.locks.StampedLock` lock is not a reentrant lock.If you call a method that tries to get the lock again, it may be blocked and you'll get a deadlock

- It does not have the notion of ownership. They can be acquired by one thread and released by another.

- Finally, it doesn't have any policy about the next thread that will get control of the lock

example

```java
public class Position {
    private int x;
    private int y;

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }

    public int getY() {
        return y;
    }

    public void setY(int y) {
        this.y = y;
    }
}
```

```java
public class PositionWriter implements Runnable {
    private final Position position;
    private final StampedLock lock;

    public PositionWriter(Position position, StampedLock lock) {
        this.position = position;
        this.lock = lock;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            long stamp = lock.writeLock();
            try {
                System.out.printf("Writer: Lock acquired %d\n", stamp);
                position.setX(position.getX() + 1);
                position.setY(position.getY() + 1);
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlockWrite(stamp);
                System.out.printf("Wirter: Lock released %d\n", stamp);
            }
        }
    }
}
```

```java
public class PositionReader implements Runnable {
    private final Position position;
    private final StampedLock lock;

    public PositionReader(Position position, StampedLock lock) {
        this.position = position;
        this.lock = lock;
    }

    @Override
    public void run() {
        for (int i = 0; i < 50; i++) {
            long stamp = lock.readLock();
            try {
                System.out.printf("Reader: %d - (%d,%d)\n",
                        stamp, position.getX(), position.getY());
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlockRead(stamp);
                System.out.printf("Reader: %d Lock released\n", stamp);
            }
        }
    }
}
```

```java
public class PositionOptimisticReader implements Runnable {

    private final Position position;
    private final StampedLock lock;

    public PositionOptimisticReader(Position position, StampedLock lock) {
        this.position = position;
        this.lock = lock;
    }

    @Override
    public void run() {
        long stamp;
        for (int i = 0; i < 100; i++) {
            try {
                stamp = lock.tryOptimisticRead();
                int x = position.getX();
                int y = position.getY();
                if (lock.validate(stamp)) {
                    System.out.printf("OptimisticReader:%d - (%d,%d)\n", stamp, x, y);
                } else {
                    System.out.printf("OptimisticReader: %d - Not Free\n", stamp);
                }
                TimeUnit.MILLISECONDS.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {

        Position position = new Position();
        StampedLock stampedLock = new StampedLock();

        Thread writeThread = new Thread(new PositionWriter(position,stampedLock));
        Thread readThread = new Thread(new PositionReader(position,stampedLock));
        Thread optimisticReadThread = new Thread(new PositionOptimisticReader(position,stampedLock));

        writeThread.start();
        readThread.start();
        optimisticReadThread.start();

        try{
            writeThread.join();
            readThread.join();
            optimisticReadThread.join();
        }catch (InterruptedException e){
            e.printStackTrace();
        }

    }
}
```

 other methods 

- `java.util.concurrent.locks.StampedLock#isReadLocked`
- `java.util.concurrent.locks.StampedLock#isWriteLocked`
- `java.util.concurrent.locks.StampedLock#tryConvertToReadLock`
- `java.util.concurrent.locks.StampedLock#tryConvertToWriteLock`
- `java.util.concurrent.locks.StampedLock#tryConvertToOptimisticRead`
- `java.util.concurrent.locks.StampedLock#unlock`  releases the corresponding mode of the lock.



## Thread Synchronization Utilities

> - `java.util.concurrent.Semaphore`
>
>   a counter that controls access to one or more shared resources
>
> - `java.util.concurrent.CountDownLatch`
>
>   allows a thread to wait for the finalization of multiple operations.
>
> - `java.util.concurrent.CyclicBarrier`
>
>   allows the synchronization of multiple threads at a common point.
>
> - `java.util.concurrent.Phaser`
>
>   controls the execution of concurrent tasks divided in phases. 
>
> - `java.util.concurrent.Exchanger`
>
>   provides a point of data interchange between two threads.
>
> - `java.util.concurrent.CompletableFuture`
>
>   one or more tasks can wait for the finalization of another task that will be explicitly completed in an asynchronous way in future.

### Controlling concurrent access to one or more copies of a resource

`java.util.concurrent.Semaphore`

 A counter bigger than 0 implies that there are free resources that can be used, so the thread can access and use one of them

example

```java
public class PrintQueue {

    private final Semaphore semaphore;
    private final boolean[] freePrinters;
    private final Lock lockPrinters;

    public PrintQueue() {
        this.semaphore = new Semaphore(3);
        freePrinters = new boolean[3];
        for (int i = 0; i < 3; i++) {
            freePrinters[i] = true;
        }
        this.lockPrinters = new ReentrantLock();
    }

    public void printJob(Object document) {
        try {
            semaphore.acquire();
            int assignedPrinter = getPrinter();
            long duration = (long) (Math.random() * 10);
            System.out.printf("%s - %s: PrintQueue: Printing a job in Printer %d during %d seconds\n",
                    new Date(), Thread.currentThread().getName(), assignedPrinter, duration);
            TimeUnit.SECONDS.sleep(duration);

            freePrinters[assignedPrinter] = true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
        }
    }

    private int getPrinter() {
        int ret = -1;
        try {
            lockPrinters.lock();
            for (int i = 0; i < freePrinters.length; i++) {
                if(freePrinters[i]){
                    ret = i;
                    freePrinters[i] = false;
                    break;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lockPrinters.unlock();
        }
        return ret;
    }
}
```

```java
public class Job implements Runnable {

    private PrintQueue printQueue;

    public Job(PrintQueue printQueue) {
        this.printQueue = printQueue;
    }

    @Override
    public void run() {
        System.out.printf("%s: Going to print a job\n", Thread.currentThread().getName());

        printQueue.printJob(new Object());

        System.out.printf("%s: The Document has been printed\n",Thread.currentThread().getName());

    }
}
```

```java
public class Main01 {

    public static void main(String[] args) {
        PrintQueue printQueue = new PrintQueue();

        Thread[] threads = new Thread[12];

        for (int i = 0; i < 12; i++) {
            threads[i] = new Thread(new Job(printQueue), "Thread" + i);
        }

        for (int i = 0; i < 12; i++) {
            threads[i].start();
        }
    }
}
```

1. you acquire the semaphore with the `java.util.concurrent.Semaphore#acquire()` method
2. you do the necessary operations with the shared resource.
3. release the semaphore with the `java.util.concurrent.Semaphore#release()` method.

three additional versions of the `java.util.concurrent.Semaphore#acquire()` method

- `java.util.concurrent.Semaphore#acquireUninterruptibly()`

  This version of the acquire operation ignores the interruption of the thread and doesn't throw any exceptions.

- `java.util.concurrent.Semaphore#tryAcquire()`

  tries to acquire the semaphore

  It's your responsibility to take correct action based on the return value

- `java.util.concurrent.Semaphore#tryAcquire(long, java.util.concurrent.TimeUnit)`

  it waits for the semaphore for the period of time specified in the parameters.

- **non-fair mode**

- **fair mode**

### Waiting for multiple concurrent events

`java.util.concurrent.CountDownLatch`

one or more threads to wait until a set of operations are made

>When a thread wants to wait for the execution of these operations, it uses the `java.util.concurrent.CountDownLatch#await()` method. This method puts the thread to sleep until the operations are completed. When one of these operations finishes, it uses the `java.util.concurrent.CountDownLatch#countDown` method to decrement the internal counter of the `CountDownLatch` class. When the counter arrives at 0 , the class wakes up all the threads that were sleeping in the `await()` method.

example

```java
public class Videoconference implements Runnable {

    private final CountDownLatch controller;

    public Videoconference(int number) {
        this.controller = new CountDownLatch(number);
    }

    public void arrive(String name) {
        System.out.printf("%s has arrived.", name);
        controller.countDown();

        System.out.printf("Videoconference: Waiting for %d participants.\n", controller.getCount());
    }

    @Override
    public void run() {
        System.out.printf("Videoconference: Initialization: %d participants.\n", controller.getCount());

        try {
            controller.await();
            System.out.print("Videoconference: All the participants have come\n");
            System.out.print("Videoconference: Let's start...\n");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Participant implements Runnable{

    private Videoconference conference;
    private String name;

    public Participant(Videoconference conference, String name) {
        this.conference = conference;
        this.name = name;
    }

    @Override
    public void run() {

        long duration = (long) (Math.random()*10);
        try {
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        conference.arrive(name);
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        Videoconference conference = new Videoconference(10);

        Thread conferenceThread = new Thread(conference);
        conferenceThread.start();

        for (int i = 0; i < 10; i++) {
            Participant participant = new Participant(conference, "Participant" + i);
            Thread t = new Thread(participant);
            t.start();
        }
    }
}
```

1. The initialization value that determines how many events the `CountDownLatch` object waits for
2. The `await()` method, called by the threads that wait for the finalization of all the events
3. The `countDown()` method, called by the events when they finish their execution

There's no way to re-initialize the internal counter of the `CountDownLatch` object or modify its value.

- It is used to synchronize one or more threads with the execution of various tasks
- It only admits one use

`java.util.concurrent.CountDownLatch#await(long, java.util.concurrent.TimeUnit)`

### Synchronizing tasks at a common point

`java.util.concurrent.CyclicBarrier`

The `CyclicBarrier` class is initialized with an integer number, which is the number of threads that will be synchronized at a determined point

example

```java
public class MatrixMock {

    private final int[][] data;

    public MatrixMock(int size, int length, int number) {
        int counter = 0;
        this.data = new int[size][length];
        Random random = new Random();

        for (int i = 0; i < size; i++) {
            for (int j = 0; j < length; j++) {
                data[i][j] = random.nextInt(10);
                if (data[i][j] == number) {
                    counter++;
                }
            }
        }

        System.out.printf("Mock: There are %d ocurrences of %d in generated data.\n", counter, number);
    }

    public int[] getRow(int row) {
        if (row >= 0 && row < data.length) {
            return data[row];
        }
        return null;
    }
}
```

```java
public class Results {
    private final int data[];

    public Results(int size) {
        data = new int[size];
    }

    public int[] getData() {
        return data;
    }

    public void setData(int position, int value) {
        data[position] = value;
    }
}
```

```java
public class Searcher implements Runnable {
    private final int firstRow;
    private final int lastRow;
    private final MatrixMock mock;
    private final Results results;
    private final int number;
    private final CyclicBarrier barrier;

    public Searcher(int firstRow, int lastRow, MatrixMock mock, Results results, int number, CyclicBarrier barrier) {
        this.firstRow = firstRow;
        this.lastRow = lastRow;
        this.mock = mock;
        this.results = results;
        this.number = number;
        this.barrier = barrier;
    }

    @Override
    public void run() {
        int counter;
        System.out.printf("%s: Processing lines from %d to %d\n", Thread.currentThread().getName(),
                firstRow, lastRow);

        for (int i = firstRow; i < lastRow; i++) {
            int[] row = mock.getRow(i);
            counter = 0;
            for (int j = 0; j < row.length; j++) {
                if (row[j] == number) {
                    counter++;
                }
            }
            results.setData(i, counter);
        }
        System.out.printf("%s: Lines processed.\n", Thread.currentThread().getName());

        try {
            barrier.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Grouper implements Runnable {

    private final Results results;

    public Grouper(Results results) {
        this.results = results;
    }

    @Override
    public void run() {
        int finalResult = 0;
        System.out.print("Grouper: Processing results...\n");

        int[] data = results.getData();
        for (int number : data) {
            finalResult += number;
        }

        System.out.printf("Grouper: Total result: %d.\n", finalResult);
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        final int ROWS = 10000;
        final int NUMBERS = 1000;
        final int SEARCH = 5;
        final int PARTICIPANTS = 5;
        final int LINES_PARTICIPANT = 2000;

        MatrixMock mock = new MatrixMock(ROWS, NUMBERS, SEARCH);

        Results results = new Results(ROWS);

        Grouper grouper = new Grouper(results);

        CyclicBarrier barrier = new CyclicBarrier(PARTICIPANTS, grouper);

        Searcher[] searchers = new Searcher[PARTICIPANTS];
        for (int i = 0; i < PARTICIPANTS; i++) {
            searchers[i] = new Searcher(i*LINES_PARTICIPANT, (i*LINES_PARTICIPANT)+LINES_PARTICIPANT,
                    mock, results, SEARCH, barrier);
            Thread thread = new Thread(searchers[i]);
            thread.start();
        }

        System.out.print("Main: The Main thread has finished.\n");

    }
}
```

```
Mock: There are 1001916 ocurrences of 5 in generated data.
Thread-0: Processing lines from 0 to 2000
Thread-4: Processing lines from 8000 to 10000
Thread-3: Processing lines from 6000 to 8000
Main: The Main thread has finished.
Thread-2: Processing lines from 4000 to 6000
Thread-1: Processing lines from 2000 to 4000
Thread-3: Lines processed.
Thread-0: Lines processed.
Thread-4: Lines processed.
Thread-1: Lines processed.
Thread-2: Lines processed.
Grouper: Processing results...
Grouper: Total result: 1001916.
```

`CyclicBarrier` object can be reset to its initial state, assigning to its internal counter the value with which it was initialized. 

When there are various threads waiting in the `await() `method and one of them is interrupted, the one that is interrupted receives an `InterruptedException` exception, but other threads receive a `BrokenBarrierException` exception; `CyclicBarrier` is placed in the broken state

### Running concurrent-phased tasks

`java.util.concurrent.Phaser`

synchronize threads at the end of each step, so no thread will start with the second step until all the threads have finished the first one

example

```java
public class FileSearch implements Runnable {

    private final String initPath;
    private final String fileExtension;
    private List<String> results;
    private Phaser phaser;

    public FileSearch(String initPath, String fileExtension, Phaser phaser) {
        this.initPath = initPath;
        this.fileExtension = fileExtension;
        this.phaser = phaser;
        this.results = new ArrayList<>();
    }

    private void directoryProcess(File file) {

        File[] list = file.listFiles();
        if (null != list) {
            for (int i = 0; i < list.length; i++) {
                if (list[i].isDirectory()) {
                    directoryProcess(list[i]);
                } else {
                    fileProcess(list[i]);
                }
            }
        }
    }

    private void fileProcess(File file) {
        if (file.getName().endsWith(fileExtension)) {
            results.add(file.getAbsolutePath());
        }
    }

    private void filterResults() {
        List<String> newResults = new ArrayList<>();
        long actualDate = new Date().getTime();

        for (int i = 0; i < results.size(); i++) {
            File file = new File(results.get(i));
            long fileDate = file.lastModified();

            if (actualDate - fileDate < TimeUnit.MILLISECONDS.convert(1, TimeUnit.DAYS)) {
                newResults.add(results.get(i));
            }
        }

        results = newResults;
    }

    private boolean checkResults() {
        if (results.isEmpty()) {
            System.out.printf("%s: Phase %d:0 results.\n",
                    Thread.currentThread().getName(), phaser.getPhase());

            System.out.printf("%s: Phase %d:0 End.\n",
                    Thread.currentThread().getName(), phaser.getPhase());
            phaser.arriveAndDeregister();
            return false;
        } else {
            System.out.printf("%s: Phase %d: %d results.\n",
                    Thread.currentThread().getName(), phaser.getPhase(), results.size());
            phaser.arriveAndAwaitAdvance();
            return true;
        }
    }

    private void showInfo() {
        for (int i = 0; i < results.size(); i++) {
            File file = new File(results.get(i));
            System.out.printf("%s: %s\n", Thread.currentThread().getName(), file.getAbsolutePath());
        }
        phaser.arriveAndAwaitAdvance();
    }

    @Override
    public void run() {
        phaser.arriveAndAwaitAdvance();
        System.out.printf("%s: Starting.\n", Thread.currentThread().getName());

        File file = new File(initPath);
        if (file.isDirectory()) {
            directoryProcess(file);
        }

        if (!checkResults()) {
            return;
        }

        filterResults();

        if (!checkResults()) {
            return;
        }

        showInfo();

        phaser.arriveAndDeregister();
        System.out.printf("%s: Work completed.\n", Thread.currentThread().getName());
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3);

        FileSearch system = new FileSearch("c:/Windows", "log", phaser);
        FileSearch apps = new FileSearch("c:/Program Files", "log", phaser);
        FileSearch documents = new FileSearch("c:/Documents And Settings", "log", phaser);

        Thread systemThread = new Thread(system, "System");
        Thread appsThread = new Thread(apps, "Apps");
        Thread documentsThread = new Thread(documents, "Documents");

        systemThread.start();
        appsThread.start();
        documentsThread.start();

        try {
            systemThread.join();
            appsThread.join();
            documentsThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("Terminated: " + phaser.isTerminated());
    }
}
```

 `Phaser` knows the number of threads that we want to synchronize.

two states

- Active
- Termination

`java.util.concurrent.Phaser#arriveAndAwaitAdvance`

> `phaser` decreases the number of threads that have to finalize the actual phase and puts this thread to sleep until all the remaining threads finish this phase.

`java.util.concurrent.Phaser#arriveAndDeregister`

> This notifies phaser that the thread has finished the actual phase, but it won't participate in future phases

`java.util.concurrent.Phaser#isTerminated`

`java.util.concurrent.Phaser#arrive`

> This method notifies the `Phaser` class that one participant has finished the actual phase but it should **not wait for the rest of the participants** to continue with their execution

`java.util.concurrent.Phaser#awaitAdvance`

> This method puts the current thread to sleep until all the participants of the `phaser` parameter have finished the current phase

`java.util.concurrent.Phaser#awaitAdvanceInterruptibly(int)`

> it throws an InterruptedException exception if the thread that is sleeping in this method is interrupted

`java.util.concurrent.Phaser#register`

> adds a new participant to `Phaser `

`java.util.concurrent.Phaser#bulkRegister`

> adds the specified number of participants to`phaser`

`java.util.concurrent.Phaser#forceTermination`

### Controlling phase change in concurrent-phased tasks

provides a method that is executed each time `phaser` changes the phase

```java
/**
 * @param phase the current phase number on entry to this method,
 * before this phaser is advanced
 * @param registeredParties the current number of registered parties
 */
protected boolean onAdvance(int phase, int registeredParties) {
    return registeredParties == 0;
}
```

you can modify this behavior if you extend the `Phaser` class and override this method

example

```java
public class MyPhaser extends Phaser {
    @Override
    protected boolean onAdvance(int phase, int registeredParties) {
        switch (phase) {
            case 0:
                return studentsArrived();
            case 1:
                return finishFirstExercise();
            case 2:
                return finishSecondExercise();
            case 3:
                return finishExam();
            default:
                return true;
        }
    }

    private boolean studentsArrived() {
        System.out.print("Phaser: The exam are going to start.The Students are ready.\n");
        System.out.printf("Phaser: We have %d students.\n",getRegisteredParties());
        return false;
    }
    private boolean finishFirstExercise() {
        System.out.print("Phaser: All students have finished the first exercise.\n");
        System.out.print("Phaser: It's time for the second one.\n");
        return false;
    }

    private boolean finishSecondExercise() {
        System.out.print("Phaser: All students have finished the second exercise.\n");
        System.out.print("Phaser: It's time for the third one.\n");
        return false;
    }

    private boolean finishExam() {
        System.out.print("Phaser: All students have finished the exam.\n");
        System.out.print("Phaser: Thank you for your time.\n");
        return true;
    }
}
```

```java
public class Student implements Runnable {
    private Phaser phaser;

    public Student(Phaser phaser) {
        this.phaser = phaser;
    }

    @Override
    public void run() {
        System.out.printf("%s: Has arrived to do the exam. %s\n",
                Thread.currentThread().getName(), new Date());
        phaser.arriveAndAwaitAdvance();

        System.out.printf("%s: Is going to do the first exercise. %s\n", Thread.currentThread().getName(), new Date());
        doExercise1();
        System.out.printf("%s: Has down the first exercise. %s\n", Thread.currentThread().getName(), new Date());
        phaser.arriveAndAwaitAdvance();

        System.out.printf("%s: Is going to do the second exercise. %s\n", Thread.currentThread().getName(), new Date());
        doExercise2();
        System.out.printf("%s: Has down the second exercise. %s\n", Thread.currentThread().getName(), new Date());
        phaser.arriveAndAwaitAdvance();

        System.out.printf("%s: Is going to do the third exercise. %s\n", Thread.currentThread().getName(), new Date());
        doExercise3();
        System.out.printf("%s: Has down the third exercise. %s\n", Thread.currentThread().getName(), new Date());
        phaser.arriveAndAwaitAdvance();
    }

    private void doExercise1() {
        doExercise();
    }

    private void doExercise2() {
        doExercise();
    }

    private void doExercise3() {
        doExercise();
    }

    private void doExercise() {
        try {
            long duration = (long) (Math.random() * 10);
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) {
        MyPhaser phaser = new MyPhaser();

        Student[] student = new Student[5];
        for (int i = 0; i < 5; i++) {
            student[i] = new Student(phaser);
            phaser.register();
        }

        Thread[] threads = new Thread[student.length];
        for (int i = 0; i < student.length; i++) {
            threads[i] = new Thread(student[i], "Student " + i);
            threads[i].start();
        }

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Main: The phaser has finished: %s.\n", phaser.isTerminated());
    }
}
```



### Exchanging data between concurrent tasks

`java.util.concurrent.Exchanger`

allows you to have a definition of a synchronization point between two threads

example

```java
public class Producer implements Runnable {
    private List<String> buffer;
    private final Exchanger<List<String>> exchanger;

    public Producer(List<String> buffer, Exchanger<List<String>> exchanger) {
        this.buffer = buffer;
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        for (int cycle = 1; cycle <= 10; cycle++) {
            System.out.printf("Producer: cycle %d \n", cycle);
            for (int j = 0; j < 10; j++) {
                String message = "Event " + ((cycle - 1) * 10 + j);
                System.out.printf("Producer: %s\n", message);
                buffer.add(message);
            }

            try {
                buffer = exchanger.exchange(buffer);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.printf("Producer: %d\n", buffer.size());
        }
    }
}
```

```java
public class Consumer implements Runnable {

    private List<String> buffer;

    private final Exchanger<List<String>> exchanger;

    public Consumer(List<String> buffer, Exchanger<List<String>> exchanger) {
        this.buffer = buffer;
        this.exchanger = exchanger;
    }

    @Override
    public void run() {
        for (int cycle = 1; cycle <= 10; cycle++) {
            System.out.printf("Consumer: Cycle %d\n", cycle);

            try {
                buffer = exchanger.exchange(buffer);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.printf("Consumer: %d\n", buffer.size());

            for (int j = 0; j < 10; j++) {
                String message = buffer.get(0);
                System.out.printf("Consumer: %s\n", message);
                buffer.remove(0);
            }
        }
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        List<String> buffer1 = new ArrayList<>();
        List<String> buffer2 = new ArrayList<>();

        Exchanger<List<String>> exchanger = new Exchanger<>();

        Producer producer = new Producer(buffer1,exchanger);
        Consumer consumer = new Consumer(buffer2,exchanger);

        Thread producerThread = new Thread(producer);
        Thread consumerThread = new Thread(consumer);

        producerThread.start();
        consumerThread.start();
    }
}
```



### Completing and linking tasks asynchronously

`java.util.concurrent.CompletableFuture`

->`java.util.concurrent.Future`

->`java.util.concurrent.CompletionStage`

- return a result sometime in future

- execute more asynchronous tasks after the completion of one or more `CompletableFuture` objects

work with a `CompletableFuture` class in different ways

- create a `CompletableFuture`object explicitly 
- use a static method of the `CompletableFuture` class to execute `Runnable` or `Supplier` with the
  `runAsync()` and `supplyAsync()` methods.
- specify other tasks to be executed in an asynchronous way after the completion of one or more `CompletableFuture` objects. 

example

```java
public class NumberListGenerator implements Supplier<List<Long>> {

    private final int size;

    public NumberListGenerator(int size) {
        this.size = size;
    }

    @Override
    public List<Long> get() {
        List<Long> ret = new ArrayList<>();
        System.out.printf("%s: NumberListGenerator : Start\n", Thread.currentThread().getName());

        for (int i = 0; i < size * 1000000; i++) {
            long number = Math.round(Math.random() * Long.MAX_VALUE);
            ret.add(number);
        }
        System.out.printf("%s: NumberListGenerator : End\n", Thread.currentThread().getName());
        return ret;
    }
}
```

```java
public class NumberSelector implements Function<List<Long>, Long> {
    @Override
    public Long apply(List<Long> list) {
        System.out.printf("%s: Step 3: Start\n", Thread.currentThread().getName());

        long max = list.stream().max(Long::compare).get();
        long min = list.stream().min(Long::compare).get();
        long result = (max + min) / 2;

        System.out.printf("%s: Step 3: Result - %d\n", Thread.currentThread().getName(), result);
        return result;
    }
}
```

```java
public class SeedGenerator implements Runnable {

    private CompletableFuture<Integer> resultCommunicator;

    public SeedGenerator(CompletableFuture<Integer> completable) {
        this.resultCommunicator = completable;
    }

    @Override
    public void run() {
        System.out.print("SeedGenerator: Generating seed ...\n");

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        int seed = (int) Math.rint(Math.random() * 10);

        System.out.printf("SeedGenerator: seed generated: %d\n", seed);

        resultCommunicator.complete(seed);
    }
}
```

```java
public class Main07 {
    public static void main(String[] args) {
        System.out.print("Main: start...\n");
        CompletableFuture<Integer> seedFuture = new CompletableFuture<>();
        Thread seedThread = new Thread(new SeedGenerator(seedFuture));
        seedThread.start();


        System.out.print("Main: Getting the seed\n");
        int seed = 0;
        try {
            seed = seedFuture.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        System.out.printf("Main: The seed is : %d\n", seed);

        System.out.print("Main: Launching the list of numbers generator\n");
        NumberListGenerator task = new NumberListGenerator(seed);
        CompletableFuture<List<Long>> startFuture = CompletableFuture.supplyAsync(task);

        System.out.print("Main: Launching step 1\n");
        CompletableFuture<Long> step1Future = startFuture.thenApplyAsync(list -> {
            System.out.printf("%s: Step 1: Start\n", Thread.currentThread().getName());
            long selected = 0;
            long selectedDistance = Long.MAX_VALUE;
            long distance;
            for (Long number : list) {
                distance = Math.abs(number - 1000);
                if (distance < selectedDistance) {
                    selected = number;
                    selectedDistance = distance;
                }
            }
            System.out.printf("%s: Step 1: Result - %d\n", Thread.currentThread().getName(), selected);
            return selected;
        });

        System.out.print("Main: Launching step 2\n");
        CompletableFuture<Long> step2Future = startFuture
                .thenApplyAsync(list -> list.stream().max(Long::compare).get());

        CompletableFuture<Void> write2Future = step2Future
                .thenAccept(selected -> System.out.printf("%s: Step 2: Result - %d\n",
                        Thread.currentThread().getName(), selected));

        System.out.print("Main: Launching step 3\n");
        NumberSelector numberSelector = new NumberSelector();
        CompletableFuture<Long> step3Future = startFuture.thenApplyAsync(numberSelector);

        System.out.print("Main: Waiting for the end of the three steps\n");
        CompletableFuture<Void> waitFuture = CompletableFuture.allOf(step1Future, step2Future, step3Future);

        CompletableFuture<Void> finalFuture = waitFuture
                .thenAcceptAsync(param -> System.out.print("Main: The CompletableFuture example has bean completed."));
        finalFuture.join();

    }
}
```

two main purposes

- Wait for a value or an event that will be produced in future (creating an object and using the`complete()`and `get()` or `join()` methods).
- To organize a set of tasks to be executed in a determined order so one or more tasks won't start their
  execution until others have finished their execution.

`java.util.concurrent.CompletableFuture#complete`

`java.util.concurrent.CompletableFuture#get()`

`java.util.concurrent.CompletableFuture#join`

`java.util.concurrent.CompletableFuture#supplyAsync(java.util.function.Supplier<U>)`

`java.util.concurrent.CompletableFuture#thenApplyAsync(java.util.function.Function<? super T,? extends U>)`

`java.util.concurrent.CompletableFuture#thenAcceptAsync(java.util.function.Consumer<? super T>)`

`java.util.concurrent.CompletableFuture#allOf`



other useful method

- complete a `CompletableFuture` object
  - `java.util.concurrent.CompletableFuture#cancel`
  - `java.util.concurrent.CompletableFuture#completeAsync(java.util.function.Supplier<? extends T>)`
  - `java.util.concurrent.CompletableFuture#completeExceptionally`
- execute a task
  - `java.util.concurrent.CompletableFuture#runAsync(java.lang.Runnable)`
- synchronize the execution of different tasks
  - `java.util.concurrent.CompletableFuture#anyOf`
  - `java.util.concurrent.CompletableFuture#runAfterBothAsync(java.util.concurrent.CompletionStage<?>, java.lang.Runnable)`
  - `java.util.concurrent.CompletableFuture#runAfterEitherAsync(java.util.concurrent.CompletionStage<?>, java.lang.Runnable)`
  - `java.util.concurrent.CompletableFuture#thenAcceptBothAsync(java.util.concurrent.CompletionStage<? extends U>, java.util.function.BiConsumer<? super T,? super U>)`
  - `java.util.concurrent.CompletableFuture#thenCombineAsync(java.util.concurrent.CompletionStage<? extends U>, java.util.function.BiFunction<? super T,? super U,? extends V>)`
  - `java.util.concurrent.CompletableFuture#thenComposeAsync(java.util.function.Function<? super T,? extends java.util.concurrent.CompletionStage<U>>)`
  - `java.util.concurrent.CompletableFuture#thenRunAsync(java.lang.Runnable)`
- obtain the completion value
  - `java.util.concurrent.CompletableFuture#getNow`

## Thread Executors

> **Executor framework**
>
> `java.util.concurrent.Executor`
>
> - `java.util.concurrent.ExecutorService`
>   - `java.util.concurrent.ThreadPoolExecutor`
>
> This mechanism separates task creation and execution.
>
> running `java.lang.Runnable` or `java.util.concurrent.Callable` with the necessary threads.
>
> it improves performance using a pool of threads
>
> `java.util.concurrent.Callable` improvements
>
> - The main method of this interface, named `call()` , may return a result
> - When you send a `Callable` object to an executor, you get an object that implements the `Future` interface. You can use this object to control the status and the result of the Callable object.



### Creating a thread executor and controlling its rejected tasks

`java.util.concurrent.ThreadPoolExecutor`

`java.util.concurrent.ExecutorService#shutdown`

执行`shutdown()` 方法 executor 等待剩余的线程执行完毕，然后结束executor。

在此期间向executor发送task会被拒绝。

example

```java
public class Task implements Runnable {
    private final Date initDate;
    private final String name;

    public Task(String name) {
        this.initDate = new Date();
        this.name = name;
    }

    @Override
    public void run() {
        System.out.printf("%s: Task %s: Created on %s\n",
                Thread.currentThread().getName(), name, initDate);
        System.out.printf("%s: Task %s: Started on %s\n",
                Thread.currentThread().getName(), name, new Date());
        try{
            long duration = (long) (Math.random() * 10);
            System.out.printf("%s: Task %s: Doing a task during %d seconds\n",
                    Thread.currentThread().getName(), name, duration);
            TimeUnit.SECONDS.sleep(duration);
        }catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.printf("%s: Task %s: Finished on %s\n",
                Thread.currentThread().getName(), name, new Date());
    }
}
```

```java
public class RejectedTaskController implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        System.out.printf("RejectedTaskController: The task %s has been rejected\n", r.toString());
        System.out.printf("RejectedTaskController: %s\n", executor.toString());
        System.out.printf("RejectedTaskController: Terminating: %s\n", executor.isTerminating());
        System.out.printf("RejectedTaskController: Terminated: %s\n", executor.isTerminated());
    }
}
```

```java
public class Server {

    private final ThreadPoolExecutor executor;

    public Server() {
        this.executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(
                Runtime.getRuntime().availableProcessors());
        RejectedTaskController controller = new RejectedTaskController();
        executor.setRejectedExecutionHandler(controller);
    }

    public void executeTask(Task task){
        System.out.print("Server: A new task has arrived\n");

        executor.execute(task);

        System.out.printf("Server: Pool Size: %d\n",executor.getPoolSize());
        System.out.printf("Server: Active Count: %d\n",executor.getActiveCount());
        System.out.printf("Server: Task Count: %d\n",executor.getTaskCount());
        System.out.printf("Server: Completed Tasks: %d\n",executor.getCompletedTaskCount());
    }

    public void endServer(){
        executor.shutdown();
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        Server server = new Server();
        System.out.print("Main: Starting...\n");

        for (int i = 0; i < 100; i++) {
            Task task = new Task("Task " + i);
            server.executeTask(task);
        }

        System.out.print("Main: Shutting down the Executor.\n");
        server.endServer();

        System.out.print("Main: Sending anotherTask.\n");
        Task task = new Task("Rejected Task");
        server.executeTask(task);

        System.out.print("Main: End.\n");
    }
}
```

`java.util.concurrent.Executors`

`java.util.concurrent.Executors#newCachedThreadPool()`

The cached thread pool you created creates new threads, if needed, to execute new tasks. Plus, it reuses the existing ones if they have finished the execution of the tasks they were running.

`java.util.concurrent.Executors#newSingleThreadExecutor()`

This is an extreme case of a fixed-size thread executor. It creates an executor with **only one thread** so it can only execute one task at a time

`java.util.concurrent.ThreadPoolExecutor`

- `java.util.concurrent.ThreadPoolExecutor#getPoolSize`
- `java.util.concurrent.ThreadPoolExecutor#getActiveCount`
- `java.util.concurrent.ThreadPoolExecutor#getCompletedTaskCount`
- `java.util.concurrent.ThreadPoolExecutor#getLargestPoolSize`
- `java.util.concurrent.ThreadPoolExecutor#shutdown`
- `java.util.concurrent.ThreadPoolExecutor#isTerminated`
- `java.util.concurrent.ThreadPoolExecutor#isShutdown`
- `java.util.concurrent.ThreadPoolExecutor#awaitTermination`



### Executing multiple tasks executor that returns a result

- `java.util.concurrent.Callable`

- `java.util.concurrent.Future`

  This interface has some methods to **obtain the result** generated by a Callable object and manage its state.

example

```java
public class FactorialCalculator implements Callable<Integer> {
    private final Integer number;

    public FactorialCalculator(Integer number) {
        this.number = number;
    }

    @Override
    public Integer call() throws Exception {
        int result = 1;
        if (number == 0 || number == 1) {
            result = 1;
        } else {
            for (int i = 2; i <= number; i++) {
                result *= i;
                TimeUnit.MILLISECONDS.sleep(20);
            }
        }

        System.out.printf("%s: %d\n", Thread.currentThread().getName(), result);

        return result;
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(2);
        List<Future<Integer>> resultList = new ArrayList<>();

        Random random = new Random();

        for (int i = 0; i < 10; i++) {
            Integer number = random.nextInt(10);
            FactorialCalculator calculator = new FactorialCalculator(number);
            Future<Integer> result = executor.submit(calculator);
            resultList.add(result);
        }

        do {
            System.out.printf("Main: Number of Completed Tasks: %d\n", executor.getCompletedTaskCount());
            for (int i = 0; i < resultList.size(); i++) {
                Future<Integer> result = resultList.get(i);
                System.out.printf("Main: Task %d: %s\n", i, result.isDone());
            }

            try {
                TimeUnit.MILLISECONDS.sleep(20);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } while (executor.getCompletedTaskCount() < resultList.size());

        System.out.print("Main: Results\n");
        for (int i = 0; i < resultList.size(); i++) {
            Future<Integer> result = resultList.get(i);
            Integer number = null;
            try {
                number = result.get();
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }

            System.out.printf("Main: Task %d: %d\n", i, number);
        }
        executor.shutdown();
    }

}
```

`java.util.concurrent.ExecutorService#submit(java.util.concurrent.Callable<T>)`

receives a `Callable` object as a parameter and returns a `Future` object 

two main objectives

- control the status of the task you can cancel the task and check whether it has finished or not.

  `java.util.concurrent.Future#isDone`

- get the result returned by the `call()` method. used the `get()` method.This method waits until the `Callable` object has finished the execution of the `call()` method and has returned its result.

When you call the `get()`method of a `Future` object and the task controlled by this object hasn't finished yet, the method is **blocked until the task is finished**.



### Running multiple tasks and processing the first result

example

```java
public class UserValidator {
    private final String name;

    public UserValidator(String name) {
        this.name = name;
    }

    public boolean validate(String name, String password) {
        Random random = new Random();
        try {
            long duration = (long) (Math.random() * 10);
            System.out.printf("Validator %s: Validating a user during %d seconds\n", this.name, duration);

            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return random.nextBoolean();
    }

    public String getName() {
        return name;
    }
}
```

```java
public class ValidatorTask implements Callable<String> {
    private final UserValidator validator;
    private final String user;
    private final String password;

    public ValidatorTask(UserValidator validator, String user, String password) {
        this.validator = validator;
        this.user = user;
        this.password = password;
    }

    @Override
    public String call() throws Exception {
        if (!validator.validate(user, password)) {
            System.out.printf("%s: The user has not been found\n", validator.getName());
            throw new Exception("Error validating user");
        }

        System.out.printf("%s: The user has been found\n", validator.getName());
        return validator.getName();
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        String name = "test";
        String password = "test";

        UserValidator ldapValidator = new UserValidator("LDAP");
        UserValidator dbValidator = new UserValidator("DataBase");

        ValidatorTask ldapTask = new ValidatorTask(ldapValidator, name, password);
        ValidatorTask dbTask = new ValidatorTask(dbValidator, name, password);

        List<ValidatorTask> taskList = new ArrayList<>();
        taskList.add(ldapTask);
        taskList.add(dbTask);

        ExecutorService executor = Executors.newCachedThreadPool();
        String result;
        try {
            result = executor.invokeAny(taskList);
            System.out.printf("Main: Result: %s\n", result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }

        executor.shutdown();
        System.out.print("Main: End of the Execution\n");
    }
}
```

four possibilities:

- Both tasks return the true value. Here, the result of the `invokeAny()` method is the name of the task that finishes in the first place.
- The first task returns the true value and the second one throws Exception . Here, the result of the `invokeAny()` method is the name of the first task.
- The first task throws Exception and the second one returns the true value. Here, the result of the `invokeAny()` method is the name of the second task.
- Both tasks throw Exception . In such a class, the `invokeAny()` method throws an `ExecutionException` exception.



### Running multiple tasks and processing all the results

example

```java
public class Result {
    private String name;
    private int value;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getValue() {
        return value;
    }

    public void setValue(int value) {
        this.value = value;
    }
}
```

```java
public class ResultTask implements Callable<Result> {

    private final String name;

    public ResultTask(String name) {
        this.name = name;
    }

    @Override
    public Result call() throws Exception {
        System.out.printf("%s: Starting \n", this.name);
        try {
            long duration = (long) (Math.random() * 10);
            System.out.printf("%s: Waiting %s seconds for results.\n", this.name, duration);
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        int value = 0;
        for (int i = 0; i < 5; i++) {
            value += (int) (Math.random() * 100);
        }

        Result result = new Result();
        result.setName(this.name);
        result.setValue(value);

        System.out.printf("%s: End.\n", this.name);
        return result;

    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        ExecutorService executorService = Executors.newCachedThreadPool();

        List<ResultTask> taskList = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            ResultTask task = new ResultTask("Task " + i);
            taskList.add(task);
        }

        List<Future<Result>> resultList = null;
        try {
            resultList = executorService.invokeAll(taskList);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        executorService.shutdown();

        System.out.println("Main: Printing the results");

        for (int i = 0; i < resultList.size(); i++) {
            Future<Result> future = resultList.get(i);
            try {
                Result result = future.get();
                System.out.println(result.getName() + ": " + result.getValue());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
    }
}
```

`java.util.concurrent.ExecutorService#invokeAll(java.util.Collection<? extends java.util.concurrent.Callable<T>>)`



### Running a task in an executor after a delay

When you send a task to the executor, **it's executed as soon as possible**, according to the configuration of the executor. There are use cases when you are not interested in executing a task as soon as possible. You may want to **execute a task after a period of time or do it periodically**.

`java.util.concurrent.ScheduledExecutorService`

->`java.util.concurrent.ScheduledThreadPoolExecutor`

example

```java
public class ScheduledTask implements Callable<String> {
    private final String name;

    public ScheduledTask(String name) {
        this.name = name;
    }

    @Override
    public String call() throws Exception {
        System.out.printf("%s: Starting at : %s\n", this.name, new Date());
        return "hello world";
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) {
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();

        System.out.printf("Main: Starting at: %s\n", new Date());
        for (int i = 0; i < 5; i++) {
            ScheduledTask task = new ScheduledTask("Scheduled Task " + i);
            executor.schedule(task, i + 1, TimeUnit.SECONDS);
        }

        executor.shutdown();

        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Ends at: %s\n", new Date());
    }
}
```

`java.util.concurrent.ScheduledExecutorService#schedule(java.util.concurrent.Callable<V>, long, java.util.concurrent.TimeUnit)`

- The task you want to execute
- The period of time you want the task to wait before its execution
- The unit of the period of time, specified as a constant of the `TimeUnit` class

configure the behavior of the `ScheduledThreadPoolExecutor` class

`java.util.concurrent.ScheduledThreadPoolExecutor#setExecuteExistingDelayedTasksAfterShutdownPolicy`

passing the false value as parameter, pending tasks won't be executed after you call the `shutdown()` method.

### Running a task in an executor periodically

example

```java
public class PeriodicallyTask implements Runnable {

    private final String name;

    public PeriodicallyTask(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        System.out.printf("%s: Executed at: %s\n",this.name,new Date());

    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();

        System.out.printf("Main: Starting at %s\n", new Date());
        PeriodicallyTask task = new PeriodicallyTask("periodically");

        ScheduledFuture<?> result = executor.scheduleAtFixedRate(task, 1, 2, TimeUnit.SECONDS);

        for (int i = 0; i < 10; i++) {
            System.out.printf("Main: Delay: %d\n", result.getDelay(TimeUnit.MILLISECONDS));

            try {
                TimeUnit.MILLISECONDS.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Finished at: %s\n", new Date());

    }
}
```

`java.util.concurrent.ScheduledExecutorService#scheduleAtFixedRate`

- the task you want to execute periodically
- the delay of time until the first execution of the task
- the period between two executions
- the time unit of the second and third parameters

`java.util.concurrent.ScheduledExecutorService#scheduleWithFixedDelay`

### Canceling a task in an executor

you may want to cancel a task that you send to the executor. In that case, you can use the `cancel()` method of `Future` ,which allows you to make the cancelation operation.

example

```java
public class CancelTask implements Callable<String> {
    @Override
    public String call() throws Exception {
        while(true){
            System.out.print("Task: test\n");
            Thread.sleep(100);
        }
    }
}
```

```java
public class Main07 {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();

        CancelTask task = new CancelTask();

        System.out.print("Main: Executing the task\n");
        Future<String> result = executor.submit(task);

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: canceling the task\n");
        result.cancel(true);

        System.out.printf("Main: Canceled: %s\n", result.isCancelled());
        System.out.printf("Main: Done: %s\n", result.isDone());

        executor.shutdown();
        System.out.print("Main: The Executor has finished\n");
    }
}
```



### Controlling a task finishing in an executor

`java.util.concurrent.FutureTask`

`java.util.concurrent.FutureTask#done`

allows you to execute some code after the finalization of a task executed in an executor. 

example

```java
public class ExecutableTask implements Callable<String> {
    private final String name;

    public ExecutableTask(String name) {
        this.name = name;
    }

    @Override
    public String call() throws Exception {
        try{
            long duration = (long)(Math.random()*10);
            System.out.printf("%s: Waiting %d seconds for results.\n",this.name,duration);

            TimeUnit.SECONDS.sleep(duration);
        }catch (InterruptedException e){}
        return "Hello world! I'm "+ this.name;
    }

    public String getName() {
        return name;
    }
}
```

```java
public class ResultFutureTask extends FutureTask<String> {
    private final String name;

    public ResultFutureTask(ExecutableTask callable) {
        super(callable);
        this.name = callable.getName();
    }

    @Override
    protected void done() {
        if (isCancelled()) {
            System.out.printf("%s: Has been canceled\n", this.name);
        }else{
            System.out.printf("%s: Has finished.\n", this.name);
        }
    }
}
```

```java
public class Main08 {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();

        ResultFutureTask[] resultTasks = new ResultFutureTask[5];

        for (int i = 0; i < 5; i++) {
            ExecutableTask task = new ExecutableTask("Task " + i);
            resultTasks[i] = new ResultFutureTask(task);
            executor.submit(resultTasks[i]);
        }

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int i = 0; i < 5; i++) {
            resultTasks[i].cancel(true);
        }

        for (int i = 0; i < resultTasks.length; i++) {
            try {
                if (!resultTasks[i].isCancelled()) {
                    System.out.printf("%s\n", resultTasks[i].get());
                }
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();
    }
}
```



### Separating the launching of tasks and the processing of their results in an executor  

`java.util.concurrent.CompletionService`

The `CompletionService` class has a method to send tasks to an executor and a method to get the `Future` object for the next task that has finished its execution.

uses an `Executor` object to execute the tasks. 

This behavior has the advantage of sharing a `CompletionService` object and sending tasks to the executor so others can process the results.

The limitation is that the second object can only get the Future objects for those tasks that have finished their execution, so these Future objects can only be used to get the results of the tasks.

example

```java
public class ReportGenerator implements Callable<String> {

    private final String sender;
    private final String title;

    public ReportGenerator(String sender, String title) {
        this.sender = sender;
        this.title = title;
    }

    @Override
    public String call() throws Exception {
        try {
            long duration = (long) (Math.random() * 10);
            System.out.printf("%s_%s: ReportGenerator: Generating a report during %d seconds\n",
                    this.sender, this.title, duration);
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        String ret = sender + ": " + title;
        return ret;
    }
}
```

```java
public class ReportRequest implements Runnable {

    private final String name;

    private final CompletionService<String> service;

    public ReportRequest(String name, CompletionService<String> service) {
        this.name = name;
        this.service = service;
    }

    @Override
    public void run() {
        ReportGenerator reportGenerator = new ReportGenerator(name, "Report");

        service.submit(reportGenerator);
    }
}
```

```java
public class ReportProcessor implements Runnable {
    private final CompletionService<String> service;
    private volatile boolean end;

    public ReportProcessor(CompletionService<String> service) {
        this.service = service;
        this.end = false;
    }

    @Override
    public void run() {
        while (!end) {
            try {
                Future<String> result = service.poll(2, TimeUnit.SECONDS);
                if (result != null) {
                    String report = result.get();
                    System.out.printf("ReportReceiver: Report Received: %s\n", report);
                }
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        System.out.print("ReportSender: End\n");
    }

    public void stopProcessing() {
        this.end = true;
    }
}
```

```java
public class Main09 {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        CompletionService<String> service = new ExecutorCompletionService<>(executor);

        ReportRequest faceRequest = new ReportRequest("Face", service);
        ReportRequest onlineRequest = new ReportRequest("Online", service);

        Thread faceThread = new Thread(faceRequest);
        Thread onlineThread = new Thread(onlineRequest);

        ReportProcessor processor = new ReportProcessor(service);
        Thread senderThread = new Thread(processor);

        System.out.print("Main: Starting the Threads\n");
        faceThread.start();
        onlineThread.start();
        senderThread.start();

        try {
            System.out.print("Main: Waiting for the report generators.\n");
            faceThread.join();
            onlineThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: Shutting down the executor.\n");
        executor.shutdown();

        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        processor.stopProcessing();
        System.out.print("Main: Ends\n");
    }
}
```

`java.util.concurrent.CompletionService`

->`java.util.concurrent.ExecutorCompletionService`

`java.util.concurrent.CompletionService#submit(java.util.concurrent.Callable<V>)`

`java.util.concurrent.CompletionService#poll()`

`java.util.concurrent.CompletionService#take`

## Fork/Join Framework

> This framework is designed to solve problems that can be broken into smaller tasks using the divide and conquer technique.
>
> two operations
>
> - **Fork operation** divide a task into smaller tasks and execute them using the framework
> - **Join operation** combine the results of those tasks
>
> when a task is waiting for the finalization of the subtasks it has created using the join operation, the thread that is executing that task (called **worker thread**) looks for other tasks that have not been executed yet and begins their execution.
>
> limitations
>
> - Tasks can only use the `fork()` and `join()` operations as synchronization mechanisms.
> - Tasks should not perform I/O operations such as read or write data in a file.
> - Tasks can't throw checked exceptions.
>
> core
>
> - `java.util.concurrent.ForkJoinPool`
>
>   implements the `ExecutorService` interface and the **work-stealing algorithm**
>
> - `java.util.concurrent.ForkJoinTask`
>
>   the base class of the tasks that will execute in the `ForkJoinPool`
>
>   It provides the mechanisms to execute the fork() and join() operations inside a task and the methods to control the status of the tasks.
>
>   Usually, to implement your fork/join tasks, you will implement a subclass of three subclasses of this class:
>
>   - `java.util.concurrent.RecursiveAction` for tasks with no return result
>   - `java.util.concurrent.RecursiveTask` for tasks that return one result
>   - `java.util.concurrent.CountedCompleter` for tasks that launch a completion action when all the subtasks have finished.
>
> You can obtain it using the static method, `commonPool()` ,of the` ForkJoinPool` class. This default fork/join executor will by default use the number of threads **determined by the available processors of your computer**. You can change this default behavior by changing the value of the system property, `java.util.concurrent.ForkJoinPool.common.parallelism` . This default pool is used internally by other classes of the Concurrency API.

### Creating a fork/join pool

Creating a `ForkJoinPool` object to execute the tasks

Creating a subclass of `ForkJoinTask` to be executed in the pool

main characteristics

- using the default constructor
- use the structure recommended by the Java API documentation
- execute the tasks in a synchronized way
- The tasks you're going to implement won't return any result,

example

```java
public class Product {
    private String name;
    private double price;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }
}
```

```java
public class ProductListGenerator {

    public List<Product> generate(int size) {
        List<Product> ret = new ArrayList<>();

        for (int i = 0; i < size; i++) {
            Product product = new Product();
            product.setName("Product " + i);
            product.setPrice(10);

            ret.add(product);
        }
        return ret;
    }
}
```

```java
public class Task extends RecursiveAction {
    private List<Product> products;

    private int first;
    private int last;
    private double increment;

    public Task(List<Product> products, int first, int last, double increment) {
        this.products = products;
        this.first = first;
        this.last = last;
        this.increment = increment;
    }

    @Override
    protected void compute() {
        if (last - first < 10) {
            updatePrices();
        } else {
            int middle = (last + first) / 2;
            System.out.printf("Task: Pending tasks: %s\n", getQueuedTaskCount());
            Task task1 = new Task(products, first, middle + 1, increment);
            Task task2 = new Task(products, middle + 1, last, increment);
            invokeAll(task1, task2);
        }
    }

    private void updatePrices() {
        for (int i = first; i < last; i++) {
            Product product = products.get(i);
            product.setPrice(product.getPrice() * (1 + increment));
        }
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        ProductListGenerator generator = new ProductListGenerator();
        List<Product> products = generator.generate(10000);
        Task task = new Task(products, 0, products.size(), 0.20);

        ForkJoinPool forkJoinPool = new ForkJoinPool();

        forkJoinPool.execute(task);

        do {
            System.out.printf("Main: Thread Count: %d\n", forkJoinPool.getActiveThreadCount());
            System.out.printf("Main: Thread Steal: %d\n", forkJoinPool.getStealCount());
            System.out.printf("Main: Parallelism: %d\n", forkJoinPool.getParallelism());

            try {
                TimeUnit.MILLISECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } while (!task.isDone());

        forkJoinPool.shutdown();

        if (task.isCompletedNormally()) {
            System.out.print("Main: The process has completed normally.\n");
        }

        for (int i = 0; i < products.size(); i++) {
            Product product = products.get(i);
            if (product.getPrice() != 12) {
                System.out.printf("Product %s: %f\n", product.getName(), product.getPrice());
            }
        }

        System.out.printf("Main: End of the program\n");

    }
}
```

`java.util.concurrent.ForkJoinTask#invokeAll(java.util.concurrent.ForkJoinTask<?>, java.util.concurrent.ForkJoinTask<?>)`

This is a synchronous call, and the task waits for the finalization of the subtasks before continuing (potentially finishing) its execution.

`java.util.concurrent.ForkJoinPool#execute(java.lang.Runnable)`

`java.util.concurrent.ForkJoinPool#invoke`

Although the `ForkJoinPool` class is designed to execute an object of `ForkJoinTask` , you can also execute the `Runnable` and `Callable` objects directly

`java.util.concurrent.ForkJoinTask#adapt(java.util.concurrent.Callable<? extends T>)`

### Joining the results of the tasks

The fork/join framework provides the ability to execute tasks that return a result.

`java.util.concurrent.RecursiveTask`

example

```java
public class DocumentMock {
    private String[] words = {"the", "hello", "goodbye", "packt",
            "java", "thread", "pool", "random", "class", "main"};

    public String[][] generateDocument(int numLines, int numWords, String word) {
        int counter = 0;
        String[][] document = new String[numLines][numWords];
        Random random = new Random();
        for (int i = 0; i < numLines; i++) {
            for (int j = 0; j < numWords; j++) {
                int index = random.nextInt(words.length);
                document[i][j] = words[index];
                if (document[i][j].equals(word)) {
                    counter++;
                }
            }
        }
        System.out.printf("DocumentMock: The word appears %d time in the document\n", counter);

        return document;
    }
}
```

```java
public class LineTask extends RecursiveTask<Integer> {

    private String line[];
    private int start, end;
    private String word;

    public LineTask(String[] line, int start, int end, String word) {
        this.line = line;
        this.start = start;
        this.end = end;
        this.word = word;
    }

    @Override
    protected Integer compute() {
        Integer result = null;
        if (end - start < 100) {
            result = count(line, start, end, word);
        } else {
            int mid = (start + end) / 2;
            LineTask task1 = new LineTask(line, start, mid, word);
            LineTask task2 = new LineTask(line, mid, end, word);

            invokeAll(task1, task2);

            try {
                result = groupResult(task1.get(), task2.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        return result;
    }

    private Integer groupResult(Integer num1, Integer num2) {
        return num1 + num2;
    }

    private Integer count(String[] line, int start, int end, String word) {
        int counter = 0;
        for (int i = start; i < end; i++) {
            if (line[i].equals(word)) {
                counter++;
            }
        }

        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return counter;
    }
}
```

```java
public class DocumentTask extends RecursiveTask<Integer> {

    private String[][] document;
    private int start, end;
    private String word;

    public DocumentTask(String[][] document, int start, int end, String word) {
        this.document = document;
        this.start = start;
        this.end = end;
        this.word = word;
    }

    @Override
    protected Integer compute() {
        Integer result = null;
        if (end - start < 10) {
            result = processLines(document, start, end, word);
        } else {
            int mid = (start + end) / 2;
            DocumentTask task1 = new DocumentTask(document, start, mid, word);
            DocumentTask task2 = new DocumentTask(document, mid, end, word);

            invokeAll(task1, task2);

            try {
                result = groupResults(task1.get(), task2.get());
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }
        return result;
    }

    private Integer groupResults(Integer num1, Integer num2) {
        return num1 + num2;
    }

    private Integer processLines(String[][] document, int start, int end, String word) {
        List<LineTask> tasks = new ArrayList<>();
        for (int i = start; i < end; i++) {
            LineTask task = new LineTask(document[i], 0, document[i].length, word);

            tasks.add(task);
        }

        invokeAll(tasks);

        int result = 0;
        for(int i = 0; i<tasks.size();i++){
            LineTask task = tasks.get(i);

            try {
                result = result+task.get();
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        return result;
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        DocumentMock mock = new DocumentMock();
        String[][] document = mock.generateDocument(100,1000,"the");

        DocumentTask task = new DocumentTask(document,0,100,"the");

        ForkJoinPool pool = new ForkJoinPool();
        pool.execute(task);

        do{
            System.out.print("****************************************\n");
            System.out.printf("Main: Active Threads: %d\n",pool.getActiveThreadCount());
            System.out.printf("Main: Task Count: %d\n",pool.getQueuedTaskCount());
            System.out.printf("Main: Steal Count: %d\n",pool.getStealCount());
            System.out.print("****************************************\n");

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }while(!task.isDone());


        pool.shutdown();

        try {
            pool.awaitTermination(1,TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }


        try {
            System.out.printf("Main: The word appears %d in the document\n",task.get());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

provides another method to finish the execution of a task and return a result

`java.util.concurrent.ForkJoinTask#complete`

`java.util.concurrent.ForkJoinTask#join`

### Running tasks asynchronously

When you execute `ForkJoinTask` in `ForkJoinPool` , you can do it in a **synchronous** or an **asynchronous** way

**synchronous** the method that sends the task to the pool doesn't return until the task sent finishes its execution.

**asynchronous** the method that sends the task to the executor returns immediately, so the task can continue with its execution

`java.util.concurrent.CountedCompleter`

`java.util.concurrent.CountedCompleter#onCompletion`

example

```java
public class FolderProcessor extends CountedCompleter<List<String>> {
    private String path;
    private String extension;
    private List<FolderProcessor> tasks;
    private List<String> resultList;

    protected FolderProcessor(CountedCompleter<?> completer, String path, String extension) {
        super(completer);
        this.path = path;
        this.extension = extension;
    }

    public FolderProcessor(String path, String extension) {
        this.path = path;
        this.extension = extension;
    }

    @Override
    public void compute() {
        resultList = new ArrayList<>();
        tasks = new ArrayList<>();
        File file = new File(path);
        File[] content = file.listFiles();

        if (null != content) {
            for (int i = 0; i < content.length; i++) {
                if (content[i].isDirectory()) {
                    FolderProcessor task = new FolderProcessor(this,
                            content[i].getAbsolutePath(), extension);

                    task.fork();
                    addToPendingCount(1);
                    tasks.add(task);
                } else {
                    if (checkFile(content[i].getName())) {
                        resultList.add(content[i].getAbsolutePath());
                    }
                }
            }

            if (tasks.size() > 50) {
                System.out.printf("%s: %d tasks ran.\n", file.getAbsolutePath(), tasks.size());
            }
        }

        tryComplete();

    }

    private boolean checkFile(String name) {
        return name.endsWith(extension);
    }

    @Override
    public void onCompletion(CountedCompleter<?> caller) {
        for (FolderProcessor childTask : tasks) {
            resultList.addAll(childTask.getResultList());
        }
    }

    public List<String> getResultList() {
        return resultList;
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();

        FolderProcessor system = new FolderProcessor("c:/Windows", "log");
        FolderProcessor apps = new FolderProcessor("c:/Program Files", "log");
        FolderProcessor documents = new FolderProcessor("c:/Documents And Settings", "log");

        pool.execute(system);
        pool.execute(apps);
        pool.execute(documents);

        do {
            System.out.print("***************************************\n");
            System.out.printf("Main: Active Threads: %d\n", pool.getActiveThreadCount());
            System.out.printf("Main: Task Count: %d\n", pool.getQueuedTaskCount());
            System.out.printf("Main: Steal Count: %d\n", pool.getStealCount());
            System.out.print("***************************************\n");

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } while (!system.isDone() || !apps.isDone() || !documents.isDone());

        pool.shutdown();

        List<String> results;

        results = system.getResultList();
        System.out.printf("System: %d files found.\n", results.size());

        results = apps.getResultList();
        System.out.printf("Apps: %d files found.\n", results.size());

        results = documents.getResultList();
        System.out.printf("Documents: %d files found.\n", results.size());
    }
}
```

`java.util.concurrent.ForkJoinTask#fork`

`java.util.concurrent.CountedCompleter#addToPendingCount`

`java.util.concurrent.CountedCompleter#tryComplete`

`java.util.concurrent.CountedCompleter#onCompletion`

Other Method

`java.util.concurrent.CountedCompleter#setPendingCount`

`java.util.concurrent.CountedCompleter#compareAndSetPendingCount`

`java.util.concurrent.CountedCompleter#decrementPendingCountUnlessZero`

`java.util.concurrent.CountedCompleter#complete`

`java.util.concurrent.CountedCompleter#onExceptionalCompletion`

### Throwing exceptions in the tasks

**Checked Exception**  These exceptions must be specified in the throws clause of a method or caught inside them.

**Unchecked Exception**  These exceptions don't have to be specified or caught.

exception

```java
public class ExceptionTask extends RecursiveTask<Integer> {

    private int[] array;
    private int start, end;

    public ExceptionTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        System.out.printf("Task: Start form %d to %d\n", start, end);
        if (end - start < 10) {
            if (3 > start && 3 < end) {
                throw new RuntimeException("This task throws an Exception: task from " + start + " to " + end);
            }

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } else {
            int mid = (start + end) / 2;
            ExceptionTask task1 = new ExceptionTask(array, start, mid);
            ExceptionTask task2 = new ExceptionTask(array, mid, end);

            invokeAll(task1, task2);
            System.out.printf("Task: Result form %d to %d: %d\n", start, mid, task1.join());
            System.out.printf("Task: Result form %d to %d: %d\n", mid, end, task2.join());
        }

        System.out.printf("Task: End form %d to %d\n", start, end);
        return 0;
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        int[] array = new int[100];
        ExceptionTask task = new ExceptionTask(array, 0, 100);
        ForkJoinPool pool = new ForkJoinPool();

        pool.execute(task);

        pool.shutdown();

        try {
            pool.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (task.isCompletedAbnormally()) {
            System.out.print("Main: An exception has occurred\n");
            System.out.printf("Main: %s\n", task.getException());
        }

        System.out.printf("Main: Result: %d\n", task.join());

    }
}
```

`java.util.concurrent.ForkJoinTask#completeExceptionally`

### Canceling a task 

`java.util.concurrent.ForkJoinTask#cancel`

 some points you have to take into account when you want to cancel a task

- The `ForkJoinPool` class doesn't provide any method **to cancel all the tasks** it has running or waiting in the pool
- When you cancel a task, you don't cancel the tasks this **task has executed**

example

```java
public class ArrayGenerator {
    public int[] generateArray(int size) {
        int[] array = new int[size];
        Random random = new Random();
        for (int i = 0; i < size; i++) {
            array[i] = random.nextInt(10);
        }
        return array;
    }
}
```

```java
public class TaskManager {

    private final ConcurrentLinkedDeque<SearchNumberTask> tasks;
    public TaskManager(){
        tasks = new ConcurrentLinkedDeque<>();
    }

    public void addTask(SearchNumberTask task){
        tasks.add(task);
    }

    public void cancelTasks(SearchNumberTask cancelTask){
        for(SearchNumberTask task: tasks){
            if(task != cancelTask){
                task.cancel(true);
                task.logCancelMessage();
            }
        }
    }
}
```

```java
public class SearchNumberTask extends RecursiveTask<Integer> {

    private int[] numbers;
    private int start, end;
    private int number;
    private TaskManager manager;
    private static final int NOT_FOUND = -1;

    public SearchNumberTask(int[] numbers, int start, int end, int number, TaskManager manager) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
        this.number = number;
        this.manager = manager;
    }

    @Override
    protected Integer compute() {
        System.out.printf("Task: %d: %d\n", start, end);
        int ret;
        if (end - start > 10) {
            ret = launchTasks();
        } else {
            ret = lookForNumber();
        }
        return ret;
    }

    private int launchTasks() {
        int mid = (start + end) / 2;
        SearchNumberTask task1 = new SearchNumberTask(numbers, start, mid, number, manager);
        SearchNumberTask task2 = new SearchNumberTask(numbers, mid, end, number, manager);

        manager.addTask(task1);
        manager.addTask(task2);
        task1.fork();
        task2.fork();

        int returnValue;
        returnValue = task1.join();
        if (returnValue != -1) {
            return returnValue;
        }
        returnValue = task2.join();
        return returnValue;
    }

    private int lookForNumber() {
        for (int i = start; i < end; i++) {
            if (numbers[i] == number) {
                System.out.printf("Task: Number %d found in position %d\n", number, i);
                manager.cancelTasks(this);
                return i;
            }
        }

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return NOT_FOUND;
    }

    public void logCancelMessage() {
        System.out.printf("Task: Canceled Task form %d to %d\n", start, end);
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) {
        ArrayGenerator generator = new ArrayGenerator();
        int[] array = generator.generateArray(1000);

        TaskManager manager = new TaskManager();
        ForkJoinPool pool = new ForkJoinPool();
        SearchNumberTask task = new SearchNumberTask(array, 0, 1000, 5, manager);

        pool.execute(task);

        pool.shutdown();

        try {
            pool.awaitTermination(1, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: The program has finished\n");
    }
}
```

## Parallel And Reactive Streams

> A stream in Java is **a sequence of elements** that can be processed (mapped, filtered, transformed,reduced, and collected) in a **pipeline** of declarative operations using **lambda expressions** in a **sequential or parallel**  way.
>
> characteristics of streams
>
> - A stream is a sequence of data, not a data structure. Elements of data are processed by the stream but not stored in it
> - create streams from different sources
> - can't access an individual element of the streams
> - You can't modify the source of the stream
> - two kinds of operations
>   - Intermediate operations
>   - Terminal operations
> - A stream pipeline is formed by zero or more intermediate operations and a final operation
> - Intermediate operations
>   - **Stateless** Processing an element of the stream is independent of the other elements
>   - **Stateful** Processing an element of the stream depends on the other elements of the stream
> - **Laziness**  Intermediate operations are lazy. They're not executed until the terminal operation begins its execution.
> - `Stream` can have an **infinite number of elements**
> - Streams can only be used once.
> - process the elements of a stream sequentially or in a parallel way without any extra effort
>
> Java 9 has included a new kind of streams-the reactive streams-that allow you to communicate information to producers and consumers in an asynchronous way. 

### Create streams from different sources

different options

- The `parallelStream()` method of the `Collection` interface
- The `Supplier` interface
- A predefined set of elements
- `File` and a directory
- An array
- A random number generator
- The concatenation of two different streams

example

```java
public class MySupplier implements Supplier<String> {

    private final AtomicInteger counter;

    public MySupplier() {
        this.counter = new AtomicInteger(0);
    }

    @Override
    public String get() {
        int value = counter.getAndAdd(1);
        return "String " + value;
    }
}
```

```java
public class Person implements Comparable<Person> {
    private int id;
    private String firstName;
    private String lastName;
    private Date birthday;
    private int salary;
    private double coefficient;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public Date getBirthday() {
        return birthday;
    }

    public void setBirthday(Date birthday) {
        this.birthday = birthday;
    }

    public int getSalary() {
        return salary;
    }

    public void setSalary(int salary) {
        this.salary = salary;
    }

    public double getCoefficient() {
        return coefficient;
    }

    public void setCoefficient(double coefficient) {
        this.coefficient = coefficient;
    }

    @Override
    public int compareTo(Person o) {
        int compareLastName = this.getLastName().compareTo(o.getLastName());
        if(compareLastName !=0){
            return compareLastName;
        }else{
            return this.getFirstName().compareTo(o.getFirstName());
        }
    }
}
```

```java
public class PersonGenerator {

    public static List<Person> generatePersonList(int size) {
        List<Person> ret = new ArrayList<>();
        String[] firstNames = {"Mary", "Patricia", "Linda", "Barbara", "Elizabeth",
                               "James", "John", "Robert", "Michael", "William"};
        String[] lastName = {"Smith", "Jones", "Taylor", "Williams", "Brown",
                             "Davies", "Evans", "Wilson", "Thomas", "Roberts"};

        Random random = new Random();
        for (int i = 0; i < size; i++) {
            Person person = new Person();
            person.setId(i);
            person.setFirstName(firstNames[random.nextInt(10)]);
            person.setLastName(lastName[random.nextInt(10)]);
            person.setSalary(random.nextInt(100000));
            person.setCoefficient(random.nextDouble() * 10);

            Calendar calendar = Calendar.getInstance();
            calendar.add(Calendar.YEAR, -random.nextInt(30));
            Date birthDay = calendar.getTime();
            person.setBirthday(birthDay);
            ret.add(person);
        }
        return ret;
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        System.out.print("From a Collection: \n");
        List<Person> persons = PersonGenerator.generatePersonList(10000);
        Stream<Person> personStream = persons.parallelStream();
        System.out.printf("Number of persons: %d\n", personStream.count());

        System.out.print("From a Supplier:\n");
        Supplier<String> supplier = new MySupplier();
        Stream<String> generatorStream = Stream.generate(supplier);
        generatorStream.parallel().limit(10)
            .forEach(s -> System.out.printf("%s\n", s));

        System.out.print("From a predefined set of elements:\n");
        Stream<String> elementStream = Stream.of("Peter", "John", "Mary");
        elementStream.parallel()
            .forEach(element -> System.out.printf("%s\n", element));


        System.out.print("From a File:\n");
        try (BufferedReader br = new BufferedReader(new FileReader("D:\\IdeaProjects\\concurrency-cook-book\\src\\main\\resources\\data\\nursery.data"))) {
            Stream<String> fileLines = br.lines();
            System.out.printf("Number of lines in the file: %d\n\n", fileLines.parallel().count());
            System.out.print("********************************************************\n\n");
        } catch (IOException e) {
            e.printStackTrace();
        }

        System.out.print("From a Directory:\n");
        try {
            Stream<Path> directoryContent = Files.list(Paths.get(System.getProperty("user.home")));
            System.out.printf("Number of elements (files and folders): %d\n\n", directoryContent.parallel().count());
            System.out.print("********************************************************\n\n");
        } catch (IOException e) {
            e.printStackTrace();
        }


        System.out.print("From an Array:\n");
        String[] array = {"1", "2", "3", "4", "5", "6"};
        Stream<String> streamFromArray = Arrays.stream(array);
        streamFromArray.parallel().forEach(s -> System.out.printf("%s : ", s));

        Random random = new Random();
        DoubleStream doubleStream = random.doubles(10);
        double doubleStreamAverage = doubleStream.parallel()
            .peek(d -> System.out.printf("%f: ", d))
            .average()
            .getAsDouble();
        System.out.printf("\nDouble Stream Average: %f\n", doubleStreamAverage);
        System.out.print("********************************************************\n\n");

        System.out.print("Concatenation streams:\n");
        Stream<String> stream1 = Stream.of("1", "2", "3", "4");
        Stream<String> stream2 = Stream.of("5", "6", "7", "8");
        Stream<String> finalStream = Stream.concat(stream1, stream2);
        finalStream.parallel().forEach(s -> System.out.printf("%s : ", s));

    }
}
```



### Reducing the elements of a stream

**MapReduce** is a programming model used to process very large datasets in distributed environments using a lot of machines working in a cluster.

- **Map** This operation filters and transforms the original elements into a form more suitable to the reduction operation
- **Reduce** This operation generates a summary result from all the elements

The `Stream` class implements two different reduce operations

- `reduce()` process a stream of elements to obtain a value
- `collect()` process a stream of elements to generate a mutable data structure

example

```java
public class DoubleGenerator {
    public static List<Double> generateDoubleList(int size, int max) {
        Random random = new Random();
        List<Double> numbers = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            double value = random.nextDouble() * max;
            numbers.add(value);
        }
        return numbers;
    }

    public static DoubleStream generateStreamFromList(List<Double> list){
        DoubleStream.Builder builder = DoubleStream.builder();
        for(Double number:list){
            builder.add(number);
        }

        return builder.build();
    }
}
```

```java
public class Point {
    private Double x;
    private Double y;

    public Double getX() {
        return x;
    }

    public void setX(Double x) {
        this.x = x;
    }

    public Double getY() {
        return y;
    }

    public void setY(Double y) {
        this.y = y;
    }
}
```

```java
public class PointGenerator {
    public static List<Point> generatePointList(int size) {
        List<Point> ret = new ArrayList<>();
        Random random = new Random();
        for (int i = 0; i < size; i++) {
            Point point = new Point();
            point.setX(random.nextDouble());
            point.setY(random.nextDouble());

            ret.add(point);
        }
        return ret;
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        List<Double> numbers = DoubleGenerator.generateDoubleList(10000, 1000);

        DoubleStream doubleStream = DoubleGenerator.generateStreamFromList(numbers);
        long numberOfElements = doubleStream.parallel().count();
        System.out.printf("The list of number has %d elements\n", numberOfElements);

        doubleStream = DoubleGenerator.generateStreamFromList(numbers);
        double sum = doubleStream.parallel().sum();
        System.out.printf("Its numbers sum %f\n", sum);

        doubleStream = DoubleGenerator.generateStreamFromList(numbers);
        double average = doubleStream.parallel().average().getAsDouble();
        System.out.printf("Its numbers have an average value of %f\n", average);

        doubleStream = DoubleGenerator.generateStreamFromList(numbers);
        double max = doubleStream.parallel().max().getAsDouble();
        System.out.printf("The maximum value in the list is %f\n", max);

        doubleStream = DoubleGenerator.generateStreamFromList(numbers);
        double min = doubleStream.parallel().min().getAsDouble();
        System.out.printf("The minimum value in the list is %f\n", min);


        List<Point> points = PointGenerator.generatePointList(10000);
        Optional<Point> point = points.parallelStream().reduce(((point1, point2) -> {
            Point p = new Point();
            p.setX(point1.getX() + point2.getX());
            p.setY(point1.getY() + point2.getY());
            return p;
        }));
        System.out.printf("(%f,%f)\n", point.get().getX(), point.get().getY());

        System.out.print("Reduce, second version\n");
        List<Person> persons = PersonGenerator.generatePersonList(10000);
        long totalSalary = persons.parallelStream()
                .map(Person::getSalary)
                .reduce(0, (s1, s2) -> s1 + s2);
        System.out.printf("Total salary: %d\n", totalSalary);

        Integer value = 0;
        value = persons.parallelStream().reduce(value, (n, p) -> {
            if (p.getSalary() > 50000) {
                return n + 1;
            } else {
                return n;
            }
        }, (n1, n2) -> n1 + n2);

        System.out.printf("The number of people with a salary bigger that 50,000 is %d\n", value);
    }
}
```

`java.util.stream.Stream#reduce(java.util.function.BinaryOperator<T>)`

`java.util.stream.Stream#reduce(T, java.util.function.BinaryOperator<T>)`

`java.util.stream.Stream#reduce(U, java.util.function.BiFunction<U,? super T,U>, java.util.function.BinaryOperator<U>)`

### Collecting the element of a stream

**Intermediate operations**  return other `Stream` as a result

**Terminal operations** return a result

The reduce operation

The collect operation

example

```java
public class Counter {
    private String value;
    private int counter;

    public Counter() {
        counter = 1;
    }

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }

    public int getCounter() {
        return counter;
    }

    public void setCounter(int counter) {
        this.counter = counter;
    }

    public void increment() {
        counter++;
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        List<Person> persons = PersonGenerator.generatePersonList(100);

        Map<String, List<Person>> personsByName = persons.parallelStream()
            .collect(Collectors.groupingBy(Person::getFirstName));

        personsByName.keySet().forEach(key -> {
            List<Person> listOfPersons = personsByName.get(key);
            System.out.printf("%s: There are %d persons with the name\n", key, listOfPersons.size());
        });

        String message = persons.parallelStream()
            .map(Person::toString)
            .collect(Collectors.joining(","));
        System.out.printf("%s\n", message);

        Map<Boolean, List<Person>> personsBySalary = persons.parallelStream()
            .collect(Collectors.partitioningBy(p -> p.getSalary() > 50000));
        personsBySalary.keySet().forEach(key -> {
            List<Person> listOfPersons = personsBySalary.get(key);
            System.out.printf("%s: %d\n", key, listOfPersons.size());
        });

        ConcurrentMap<String, String> nameMap = persons.parallelStream()
            .collect(Collectors.toConcurrentMap(
                Person::getFirstName, Person::getLastName,
                (s1, s2) -> s1 + ", " + s2));

        nameMap.forEach((key, value) -> System.out.printf("%s: %s\n", key, value));

        List<Person> highSalaryPeople = persons.parallelStream()
            .collect(ArrayList::new, (list, person) -> {
                if (person.getSalary() > 50000) {
                    list.add(person);
                }
            }, ArrayList::addAll);

        System.out.printf("High Salary People: %d\n", highSalaryPeople.size());


        System.out.print("Collect, Second example\n");
        ConcurrentMap<String, Counter> peopleNames = persons.parallelStream()
            .collect(ConcurrentHashMap::new, (hash, person) -> {
                hash.computeIfPresent(person.getFirstName(), (name, counter) -> {
                    counter.increment();
                    return counter;
                });

                hash.computeIfAbsent(person.getFirstName(), name -> {
                    Counter c = new Counter();
                    c.setValue(name);
                    return c;
                });
            }, (hash1, hash2) -> {
                hash2.forEach(10, (key, value) -> {
                    hash1.merge(key, value, (v1, v2) -> {
                        v1.setCounter(v1.getCounter() + v2.getCounter());
                        return v1;
                    });
                });
            });


        peopleNames.forEach((name, counter) -> System.out.printf("%s: %d\n", name, counter.getCounter()));

    }
}
```

`java.util.stream.Stream#collect(java.util.stream.Collector<? super T,A,R>)`

use the utility class  `java.util.stream.Collectors`

- `groupingByConcurrent()`
- `joining()`
- `partitioningBy()`
- `toConcurrentMap()`

`java.util.stream.Collector`

the most important is the CONCURRENT one that indicates if the collector can work in a concurrent way or not

We will only have a concurrent reduction if the next three conditions are true:

- The `Stream` is paralle
- The collector has the **CONCURRENT** characteristic
- Either the stream is unordered, or the collector has the **UNORDERED** characteristic

`java.util.stream.Stream#collect(java.util.function.Supplier<R>, java.util.function.BiConsumer<R,? super T>, java.util.function.BiConsumer<R,R>)`

- A supplier function 

  generates a data structure of the type of the final result of the collect operation. With parallel streams, this function will be called as many times as there are threads executing the operation

- An accumulator function 

  receives a data structure and an element of the stream and makes the process of the element

- A combiner function

  receives two data structures and generates a unique data structure of the same type

### Applying an action to every element of a stream

terminal operations

`forEach()`

`forEachOrdered()`

an intermediate operation

`peek()`

example

```java
public class Main04 {
    public static void main(String[] args) {
        List<Person> persons = PersonGenerator.generatePersonList(10);

        persons.parallelStream().forEach(p -> System.out.printf("%s: %s\n", p.getLastName(), p.getFirstName()));


        List<Double> doubles = DoubleGenerator.generateDoubleList(10, 100);
        System.out.print("Parallel forEachOrdered() with numbers\n");
        doubles.parallelStream().sorted().forEachOrdered(n -> System.out.printf("%f\n", n));

        System.out.print("Parallel forEach() after sorted() with numbers\n");
        doubles.parallelStream().sorted().forEach(n -> System.out.printf("%f\n", n));

        persons.parallelStream().sorted().forEachOrdered(p ->
                System.out.printf("%s,%s\n", p.getLastName(), p.getFirstName()));


        doubles.parallelStream()
                .peek(d -> System.out.printf("Step1: Number:%f\n",d))
                .peek(d -> System.out.printf("Step2: Number:%f\n",d))
                .forEach(d -> System.out.printf("Final Step: Number:%f\n",d));
    }
}
```



### Filtering the element of a stream

example

```java
public class Main05 {
    public static void main(String[] args) {
        List<Person> persons = PersonGenerator.generatePersonList(10);
        persons.parallelStream()
            .forEach(p -> System.out.printf("%s, %s\n", p.getLastName(), p.getFirstName()));

        persons.parallelStream()
            .distinct()
            .forEach(p -> System.out.printf("%s, %s\n", p.getLastName(), p.getFirstName()));

        Integer[] numbers = {1, 3, 2, 1, 2, 2, 1, 1, 3, 3, 1, 1, 2, 2, 1};
        Arrays.asList(numbers).parallelStream()
            .mapToInt(n -> n)
            .distinct()
            .forEach(n -> System.out.printf("Number: %d\n", n));

        persons.parallelStream()
            .filter(p -> p.getSalary() < 30000)
            .forEach(p -> System.out.printf("%s, %s\n", p.getLastName(), p.getFirstName()));

        Arrays.asList(numbers).parallelStream()
            .mapToInt(n -> n)
            .filter(n -> n < 2)
            .forEach(n -> System.out.printf("%d\n", n));

        persons.parallelStream()
            .mapToDouble(Person::getSalary)
            .sorted()
            .limit(5)
            .forEachOrdered(s ->System.out.printf("Limit: %f\n",s));

        persons.parallelStream()
            .mapToDouble(Person::getSalary)
            .sorted()
            .skip(5)
            .forEachOrdered(s -> System.out.printf("Skip: %f\n",s));
    }
}
```

`distinct()` This method returns a stream with the distinct elements of the current stream according to the `equals()` method of the elements of the `Stream` class

`filter()`The `filter()` method returns a stream with the elements that make the `Predicate` true.

`limit()` This method receives an int value as a parameter and returns a stream with no more than as many number of elements

`skip()` This method returns a stream with the elements of the original stream after discarding the first elements. 

`java.util.stream.Stream#dropWhile`

With ordered streams, the method deletes the first elements that match the predicate from the stream.When it finds an element that doesn't match the predicate, it stops deleting the elements and returns the rest of the stream.

With unordered streams, its behavior is not deterministic.

`java.util.stream.Stream#takeWhile`

This method is equivalent to the previous one, but it takes the elements instead of deleting
them.

### Transforming the elements of a stream

Some of the most useful intermediate operations you can use with streams are those that allow you to transform the elements of the stream.

example

```java
public class FileGenerator {
    public static List<String> generateFile(int size){
        List<String> file = new ArrayList<>();
        for(int i = 0;i<size;i++){
            file.add("Lorem ipsum dolor sit amet,\n" +
                     "consectetur adipiscing elit. Morbi lobortis\n" +
                     "cursus venenatis. Mauris tempus elit ut\n" +
                     "malesuada luctus. Interdum et malesuada fames\n" +
                     "ac ante ipsum primis in faucibus. Phasellus\n" +
                     "laoreet sapien eu pulvinar rhoncus. Integer vel\n" +
                     "ultricies leo. Donec vel sagittis nibh.\n" +
                     "Maecenas eu quam non est hendrerit pu");
        }
        return file;
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        List<Person> persons = PersonGenerator.generatePersonList(100);
        DoubleStream ds = persons.parallelStream()
                .mapToDouble(Person::getSalary);
        long size = ds.distinct().count();
        System.out.printf("Size: %d\n", size);

        List<BasicPerson> basicPersons = persons.parallelStream().map(p -> {
            BasicPerson bp = new BasicPerson();
            bp.setName(p.getFirstName() + " " + p.getLastName());
            bp.setAge(getAge(p.getBirthday()));
            return bp;
        }).collect(Collectors.toList());

        basicPersons.forEach(bp -> System.out.printf("%s: %d\n", bp.getName(), bp.getAge()));

        List<String> file = FileGenerator.generateFile(100);
        Map<String, Long> wordCount = file.parallelStream()
                .flatMap(line -> Stream.of(line.split("[,.]")))
                .filter(w -> w.length() > 0)
                .sorted()
                .collect(Collectors.groupingByConcurrent(e -> e, Collectors.counting()));

        wordCount.forEach((k, v) -> System.out.printf("%s: %d\n", k, v));
    }

    private static long getAge(Date birthday) {
        LocalDate start = birthday.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
        LocalDate now = LocalDate.now();
        return ChronoUnit.YEARS.between(start, now);
    }
}
```

`mapToDouble()`

`map()` use this method when we have to convert the elements of `Stream` to a different class

`flatMap()` This method is useful in a more complex situation when you have to work with a `Stream` of `Stream` objects and you want to convert them to a unique Stream .automatically concatenate all those streams into a unique `Stream` 



### Sorting the elements of a stream

Another typical operation you will want to do with a `Stream` is sorting its elements.

example

```java
public class Main07 {
    public static void main(String[] args) {
        int[] numbers = {9, 8, 7, 6, 5, 4, 3, 2, 1, 0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
        Arrays.stream(numbers).parallel().sorted()
                .forEachOrdered(n -> System.out.printf("%d\n", n));

        List<Person> persons = PersonGenerator.generatePersonList(10);
        persons.parallelStream().sorted()
                .forEachOrdered(p -> System.out.printf("%s, %s\n", p.getLastName(), p.getFirstName()));

        TreeSet<Person> personSet = new TreeSet<>(persons);
        for (int i = 0; i < 10; i++) {
            Person person = personSet.stream().parallel().limit(1)
                    .collect(Collectors.toList()).get(0);
            System.out.printf("%s, %s\n", person.getFirstName(), person.getLastName());

            person = personSet.stream().unordered().parallel().limit(1)
                    .collect(Collectors.toList()).get(0);
            System.out.printf("%s, %s\n", person.getFirstName(), person.getLastName());
        }
    }
}
```

There are `Stream` objects that may have an encounter order depending on its source and the intermediate operations you have applied before.

This encounter order imposes a restriction about the order in which the elements must be processed by certain methods. 

 the main purpose of the `unordered()` method is to delete a constraint that limits the
performance of parallel streams.

### Verifying conditions in the elements of a stream

example

```java
public class Main08 {
    public static void main(String[] args) {
        List<Person> persons = PersonGenerator.generatePersonList(10);

        int maxSalary = persons.parallelStream().map(Person::getSalary)
                .max(Integer::compare).get();

        int minSalary = persons.parallelStream().map(Person::getSalary)
                .min(Integer::compare).get();

        System.out.printf("Salaries are between %d and %d\n", minSalary, maxSalary);

        boolean condition;
        condition = persons.parallelStream().allMatch(p -> p.getSalary() > 0);
        System.out.printf("Salary > 0: %b\n", condition);

        condition = persons.parallelStream().allMatch(p -> p.getSalary() > 10000);
        System.out.printf("Salary > 10000: %b\n", condition);

        condition = persons.parallelStream().allMatch(p -> p.getSalary() > 30000);
        System.out.printf("Salary > 30000: %b\n", condition);


        condition = persons.parallelStream().anyMatch(p -> p.getSalary() > 50000);
        System.out.printf("Any with salary > 50000: %b\n", condition);

        condition = persons.parallelStream().anyMatch(p -> p.getSalary() > 100000);
        System.out.printf("Any with salary > 100000: %b\n", condition);

        condition = persons.parallelStream().noneMatch(p -> p.getSalary() > 100000);
        System.out.printf("None with salary > 100000: %b\n", condition);

        Person person = persons.parallelStream().findAny().get();
        System.out.printf("Any: %s %s: %d\n", person.getFirstName(), person.getLastName(), person.getSalary());

        person = persons.parallelStream().findFirst().get();
        System.out.printf("First: %s %s: %d\n", person.getFirstName(),
                person.getLastName(), person.getSalary());

        person = persons.parallelStream().sorted(Comparator.comparingInt(Person::getSalary))
                .findFirst().get();

        System.out.printf("First Sorted: %s %s: %d\n", person.getFirstName(), person.getLastName(), person.getSalary());

    }
}
```



### Reactive programming with reactive streams

**Reactive streams**

provide **asynchronous** stream processing with **non-blocking back pressure**.

Reactive streams are based on the following three elements:

- A publisher of information
- One or more subscribers of that information
- A subscription between the publisher and a consumer

how these classes should interact among them

- The publisher will add the subscribers that want to be notified
- The subscriber receives a notification when they're added to a publisher
- The subscribers request one or more elements from the publisher in an asynchronous way, that is to say, the subscriber requests the element and continues with the execution
- When the publisher has an element to publish, it sends it to all its subscribers that have requested an element

`java.util.concurrent.Flow.Publisher`

`java.util.concurrent.Flow.Subscriber`

`java.util.concurrent.Flow.Subscription`

`java.util.concurrent.SubmissionPublisher`



example

```java
public class Consumer1 implements Flow.Subscriber<Item> {
    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        System.out.printf("%s: Consumer 1: Subscription received\n", Thread.currentThread().getName());
        System.out.printf("%s: Consumer 1: No Items requested\n", Thread.currentThread().getName());
    }

    @Override
    public void onNext(Item item) {
        System.out.printf("%s: Consumer 1: Item received\n", Thread.currentThread().getName());
        System.out.printf("%s: Consumer 1: %s\n", Thread.currentThread().getName(), item.getTitle());
        System.out.printf("%s: Consumer 1: %s\n", Thread.currentThread().getName(), item.getContent());
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.printf("%s: Consumer 1: Error\n", Thread.currentThread().getName());
        throwable.printStackTrace(System.err);
    }

    @Override
    public void onComplete() {
        System.out.printf("%s: Consumer 1: Completed\n", Thread.currentThread().getName());
    }
}
```

```java
public class Consumer2 implements Flow.Subscriber<Item> {
    private Flow.Subscription subscription;

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        System.out.printf("%s: Consumer 2: Subscription received\n", Thread.currentThread().getName());
        this.subscription = subscription;
        subscription.request(1);
    }

    @Override
    public void onNext(Item item) {
        System.out.printf("%s: Consumer 2: Item received\n", Thread.currentThread().getName());
        System.out.printf("%s: Consumer 2: %s\n", Thread.currentThread().getName(), item.getTitle());
        System.out.printf("%s: Consumer 2: %s\n", Thread.currentThread().getName(), item.getContent());
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.printf("%s: Consumer 2: Error\n", Thread.currentThread().getName());
        throwable.printStackTrace(System.err);
    }

    @Override
    public void onComplete() {
        System.out.printf("%s: Consumer 2: Completed\n", Thread.currentThread().getName());
    }
}
```

```java
public class Consumer3 implements Flow.Subscriber<Item> {
    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        System.out.printf("%s: Consumer 3: Subscription received\n", Thread.currentThread().getName());
        System.out.printf("%s: Consumer 3: Requested three items\n", Thread.currentThread().getName());
        subscription.request(3);
    }

    @Override
    public void onNext(Item item) {
        System.out.printf("%s: Consumer 3: Item received\n", Thread.currentThread().getName());
        System.out.printf("%s: Consumer 3: %s\n", Thread.currentThread().getName(), item.getTitle());
        System.out.printf("%s: Consumer 3: %s\n", Thread.currentThread().getName(), item.getContent());
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.printf("%s: Consumer 3: Error\n", Thread.currentThread().getName());
        throwable.printStackTrace(System.err);
    }

    @Override
    public void onComplete() {
        System.out.printf("%s: Consumer 3: Completed\n", Thread.currentThread().getName());
    }
}
```

```java
public class Main09 {
    public static void main(String[] args) {
        Consumer1 consumer1 = new Consumer1();
        Consumer2 consumer2 = new Consumer2();
        Consumer3 consumer3 = new Consumer3();

        SubmissionPublisher<Item> publisher = new SubmissionPublisher<>();
        publisher.subscribe(consumer1);
        publisher.subscribe(consumer2);
        publisher.subscribe(consumer3);

        for (int i = 0; i < 10; i++) {
            Item item = new Item();
            item.setTitle("Item "+ i);
            item.setContent("This is the item " +i);
            publisher.submit(item);

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        publisher.close();
    }
}
```

`java.util.concurrent.Flow.Publisher#subscribe`

This method receives a `Subscriber` object as parameter. The publisher should take this
subscriber into account when it publishes an Item.

`java.util.concurrent.Flow.Subscriber#onComplete`

This method will be called when the `Publisher` has finished its execution

`java.util.concurrent.Flow.Subscriber#onError`

This method will be called when there is an error that must be notified to the
subscribers

`java.util.concurrent.Flow.Subscriber#onNext`

This method will be called when the `Publisher` has a new element

`java.util.concurrent.Flow.Subscriber#onSubscribe`

This method will be called when the publisher has added the subscriber with the
`subscribe()` method

`java.util.concurrent.Flow.Subscription#request`

This method is used by the `Subscriber` to request an element from the publisher

Java API provides the `SubmissionPublisher` class that implements the Publisher interface and implements this behavior

`java.util.concurrent.Flow.Processor`

Its main purpose is to be an element between a publisher and a subscriber to transform the elements produced by the first one into a format that can be processed by the second one.

## Concurrent Collections

**Data structure** is a basic element of programming.Almost every program uses one or more types of data structure to store and manage data. The Java API provides the **Java Collections framework**.

When you need to work with data collections in a concurrent program, you must be very careful with the implementation you choose. Most collection classes do not work with concurrent applications because they can't control concurrent access to their data. 

Java provides two kinds of collections to use in concurrent applications:

- Blocking collections

  This kind of collection includes operations to add and remove data. If the operation can't be done immediately, because the collection is either full or empty, the thread that makes the call will **be blocked until the operation could be carried out**

- Non-blocking collections

  This kind of collection also includes operations to add and remove data. But in this case, if the operation can't be done immediately, **it returns a null value or throws an exception**; the thread that makes the call won't be blocked here



### Using non-blocking thread-safe deques

`java.util.concurrent.ConcurrentLinkedDeque`

`java.util.concurrent.ConcurrentLinkedQueue`

example

```java
public class AddTask implements Runnable {
    private final ConcurrentLinkedDeque<String> list;

    public AddTask(ConcurrentLinkedDeque<String> list) {
        this.list = list;
    }

    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        for (int i = 0; i < 10000; i++) {
            list.add(name + ": Element " + i);
        }
    }
}
```

```java
public class PollTask implements Runnable{
    private final ConcurrentLinkedDeque<String> list;

    public PollTask(ConcurrentLinkedDeque<String> list) {
        this.list = list;
    }

    @Override
    public void run() {
        for(int i = 0; i< 5000;i++){
            list.pollFirst();
            list.pollLast();
        }

    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        ConcurrentLinkedDeque<String> list = new ConcurrentLinkedDeque<>();

        Thread[] threads = new Thread[100];

        for (int i = 0; i < threads.length; i++) {
            AddTask task = new AddTask(list);
            threads[i] = new Thread(task);
            threads[i].start();
        }
        System.out.printf("Main: %d AddTask threads have been launched\n", threads.length);

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Main: Size of List: %d\n", list.size());

        for (int i = 0; i < threads.length; i++) {
            PollTask task = new PollTask(list);
            threads[i] = new Thread(task);
            threads[i].start();
        }
        System.out.printf("Main: %d PollTask threads have been launched\n", threads.length);

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Main: Size of the List: %d\n", list.size());
    }
}
```

`getFirst()` `getLast()` 

return the first and last element from the deque

don't remove the returned element from the deque

`peek()` `peekFirst()` `peekLast()`

return the first and the last element of the deque

don't remove the returned element from the deque

`remove()` `removeFirst()` `removeLast()`

return the first and last element of the deque

remove the returned element as well

### Using blocking thread-safe deques

example

```java
public class Client implements Runnable {

    private final LinkedBlockingDeque<String> requestList;

    public Client(LinkedBlockingDeque<String> requestList) {
        this.requestList = requestList;
    }

    @Override
    public void run() {
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 5; j++) {
                StringBuilder request = new StringBuilder();
                request.append(i);
                request.append(": ");
                request.append(j);
                try {
                    requestList.put(request.toString());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.print("Client: End\n");
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) throws InterruptedException {
        LinkedBlockingDeque<String> list = new LinkedBlockingDeque<>(3);
        Client client = new Client(list);
        Thread thread = new Thread(client);
        thread.start();

        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 3; j++) {
                String request = list.take();

                System.out.printf("Main: Removed: %s at %s. Size: %d\n", request, new Date(), list.size());
            }
            TimeUnit.MILLISECONDS.sleep(300);
        }

        System.out.print("Main: End of the program.\n");
    }
}
```

`taskFirst()` `takeLast()`

`getFirst()` `getLast()`

`peek()` `peekFirst()` `peekLast()`

`poll()` `pollFirst()` `pollLast()`

`add()` `addFirst()` `addLast()`



### Using blocking thread-safe queue ordered by priority

`java.util.concurrent.PriorityBlockingQueue`

All the elements you want to add to `PriorityBlockingQueue` have to implement the Comparable interface.

example

```java
public class Event implements Comparable<Event> {

    private final int thread;
    private final int priority;

    public Event(int thread, int priority) {
        this.thread = thread;
        this.priority = priority;
    }

    public int getThread() {
        return thread;
    }

    public int getPriority() {
        return priority;
    }

    @Override
    public int compareTo(Event o) {
        if (this.priority > o.getPriority()) {
            return -1;
        } else if (this.priority < o.getPriority()) {
            return 1;
        } else {
            return 0;
        }
    }
}
```

```java
public class Task  implements Runnable{
    private final int id;
    private final PriorityBlockingQueue<Event> queue;

    public Task(int id, PriorityBlockingQueue<Event> queue) {
        this.id = id;
        this.queue = queue;
    }

    @Override
    public void run() {
        for(int i = 0; i<1000; i++){
            Event event = new Event(id, i);
            queue.add(event);
        }
    }
}

```

```java
public class Main03 {
    public static void main(String[] args) {
        PriorityBlockingQueue<Event> queue = new PriorityBlockingQueue<>();
        Thread[] threads = new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            Task task = new Task(i, queue);

            threads[i] = new Thread(task);
        }

        for (int i = 0; i < threads.length; i++) {
            threads[i].start();
        }

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Main: Queue Size: %d\n", queue.size());
        for (int i = 0; i < threads.length * 1000; i++) {
            Event event = queue.poll();
            System.out.printf("Thread %d: Priority %d\n", event.getThread(), event.getPriority());

        }

        System.out.printf("Main: Queue Size: %d\n", queue.size());
        System.out.print("Main: End of the program\n");
    }
}
```

`clear()`

`take()`

`put()`

`peek()`

### Using thread-safe lists with delayed elements

`java.util.concurrent.DelayQueue`

The methods that return or extract elements from the queue will ignore these elements whose data will appear in the future

the elements you want to store in the `DelayQueue` class need to have the `Delayed` interface implemented

example

```java
public class DelayedEvent implements Delayed {
    private final Date startDate;

    public DelayedEvent(Date startDate) {
        this.startDate = startDate;
    }


    @Override
    public long getDelay(TimeUnit unit) {
        Date now = new Date();
        long diff = startDate.getTime() - now.getTime();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        long result = this.getDelay(TimeUnit.NANOSECONDS) - o.getDelay(TimeUnit.NANOSECONDS);
        if (result < 0) {
            return -1;
        } else if (result > 0) {
            return 1;
        } else {
            return 0;
        }
    }
}

```

```java
public class DelayedEventTask implements Runnable {
    private final int id;
    private final DelayQueue<DelayedEvent> queue;

    public DelayedEventTask(int id, DelayQueue<DelayedEvent> queue) {
        this.id = id;
        this.queue = queue;
    }

    @Override
    public void run() {
        Date now = new Date();
        Date delay = new Date();
        delay.setTime(now.getTime() + (id * 1000));
        System.out.printf("Thread %s: %s\n", id, delay);

        for (int i = 0; i < 100; i++) {
            DelayedEvent event = new DelayedEvent(delay);
            queue.add(event);
        }
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) throws InterruptedException {
        DelayQueue<DelayedEvent> queue = new DelayQueue<>();

        Thread[] threads = new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            DelayedEventTask task = new DelayedEventTask(i + 1, queue);
            threads[i] = new Thread(task);
        }

        for (int i = 0; i < threads.length; i++) {
            threads[i].start();
        }

        for (int i = 0; i < threads.length; i++) {
            threads[i].join();
        }

        do {
            int counter = 0;
            DelayedEvent event;
            do {
                event = queue.poll();
                if (event != null) {
                    counter++;
                }
            } while (event != null);
            System.out.printf("At %s you have read %d events\n", new Date(), counter);
            TimeUnit.MILLISECONDS.sleep(500);

        } while (queue.size() > 0);
    }
}
```

You must be very careful with the size() method. It returns the total number of elements in the list that includes both active and non-active elements.

### Using thread-safe navigable maps

`java.util.concurrent.ConcurrentNavigableMap`

->`java.util.concurrent.ConcurrentSkipListMap`

**Skip List**

A Skip List is a data structure based on parallel lists that allow us to get the kind of efficiency that is associated with a binary tree.

example

```java
public class Contact {

    private final String name;
    private final String phone;

    public Contact(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }

    public String getName() {
        return name;
    }

    public String getPhone() {
        return phone;
    }
}
```

```java
public class ContactTask implements Runnable {
    private final ConcurrentSkipListMap<String, Contact> map;
    private final String id;

    public ContactTask(ConcurrentSkipListMap<String, Contact> map, String id) {
        this.map = map;
        this.id = id;
    }

    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            Contact contact = new Contact(id, String.valueOf(i + 1000));
            map.put(id + contact.getPhone(), contact);
        }
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) {
        ConcurrentSkipListMap<String, Contact> map = new ConcurrentSkipListMap<>();
        Thread[] threads = new Thread[26];
        int counter = 0;
        for (char i = 'A'; i <= 'Z'; i++) {
            ContactTask task = new ContactTask(map, String.valueOf(i));
            threads[counter] = new Thread(task);
            threads[counter].start();
            counter++;
        }

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.printf("Main: Size of the map: %d\n", map.size());

        Map.Entry<String, Contact> element;
        Contact contact;

        element = map.firstEntry();
        contact = element.getValue();
        System.out.printf("Main: first entry: %s: %s\n", contact.getName(), contact.getPhone());


        element = map.lastEntry();
        contact = element.getValue();
        System.out.printf("Main: last entry: %s: %s\n", contact.getName(), contact.getPhone());


        System.out.print("Main: Submap form A1996 to B 1002:\n");
        ConcurrentNavigableMap<String, Contact> subMap = map.subMap("A1996", "B1002");

        do {

            element = subMap.pollFirstEntry();
            if (element != null) {
                contact = element.getValue();
                System.out.printf("%s: %s\n", contact.getName(), contact.getPhone());
            }

        } while (element != null);

    }
}
```

`headMap()`

`tailMap()`

`putIfAbsent()`

`pollLastEnrty()`

`pollLastEntry()`

`replace()`



### Using thread-safe HashMaps

A hash table is a data structure that allows you to map a key to a value. Internally, it usually uses an **array to store the elements** and a **hash function to calculate the position** of the element in the array, using its key.The main advantage of this data structure is that the **insert, delete, and search operations are very fast** here, so it's very useful in situations when you have to carry out **a lot of search operations**.

`java.util.concurrent.ConcurrentHashMap`

- Full concurrency of read operations
- High expected concurrency for insert and delete operations

example

```java
public class HashFiller implements Runnable {
    private ConcurrentHashMap<String, ConcurrentLinkedDeque<Operation>> userHash;

    public HashFiller(ConcurrentHashMap<String, ConcurrentLinkedDeque<Operation>> userHash) {
        this.userHash = userHash;
    }

    @Override
    public void run() {
        Random randomGenerator = new Random();
        for (int i = 0; i < 100; i++) {
            Operation operation = new Operation();
            String user = "USER" + randomGenerator.nextInt(100);
            operation.setUser(user);
            String action = "OP" + randomGenerator.nextInt(10);
            operation.setOperation(action);
            operation.setTime(new Date());

            addOperationToHash(userHash, operation);
        }
    }

    private void addOperationToHash(ConcurrentHashMap<String, ConcurrentLinkedDeque<Operation>> userHash,
                                   Operation operation) {
        ConcurrentLinkedDeque<Operation> opList = userHash.computeIfAbsent(operation.getUser(),
                user -> new ConcurrentLinkedDeque<>());
        opList.add(operation);
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        ConcurrentHashMap<String, ConcurrentLinkedDeque<Operation>>
                userHash = new ConcurrentHashMap<>();
        HashFiller hashFiller = new HashFiller(userHash);

        Thread[] threads = new Thread[10];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(hashFiller);
            threads[i].start();
        }

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Size: %d\n", userHash.size());
        userHash.forEach(10, (user, list) ->
                System.out.printf("%s: %s: %d\n", Thread.currentThread().getName(), user, list.size()));

        userHash.forEachEntry(10, entry ->
                System.out.printf("%s: %s: %d\n", Thread.currentThread().getName(), entry.getKey(), entry.getValue().size()));

        Operation op = userHash.search(10, (user, list) -> {
            for (Operation operation : list) {
                if (operation.getOperation().endsWith("1")) {
                    return operation;
                }
            }
            return null;
        });

        System.out.printf("The operation we have found is: %s, %s, %s,\n",
                op.getUser(), op.getOperation(), op.getTime());

        ConcurrentLinkedDeque<Operation> operations = userHash.search(10,
                (user, list) -> {
                    if (list.size() > 10) {
                        return list;
                    }
                    return null;
                });
        System.out.printf("The user we have found is: %s: %d operations\n", operations.getFirst().getUser(), operations.size());


        int totalSize = userHash.reduce(10,
                (user, list) -> list.size(), (n1, n2) -> n1 + n2);

        System.out.printf("The total size is: %d\n", totalSize);
    }
}
```



### using atomic variables

**Atomic variables** were introduced in Java version 5 to provide atomic operations on single variables.

When you work with a normal variable, each operation that you implement in Java is transformed into **several instructions of Java byte code that is understandable by the JVM** when you compile the program.

When a thread is doing an operation with an atomic variable and if other threads want to do an operation with the same variable, the implementation of the class includes a mechanism to check that the operation is done atomically.

**Compare and Set** 

Basically, the operation gets the value of the variable, changes the value to a local variable, and then tries to change the old value with the new one. three steps:

1. get the value of the variable, which is the old value of the variable.
2. change the value of the variable in a temporal variable, which is the new value of the variable.
3. substitute the old value with the new value if the old value is equal to the actual value of the variable. The old value may be different from the actual value if another thread changes the value of the variable.

example

```java
public class Account {

    private final AtomicLong balance;
    private final LongAdder operations;
    private final DoubleAccumulator commission;

    public Account() {
        this.balance = new AtomicLong();
        this.operations = new LongAdder();
        this.commission = new DoubleAccumulator((x, y) -> x + y * 0.2, 0);
    }

    public long getBalance() {
        return this.balance.get();
    }

    public void setBalance(long balance) {
        this.balance.set(balance);
        this.operations.reset();
        this.commission.reset();
    }

    public long getOperations() {
        return this.operations.longValue();
    }

    public void addAmount(long amount) {
        this.balance.getAndAdd(amount);
        this.operations.increment();
        this.commission.accumulate(amount);
    }

    public void subtractAmount(long amount) {
        this.balance.getAndAdd(-amount);
        this.operations.increment();
        this.commission.accumulate(amount);
    }

    public double getCommission() {
        return this.commission.get();
    }

}
```

```java
public class Bank implements Runnable {
    private final Account account;

    public Bank(Account account) {
        this.account = account;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            account.subtractAmount(1000);
        }
    }
}
```

```java
public class Company implements Runnable {
    private final Account account;

    public Company(Account account) {
        this.account = account;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            account.addAmount(1000);
        }
    }
}
```

```java
public class Main07 {
    public static void main(String[] args) {
        Account account = new Account();
        account.setBalance(1000);

        Company company = new Company(account);
        Thread companyThread = new Thread(company);

        Bank bank = new Bank(account);
        Thread bankThread = new Thread(bank);

        System.out.printf("Account: Initial Balance: %d\n", account.getBalance());

        companyThread.start();
        bankThread.start();

        try {
            companyThread.join();
            bankThread.join();
            System.out.printf("Account: Final Balance: %d\n",account.getBalance());
            System.out.printf("Account: Numbers of operations: %d\n",account.getOperations());
            System.out.printf("Account: Accumulated Commissions: %f\n",account.getCommission());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```
Account: Initial Balance: 1000
Account: Final Balance: 1000
Account: Numbers of operations: 200
Account: Accumulated Commissions: 40000.000000
```

### Using atomic arrays

synchronization mechanism, such as locks or the synchronized keyword

- Deadlock: This situation occurs when a thread is blocked waiting for a lock that is locked by other threads that will never free it. This situation blocks the program, so it will never finish.
- If only one thread is accessing the shared object, it has to execute the code necessary to get and release the lock.

**compare-and-swap operation**

This operation implements the modification of the value of a variable in the following three steps:

1. get the value of the variable, which is the old value of the variable.
2. change the value of the variable in a temporal variable, which is the new value of the variable.
3. substitute the old value with the new value if the old value is equal to the actual value of the variable. The old value may be different from the actual value if another thread has changed it.

avoid deadlocks and obtain better performance

drawbacks

Operations must be free from any side effects as they might be retried using livelocks with highly contended resources

they are also harder to monitor for performance when compared with standard locks

example

```java
public class Incrementer implements Runnable {
    private final AtomicIntegerArray vector;

    public Incrementer(AtomicIntegerArray vector) {
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i < vector.length(); i++) {
            vector.getAndIncrement(i);
        }
    }
}
```

```java
public class Decrementer implements Runnable {
    private final AtomicIntegerArray vector;

    public Decrementer(AtomicIntegerArray vector) {
        this.vector = vector;
    }

    @Override
    public void run() {
        for (int i = 0; i < vector.length(); i++) {
            vector.getAndDecrement(i);
        }
    }
}
```

```java
public class Main08 {
    public static void main(String[] args) {

        final int THREADS = 100;
        AtomicIntegerArray vector = new AtomicIntegerArray(1000);

        Incrementer incrementer = new Incrementer(vector);
        Decrementer decrementer = new Decrementer(vector);

        Thread[] incrementerThreads = new Thread[THREADS];
        Thread[] decrementerThreads = new Thread[THREADS];

        for (int i = 0; i < THREADS; i++) {
            incrementerThreads[i] = new Thread(incrementer);
            decrementerThreads[i] = new Thread(decrementer);

            incrementerThreads[i].start();
            decrementerThreads[i].start();
        }


        for (int i = 0; i < THREADS; i++) {
            try {
                incrementerThreads[i].join();
                decrementerThreads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        int errors = 0;
        for (int i = 0; i < vector.length(); i++) {
            if (vector.get(i) != 0) {
                System.out.printf("vector[%d]: %d\n", i, vector.get(i));
                errors++;
            }
        }
        if (errors == 0) {
            System.out.print("No errors found\n");
        }
    }
}
```

### Using the volatile keyword

Almost every application reads and writes data to the main memory of the computer. For performance reasons, these operations aren't performed directly in the memory. CPUs have a system of cache memory, so applications write data in the cache and then the data is moved from the cache to the main memory.

When a thread modifies a variable stored in the memory, the modification is made in the cache or the CPU or core where it's running. However, there's no guarantee about when that modification would reach the main memory. If another thread wants to read the value of the data, it's possible that it would not read the
modified value because it's not in the main memory of the computer.

`volatile`

It's a modifier that allows you to specify that a variable must always be read from and stored in the main memory, not the cache of your CPU.

You should use the volatile keyword when it's important that other threads have visibility of the actual value of the variable; however, **order of access to that variable is not important**. In this scenario, the volatile keyword will give you better performance because **it doesn't need to get any monitor or lock to access the variable**. On the contrary, if the order of access to the variable is important, you must use another synchronization mechanism.

example

```java
public class Flag {
    public boolean flag = true;
}
```

```java
public class FlagTask implements Runnable {
    private Flag flag;

    public FlagTask(Flag flag) {
        this.flag = flag;
    }

    @Override
    public void run() {
        int i = 0;
        while (flag.flag) {
            i++;
        }
        System.out.printf("Task: Stopped %d - %s\n", i, new Date());
    }
}
```

```java
public class VolatileFlag {
    public volatile boolean flag = true;
}
```

```java
public class VolatileFlagTask implements Runnable {
    private VolatileFlag flag;

    public VolatileFlagTask(VolatileFlag flag) {
        this.flag = flag;
    }

    @Override
    public void run() {
        int i = 0;
        while (flag.flag) {
            i++;
        }
        System.out.printf("Volatile: Stopped %d - %s\n", i, new Date());
    }
}
```

```java
public class Main09 {
    public static void main(String[] args) {
        VolatileFlag volatileFlag = new VolatileFlag();
        Flag flag = new Flag();

        VolatileFlagTask volatileTask = new VolatileFlagTask(volatileFlag);
        FlagTask task = new FlagTask(flag);

        Thread thread = new Thread(volatileTask);
        thread.start();
        thread = new Thread(task);
        thread.start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Going to stop volatile task: %s\n", new Date());
        volatileFlag.flag = false;
        System.out.printf("Main: Volatile task stopped: %s\n", new Date());

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Going to stop task: %s\n", new Date());
        flag.flag = false;
        System.out.printf("Main: Task flag changed: %s\n", new Date());

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

Take into account that the `volatile` keyword guarantees that modifications are written in the main memory,but its contrary is not always true.

The `volatile` keyword only works well when the value of the shared variable is only modified by one thread.

Since Java 5, **Java Memory Model** has a happens--before guarantee established with the `volatile`
keyword. This fact has two implications:

- When you modify a volatile variable, its value is sent to the main memory. The value of all the variables modified previously by the same thread are sent too.
- Compilers can't reorder sentences that modify a volatile variable for an optimization purpose. It can reorder the previous operations and the later ones, but not the modifications of a volatile variable. The changes that happen before these modifications will be visible to those instructions.

### Using variable handles

Variable handles are a new feature of Java 9 that allow you to get a typed reference to a variable (attribute, static field, or array element) in order to access it in different modes. 

example

```java
public class VariableHandleAccount {
    public double amount;
    public double unsafeAmount;

    public VariableHandleAccount() {
        this.amount = 0;
        this.unsafeAmount = 0;
    }
}
```

```java
public class VariableHandleIncrementer implements Runnable {
    private VariableHandleAccount account;

    public VariableHandleIncrementer(VariableHandleAccount account) {
        this.account = account;
    }

    @Override
    public void run() {
        VarHandle handler;
        try {
            handler = MethodHandles.lookup().in(VariableHandleAccount.class)
                    .findVarHandle(VariableHandleAccount.class, "amount", double.class);
            for (int i = 0; i < 10000; i++) {
                handler.getAndAdd(account, +100);
                account.unsafeAmount += 100;
            }
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class VariableHandleDecrementer implements Runnable {
    private VariableHandleAccount account;

    public VariableHandleDecrementer(VariableHandleAccount account) {
        this.account = account;
    }

    @Override
    public void run() {
        VarHandle handler;
        try {
            handler = MethodHandles.lookup().in(VariableHandleAccount.class)
                    .findVarHandle(VariableHandleAccount.class, "amount", double.class);
            for (int i = 0; i < 10000; i++) {
                handler.getAndAdd(account, -100);
                account.unsafeAmount -= 100;
            }
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Main10 {
    public static void main(String[] args) {
        VariableHandleAccount account = new VariableHandleAccount();
        Thread incrementerThread = new Thread(new VariableHandleIncrementer(account));
        Thread decrementerThread = new Thread(new VariableHandleDecrementer(account));

        incrementerThread.start();
        decrementerThread.start();

        try{
            incrementerThread.join();
            decrementerThread.join();
        }catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.printf("Safe amount: %f\n",account.amount);
        System.out.printf("UnSafe amount: %f\n",account.unsafeAmount);

    }
}
```

four different access types to a variable with a variable handle:

- Read mode

  - `get() ` Read the value of the variable as if it was declared non-volatile
  - `getVolatile()` Read the value of the variable as if it was declared volatile
  - `getAcquire()`  Read the value of the variable and guarantee that the following instructions that
    modify or access this variable are not reordered before the instructions for optimization purposes
  - `getOpaque()` Read the value of variable and guarantee that the instructions of the current thread
    are not reordered; no guarantee is provided for other threads

- Write mode

  - ` set()`
  - `setVolatile()`
  - `setRelease() `
  - `setOpaque() `

- Atomic access mode

  - `compareAndSet() ` Change the value of the variable as it was declared as a volatile variable if the
    expected value passed as parameter is equal to the current value of the variable

  - `weakCompareAndSet()`  `weakCompareAndSetVolatile() `  

    Possibly atomically' changes the value of the variable as it was declared as non-volatile or volatile variables respectively if the expected value passed as parameter is equals to the current value of the variable

- Numerical update access mode



## Customizing Concurrency Classes

> low-level mechanisms
>
> `Thread`
>
> `Runnable`
>
> `Callable`
>
> high-level mechanisms
>
> `Executor` framework
>
> fork/join framework
>
> `Stream` framework
>
>

### Customizing the ThreadPoolExecutor class

example

```java
public class MyExecutor extends ThreadPoolExecutor {
    private final ConcurrentHashMap<Runnable, Date> startTimes;

    public MyExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        this.startTimes = new ConcurrentHashMap<>();
    }

    @Override
    public void shutdown() {
        System.out.print("MyExecutor: Going to shutdown.\n");
        System.out.printf("MyExecutor: Executed tasks: %d.\n", getCompletedTaskCount());
        System.out.printf("MyExecutor: Running tasks: %d.\n", getActiveCount());
        System.out.printf("MyExecutor: Pending tasks: %d.\n", getQueue().size());
        super.shutdown();
    }

    @Override
    public List<Runnable> shutdownNow() {
        System.out.print("MyExecutor: Going to immediately shutdown.\n");
        System.out.printf("MyExecutor: Executed tasks: %d.\n", getCompletedTaskCount());
        System.out.printf("MyExecutor: Running tasks: %d.\n", getActiveCount());
        System.out.printf("MyExecutor: Pending tasks: %d.\n", getQueue().size());
        return super.shutdownNow();
    }

    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        System.out.printf("My Executor: A task is beginning: %s : %s\n",t.getName(),r.hashCode());
        startTimes.put(r,new Date());
    }

    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        Future<?> result = (Future<?>) r;
        try {
            System.out.print("*****************************************\n");
            System.out.print("My Executor: A task is finishing.\n");
            System.out.printf("My Executor: Result: %s.\n", result.get());
            Date startDate = startTimes.remove(r);
            Date finishDate = new Date();
            long diff = finishDate.getTime() - startDate.getTime();
            System.out.printf("My Executor: duration: %d\n",diff);
            System.out.print("*****************************************\n");
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
        super.afterExecute(r, t);
    }
}
```

```java
public class SleepTwoSecondsTask implements Callable<String> {
    @Override
    public String call() throws Exception {
        TimeUnit.SECONDS.sleep(2);
        return new Date().toString();
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        MyExecutor executor = new MyExecutor(4, 8, 1000,
                TimeUnit.MILLISECONDS, new LinkedBlockingDeque<>());
        List<Future<String>> results = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            SleepTwoSecondsTask task = new SleepTwoSecondsTask();
            Future<String> result = executor.submit(task);

            results.add(result);
        }
        for (int i = 0; i < 5; i++) {
            try {
                String result = results.get(i).get();
                System.out.printf("Main: Result for task %d : %s\n", i, result);
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();

        for (int i = 5; i < 10; i++) {
            try {
                String result = results.get(i).get();
                System.out.printf("Main: Result for task %d : %s\n", i, result);
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }

        try {
            executor.awaitTermination(1,TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: End of the program.\n");
    }
}
```

`beforeExecute()`

`afterExecute()`



### Implementing a priority-based Executor class

example

```java
public class MyPriorityTask implements Runnable, Comparable<MyPriorityTask> {

    private int priority;
    private String name;

    public MyPriorityTask(int priority, String name) {
        this.priority = priority;
        this.name = name;
    }

    public int getPriority() {
        return priority;
    }

    public String getName() {
        return name;
    }

    @Override
    public int compareTo(MyPriorityTask o) {
        return Integer.compare(o.getPriority(),this.priority);
    }

    @Override
    public void run() {
        System.out.printf("MyPriorityTask: %s Priority : %d\n",name, priority);
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(4, 4, 1,
                TimeUnit.SECONDS, new PriorityBlockingQueue<>());

        for (int i = 0; i < 10; i++) {
            MyPriorityTask task = new MyPriorityTask(i, "Task " + i);
            executor.execute(task);
        }

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int i = 10; i < 20; i++) {
            MyPriorityTask task = new MyPriorityTask(i, "Task " + i);
            executor.execute(task);
        }

        executor.shutdown();

        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: End of the program.\n");
    }
}
```



### Implementing the ThreadFactory interface to generate custom threads

example

```java
public class MyThreadFactory implements ThreadFactory {
    private AtomicInteger counter;
    private String prefix;

    public MyThreadFactory(String prefix) {
        this.prefix = prefix;
        this.counter = new AtomicInteger(1);
    }

    @Override
    public Thread newThread(Runnable r) {
        return new MyThread(r, prefix + " - " + counter.getAndIncrement());
    }
}
```

```java
public class MyThread extends Thread {
    private final Date creationDate;
    private Date startDate;
    private Date finishDate;

    public MyThread(Runnable target, String name) {
        super(target, name);
        this.creationDate = new Date();
    }

    @Override
    public void run() {
        setStartDate();
        super.run();
        setFinishDate();
    }

    public synchronized void setFinishDate() {
        this.finishDate = new Date();
    }

    public synchronized void setStartDate() {
        this.startDate = new Date();
    }

    public synchronized long getExecutionTime() {
        return finishDate.getTime() - startDate.getTime();
    }

    @Override
    public synchronized String toString() {
        StringBuilder buffer = new StringBuilder();
        buffer.append(getName()).append(": Creation Date: ")
                .append(creationDate)
                .append(" : RunningTime: ")
                .append(getExecutionTime())
                .append(" Milliseconds.");
        return buffer.toString();
    }
}
```

```java
public class MyTask implements Runnable {
    @Override
    public void run() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        MyThreadFactory myThreadFactory = new MyThreadFactory("MyThreadFactory");
        MyTask task = new MyTask();
        Thread thread = myThreadFactory.newThread(task);

        thread.start();
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: Thread information.\n");
        System.out.printf("%s\n",thread);
        System.out.print("End of example.\n");
    }
}
```



### Using our ThreadFactory in an Executor object

example

```java
public class Main04{
    public static void main(String[] args) {
        MyThreadFactory threadFactory = new MyThreadFactory("MyThreadFactory");

        ExecutorService executorService = Executors.newCachedThreadPool(threadFactory);

        MyTask task = new MyTask();

        executorService.submit(task);

        executorService.shutdown();

        try {
            executorService.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.print("Main: End of the program.\n");
    }
}
```



### Customizing tasks running in a scheduled thread pool

`java.util.concurrent.ScheduledThreadPoolExecutor`

- Delayed tasks
- Periodic tasks

example

```java
public class MyScheduledTask<V> extends FutureTask<V> implements RunnableScheduledFuture<V> {
    private RunnableScheduledFuture<V> task;
    private ScheduledThreadPoolExecutor executor;
    private long period;
    private long startDate;

    public MyScheduledTask(Runnable runnable, V result, RunnableScheduledFuture<V> task,
                           ScheduledThreadPoolExecutor executor) {
        super(runnable, result);
        this.task = task;
        this.executor = executor;
    }

    @Override
    public void run() {
        if (isPeriodic() && (!executor.isShutdown())) {
            Date now = new Date();
            startDate = now.getTime() + period;
            executor.getQueue().add(this);
        }
        System.out.printf("Per-MyScheduledTask: %s\n", new Date());
        System.out.printf("MyScheduledTask: Is Periodic: %s\n", new Date());
        super.runAndReset();

        System.out.printf("Post-MyScheduledTask: %s\n", new Date());
    }

    public void setPeriod(long period) {
        this.period = period;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        if (!isPeriodic()) {
            return task.getDelay(unit);
        } else {
            if (startDate == 0L) {
                return task.getDelay(unit);
            } else {
                Date now = new Date();
                long delay = startDate - now.getTime();
                return unit.convert(delay, TimeUnit.MILLISECONDS);
            }
        }
    }

    @Override
    public int compareTo(Delayed o) {
        return task.compareTo(o);
    }

    @Override
    public boolean isPeriodic() {
        return task.isPeriodic();
    }
}
```

```java
public class MyScheduledThreadPoolExecutor extends ScheduledThreadPoolExecutor {
    public MyScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize);
    }

    @Override
    protected <V> RunnableScheduledFuture<V> decorateTask(Runnable runnable,
                                                          RunnableScheduledFuture<V> task) {

        return new MyScheduledTask<>(runnable, null, task, this);
    }

    @Override
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
        ScheduledFuture<?> task = super.scheduleAtFixedRate(command, initialDelay, period, unit);
        MyScheduledTask<?> myTask = (MyScheduledTask<?>) task;
        myTask.setPeriod(TimeUnit.MILLISECONDS.convert(period, unit));
        return task;
    }
}
```

```java
public class Task implements Runnable {
    @Override
    public void run() {
        System.out.print("Task: Begin.\n");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.print("Task: End.\n");
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) {
        MyScheduledThreadPoolExecutor executor = new MyScheduledThreadPoolExecutor(4);

        Task task = new Task();

        System.out.printf("Main: %s\n",new Date());

        executor.schedule(task,1, TimeUnit.SECONDS);

        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        executor.scheduleAtFixedRate(task, 1,3,TimeUnit.SECONDS);

        try {
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        executor.shutdown();

        try {
            executor.awaitTermination(1,TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: End of the program.\n");
    }
}
```

`java.util.concurrent.FutureTask`

`java.util.concurrent.RunnableScheduledFuture`

all the tasks executed in a scheduled executor must implement this interface and extend the `FutureTask `class.

`decorateTask()`

provides a mechanism to convert the default scheduled tasks implemented by the `ScheduledThreadPoolExecutor` executor into other tasks.So, when you implement your own version of scheduled tasks, you have to implement your own version of a scheduled executor.

`getDelay() `

called by the scheduled executor to know whether it has to execute a task.



### Implementing the ThreadFactory interface to generate custom threads for the fork/join framework

One of the most interesting features of Java 9 is the fork/join framework. It's an implementation of the `Executor` and `ExecutorService` interfaces that allows you to execute the Callable and Runnable tasks without managing the threads that execute them

This executor is oriented to execute tasks that can be divided into smaller parts.

- It's a special kind of task, which is implemented by the `ForkJoinTask` class
- It provides two operations for dividing a task into subtasks (the fork operation) and to wait for the
  finalization of these subtasks (the join operation)
- It's an algorithm, denominating the work-stealing algorithm, that optimizes the use of the threads of the pool. When a task waits for its subtasks, the thread that was executing it is used to execute another thread.

The main class of the fork/join framework is the `ForkJoinPool` class

- A queue of tasks that are waiting to be executed
- A pool of threads that execute the tasks

`java.util.concurrent.ForkJoinWorkerThread`

`java.util.concurrent.ForkJoinPool.ForkJoinWorkerThreadFactory`

example

```java
public class MyRecursiveTask extends RecursiveTask<Integer> {
    private int[] array;
    private int start, end;

    public MyRecursiveTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        Integer ret;
        MyWorkerThread thread = (MyWorkerThread) Thread.currentThread();
        thread.addTask();
        if (end - start > 100) {
            int mid = (start + end) / 2;
            MyRecursiveTask task1 = new MyRecursiveTask(array, start, mid);
            MyRecursiveTask task2 = new MyRecursiveTask(array, mid, end);
            invokeAll(task1, task2);
            ret = addResults(task1, task2);
        } else {
            int add = 0;
            for (int i = start; i < end; i++) {
                add += array[i];
            }
            ret = add;
        }

        try {
            TimeUnit.MILLISECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return ret;
    }

    private Integer addResults(MyRecursiveTask task1, MyRecursiveTask task2) {
        int value;
        try {
            value = task1.get() + task2.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
            value = 0;
        }
        return value;
    }
}
```

```java
public class MyWorkerThread extends ForkJoinWorkerThread {

    private final static ThreadLocal<Integer> taskCounter = new ThreadLocal<>();

    public MyWorkerThread(ForkJoinPool pool) {
        super(pool);
    }

    public void addTask() {
        taskCounter.set(taskCounter.get() + 1);
    }

    @Override
    protected void onStart() {
        super.onStart();
        System.out.printf("MyWorkerThread %d: Initializing task counter.\n", getId());
        taskCounter.set(0);
    }


    @Override
    protected void onTermination(Throwable exception) {
        System.out.printf("MyWorkerThread %d: %d\n", getId(), taskCounter.get());
        super.onTermination(exception);
    }
}
```

```java
public class MyWorkerThreadFactory implements ForkJoinPool.ForkJoinWorkerThreadFactory {
    @Override
    public ForkJoinWorkerThread newThread(ForkJoinPool pool) {
        return new MyWorkerThread(pool);
    }
}
```

```java
public class Main06 {

    public static void main(String[] args) {
        MyWorkerThreadFactory factory = new MyWorkerThreadFactory();
        ForkJoinPool pool = new ForkJoinPool(4, factory, null, false);

        int[] array = new int[100000];
        for (int i = 0; i < array.length; i++) {
            array[i] = 1;
        }

        MyRecursiveTask task = new MyRecursiveTask(array, 0, array.length);
        pool.execute(task);

        task.join();
        pool.shutdown();

        try {
            pool.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            System.out.printf("Main: Result: %d\n",task.get());
            System.out.print("End of the program.\n");
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```



### Customizing tasks running in the fork/join framework

Java 9 provides a special kind of executor in the fork/join framework (introduced in Java 7). This framework is designed to solve problems that can be broken down into smaller tasks using the divide and conquer technique.

`ForkJoinPool`

- A queue of tasks that are waiting to be executed
- A pool of threads that execute the tasks

example

```java
public abstract class MyWorkerTask extends ForkJoinTask<Void> {
    private String name;

    public MyWorkerTask(String name) {
        this.name = name;
    }

    @Override
    public Void getRawResult() {
        return null;
    }

    @Override
    protected void setRawResult(Void value) {

    }

    @Override
    protected boolean exec() {
        Date startDate = new Date();
        compute();
        Date finishDate = new Date();
        long diff = finishDate.getTime() - startDate.getTime();
        System.out.printf("MyWorkerTask: %s: %d Milliseconds to complete.\n ", name, diff);
        return true;
    }

    protected abstract void compute();

    public String getName() {
        return name;
    }
}
```

```java
public class WorkerTask extends MyWorkerTask {
    private int[] array;

    private int start;
    private int end;

    public WorkerTask(String name, int[] array, int start, int end) {
        super(name);
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if(end - start > 100){
            int mid = (start+end)/2;
            WorkerTask task1 = new WorkerTask(this.getName(),array,start,mid);
            WorkerTask task2 = new WorkerTask(this.getName(),array,mid,end);
            invokeAll(task1,task2);
        }else{
            for(int i = start; i<end;i++){
                array[i]++;
            }

            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class Main07 {
    public static void main(String[] args) {
        int[] array = new int[10000];

        ForkJoinPool forkJoinPool = new ForkJoinPool();

        WorkerTask workerTask = new WorkerTask("WorkerTask", array, 0, array.length);

        forkJoinPool.invoke(workerTask);

        forkJoinPool.shutdown();

        System.out.print("Main: End of the program.\n");
    }
}
```

`java.util.concurrent.ForkJoinTask#setRawResult`

This method is used to establish the result of the task. As your tasks don't return any results, leave this method empty

`java.util.concurrent.ForkJoinTask#getRawResult`

This method is used to return the result of the task. As your tasks don't return any results, this method returns null.

`java.util.concurrent.ForkJoinTask#exec`

This method implements the logic of the task.

### Implementing a custom Lock class

`lock()` call this operation when you want to access a critical section

`unlock() ` call this operation at the end of a critical section to allow other threads to access it

example

```java
public class MyLock implements Lock {
    private final AbstractQueuedSynchronizer sync;

    public MyLock() {
        sync = new MyAbstractQueuedSynchronizer();
    }

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        try {
            return sync.tryAcquireNanos(1, 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupted();
            return false;
        }
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, TimeUnit.NANOSECONDS.convert(time, unit));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.new ConditionObject();
    }
}
```

```java
public class MyLockTask implements Runnable{

    private final MyLock lock;
    private final String name;

    public MyLockTask(MyLock lock, String name) {
        this.lock = lock;
        this.name = name;
    }

    @Override
    public void run() {
        lock.lock();

        System.out.printf("Task: %s: Take the lock\n", name);

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }

    }
}
```

```java
public class Main08 {
    public static void main(String[] args) {
        MyLock lock = new MyLock();

        for (int i = 0; i < 10; i++) {
            MyLockTask task = new MyLockTask(lock, "Task-" + i);
            Thread thread = new Thread(task);
            thread.start();
        }

        boolean value;
        do {
            try {
                value = lock.tryLock(1, TimeUnit.SECONDS);
                if (!value) {
                    System.out.print("Main: Trying to get the Lock\n");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                value = false;
            }
        } while (!value);

        System.out.print("Main: Got the Lock\n");

        System.out.print("Main: end of the program\n");
    }
}
```

The operations are based on two abstract methods

- `tryAcquire() `

  This method is called to try and get access to a critical section.

- `tryRelease()`

  This method is called to try and release access to a critical section



### Implementing a transfer queue-based on priorities

`java.util.concurrent.LinkedTransferQueue`

This data structure is supposed to be used in programs that have a producer/consumer structure

`java.util.concurrent.PriorityBlockingQueue`

In this data structure, elements are stored in an ordered way.

exmaple

```java
public class MyPriorityTransferQueue<E> extends PriorityBlockingQueue<E> implements TransferQueue<E> {

    private final AtomicInteger counter;
    private final LinkedBlockingQueue<E> transered;

    private final ReentrantLock lock;

    public MyPriorityTransferQueue() {
        this.counter = new AtomicInteger(0);
        this.lock = new ReentrantLock();
        this.transered = new LinkedBlockingQueue<>();
    }

    @Override
    public boolean tryTransfer(E e) {
        boolean value = false;
        try {
            lock.lock();
            if (counter.get() == 0) {
                value = false;
            } else {
                put(e);
                value = true;
            }
        } finally {
            lock.unlock();
        }
        return value;
    }

    @Override
    public void transfer(E e) throws InterruptedException {
        lock.lock();
        if (counter.get() != 0) {
            try {
                put(e);
            } finally {
                lock.unlock();
            }
        } else {
            try {
                transered.add(e);
            } finally {
                lock.unlock();
            }
            synchronized (e) {
                e.wait();
            }
        }
    }

    @Override
    public boolean tryTransfer(E e, long timeout, TimeUnit unit) throws InterruptedException {
        lock.lock();
        if (counter.get() != 0) {
            try {
                put(e);
            } finally {
                lock.unlock();
            }
            return true;
        } else {
            long newTimeout;
            try {
                transered.add(e);
                newTimeout = TimeUnit.MILLISECONDS.convert(timeout, unit);
            } finally {
                lock.unlock();
            }
            e.wait(newTimeout);

            lock.lock();
            boolean value;
            try {
                if (transered.contains(e)) {
                    transered.remove(e);
                    value = false;
                } else {
                    value = true;
                }
            } finally {
                lock.unlock();
            }
            return value;
        }
    }

    @Override
    public E take() throws InterruptedException {
        lock.lock();
        E value = null;
        try {
            counter.incrementAndGet();
            value = transered.poll();
            if (value == null) {
                lock.unlock();
                value = super.take();
                lock.lock();
            } else {
                synchronized (value) {
                    value.notify();
                }
            }
            counter.decrementAndGet();
        } finally {
            lock.unlock();
        }

        return value;
    }

    @Override
    public boolean hasWaitingConsumer() {
        return counter.get() != 0;
    }

    @Override
    public int getWaitingConsumerCount() {
        return counter.get();
    }
}
```

```java
public class Producer implements Runnable {

    private final MyPriorityTransferQueue<Event> buffer;

    public Producer(MyPriorityTransferQueue<Event> buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            Event event = new Event(Thread.currentThread().getName(), i);
            buffer.put(event);
        }
    }
}
```

```java
public class Consumer implements Runnable {

    private final MyPriorityTransferQueue<Event> buffer;

    public Consumer(MyPriorityTransferQueue<Event> buffer) {
        this.buffer = buffer;
    }

    @Override
    public void run() {
        for (int i = 0; i < 1002; i++) {
            try {
                Event event = buffer.take();
                System.out.printf("Consumer: %s: %d\n", event.getThread(), event.getPriority());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class Main09 {
    public static void main(String[] args) {
        MyPriorityTransferQueue<Event> buffer = new MyPriorityTransferQueue<>();

        Producer producer = new Producer(buffer);
        Thread[] producerThreads = new Thread[10];
        for (int i = 0; i < producerThreads.length; i++) {
            producerThreads[i] = new Thread(producer);
            producerThreads[i].start();
        }

        Consumer consumer = new Consumer(buffer);
        Thread consumerThread = new Thread(consumer);
        consumerThread.start();
        System.out.printf("Main: Buffer: Consumer count: %d\n", buffer.getWaitingConsumerCount());

        Event event = new Event("Core Event", 0);
        try {
            buffer.transfer(event);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.print("Main: My Event has been transfered.\n");

        for (int i = 0; i < producerThreads.length; i++) {
            try {
                producerThreads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("Main: Buffer: Consumer count: %d\n", buffer.getWaitingConsumerCount());

        event = new Event("Core Event 2", 0);
        try {
            buffer.transfer(event);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        try {
            consumerThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: End of the program\n");
    }
}
```



### Implementing your own atomic object

example

```java
public class ParkingCounter extends AtomicInteger {

    private final int maxNumber;

    public ParkingCounter(int maxNumber) {
        set(0);
        this.maxNumber = maxNumber;
    }

    public boolean carIn() {
        for (; ; ) {
            int value = get();
            if (value == maxNumber) {
                System.out.print("ParkingCounter: The parking lot is full.\n");
                return false;
            } else {
                int newValue = value + 1;
                boolean changed = compareAndSet(value, newValue);
                if (changed) {
                    System.out.print("ParkingCounter: a car has entered.\n");
                    return true;
                }
            }
        }
    }

    public boolean carOut() {
        for (; ; ) {
            int value = get();
            if (value == 0) {
                System.out.print("ParkingCounter:The parking lot is empty.\n");
                return false;
            } else {
                int newValue = value - 1;
                boolean changed = compareAndSet(value, newValue);
                if (changed) {
                    System.out.print("ParkingCounter: a car has gone out.\n");
                    return true;
                }
            }
        }
    }
}
```

```java
public class Sensor1 implements Runnable{

    private final ParkingCounter counter;

    public Sensor1(ParkingCounter counter) {
        this.counter = counter;
    }

    @Override
    public void run() {
        counter.carIn();
        counter.carIn();
        counter.carIn();
        counter.carIn();
        counter.carOut();
        counter.carOut();
        counter.carOut();
        counter.carIn();
        counter.carIn();
        counter.carIn();
    }
}
```

```java
public class Sensor2 implements Runnable{

    private final ParkingCounter counter;

    public Sensor2(ParkingCounter counter) {
        this.counter = counter;
    }

    @Override
    public void run() {
        counter.carIn();
        counter.carOut();
        counter.carOut();
        counter.carIn();
        counter.carIn();
        counter.carIn();
        counter.carIn();
        counter.carIn();
    }
}
```

```java
public class Main10 {
    public static void main(String[] args) {
        ParkingCounter counter = new ParkingCounter(5);
        Sensor1 sensor1 =new Sensor1(counter);
        Sensor2 sensor2 =new Sensor2(counter);

        Thread thread1 = new Thread(sensor1);
        Thread thread2 = new Thread(sensor2);

        thread1.start();
        thread2.start();

        try{
            thread1.join();
            thread2.join();
        }catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.printf("Main:Number of cars: %d\n",counter.get());

        System.out.print("Main: End of thr program.\n");
    }
}
```



### Implementing your own stream generator

example

```java
public class Item {
    private String name;
    private int row;
    private int column;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getRow() {
        return row;
    }

    public void setRow(int row) {
        this.row = row;
    }

    public int getColumn() {
        return column;
    }

    public void setColumn(int column) {
        this.column = column;
    }
}
```

```java
public class MySpliterator implements Spliterator<Item> {

    private Item[][] items;
    private int start, end, current;

    public MySpliterator(Item[][] items, int start, int end) {
        this.items = items;
        this.start = start;
        this.end = end;
        this.current = start;
    }

    @Override
    public boolean tryAdvance(Consumer<? super Item> action) {
        System.out.printf("MySpliterator. tryAdvance.start: %d, %d, %d\n", start, end, current);

        if (current < end) {
            for (int i = 0; i < items[current].length; i++) {
                action.accept(items[current][i]);
            }
            current++;
            System.out.print("MySpliteartor.tryAdvance.end:true\n");
            return true;
        }
        System.out.print("MySpliteartor.tryAdvance.end:false.\n");
        return false;
    }

    @Override
    public void forEachRemaining(Consumer<? super Item> action) {
        System.out.print("MySpliterator.forEachRemaining.start\n");
        boolean ret;
        do {
            ret = tryAdvance(action);
        } while (ret);
        System.out.print("MySpliterator.forEachRemaining.end\n");
    }

    @Override
    public Spliterator<Item> trySplit() {

        System.out.print("MySpliterator.trySplit.start\n");

        if (end - start <= 2) {
            System.out.print("MySpliterator.trySplit.end\n");
            return null;
        }

        int mid = start + (end - start) / 2;
        int newStart = mid;
        int newEnd = end;
        end = mid;
        System.out.printf("MySpliterator.trySplit.end: %d, %d, %d, %d, %d, %d\n",
                start, mid, end, newStart, newEnd, current);
        return new MySpliterator(items, newStart, newEnd);
    }

    @Override
    public long estimateSize() {
        return end - current;
    }

    @Override
    public int characteristics() {
        return ORDERED | SIZED | SUBSIZED;
    }
}
```

```java
public class Main11 {
    public static void main(String[] args) {
        Item[][] items = new Item[10][10];
        for (int i = 0; i < 10; i++) {
            for (int j = 0; j < 10; j++) {
                items[i][j] = new Item();
                items[i][j].setRow(i);
                items[i][j].setColumn(j);
                items[i][j].setName("Item " + i + " " + j);
            }
        }

        MySpliterator mySpliterator = new MySpliterator(items, 0, items.length);

        StreamSupport.stream(mySpliterator, true)
                .forEach(item -> System.out.printf("%s: %s\n", Thread.currentThread().getName(), item.getName()));
    }
}
```

`java.util.Spliterator`

This interface defines methods that can be used to process and partition a source of elements to be used

has a set of characteristics that defines its behavior

- CONCURRENT  The data source can be safely modified concurrently
- DISTINCT All the elements of the data source are distinct
- IMMUTABLE  Elements cannot be added, deleted, or replaced in the data source
- NONNULL There's no null element in the data source
- ORDERED There's an encounter ordered in the elements of the data source
- SIZED The value returned by the `estimateSize()` method is the exact size of the `Spliterator`
- SORTED  The elements of `Spliterator` are sorted
- SUBSIZED After you call the `trySplit()` method, you can obtain the exact size of both the parts of `Spliterator`

we implemented all the methods defined by the `Spliterator` interface that don't have a default implementation:

- `java.util.Spliterator#tryAdvance`

  applies the function specified as a parameter to the next element to be processed

- `java.util.Spliterator#trySplit`

  This method is used to divide the current `Spliterator` into two different parts so each one can be processed by different threads. 

- `java.util.Spliterator#estimateSize`

  This method returns the number of elements that would be processed by the `forEachRemaining()` method if it were called at the current moment

- `java.util.Spliterator#characteristics`

  returns the characteristics of the `Spliterator` object.returns an integer value you calculate using the bitwise or operator ( | ) between the individual characteristics of your `Spliterator` object.

- `java.util.Spliterator#forEachRemaining`

  This method applies the function received as a parameter (an implementation of the `Consumer `interface) to the elements of the `Spliterator` that haven't been processed yet



### Implementing your own asynchronous stream

example

```java
public class News {
    private String title;
    private String content;
    private Date date;

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getContent() {
        return content;
    }

    public void setContent(String content) {
        this.content = content;
    }

    public Date getDate() {
        return date;
    }

    public void setDate(Date date) {
        this.date = date;
    }
}
```

```java
public class Consumer implements Flow.Subscriber<News> {

    private Flow.Subscription subscription;
    private String name;

    public Consumer(String name) {
        this.name = name;
    }


    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        this.subscription = subscription;
        subscription.request(1);
        System.out.printf("%s: Consumer - Subscription\n", Thread.currentThread().getName());
    }

    @Override
    public void onNext(News item) {
        System.out.printf("%s - %s: Consumer - News\n",
                name, Thread.currentThread().getName());
        System.out.printf("%s - %s: Title: %s\n",
                name, Thread.currentThread().getName(), item.getTitle());
        System.out.printf("%s - %s: Content: %s\n",
                name, Thread.currentThread().getName(), item.getContent());
        System.out.printf("%s - %s: Date: %s\n",
                name, Thread.currentThread().getName(), item.getDate());
        subscription.request(1);
    }

    @Override
    public void onError(Throwable throwable) {
        System.out.printf("%s - %s: Consumer - Error: %s\n",
                name, Thread.currentThread().getName(), throwable.getMessage());
    }

    @Override
    public void onComplete() {
        System.out.printf("%s - %s: Consumer - Completed\n",
                name, Thread.currentThread().getName());
    }
}
```

```java
public class ConsumerData {
    private Consumer consumer;
    private MySubscription subscription;

    public Consumer getConsumer() {
        return consumer;
    }

    public void setConsumer(Consumer consumer) {
        this.consumer = consumer;
    }

    public MySubscription getSubscription() {
        return subscription;
    }

    public void setSubscription(MySubscription subscription) {
        this.subscription = subscription;
    }
}
```

```java
public class MyPublisher implements Flow.Publisher<News> {
    private ConcurrentLinkedDeque<ConsumerData> consumers;
    private ThreadPoolExecutor executor;

    public MyPublisher() {
        this.consumers = new ConcurrentLinkedDeque<>();
        this.executor = (ThreadPoolExecutor) Executors.newFixedThreadPool(
                Runtime.getRuntime().availableProcessors());
    }

    @Override
    public void subscribe(Flow.Subscriber<? super News> subscriber) {
        ConsumerData consumerData = new ConsumerData();
        consumerData.setConsumer((Consumer) subscriber);

        MySubscription subscription = new MySubscription();
        consumerData.setSubscription(subscription);

        subscriber.onSubscribe(subscription);
        consumers.add(consumerData);
    }

    public void publish(News news){
        consumers.forEach(consumerData -> {
            try{
                executor.execute(new PublishTask(consumerData,news));
            }catch (Exception e){
                consumerData.getConsumer().onError(e);
            }
        });
    }
}
```

```java
public class MySubscription implements Flow.Subscription {

    private boolean canceled = false;
    private long requested = 0;

    @Override
    public void request(long n) {
        requested += n;
    }

    @Override
    public void cancel() {
        this.canceled = true;
    }

    public boolean isCanceled() {
        return canceled;
    }

    public long getRequested() {
        return requested;
    }

    public void decreaseRequested() {
        requested--;
    }
}
```

```java
public class PublishTask implements Runnable{

    private ConsumerData consumerData;
    private News news;

    public PublishTask(ConsumerData consumerData, News news) {
        this.consumerData = consumerData;
        this.news = news;
    }

    @Override
    public void run() {

        MySubscription subscription = consumerData.getSubscription();
        if(!(subscription.isCanceled()&&(subscription.getRequested()>0))){
            consumerData.getConsumer().onNext(news);
            subscription.decreaseRequested();
        }

    }
}
```

```java
public class Main12 {
    public static void main(String[] args) {
        MyPublisher publisher = new MyPublisher();
        Flow.Subscriber<News> consumer1,consumer2;
        consumer1 = new Consumer("Consumer1");
        consumer2 = new Consumer("Consumer2");

        publisher.subscribe(consumer1);
        publisher.subscribe(consumer2);

        System.out.print("Main: Start\n");
        News news = new News();
        news.setTitle("My First News");
        news.setContent("This is the content");
        news.setDate(new Date());

        publisher.publish(news);

        try{
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        news=new News();
        news.setTitle("My second news");
        news.setContent("This is the content of the second news");
        news.setDate(new Date());
        publisher.publish(news);
        System.out.print("Main: End\n");

    }
}
```



## Testing Concurrent Applications

> Testing an application is a critical task. Before you make an application ready for end users, you have to demonstrate its correctness. 

### Monitoring a Lock interface

A **critical section** is a block of code that accesses a shared resource and can't be executed by more than one thread at the same time. 

example

```java
public class MyLock extends ReentrantLock {
    public String getOwnerName(){
        if(this.getOwner() == null){
            return "None";
        }
        return this.getOwner().getName();
    }

    public Collection<Thread> getThreads(){
        return this.getQueuedThreads();
    }
}
```

```java
public class Task implements Runnable{
    private final Lock lock;

    public Task(Lock lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        for(int i = 0; i< 5;i++){
            lock.lock();
            System.out.printf("%s: Get the Lock.\n",Thread.currentThread().getName());

            try {
                TimeUnit.MILLISECONDS.sleep(500);
                System.out.printf("%s: Free the lock.\n",Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                lock.unlock();
            }
        }
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        MyLock lock = new MyLock();

        Thread[] threads = new Thread[5];

        for (int i = 0; i < 5; i++) {
            Task task = new Task(lock);
            threads[i] = new Thread(task);
            threads[i].start();
        }

        for (int i = 0; i < 15; i++) {
            System.out.println("Main: Logging the Lock.");
            System.out.println("************************");
            System.out.printf("Lock: Owner : %s\n", lock.getOwnerName());
            System.out.printf("Lock: Queued Threads: %s\n", lock.hasQueuedThreads());

            if (lock.hasQueuedThreads()) {
                System.out.printf("Lock: Queue Length: %d\n", lock.getQueueLength());
                System.out.println("Lock: Queued Threads:");
                Collection<Thread> lockedThreads = lock.getThreads();
                lockedThreads.forEach(thread -> System.out.printf("%s ", thread.getName()));
                System.out.print("\n");
            }

            System.out.printf("Lock: Fairness: %s\n", lock.isFair());
            System.out.printf("Lock: Locked: %s\n", lock.isLocked());
            System.out.println("************************");

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

`java.util.concurrent.locks.ReentrantLock#hasQueuedThreads`

`java.util.concurrent.locks.ReentrantLock#getQueueLength`

`java.util.concurrent.locks.ReentrantLock#isLocked`

`java.util.concurrent.locks.ReentrantLock#isFair`

`java.util.concurrent.locks.ReentrantLock#getHoldCount`

`java.util.concurrent.locks.ReentrantLock#isHeldByCurrentThread`

### Monitoring a Phaser Class

The `Phaser` class provides the mechanism to synchronize threads at the end of each step so no thread starts its second step until all the threads have finished the first one.

example

```java
public class Task implements Runnable{

    private final int time;
    private final Phaser phaser;

    public Task(int time, Phaser phaser) {
        this.time = time;
        this.phaser = phaser;
    }

    @Override
    public void run() {
        phaser.arrive();

        System.out.printf("%s: Entering phase 1.\n",Thread.currentThread().getName());

        try {
            TimeUnit.SECONDS.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("%s: Finishing phase 1.\n",Thread.currentThread().getName());
        phaser.arriveAndAwaitAdvance();

        System.out.printf("%s: Entering phase 2.\n",Thread.currentThread().getName());

        try {
            TimeUnit.SECONDS.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("%s: Finishing phase 2.\n",Thread.currentThread().getName());
        phaser.arriveAndAwaitAdvance();

        System.out.printf("%s: Entering phase 3.\n",Thread.currentThread().getName());

        try {
            TimeUnit.SECONDS.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("%s: Finishing phase 3.\n",Thread.currentThread().getName());

        phaser.arriveAndDeregister();

    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3);
        for (int i = 0; i < 3; i++) {
            Task task = new Task(i + 1, phaser);
            Thread thread = new Thread(task);
            thread.start();
        }

        for (int i = 0; i < 10; i++) {
            System.out.println("********************");
            System.out.println("Main: Phaser Log\n");
            System.out.printf("Main: Phaser: Phase: %d\n", phaser.getPhase());
            System.out.printf("Main: Phaser: Registered Parties: %d\n", phaser.getRegisteredParties());
            System.out.printf("Main: Phaser: Arrived Parties: %d\n", phaser.getArrivedParties());
            System.out.printf("Main: Phaser: UnArrived Parties: %d\n", phaser.getUnarrivedParties());
            System.out.println("********************");

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

`java.util.concurrent.Phaser#getPhase`

`java.util.concurrent.Phaser#getRegisteredParties`

`java.util.concurrent.Phaser#getArrivedParties`

`java.util.concurrent.Phaser#getUnarrivedParties`

### Monitoring an Executor framework

example

```java
public class Task implements Runnable {
    private final long milliseconds;

    public Task(long milliseconds) {
        this.milliseconds = milliseconds;
    }

    @Override
    public void run() {
        System.out.printf("%s: Begin\n",Thread.currentThread().getName());
        try {
            TimeUnit.MILLISECONDS.sleep(milliseconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.printf("%s: End\n",Thread.currentThread().getName());
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newCachedThreadPool();
        Random random = new Random();
        for (int i = 0; i < 10; i++) {
            Task task = new Task(random.nextInt(10000));

            executor.submit(task);
        }

        for (int i = 0; i < 5; i++) {
            showLog(executor);

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();

        for (int i = 0; i < 5; i++) {
            showLog(executor);

            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.print("Main: End of the program.\n");
    }

    private static void showLog(ThreadPoolExecutor executor) {
        System.out.println("*********************");
        System.out.println("Main: Executor Log");
        System.out.printf("Main: Executor: Core Pool Size: %d\n", executor.getCorePoolSize());
        System.out.printf("Main: Executor: Pool Size: %d\n", executor.getPoolSize());
        System.out.printf("Main: Executor: Active Count: %d\n", executor.getActiveCount());
        System.out.printf("Main: Executor: Task Count: %d\n", executor.getTaskCount());
        System.out.printf("Main: Executor: Completed Task Count: %d\n", executor.getCompletedTaskCount());
        System.out.printf("Main: Executor: Shutdown: %s\n", executor.isShutdown());
        System.out.printf("Main: Executor: Terminating: %s\n", executor.isTerminating());
        System.out.printf("Main: Executor: Terminated: %s\n", executor.isTerminated());
        System.out.println("*********************");
    }
}
```

`java.util.concurrent.ThreadPoolExecutor#getCorePoolSize`

`java.util.concurrent.ThreadPoolExecutor#getPoolSize`

`java.util.concurrent.ThreadPoolExecutor#getActiveCount`

`java.util.concurrent.ThreadPoolExecutor#getTaskCount`

`java.util.concurrent.ThreadPoolExecutor#getCompletedTaskCount`

`java.util.concurrent.ThreadPoolExecutor#isShutdown`

`java.util.concurrent.ThreadPoolExecutor#isTerminating`

`java.util.concurrent.ThreadPoolExecutor#isTerminated`



### Monitoring a fork/join pool

example

```java
public class Task extends RecursiveAction {

    private final int[] array;
    private final int start;
    private final int end;

    public Task(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if(end -start>100){
            int mid = (start+end)/2;
            Task task1 = new Task(array,start,mid);
            Task task2 = new Task(array,mid, end);

            task1.fork();
            task2.fork();

            task1.join();
            task2.join();
        }else{
            for(int i = start;i<end;i++){
                array[i]++;
                try {
                    Thread.sleep(5);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        ForkJoinPool pool = new ForkJoinPool();
        int[] array = new int[10000];
        Task task = new Task(array, 0, array.length);
        pool.execute(task);

        while (!task.isDone()) {
            showLog(pool);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        pool.shutdown();
        try {
            pool.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        showLog(pool);

        System.out.print("Main: end of the program.\n");
    }

    private static void showLog(ForkJoinPool pool) {
        System.out.print("**********************\n");
        System.out.print("Main: Fork/Join Pool log\n");
        System.out.printf("Main: Fork/Join Pool: Parallelism: %d\n", pool.getParallelism());
        System.out.printf("Main: Fork/Join Pool: Pool Size: %d\n", pool.getPoolSize());
        System.out.printf("Main: Fork/Join Pool: Active Thread Count: %d\n", pool.getActiveThreadCount());
        System.out.printf("Main: Fork/Join Pool: Running Thread Count: %d\n", pool.getRunningThreadCount());
        System.out.printf("Main: Fork/Join Pool: Queued Submission: %d\n", pool.getQueuedSubmissionCount());
        System.out.printf("Main: Fork/Join Pool: Queued Tasks: %d\n", pool.getQueuedTaskCount());
        System.out.printf("Main: Fork/Join Pool: Queued Submissions: %s\n", pool.hasQueuedSubmissions());
        System.out.printf("Main: Fork/Join Pool: Steal Count: %d\n", pool.getStealCount());
        System.out.printf("Main: Fork/Join Pool: Terminated: %s\n", pool.isTerminated());
        System.out.print("**********************\n");
    }
}
```

`java.util.concurrent.ForkJoinPool#getParallelism`

`java.util.concurrent.ForkJoinPool#getPoolSize`

`java.util.concurrent.ForkJoinPool#getActiveThreadCount`

`java.util.concurrent.ForkJoinPool#getRunningThreadCount`

`java.util.concurrent.ForkJoinPool#getQueuedSubmissionCount`

`java.util.concurrent.ForkJoinPool#getQueuedTaskCount`

`java.util.concurrent.ForkJoinPool#hasQueuedSubmissions`

`java.util.concurrent.ForkJoinPool#getStealCount`

`java.util.concurrent.ForkJoinPool#isTerminated`



### Monitoring a stream

```java
public class Main05 {
    public static void main(String[] args) {
        AtomicLong counter = new AtomicLong(0);
        Random random = new Random();
        long streamCounter = random.doubles(1000).parallel()
                .peek(number -> {
                    long actual = counter.incrementAndGet();
                    System.out.printf("%d - %f\n", actual, number);
                }).count();

        System.out.printf("Counter: %d\n", counter.get());
        System.out.printf("Stream Counter: %d\n", streamCounter);

        counter.set(0);
        random.doubles(1000).parallel()
                .peek(number -> {
                    long actual = counter.incrementAndGet();
                    System.out.printf("Peek: %d - %f\n", actual, number);
                }).forEach(number -> System.out.printf("For Each: %f\n", number));

        System.out.printf("Counter: %d\n", counter.get());
    }
}
```

In the first case, our final operation is the `count()` method. This method doesn't need to process the elements to calculate the returned value, so the `peek()` method will never be executed. 

The second case is different. The final operation is the `forEach()` method, and in this case, all the elements of the stream will be processed. 

The `peek()` method is an intermediate operation of a stream. Like with all intermediate operations, they are executed lazily, and they only process the necessary elements. 



### Writing effective log messages

You should use the log system because of the following two main reasons

- Write as much information as you can when an exception is caught. This will help you localize the error and resolve it.
- Write information about the classes and methods that the program is executing.

example

```java
public class MyFormatter extends Formatter {
    @Override
    public String format(LogRecord record) {
        StringBuilder sb = new StringBuilder();
        sb.append("[" + record.getLevel() + "] - ");
        sb.append(new Date(record.getMillis()) + "." + record.getSourceMethodName() + " : ");
        sb.append(record.getMessage() + "\n");
        return sb.toString();
    }
}
```

```java
public class MyLoggerFactory {
    private static Handler handler;

    public synchronized static Logger getLogger(String name) {
        Logger logger = Logger.getLogger(name);
        logger.setLevel(Level.ALL);
        try {
            if (handler == null) {
                handler = new FileHandler("recipe6.log");
                Formatter format = new MyFormatter();
                handler.setFormatter(format);
            }
            if (logger.getHandlers().length == 0) {
                logger.addHandler(handler);
            }
        } catch (SecurityException | IOException e) {
            e.printStackTrace();
        }

        return logger;
    }
}
```

```java
public class Task implements Runnable{

    @Override
    public void run() {
        Logger logger = MyLoggerFactory.getLogger(this.getClass().getName());
        logger.entering(Thread.currentThread().getName(),"run()");

        try{
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        logger.exiting(Thread.currentThread().getName(),"run()",Thread.currentThread());
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        Logger logger = MyLoggerFactory.getLogger(Main06.class.getName());
        logger.entering(Main06.class.getName(), "main()", args);

        Thread[] threads = new Thread[5];
        for (int i = 0; i < threads.length; i++) {
            logger.log(Level.INFO, "Launching thread: " + i);
            Task task = new Task();
            threads[i] = new Thread(task);
            logger.log(Level.INFO, "Thread created: " + threads[i].getName());
            threads[i].start();
        }

        logger.log(Level.INFO, "Ten Threads created.Waiting for its finalization");

        for (int i = 0; i < threads.length; i++) {
            try {
                threads[i].join();
                logger.log(Level.INFO,"Ten Threads created. Waiting for its finalization");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        logger.exiting(Main06.class.getName(), "main()");
    }
}
```

take into consideration two important points

- Write the necessary information
- Use the adequate level for the messages



### Analyzing concurrent code with FindBugs

### Configuring Eclipse for debugging concurrency code

### Configuring NetBeans for debugging concurrency code

### Test concurrency code with MultithreadedTC

### Monitoring with JConsole

JConsole is a monitoring tool that follows the JMX specification that allows you to get information about the execution of an application as the number of threads, memory use, or class loading. It is included with the JDK and it can be used to monitor local or remote applications.

six tabs

- The Overview tab provides an overview of **memory use**, **the number of threads running in the application**, **the number of objects created**, and **CPU usage of the application**.
- The Memory tab shows the amount of memory used by the application. It has a combo where you can select the type of memory you want to monitor (heap, non-heap, or pools)
- The Threads tab shows you information about the number of threads in the application and detailed
  information about each thread
- The Classes tab shows you information about the number of objects loaded in the application.
- The VW Summary tab provides a summary of the JVM running the application
- The MBeans tab shows you information about the managed beans of the application



# Additional Information

### Processing results for Runnable objects in the Executor framework

example

```java
public class FileSearch implements Runnable {

    private String initPath;
    private String end;
    private List<String> results;

    public FileSearch(String initPath, String end) {
        this.initPath = initPath;
        this.end = end;
        results = new ArrayList<>();
    }

    @Override
    public void run() {
        System.out.printf("%s: Starting\n", Thread.currentThread().getName());
        File file = new File(initPath);
        if (file.isDirectory()) {
            directoryProcess(file);
        }

    }

    private void directoryProcess(File file) {
        File[] list = file.listFiles();
        if (list != null) {
            for (int i = 0; i < list.length; i++) {
                if (list[i].isDirectory()) {
                    directoryProcess(list[i]);
                } else {
                    fileProcess(list[i]);
                }
            }
        }
    }

    private void fileProcess(File file) {
        if (file.getName().endsWith(end)) {
            results.add(file.getAbsolutePath());
        }
    }

    public List<String> getResults() {
        return results;
    }
}
```

```java
public class FileTask extends FutureTask<List<String>> {
    private FileSearch fileSearch;

    public FileTask(Runnable runnable, List<String> result) {
        super(runnable, result);
        this.fileSearch = (FileSearch) runnable;
    }

    @Override
    protected void set(List<String> v) {
        v = fileSearch.getResults();
        super.set(v);
    }
}
```

```java
public class Main01 {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        FileSearch system = new FileSearch("C:/Windows", "log");
        FileSearch apps = new FileSearch("C:/Program Files", "log");
        FileSearch documents = new FileSearch("C:/Documents And Settings"," log");

        FileTask systemTask = new FileTask(system,null);
        FileTask appsTask = new FileTask(apps,null);
        FileTask documentsTask = new FileTask(documents,null);

        executor.submit(systemTask);
        executor.submit(appsTask);
        executor.submit(documentsTask);

        executor.shutdown();

        try {
            executor.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        try {
            System.out.printf("Main: System Task: Number of Results: %d\n",
                    systemTask.get().size());
            System.out.printf("Main: App Task: Number of Results: %d\n",
                    appsTask.get().size());
            System.out.printf("Main: Documents Task: Number of Results: %d\n",documentsTask.get().size());
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```



### Processing uncontrolled exceptions in a `ForkJoinPool` class

```java
public class OneSecondLongTask  extends RecursiveAction {
    @Override
    protected void compute() {
        System.out.print("Task: Starting.\n");
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.print("Task: Finish.\n");
    }
}
```

```java
public class Handler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        System.out.printf("Handler: Thread %s has thrown an Exception.\n", t.getName());
        System.out.printf("%s\n", e);
        System.exit(-1);
    }
}
```

```java
public class AlwaysThrowsExceptionWorkerThread extends ForkJoinWorkerThread {
    public AlwaysThrowsExceptionWorkerThread(ForkJoinPool pool) {
        super(pool);
    }

    @Override
    protected void onStart() {
        super.onStart();
        throw new RuntimeException("Exception from worker thread");
    }
}
```

```java
public class AlwaysThrowsExceptionWorkerThreadFactory implements ForkJoinPool.ForkJoinWorkerThreadFactory {
    @Override
    public ForkJoinWorkerThread newThread(ForkJoinPool pool) {
        return new AlwaysThrowsExceptionWorkerThread(pool);
    }
}
```

```java
public class Main02 {
    public static void main(String[] args) {
        OneSecondLongTask task = new OneSecondLongTask();
        Handler handler = new Handler();
        AlwaysThrowsExceptionWorkerThreadFactory factory = new AlwaysThrowsExceptionWorkerThreadFactory();

        ForkJoinPool pool = new ForkJoinPool(2, factory, handler, false);

        pool.execute(task);

        pool.shutdown();

        try {
            pool.awaitTermination(1, TimeUnit.DAYS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.print("Task: Finish.\n");
    }
}
```



### Using a blocking thread-safe queue to communicate with producers and consumers

example

```java
public class Consumer implements Runnable {

    private LinkedTransferQueue<String> buffer;
    private String name;

    public Consumer(LinkedTransferQueue<String> buffer, String name) {
        this.buffer = buffer;
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            try {
                buffer.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.printf("Consumer: %s: Consumer done\n", name);
    }
}
```

```java
public class Producer implements Runnable {
    private LinkedTransferQueue<String> buffer;
    private String name;

    public Producer(LinkedTransferQueue<String> buffer, String name) {
        this.buffer = buffer;
        this.name = name;
    }

    @Override
    public void run() {
        for (int i = 0; i < 10000; i++) {
            buffer.put(name + ":Element " + i);
        }
        System.out.printf("Producer: %s: Producer done.\n", name);
    }
}
```

```java
public class Main03 {
    public static void main(String[] args) {
        final int THREADS = 100;
        LinkedTransferQueue<String> buffer = new LinkedTransferQueue<>();
        Thread[] producerThreads = new Thread[THREADS];
        Thread[] consumerThreads = new Thread[THREADS];


        for (int i = 0; i < THREADS; i++) {
            producerThreads[i] = new Thread(new Producer(buffer, "Producer: " + i));
            producerThreads[i].start();
        }

        for (int i = 0; i < THREADS; i++) {
            consumerThreads[i] = new Thread(new Consumer(buffer, "Consumer: " + i));
            consumerThreads[i].start();
        }

        for (int i = 0; i < THREADS; i++) {
            try {
                producerThreads[i].join();
                consumerThreads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.printf("Main: Size of the buffer: %d\n", buffer.size());
        System.out.print("Main: End of the example\n");
    }
}
```



### Monitoring a Thread class

example

```java
public class RunningTask implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 100; i++) {
            try {
                TimeUnit.MILLISECONDS.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.printf("%s: %d\n", Thread.currentThread().getName(), i);
        }
    }
}
```

```java
public class Main04 {
    public static void main(String[] args) {
        RunningTask task = new RunningTask();

        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(task);
            threads[i].setPriority(i + 1);
            threads[i].start();
        }

        for (int j = 0; j < 10; j++) {
            System.out.print("Main: Logging threads\n");
            for (int i = 0; i < threads.length; i++) {
                System.out.print("**********************\n");
                System.out.printf("Main: %d: Id: %d Name: %s: Priority: %d\n", i, threads[i].getId(), threads[i].getName(), threads[i].getPriority());
                System.out.printf("Main: Status: %s\n", threads[i].getState());
                System.out.printf("Main: Thread Group: %s\n", threads[i].getThreadGroup());
                System.out.print("Main: Stack Trace: \n");
                for (int t = 0; t < threads[i].getStackTrace().length; t++) {
                    System.out.printf("Main: %s\n", threads[i].getStackTrace()[t]);
                }
                System.out.print("**********************\n");
            }
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```



### Monitoring a Semaphore class

example

```java
public class SemaphoreTask implements Runnable {

    private final Semaphore semaphore;

    public SemaphoreTask(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    @Override
    public void run() {
        try {
            semaphore.acquire();
            System.out.printf("%s: Get the semaphore.\n", Thread.currentThread().getName());
            TimeUnit.SECONDS.sleep(2);
            System.out.println(Thread.currentThread().getName()+": Release the semaphore.");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            semaphore.release();
        }
    }
}
```

```java
public class Main05 {
    public static void main(String[] args) throws InterruptedException {
        Semaphore semaphore=new Semaphore(3);
        Thread[] threads = new Thread[10];
        for (int i=0; i<threads.length; i++) {
            SemaphoreTask task=new SemaphoreTask(semaphore);
            threads[i]=new Thread(task);
            threads[i].start();
            TimeUnit.MILLISECONDS.sleep(200);
            showLog(semaphore);
        }
        for (int i=0; i<5; i++) {
            showLog(semaphore);
            TimeUnit.SECONDS.sleep(1);
        }


    }
    private static void showLog(Semaphore semaphore) {
        System.out.print("********************\n");
        System.out.print("Main: Semaphore Log\n");
        System.out.printf("Main: Semaphore: Avalaible Permits: %d\n",
                semaphore.availablePermits());
        System.out.printf("Main: Semaphore: Queued Threads: %s\n",
                semaphore.hasQueuedThreads());
        System.out.printf("Main: Semaphore: Queue Length: %d\n",
                semaphore.getQueueLength());
        System.out.printf("Main: Semaphore: Fairness: %s\n",
                semaphore.isFair());
        System.out.print("********************\n");
    }
}
```



### Generating concurrent random numbers

```java
public class TaskLocalRandom implements Runnable {
    @Override
    public void run() {
        String name = Thread.currentThread().getName();
        for (int i = 0; i < 10; i++) {
            System.out.printf("%s: %d\n", name, ThreadLocalRandom.current().nextInt(10));
        }
    }
}
```

```java
public class Main06 {
    public static void main(String[] args) {
        Thread threads[]=new Thread[3];
        for (int i=0; i<3; i++) {
            TaskLocalRandom task=new TaskLocalRandom();
            threads[i]=new Thread(task);
            threads[i].start();
        }
    }
}
```

`java.util.concurrent.ThreadLocalRandom#current`

returns the `ThreadLocalRandom` object associated with the current thread, so you can generate random numbers using that object.

## Concurrent Programming Design

### Using immutable objects when possible

**immutable objects**; their main characteristic is that they can't be modified after they are created. If you need to change an immutable object, you must create a new one instead of changing the values of the attributes of the object

 advantages

- These objects cannot be modified by any thread once they are created, so you won't need to use any synchronization mechanism to protect access to their attributes
- You won't have any data inconsistency problems. As the attributes of these objects cannot be modified, you will always have access to a coherent copy of the data



### Avoiding deadlocks by ordering locks

- If you have to get control of more than one lock in different operations, try to lock them in the same order in all methods

- Then, release them in inverse order and encapsulate the locks and their unlocks in a single
  class. This is so that you don't have synchronization-related code distributed throughout the
  code.

### Using atomic variables instead of synchronization



### Holding locks for as short time as possible

Split the method into various critical sections, and use more than one lock if necessary to get the best performance of your application

### Delegating the management of threads to executors



### Using concurrent data structures instead of programming yourselves

**Non-blocking data structures**

All the operations provided by these data structures to either insert in or take off elements from the data structure return a null value if they can't be done currently because the data structure is full or empty respectively

**Blocking data structures**

These data structures provide the same operations that are provided by non-blocking data structures. However, they also provide operations to insert and take off data that, if not done immediately, would block the thread until you're able to do the operations

### Taking precautions using lazy initialization

**Lazy initialization** is a common programming technique that delays object creation until it is needed for the first time. 

The main advantage of this technique is that you can save memory

### Using the fork/join framework instead of executors



### Avoiding the use of blocking operations inside a lock

Blocking operations are operations that block the execution of the current thread until an event occurs. Typical blocking operations are those that involve input or output operations with the console, a file, or network

### Avoiding the use of deprecated methods

### Using executors instead of thread groups

### Using streams to process big data sets

### Other tips and tricks

Whenever possible, use concurrent design patterns

Implement concurrency at the highest possible level

Take scalability into account

Prefer local thread variables over static and shared when possible









