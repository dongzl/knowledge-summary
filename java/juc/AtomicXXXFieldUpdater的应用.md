# Java中AtomicXXXFieldUpdater的应用

在java.util.concurrent.atomic包下有一些用于并发进行原子更新的类。例如：
- AtomicInteger.java / AtomicLong.java / AtomicBoolean.java / AtomicReference.java 是关于对变量的原子更新；
- AtomicIntegerArray.java / AtomicLongArray.java / AtomicReferenceArray.java 是关于对数组的原子更新；
- AtomicIntegerFieldUpdater<T>.java / AtomicLongFieldUpdater<T>.java / AtomicReferenceFieldUpdater<T,V>.java 是基于反射的原子更新字段的值。

关于 AtomicInteger.java / AtomicIntegerArray.java 都比较简单，如何使用可以查相应的 API 文档，对于 AtomicIntegerFieldUpdater.java 的使用稍微有一些限制和约束，约束如下：
1. **字段必须是 volatile 类型的**，在线程之间共享变量时保证立即可见。eg：`volatile int value = 3;`；
2. 字段的描述类型（修饰符 public / protected / default / private）是与调用者与操作对象字段的关系一致。也就是说调用者能够直接操作对象字段，那么就可以反射进行原子操作。但是对于父类的字段，子类是不能直接操作的，尽管子类可以访问父类的字段；
3. **只能是实例变量，不能是类变量**，也就是说不能加 static 关键字；
4. 只能是可修改变量，**不能是 final 变量**，因为 final 的语义就是不可修改。实际上 final 的语义和 volatile 是有冲突的，这两个关键字不能同时存在；
5. 对于 AtomicIntegerFieldUpdater.java 和 AtomicLongFieldUpdater.java 只能修改 int / long 类型的字段，不能修改其包装类型（Integer / Long）。如果要修改包装类型就需要使用AtomicReferenceFieldUpdater.java。

```java
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

public class AtomicIntegerFieldUpdaterAnalyzeTest {

    public static void main(String[] args) throws Exception {
        AtomicIntegerFieldUpdaterAnalyzeTest test = new AtomicIntegerFieldUpdaterAnalyzeTest();
        test.testValue();
    }

    public AtomicIntegerFieldUpdater<DataDemo> updater(String name) {
        return AtomicIntegerFieldUpdater.newUpdater(DataDemo.class, name);
    }

    public void testValue() {
        DataDemo data = new DataDemo();
        
        // 访问父类的public 变量，报错：java.lang.NoSuchFieldException
        // System.out.println("fatherVar = " + updater("fatherVar").getAndIncrement(data));
        //访问普通 变量，报错：java.lang.IllegalArgumentException: Must be volatile type
        // System.out.println("intVar = " + updater("intVar").getAndIncrement(data));

        // 访问public volatile int 变量，成功
        System.out.println("publicVar = " + updater("publicVar").incrementAndGet(data));
        // 访问protected volatile int 变量，成功
        System.out.println("protectedVar = " + updater("protectedVar").incrementAndGet(data));

        // 访问其他类private volatile int变量，失败：java.lang.IllegalAccessException
        // System.out.println("privateVar = " + updater("privateVar").getAndIncrement(data));
        
        // 访问，static volatile int，失败，只能访问实例对象：java.lang.IllegalArgumentException
        // System.out.println("staticVar = " + updater("staticVar").getAndIncrement(data));
        
        // 访问integer变量，失败， Must be integer type
        // System.out.println("integerVar = " + updater("integerVar").getAndIncrement(data));

        // 访问long 变量，失败， Must be integer type
        // System.out.println("longVar = " + updater("longVar").getAndIncrement(data));

        data.testPrivate();
    }

    static class Father {
        public volatile int fatherVar = 4;
    }

    static class DataDemo extends Father {

        public int intVar = 4;
        public volatile int publicVar = 3;
        protected volatile int protectedVar = 4;
        private volatile int privateVar = 5;
        public volatile static int staticVar = 10;
        // public final volatile int finalVar = 11;
        public volatile Integer integerVar = 18;
        public volatile Long longVar = 19L;
        
        public void testPrivate() {
            DataDemo data = new DataDemo();
            System.out.println(AtomicIntegerFieldUpdater.newUpdater(DataDemo.class, "privateVar").incrementAndGet(data));
        }
    }
}

```