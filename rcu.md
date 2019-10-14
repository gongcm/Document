## RCu简介
  1. RCU 机制（read-copy-update）为了提高读写效率，在一个项目中可能存在多个读者和少数写者，此时若使用其他锁来进行互斥访问临界资源，务必会有多个读者和写者线程
  处于阻塞状态，而RCU 出现正是针对这样的问题。
  
  a. rcu 可以不用锁(mutex等)，可以达到互斥的效果。
  b. rcu对于临界资源访问时，可以允许同时读写，一般情况存在多个写者时，这个几个写者之间要进行互斥。


### 内核api
1. 读 ``
  rcu_read_lock 
  rcu_derefrence // 内存屏障，编译器不会优化
  rcu_read_unlock


```
// 2 写
  rcu_assign_pointer(p,v) // 内存屏障，防止编译器优化 
  synchronize_rcu or call_rcu



```
```

### rcu 宽限期

宽限期 ： 当前有多个读者访问临界资源，此时要更新临界资源，
为了保证读者和写者能够访问到正确临界资源，此时写者必须等待
所有访问该临界资源的读者退出临界资源的访问，而写者等待的时间
就是宽限期。那麽问题来了，如何判断读者是否退出临界区呢？

所以，要提供一种有效的检测机制检测读者已经退出临界区。

其实这个很简单，就是对读者进行计数，进入临界区的时候加 1,退出临界区
再减 1,那麽写者就很容易判断是否有读者正在临界区了。

那麽，内核中是否是这样实现的呢？


第二种情况，当写者进入临界区，然后读者怎么知道写者已经退出临界区呢？

那麽，写者是否也可以采用引用计数呢？ 答案是可以的。

`
```

```
struct foo{
    int a;
    char b;
    long c;
};

DEFINE_SPINLOCK(foo_mutex); //所有写者互相互斥

struct foo gbl_foo; // 全局变量，读者写者共享

void foo_read(void)
{
    foo *fp = gbl_foo;
    if( fp != NULL )
    {
        dosomthing(fp->a, fp->b, fp->c); // 这里执行时，可能被 foo_update 
                                         // 打断运行，造成fp中部分置被修改
                                         // 可能造成dosomething 出错。
    }
}

void foo_update(foo * new_fp)
{
    spin_lock(&foo_mutex);
    foo *old_fp = gbl_foo;
    gbl_foo = new_fp;
    spin_unlock(&foo_mutex);
}
``

加锁后代码：
```
void foo_read(void)
{
    rcu_read_lock();
    foo *fp = gbl_foo;
    if( fp != NULL )
        dosomthing(fp->a, fp->b, fp->c);
    rcu_read_unlock();
}

void foo_update(foo *new_fp)
{
    spin_lock(&foo_mutex);
    foo *old_fp = gbl_foo;
    gbl_foo = new_fp;
    spin_unlock(&foo_mutex);
    synchronize_rcu();
    kfree(old_fp);
}
```

当存在多个读者写者时，要保证读者和写者都能互斥访问临界资源时，
kernel 提供了一种 效率比较高的锁 rcu 来保证多个读者写者互斥访问
临界资源。


`
