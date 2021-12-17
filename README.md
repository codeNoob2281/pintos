# OS课程设计

[Project : Shell](http://gitlab.etao.net/osd20f/agf/blob/master/hw2/Shell.md)

## Project 1 THREADS

### 一. 设计概览

pintos 的第一个项目是实现线程，主要实现了以下部分:

* 修改 timer_sleep 函数，设置thread阻塞时间。
* 实现优先级调度，完成抢占机制功能和优先级捐赠功能。 
* 实现多级反馈调度，更新线程的优先级。

### 二. 设计细节

#### 1. Timer_sleep

基本思想：

设置阻塞时间，阻塞线程

结构设计：

```c++
struct thread
{
    ···
    int64_t blocking_ticks;                /* Ticks that the thread to block. */
    ···
}; 
static void thread_blocking_ticks_handler(struct thread *t, void *args UNUSED);
```

1. 在 `thread` 类中添加成员变量 `blocking_ticks` 来记录线程的剩余阻塞时间。
2. 增加 `thread_blocking_ticks_handler` 函数用来更新 `blocking_ticks`。

算法设计：
调用 `timer_sleep` 函数时设置阻塞时间把线程阻塞， 在系统自身的`timer_interrupt`函数中加入对线程状态的检测函数`thread_timer`。

```c++
void timer_sleep(int64_t ticks)
{
  if (ticks <= 0)
  {
    return;
  }
  ASSERT(intr_get_level() == INTR_ON);
  enum intr_level old_level = intr_disable();
  struct thread *current_thread = thread_current();
  current_thread->blocking_ticks = ticks;
  thread_block();
  intr_set_level(old_level);
}
static void
timer_interrupt(struct intr_frame *args UNUSED)
{
  ticks++;
  enum intr_level old_level = intr_disable();
  thread_timer(timer_ticks() % TIMER_FREQ == 0);
  intr_set_level(old_level);
  thread_tick ();
}
```

每次检测时通过`thread_foreach`调用 `thread_blocking_ticks_handler` 函数更新所有线程的`blocking_ticks`，若 `blocking_ticks` 为零则唤醒这个线程。

```c++
void
thread_timer(bool full_second){
  if (thread_mlfqs){
    thread_add_recent_cpu();
    if (full_second){
      thread_update_load_avg();
      thread_foreach(thread_update_recent_cpu, NULL);
      thread_foreach(thread_update_priority, NULL);
      thread_ready_list_sort();
    }
  }
  thread_foreach(thread_blocking_ticks_handler, NULL);
}
```



#### 2. Priority

1）Preemption

基本思想：
维护就绪队列为一个优先级队列**，**且正在运行的进程的优先级是最高的。

结构设计：

```c++
bool thread_priority_cmp (const struct list_elem *a_, const struct list_elem *b_,
                           void *aux UNUSED);
void thread_insert_ready_list (struct list_elem *elem);
```

2. 增加`thread_priority_cmp`函数比较优先级大小
3. 增加`thread_insert_ready_list`插入线程到就绪队列

算法设计：

通过获取`thread`结构体下的`priority`变量进行比较

```c++
bool
thread_priority_cmp (const struct list_elem *a_, const struct list_elem *b_,
                      void *aux UNUSED)
{
  const struct thread *a = list_entry (a_, struct thread, elem);
  const struct thread *b = list_entry (b_, struct thread, elem);

  return thread_get_certain_priority (a) > thread_get_certain_priority (b);
}
```

使用list的`list_instert_ordered`函数有序插入线程，通过`thread_priority_cmp`函数比较线程优先级。

```c++
void
thread_insert_ready_list (struct list_elem *elem)
{
  if (!list_empty (&ready_list))
    list_insert_ordered (&ready_list, elem, thread_priority_cmp, NULL);
  else
    list_push_back (&ready_list, elem);
}
```

考虑一个进程在以下两种情况下进入就绪队列:

1. `thread_unblock`

   ```c++
   void
   thread_unblock (struct thread *t)
   {
     enum intr_level old_level;
   
     ASSERT (is_thread (t));
   
     old_level = intr_disable ();
     ASSERT (t->status == THREAD_BLOCKED);
     thread_insert_ready_list (&t->elem);
     t->status = THREAD_READY;
     intr_set_level (old_level);
   }
   ```

2. `thread_yield`

   ```c++
   void
   thread_yield (void)
   {
     struct thread *cur = thread_current ();
     enum intr_level old_level;
   
     ASSERT (!intr_context ());
   
     old_level = intr_disable ();
     if (cur != idle_thread)
       thread_insert_ready_list (&cur->elem);
     cur->status = THREAD_READY;
     schedule ();
     intr_set_level (old_level);
   }
   
   ```

因此在这两个函数内使用`thread_insert_ready_list`。



2）Donation

基本思想：
当高优先级的线程因为低优先级线程占用资源而阻塞时，就将低优先级线程的优先级提升到等待它所占有资源的最高优先级线程的优先级。

结构设计：

```c++
struct lock
  {
    struct thread *holder;      /* Thread holding lock (for debugging). */
    struct semaphore semaphore; /* Binary semaphore controlling access. */
    int max_donate;             /* Max donation. */
    struct list donaters;       /* Donater list. */
    struct list_elem elem;      /* List element for lock list. */
  };
struct thread
{
    ···
    struct list lock_list;              /* Locks owned by this thread. */
    int priority_to_set;                /* Priority to be set. */
    int max_donate;                     /* Max Donation. */
    struct thread *father;              /* Thread who locks this thread. */
    struct list_elem donate_elem;       /* List element for donation list of locks. */
    ···
};
```

1. 在`look`类内增加整型`max_donate`记录最大捐赠的优先级，链表`donaters`,`elem`记录捐赠队列和等待队列。
2. 在 `thread `类加入`lock_list`队列记录获得的锁，`priority_to_set`整型记录临时优先级，整型`max_donate`记录最大捐赠的优先级，`father`指针记录当前占用`lock`的线程，`donate_elem`记录捐赠队列
3. 重写 `lock_acquire`, `lock_release` 函数修改最大捐赠值，并修改优先级。
4. 增加 `thread_update_priority` 函数更新优先级。

算法设计：

1. 在一个线程申请获取一个锁的时候， 如果拥有这个锁的线程优先级比自己低就提高它的优先级，使用`father`指针记录当前占有锁的线程，通过`list_push_back`函数把新线程放入捐赠队列中，并且如果这个锁还被别的锁锁着， 将会递归地捐赠优先级即调用`notify_father`函数，。

   ```c++
   void
   lock_acquire (struct lock *lock)
   {
     ASSERT (lock != NULL);
     ASSERT (!intr_context ());
     ASSERT (!lock_held_by_current_thread (lock));
   
     /* The lock has been locked, so deal with the waiting relation and donation stuff. */
     if (lock->holder != NULL && thread_get_priority () > lock->holder->priority)
       {
         int delta = thread_get_priority () - lock->holder->priority;
         struct thread *t = thread_current ();
         t->father = lock->holder;
   
         list_push_back (&lock->donaters, &t->donate_elem);
   
         if (delta > lock->max_donate)
           {
             lock->max_donate = delta;
           }
   
         if (delta > lock->holder->max_donate)
           {
             lock->holder->max_donate = delta;
             notify_father (lock->holder);
           }
   
       }
   
     sema_down (&lock->semaphore);
     lock->holder = thread_current ();
   
     /* Got the new lock successfully. Initialize it. */
     list_push_back (&lock->holder->lock_list, &lock->elem);
   }
   ```

2. 使用迭代法遍历，如果不存在`father`则停止，否则就判断当前线程被捐赠的最大优先级是否小于等待队列中的优先级，若持有锁的线程优先级低，则把等待队列中最高优先级的优先级捐赠给持有锁的当前线程。

   ```c++
   void
   notify_father (struct thread *t)
   {
     if (t == NULL)
       return;
     struct thread *father = t->father;
     if (father == NULL)
       return;
     int old_donate = father->max_donate;
   
     struct list_elem *el, *ed;
     struct lock *l;
     struct thread *tmp;
     for (el = list_begin (&father->lock_list); el != list_end (&father->lock_list); el = list_next (el))
       {
         l = list_entry (el, struct lock, elem);
         for (ed = list_begin (&l->donaters); ed != list_end (&l->donaters); ed = list_next (ed))
           {
             tmp = list_entry (ed, struct thread, donate_elem);
             if (tmp->priority + tmp->max_donate - father->priority > l->max_donate)
               l->max_donate = tmp->priority + tmp->max_donate - father->priority;
           }
         if (l->max_donate > father->max_donate)
           father->max_donate = l->max_donate;
       }
     if (father->max_donate != old_donate)
       notify_father (father);
   }
   
   ```

   

3. 遍历捐赠队列，使其`father`均指向NULL,并清空此队列。在释放锁对一个锁优先级有改变的时候应考虑其余被捐赠优先级和当前优先级。为保证优先队列在释放资源后调用`thread_revolt`来激活线程。

   ```c++
   void
   lock_release (struct lock *lock)
   {
     ASSERT (lock != NULL);
     ASSERT (lock_held_by_current_thread (lock));
   
     struct thread *t = lock->holder;
   
     /* Cut down the donation. */
     struct list_elem *e;
     for (e = list_begin (&lock->donaters); e != list_end (&lock->donaters); e = list_next (e))
       {
         struct thread *son = list_entry (e, struct thread, donate_elem);
         son->father = NULL;
       }
     while (!list_empty (&lock->donaters))
       list_pop_front (&lock->donaters);
     list_remove (&lock->elem);
   
     lock->max_donate = 0;
     lock->holder = NULL;
     sema_up (&lock->semaphore);
   
     /* Update the lock holder's donation information and notify its father if necessary. */
     int old_donate = t->max_donate;
     t->max_donate = 0;
     for (e = list_begin (&t->lock_list); e != list_end (&t->lock_list); e = list_next (e))
       {
         struct lock *l = list_entry (e, struct lock, elem);
         if (l->max_donate > t->max_donate)
           t->max_donate = l->max_donate;
       }
     if (t->max_donate != old_donate)
       {
         notify_father (t);
       }
   
     if (t->max_donate == 0 && t->priority_to_set > -1)
       {
         t->priority = t->priority_to_set;
         t->priority_to_set = -1;
       }
     thread_revolt ();
   }
   ```

   4.为保证优先级队列，比较当前线程和就绪队列第一个线程的优先级大小，如果当前执行优先级低则调用`thread_yield`让步。

   ```c++
   void
   thread_revolt (void)
   {
     if (thread_current () != idle_thread &&
         !list_empty (&ready_list) &&
         thread_get_priority () <
         thread_get_certain_priority (
                 list_entry (list_begin (&ready_list), struct thread, elem)))
     {
       thread_yield ();
     }
   }
   ```

   

#### 3 BSD Scheduler

基本思想：
维护了64个队列， 每个队列对应一个优先级， 从PRI_MIN到PRI_MAX。每隔一定时间，通过一些公式计算来计算出线程当前的优先级， 系统调度的时候会从高优先级队列开始选择线程执行， 这里线程的优先级随着操作系统的运转数据而动态改变。

结构设计：

```
struct thread
{
    ···
    int nice;                           /* The nice level of thread, the higher the lower priority */
    fixed_point_t recent_cpu; /* Thread recent cpu usage */
    ···
}；
int thread_get_nice (void);
void thread_set_nice (int);
int thread_get_recent_cpu (void);
int thread_get_load_avg (void);
static void thread_update_load_avg(void);
static void thread_update_recent_cpu(struct thread*, void*);
void thread_add_recent_cpu(void);
```

1. 

算法设计：
 根据http://web.stanford.edu/class/cs140/projects/pintos/pintos_7.html#SEC131提供的BSD调度算法实现更新

 priority = PRI_MAX - (recent_cpu / 4) - (nice * 2)

 recent_cpu = (2*load_avg)/(2*load_avg + 1) * recent_cpu + nice

 load_avg = (59/60)*load_avg + (1/60)*ready_threads.

1. 使用浮点类`fixed-point.h`，用于计算优先级。

2. 通过`nice`和`recent_cpu`计算`load_avg`和`priority`。

3. 增加`thread_set_nice`,`thread_get_nice`获取和设置nice值

   ```c++
   int
   thread_get_nice (void)
   {
     struct thread *cur = thread_current();
     return cur->nice;
   }
   
   void
   thread_set_nice (int nice)
   {
     thread_current()->nice = nice;
     if(thread_mlfqs){
       thread_update_priority(thread_current(), NULL);
     }
   
   }
   
   ```

4. 增加`thread_get_recent_cpu ()`，`thread_get_load_avg ()`获取recent_cpu,load_avg值

   ```c++
   /* Returns 100 times the current thread's recent_cpu value. */
   int
   thread_get_recent_cpu (void)
   {
     ASSERT(thread_mlfqs);
     return fix_round(fix_scale(thread_current()->recent_cpu, 100));
     
   }
   
   
   /* Returns 100 times the system load average. */
   int
   thread_get_load_avg (void)
   {
     ASSERT(thread_mlfqs);
     return fix_round(fix_scale(load_avg, 100));
   }
   ```

5. 在每一次时间中断时，`thread_timer`先是通过``thread_add_recent_cpu()`函数更新`recent_cpt`,之后再通过调用`thread_foreach`函数遍历所有线程执行以下`thread_update_load_avg()`, `thread_update_recent_cpu()`, `thread_update_priority`函数更新计算`recent_cpu`,`load_avg`,`priority`

   ```c++
   /* Add recent_cpu of the running thread by 1 per second */
   void thread_add_recent_cpu(void)
   {
     ASSERT(thread_mlfqs);
     struct thread* cur = thread_current();
     if (cur != idle_thread)
       cur->recent_cpu = fix_add(cur->recent_cpu, fix_int(1));
     thread_update_priority(thread_current(), NULL);
   }
   /* Update load_avg
    * load_avg = 59/60 * load_avg + 1/60 * ready_threads
    * */
   static void
   thread_update_load_avg(void)
   {
   
     ASSERT(thread_mlfqs);
     int ready_threads = 0;
     struct list_elem *e;
     for (e = list_begin (&all_list); e != list_end (&all_list);
          e = list_next (e))
     {
       struct thread *t = list_entry (e, struct thread, allelem);
       if ((t->status == THREAD_RUNNING || t->status == THREAD_READY) && t != idle_thread)
         ready_threads++;
     }
     load_avg = fix_add(fix_unscale(fix_scale(load_avg, 59), 60), fix_unscale(fix_int(ready_threads), 60));
   }
   
   /* Update recent_cpu
    * recent_cpu = recent_cpu * (load_avg * 2 / (load_avg * 2 + 1)) + nice
    * Nice is thread's nice value
    * */
   static void
   thread_update_recent_cpu(struct thread *t, void* aux UNUSED)
   {
     ASSERT(thread_mlfqs);
     int load_avg2 = thread_get_load_avg() * 2;
      t->recent_cpu = fix_add(fix_mul(fix_div(fix_scale(load_avg, 2), fix_add(fix_scale(load_avg, 2), fix_int(1))),t->recent_cpu), fix_int(t->nice));
   }
   
   /* Returns 100 times the current thread's recent_cpu value. */
   int
   thread_get_recent_cpu (void)
   {
     ASSERT(thread_mlfqs);
     return fix_round(fix_scale(thread_current()->recent_cpu, 100));
     
   }
   static void
   thread_update_priority(struct thread* t, void *args UNUSED){
     if(t == idle_thread)
       return;
     int recent_cpu_div4 = fix_trunc(fix_unscale(t->recent_cpu, 4));
     t->priority =  PRI_MAX - recent_cpu_div4 - t->nice * 2;
   }
   ```

6. 由于增加了调度算法，所以需要增加`thread_get_certain_priority`判断调度算法，返回当前优先级

   ``` c++
   /* Returns the certain thread's priority. */
   int
   thread_get_certain_priority (const struct thread *t)
   {
     return t->priority + (thread_mlfqs ? 0 : t->max_donate) ;
   }
   ```

   

## Project 2 USERPROG

### 一. 设计概览

pintos 的第二个项目是实现用户程序，主要实现了以下部分:

- 读取cmdline，并执行对应的应用程序(包含传参)
- 通过syscall处理用户的进程控制操作， 例如终止当前进程 建立新的进程等
- 通过syscall处理用户的文件操作，例如新建文件 删除文件 读写文件等
- bonus 部分 (禁止写入正在执行的可执行文件)

### 二. 设计细节

#### 1. Process Termination Messages

基本思想：

设置变量记录返回值,并在process_eixt时输出

结构设计：

```c++
struct thread
{
    ···
    int return_value;     /* Return value of the thread (anyway, nobody cares)*/
    ···
}; 
```

1. 在 `thread` 类中添加成员变量 `return_value `来记录结束返回值0或-1，表示是否为正常结束。
2. 增加 `thread_exit_with_return_value` 函数来设置返回值。

算法设计：
在线程退出时通过判断是否存在异常，使用`thread_exit_with_return_value`来改变返回值并退出线程。

```c++
/* Terminate thread with a return value FINAL_VALUE */
void
thread_exit_with_return_value(struct intr_frame *f, int return_value) {
  struct thread *cur = thread_current();
  cur->return_value = return_value;
  f->eax = (uint32_t)return_value;
  thread_exit();
}
```



#### 2. Argument Passing

基本思想：

传递参数主要分为两步，第一步是分离参数，第二步是把参数告诉要执行的程序。
使用`strtok_r`函数可以完成实现`command line`中参数的切割，将一整行命令切成可执行文件名和参数。参数分离完成之后，采用栈来使得用户程序能够读取到这些参数。

算法设计：

以命令`/bin/ls -l foo bar`为例, 参数在栈中的位置如下所示:

    Address         Name                Data            Type
    0xbffffffc      argv[3][...]        bar             char[4]
    0xbffffff8      argv[2][...]        foo             char[4]
    0xbffffff5      argv[1][...]        -l              char[3]
    0xbfffffed      argv[0][...]        /bin/ls         char[8]
    0xbfffffec      word-align          0               uint8_t
    0xbfffffe8      argv[4]             0               char *
    0xbfffffe4      argv[3]             0xbffffffc      char *
    0xbfffffe0      argv[2]             0xbffffff8      char *
    0xbfffffdc      argv[1]             0xbffffff5      char *
    0xbfffffd8      argv[0]             0xbfffffed      char *
    0xbfffffd4      argv                0xbfffffd8      char **
    0xbfffffd0      argc                4               int
    0xbfffffcc      return address      0               void (*) ()

只需要将分离出来的参数按照 从右到左、data-align-pointer-argv-argc-return_addr 的顺序压入栈中，程序就可以正确地读取到参数

1.在`process_execute`中对`file_name_`使用`strtok_r`进行切割划分出第一个值，作为为`thread_name`，在创建线程时使用。

```c++
tid_t
process_execute (const char *file_name_)
{
  /* Pudding by Chen Started */
  char *file_name;
  char *fn_copy;
  tid_t tid;

  /* Make a copy of FILE_NAME.
     Otherwise there's a race between the caller and load(). */
  file_name = palloc_get_page(0);
  strlcpy (file_name, file_name_, PGSIZE);
  /* Pudding by Chen Ended */

  fn_copy = palloc_get_page (0);
  if (fn_copy == NULL)
    return TID_ERROR;
  strlcpy (fn_copy, file_name, PGSIZE);


  char *thread_name, *save_ptr;
  thread_name = strtok_r (file_name, " ", &save_ptr);

  /* Create a new thread to execute FILE_NAME. */
  tid = thread_create (thread_name, PRI_DEFAULT, start_process, fn_copy);
  if (tid == TID_ERROR)
    palloc_free_page (fn_copy);
  palloc_free_page(file_name); /* this is part of pudding */

  list_push_back (&thread_current ()->child_list,
                  &thread_get_child_message (tid)->elem);


  return tid;
}
```



2.在`start_process`中通过`load`函数给`esp`赋值分配空间。获取`esp`堆栈指针后将`cmd`逐步切割获取所有参数放入用户栈，临时变量`args`数组存储参数地址，并使用`top`整型记录参数个数存入`argc`。由于需要加入双字的对齐，根据原始`esp`指针位置和当前位置继续减小栈指针位置使其为4的倍数。先放入0到用户栈后，逆序放入`args`内的指针以及`top`变量值，最后放入0作为`return address`。把esp指针赋值给`if_.esp`以此完成参数的切割并入栈

```c++
/* A thread function that loads a user process and starts it
   running. */
static void
start_process (void *file_name_)
{
  char *file_name = file_name_;
  struct intr_frame if_;
  bool success;


  char *text, *save_ptr;
  text = strtok_r (file_name, " ", &save_ptr);

  /* Initialize interrupt frame and load executable. */
  memset (&if_, 0, sizeof if_);
  if_.gs = if_.fs = if_.es = if_.ds = if_.ss = SEL_UDSEG;
  if_.cs = SEL_UCSEG;
  if_.eflags = FLAG_IF | FLAG_MBS;
  success = load (text, &if_.eip, &if_.esp);

  if (success)
  {

#define MAX_ARGC 32
    char *esp = if_.esp;
    char *args[MAX_ARGC], *arg;   /* Args is a stack.  */
    int top = 0;

    for (arg = text; arg != NULL; arg = strtok_r (NULL, " ", &save_ptr))
    {
      int l = strlen (arg);
      esp -= l + 1;
      strlcpy (esp, arg, l + 1);
      args[top] = esp, top++;
    }

    while (((char *) if_.esp - esp) % 4 != 0) esp--;
    esp -= 4;
    *((int *) esp) = 0;  /* alignment */

    int argc = top;
    while (top > 0)
    {
      esp -= 4;
      --top, *((char **) esp) = args[top];
    }
    char **argv = (char **) esp;

    esp -= 4;
    *((char ***) esp) = argv;
    esp -= 4;
    *((int *) esp) = argc;
    esp -= 4;
    *((int *) esp) = 0;

    if_.esp = esp;
#undef MAX_ARGC

  }
  /* If load failed, quit. */
  palloc_free_page (file_name);
  if (!success)
  {
    thread_current ()->message_to_grandpa->load_failed = true;
    thread_current ()->message_to_grandpa->return_value = -1;
    thread_current ()->return_value = -1;
  }
  sema_up (thread_current ()->message_to_grandpa->sema_started);
  if (!success)
    thread_exit ();


  /* Start the user process by simulating a return from an
     interrupt, implemented by intr_exit (in
     threads/intr-stubs.S).  Because intr_exit takes all of its
     arguments on the stack in the form of a `struct intr_frame',
     we just point the stack pointer (%esp) to our stack frame
     and jump to it. */
  asm volatile ("movl %0, %%esp; jmp intr_exit" : : "g" (&if_) : "memory");
  NOT_REACHED ();
}
```



#### 3. System call

基本思想：

用户程序有时候需要“借用”内核的权限，来进行一些比较底层的操作，比如执行一个新的程序、读写I\O等等。Syscall 提供了一个给用户程序内核权限的接口，并且要进行严格的检查以保证用户程序的所有操作都是合法的。

结构设计：

```c++
static struct lock filesys_lock;
```

定义filesys_lock变量为文件添加锁预防读写冲突。

```c++
struct child_message
{
    struct thread *tchild;              /* Thread pointer to the child. */
    tid_t tid;                          /* Thread ID. */
    bool exited;                        /* If syscall exit() is called. */
    bool terminated;                    /* If the child finishes running. */
    bool load_failed;                   /* If the child has a load fail. */
    int return_value;                   /* Return value. */
    struct semaphore *sema_finished;    /* Semaphore to finish. */
    struct semaphore *sema_started;     /* Semaphore to finish loading. */
    struct list_elem elem;              /* List element for grandpa's child list. */
    struct list_elem allelem;           /* List element for global child list. */
};
```

定义`child_message`结构体记录子进程的相关信息，方便父进程对其的控制。

```c++
struct file_handle{
    int fd;
    struct file* opened_file;
    struct thread* owned_thread;
    /* Implementation by ymt Started */
#ifdef FILESYS
    struct dir* opened_dir;
#endif
    /* Implementation by ymt Ended */
    struct list_elem elem;
};
```

定义`file_handle`结构体，作为文件句柄，在文件操作是记录打开的文件。

在pintos中，一共涉及到了16个syscall操作，如下所示：

```c++
static void syscall_halt(struct intr_frame *f);
static void syscall_exit(struct intr_frame *f, int return_value);
static void syscall_exec(struct intr_frame *f, const char *cmd_line);
static void syscall_wait(struct intr_frame *f, pid_t pid);
static void syscall_open(struct intr_frame *f, const char *name);
static void syscall_create(struct intr_frame *f, const char *name, unsigned initial_size);
static void syscall_remove(struct intr_frame *f, const char *name);
static void syscall_filesize(struct intr_frame *f, int fd);
static void syscall_read(struct intr_frame *f, int fd, const void *buffer, unsigned size);
static void syscall_write(struct intr_frame *f, int fd, const void *buffer, unsigned size);
static void syscall_seek(struct intr_frame *f, int fd, unsigned position);
static void syscall_tell(struct intr_frame *f, int fd);
static void syscall_close(struct intr_frame *f, int fd);

bool syscall_translate_vaddr(const void *vaddr, bool write);
bool syscall_check_user_string(const char *str);
bool syscall_check_user_buffer (const char *str, int size, bool write); 
```

算法设计：

1.`syscall_translate_vaddr`：判断用户占指针是否为空或错误，若是则结束执行，不是则把指针给内核

```c++
/* Transfer user Vaddr to kernel vaddr
 * Return NULL if user Vaddr is invalid
 * */
bool
syscall_translate_vaddr(const void *vaddr, bool write){
  if (vaddr == NULL || !is_user_vaddr(vaddr))
    return false;
  ASSERT(vaddr != NULL);
  return pagedir_get_page(thread_current()->pagedir, vaddr) != NULL;

}
```

2.`syscall_check_user_string`：判断cmd字符串是否过长

```c++
bool
syscall_check_user_string(const char *ustr){
  if (!syscall_translate_vaddr(ustr, false))
    return false;
  int cnt = 0;
  while(*ustr != '\0'){
    if(cnt == 4095){
      puts("String to long, please make sure it is no longer than 4096 Bytes!\n");
      return false;
    }
    cnt++;
    ustr++;
    if (((int)ustr & PGMASK) == 0){
      if (!syscall_translate_vaddr(ustr, false))
        return false;
    }
  }
  return true;
}
```

3.`syscall_check_user_buffer`：判断缓冲是否溢出

```c++
bool
syscall_check_user_buffer(const char* ustr, int size, bool write){
  if (!syscall_translate_vaddr(ustr + size - 1, write))
    return false;

  size >>= 12;
  do{
    if (!syscall_translate_vaddr(ustr, write))
      return false;
    ustr += 1 << 12;
  }while(size--);
  return true;
}
```

4.`syscall_write` ：指定输出对象和内容，获取文件操作资源的`lock`通过调用文件系统中得`file_write`完成写文件操作

```c++
static void
syscall_write(struct intr_frame *f, int fd, const void* buffer, unsigned size){
  if (!syscall_check_user_buffer(buffer, size, false))
    thread_exit_with_return_value(f, -1);

  if (fd == STDIN_FILENO)
    thread_exit_with_return_value(f, -1);

  if (fd == STDOUT_FILENO)
    putbuf(buffer, size);
  else{
    struct file_handle* t = syscall_get_file_handle(fd);
    if (t != NULL && !inode_isdir(file_get_inode(t->opened_file))){
      lock_acquire(&filesys_lock);
      f->eax = (uint32_t)file_write(t->opened_file, (void*)buffer, size);
      lock_release(&filesys_lock);
    }
    else
      thread_exit_with_return_value(f, -1);
  }
}
```

5.`syscall_exit` ：结束当前的进程，如果是子进程，则将自身的返回值和状态告诉父进程，并设置返回值。调用`thread_exit`让线程结束

```c++
static void
syscall_exit(struct intr_frame *f, const int return_value){


  struct thread *cur = thread_current ();
  if (!cur->grandpa_died)
  {
    cur->message_to_grandpa->exited = true;
    cur->message_to_grandpa->return_value = return_value;
  }


  thread_exit_with_return_value(f, return_value);
}
```

6.`syscall_create` ：指定要创建的文件名和初始大小，获取文件操作资源的`lock`后调用 `filesys_create` 函数创建文件

```c++
static void
syscall_create(struct intr_frame *f, const char * name, unsigned initial_size){
  if (!syscall_check_user_string(name))
    thread_exit_with_return_value(f, -1);

  lock_acquire(&filesys_lock);
  f->eax = (uint32_t)filesys_create(name, initial_size);
  lock_release(&filesys_lock);
}
```

7.`syscall_open` ：指定文件名，获取文件操作资源的`lock`后调用`filesys_open`打开某个文件。此处需要一个`list`记录开过的文件

```c++
static void
syscall_open(struct intr_frame *f, const char* name){
  if (!syscall_check_user_string(name))
    thread_exit_with_return_value(f, -1);
  lock_acquire(&filesys_lock);
  struct file* tmp_file = filesys_open(name);
  lock_release(&filesys_lock);
  if (tmp_file == NULL){
    f->eax = (uint32_t)-1;
    return;
  }
```

8.`syscall_close` ：指定文件名，获取文件操作资源的`lock`后调用`filesys_close`关闭某个文件， 并且需要在已打开的文件`list`中移除这个文件句柄

```c++
static void
syscall_close(struct intr_frame *f, int fd){
  struct file_handle* t = syscall_get_file_handle(fd);
  if(t != NULL){
    lock_acquire(&filesys_lock);

#ifdef FILESYS
    if (inode_isdir(file_get_inode(t->opened_file)))
      dir_close(t->opened_dir);
#endif
    file_close(t->opened_file);
    lock_release(&filesys_lock);
    list_remove(&t->elem);
    free(t);
  }
  else
    thread_exit_with_return_value(f, -1);
}
```

9.`syscall_read` ：指定缓冲大小读取输入源中相应大小的内容，获取文件操作资源的`lock`后调用`file_read`读取内容

```c++
static void
syscall_read(struct intr_frame *f, int fd, const void* buffer, unsigned size){
  if (!syscall_check_user_buffer(buffer, size, true))
    thread_exit_with_return_value(f, -1);

  if (fd == STDOUT_FILENO)
    thread_exit_with_return_value(f, -1);

  uint8_t * str = buffer;
  if (fd == STDIN_FILENO){
    while(size-- != 0)
      *(char *)str++ = input_getc();
  }
  else{
    struct file_handle* t = syscall_get_file_handle(fd);
    if (t != NULL && !inode_isdir(file_get_inode(t->opened_file))){
      lock_acquire(&filesys_lock);
      f->eax = (uint32_t)file_read(t->opened_file, (void*)buffer, size);
      lock_release(&filesys_lock);
    }
    else
      thread_exit_with_return_value(f, -1);
  }
}
```

10.`syscall_filesize` ：指定文件句柄，获取文件操作资源的`lock`后调用`filesys_length`获得文件的大小

```c++
static void
syscall_filesize(struct intr_frame *f, int fd){
  struct file_handle* t = syscall_get_file_handle(fd);
  if(t != NULL){
    lock_acquire(&filesys_lock);
    f->eax = (uint32_t)file_length(t->opened_file);
    lock_release(&filesys_lock);
  }

  else
    thread_exit_with_return_value(f, -1);
}
```

11.`syscall_exec` ：指定cmd内容，获取文件操作资源的`lock`后调用`process_execute`实现。但是由于孩子与父亲之间需要进行充分的信息交流，所以借助信号量来保持孩子与父亲之间的同步问题，确保进程之间的正确调度

```c++
static void
syscall_exec (struct intr_frame *f, const char *cmd_line)
{
  if (!syscall_check_user_string(cmd_line))
    thread_exit_with_return_value(f, -1);
  lock_acquire(&filesys_lock);
  f->eax = (uint32_t)process_execute (cmd_line);
  lock_release(&filesys_lock);
  struct list_elem *e;
  struct thread *cur = thread_current ();
  struct child_message *l;
  for (e = list_begin (&cur->child_list); e != list_end (&cur->child_list); e = list_next (e))
  {
    l = list_entry (e, struct child_message, elem);
    if (l->tid == f->eax)
    {
      sema_down (l->sema_started);
      if (l->load_failed)
        f->eax = (uint32_t)-1;
      return;
    }
  }
}
```

12.`syscall_wait`：调用 `process_wait` 函数等子进程执行结束，`process_wait` 中，父进程会先判断子进程是否已返回，如果子进程已返回，父进程就会直接将该进程从它的子进程列表中移除。子进程若未返回，则父进程就会在子进程信号量上等待，直到子进程返回为止

```c++
static void
syscall_wait (struct intr_frame *f, pid_t pid)
{
  f->eax = (uint32_t)process_wait (pid);
}
```

13.`syscall_seek` ：指定位置，获取文件操作资源的`lock`后调用 `file_seek` 将文件指针跳转到文件中的指定位置

```c++
static void
syscall_seek(struct intr_frame *f, int fd, unsigned position){
  struct file_handle* t = syscall_get_file_handle(fd);
  if (t != NULL){
    lock_acquire(&filesys_lock);
    file_seek(t->opened_file, position);
    lock_release(&filesys_lock);
  }

  else
    thread_exit_with_return_value(f, -1);
}
```

14.`syscall_remove` ：指定文件名，获取文件操作资源的`lock`后调用 `filesys_remove` 删除文件

```c++
static void
syscall_remove(struct intr_frame *f, const char* name){
  if (!syscall_check_user_string(name))
    thread_exit_with_return_value(f, -1);
  lock_acquire(&filesys_lock);
  f->eax = (uint32_t)filesys_remove(name);
  lock_release(&filesys_lock);
}
```

15.`syscall_tell` ：指定句柄，获取文件操作资源的`lock`后调用 `file_tell`获取文件指针当前所在文件中的位置

```c++
static void
syscall_tell(struct intr_frame *f, int fd){
  struct file_handle* t = syscall_get_file_handle(fd);
  if (t != NULL && !inode_isdir(file_get_inode(t->opened_file))){
    lock_acquire(&filesys_lock);
    f->eax = (uint32_t)file_tell(t->opened_file);
    lock_release(&filesys_lock);
  }
  else
    thread_exit_with_return_value(f, -1);
}
```

16.`syscall_halt` ：调用`shutdown_power_off`函数实现关机

```c++
static void
syscall_halt(struct intr_frame *f){
  shutdown_power_off();
}
```

#### 4. Denying Writes to Executables

基本思想：

通过加锁控制文件访问

结构设计：

```c++
/* An open file. */
struct file 
  {
    struct inode *inode;        /* File's inode. */
    off_t pos;                  /* Current position. */
    bool deny_write;            /* Has file_deny_write() been called? */

#ifdef FILESYS
    struct dir* dir; // for process_load
#endif

  };
```

对文件通过`deny_write`变量判断是否可写

算法设计：

在 `thread_exec` 中，在`load` 操作时调用 `file_deny_write` 加锁，禁止写入文件。在 `thread_exit` 中，将这个可执行文件调用 `file_allow_write` 解锁，允许写入文件，再调用 `file_close` 关闭文件。

```c++
bool
load (const char *file_name, void (**eip) (void), void **esp)
{
  ···
  done:
  /* We arrive here whether the load is successful or not. */
  if (success)
  {
    t->exec_file = file;

#ifdef FILESYS
    t->current_dir = get_file_dir(file);
#endif

    file_deny_write(file);
  }
  return success;
}

void
file_close (struct file *file) 
{
  if (file != NULL)
    {
      file_allow_write (file);
   
#ifdef FILESYS
      dir_close(file->dir);
#endif

      inode_close (file->inode);
      free (file);
    }
}
```



## Project 4 FILESYS

------

### 一. 设计概览

* 设计文件索引和可拓展文件，实现文件大小的自动增长，改变文件组织方式，使用inode分散的在硬盘中记录文件。  
* 设计目录组织方式，实现分层目录空间，将目录设计成文件的格式，并使用固定长度的条目进行索引。
* 实现硬盘缓存，使用32KB的空间实现了基本的硬盘缓存，使用时钟算法进行缓存替换。

### 二. 设计细节

#### 1.索引与可拓展文件

基本思想：

可以假设文件系统分区不会大于8MB。必须支持与分区一样大的文件（减去元数据）。每个索引节点是存储在一个磁盘扇区中，限制了它的块指针数量可以包含。支持8MB的文件将需要您实施二次间接块。

1) 通过二级inode索引来实现文件索引，文件增长就是扩展inode。
2) 文件写入溢出时，申请新的block并修改inode，实现文件增长。

结构设计：

![image-20210202121527526](https://raw.githubusercontent.com/Reborn1998/Img/master/20210202121754.png)



算法设计：

```c++
bool
inode_create (block_sector_t sector, off_t length)
{
  struct inode_disk *disk_inode = NULL;
  bool success = false;

  ASSERT (length >= 0);

  /* If this assertion fails, the inode structure is not exactly
     one sector in size, and you should fix that. */
  ASSERT (sizeof *disk_inode == BLOCK_SECTOR_SIZE);

  disk_inode = calloc (1, sizeof *disk_inode);
  if (disk_inode != NULL)
    {
      disk_inode->length = length;
      disk_inode->magic = INODE_MAGIC;
      disk_inode->is_dir = false;
      if (free_map_allocate (1, &disk_inode->table)) 
        {
          cache_write (sector, disk_inode);
          cache_write (disk_inode->table, empty);
          
          if(length > 0)
            {
              block_sector_t *t1 = calloc(TABLE_SIZE, sizeof *t1);
              block_sector_t *t2 = calloc(TABLE_SIZE, sizeof *t2);
              
              int i, j;
              off_t t1_t = byte_to_t1(length - 1);
              off_t t2_t = byte_to_t2(length - 1);
          
              cache_read (disk_inode->table, t1);
              for(i = 0; i <= t1_t; i++)
                {
                  off_t r = (i == t1_t ? t2_t : TABLE_SIZE - 1);
              
                  if (!free_map_allocate (1, &t1[i]))
                  {
                    free(t1);
                    free(t2);
                    free (disk_inode);
                    return false;
                  }
                  cache_write(t1[i], empty);
                 
                  cache_read (t1[i], t2);
                  for(j = 0; j <= r; j++)
                    {
                      if (!free_map_allocate (1, &t2[j]))
                      {
                        free(t1);
                        free(t2);
                        free (disk_inode);
                        return false;
                      }
                      cache_write(t2[j], zeros);
                    }
                  cache_write (t1[i], t2);
                }
              cache_write (disk_inode->table, t1);
             
              free(t1);
              free(t2);
            }      
          success = true; 
        } 
      free (disk_inode);
    }
  return success;
}
```

1.在`inode_create`时，给t1，t2指针分配内存空间读取内容，若在`cache`中则从`cache`中获取，不在则从磁盘获取

2.根据文件大小length计算二级索引的个数和物理存储块最后一个块的数据大小

3.把t1作为二级索引指针，t2存储cache地址指针

4.在分配块时先通过位图判断是否有足够空间分配，若无足够空间则释放所有占有的空间，返回false

5.若空间足够分配，则分配的t2指向的存储区全部写入0，将t1的内容写入

