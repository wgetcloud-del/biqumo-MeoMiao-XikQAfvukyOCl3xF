
如果想在spring操作事务结束后执行一些代码，应该怎么办？



> 为什么要这样？比如我们在事务中给其他系统发了消息，期望事务提交后过一会收到这个系统的回应，然后操作刚刚提交的数据。但是如果回应来的太快就像龙卷风，我们的事务是托管给Spring的可能还没提交，也就没法操作了


一个方案是使用 `ApplicationEventPublisher`，可以参考我之前的千万访问量博客
[https://www.iteye.com/blog/somefuture\-2405963](https://github.com)



> 登陆访问量是100多万，我就假设总访问量是10倍吧哈哈
> ![image](https://img2024.cnblogs.com/blog/2157887/202410/2157887-20241018165236422-849286192.png)


这个API是 Spring 1 就提供的，从 Spring 5 开始，提供了一个新的事物相关的API，叫 `TransactionSynchronization` 事物同步机制。


# 上代码


先编写一个Bean实`TransactionSynchronization`接口



```
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;
import org.springframework.stereotype.Component;

@Component
public class AfterTransactionCommitExecutor implements TransactionSynchronization {

    @Override
    public void afterCommit() {
        // 事务提交后执行的操作
        System.out.println("事务已提交，执行后续操作");
    }

    // 其他需要重写的方法...

    public void registerSynchronization() {
        // 注册当前实例到事务同步管理器
        TransactionSynchronizationManager.registerSynchronization(this);
    }
}

```

然后，你可以在服务层或者合适的地方调用`registerSynchronization()`方法来注册事务同步回调



```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SomeService {

    @Autowired
    private AfterTransactionCommitExecutor afterTransactionCommitExecutor;

    @Transactional
    public void doWork() {
        // 业务逻辑...

        // 注册事务同步回调
        afterTransactionCommitExecutor.registerSynchronization();
    }
}

```



---


基本上使用它还是为了操作数据，所以需要把参数传给他。


## 一 成员变量


最简单的就是加一个成员属性。



```
@Component
public class AfterTransactionCommitExecutor extends TransactionSynchronizationAdapter {

    private Object parameter;

    @Override
    public void afterCommit() {
        // 事务提交后使用参数执行操作
        doSomethingWithParameter(parameter);
    }

    public void setParameter(Object parameter) {
        this.parameter = parameter;
    }

    private void doSomethingWithParameter(Object parameter) {
    }

    public void registerSynchronization() {
        TransactionSynchronizationManager.registerSynchronization(this);
    }
}
@Service
public class SomeService {

    @Autowired
    private AfterTransactionCommitExecutor afterTransactionCommitExecutor;

    @Transactional
    public void doWork(Object parameter) {
        // 设置参数
        afterTransactionCommitExecutor.setParameter(parameter);
        // 注册事务同步回调
        afterTransactionCommitExecutor.registerSynchronization();
    }
}


```

## 二 每次创建匿名类对象



```
@Service
public class SomeService {

    @Transactional
    public void doWork(final Object parameter) {
        // 业务逻辑...

        // 注册事务同步回调并传递参数
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                doSomethingWithParameter(parameter);
            }
        });
    }

    private void doSomethingWithParameter(Object parameter) {
        // 使用参数执行相关操作
    }
}


```

注意，在使用成员变量传递参数时，如果多个事务并发执行，可能会存在线程安全问题。为了避免这个问题，可以使用ThreadLocal来存储参数，或者在事务方法中每次都创建一个新的TransactionSynchronization实例。


 本博客参考[slower加速器](https://chundaotian.com)。转载请注明出处！
