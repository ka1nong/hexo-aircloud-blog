---
title: brpc框架学习之线程库
date: 2018-08-09 20:04:39
tags:
---
<h2>bthread简介</h2>

bthread是brpc使用的M:N线程库，目的是在提高程序的并发度的同时，降低编码难度，并在核数日益增多的CPU上提供更好的scalability（扩展性）和cache locality（局部缓存）。”M:N“是指M个bthread会映射至N个pthread，一般M远大于N。由于linux当下的pthread实现(NPTL: Native POSIX Thread Library 原生POSIX线程库)是1:1的，M个bthread也相当于映射至N个LWP(轻量级进程)。bthread的前身是Distributed Process(DP)中的fiber，一个N:1的合作式线程库，等价于event-loop库，但写的是同步代码

<h4>什么是NPTL</h4>
提到NPTL与LinuxThreads的区别，很多人会说到1:1和M:N，即用户态线程与内核态进程的比例问题，用户态看到的那个线程与内核态的进程本来就是同一个进程。如下图：
![](./nptl.jpg)
图中，红色线以上是用户态，下面是内核态。左边虚线框内是单线程的，右边是多线程的（2个线程）。

在Linux的眼里，不会区别进程和线程，在它眼里只有task_struct。task_struct是用来描述进程（线程）的结构体，其中会记录一切关于进程的信息。内核做任务调度的时候，仅仅是选择一个task_struct而已。这个task_struct是属于进程还是线程的，内核并不关心。正因为如此，多线程下的无差别调度才能保证（所以，如果你想拖慢别人程序的速度，你可以创建大量的线程）

参考：https://blog.csdn.net/joseph_1118/article/details/47275869

<h2>bthread结构</h2>

bthread主要的类有两个TaskControl和TaskGroup，TaskControl采用单例模式，对TaskGroup进行管理，TaskGroup则1：1对应pthread

<h4>TaskControl详解</h4>

在程序启动时初始化线程库，调用get_or_new_task_control函数
	```c++
    inline TaskControl* get_or_new_task_control() {
        butil::atomic<TaskControl*>* p = (butil::atomic<TaskControl*>*)&g_task_control;
        TaskControl* c = p->load(butil::memory_order_consume);
        if (c != NULL) {
            return c;
        }
        BAIDU_SCOPED_LOCK(g_task_control_mutex);
        c = p->load(butil::memory_order_consume);
        if (c != NULL) {
            return c;
        }
        c = new (std::nothrow) TaskControl;
        if (NULL == c) {
            return NULL;
        }
        int concurrency = FLAGS_bthread_min_concurrency > 0 ?
            FLAGS_bthread_min_concurrency :
            FLAGS_bthread_concurrency;
        if (c->init(concurrency) != 0) {
            LOG(ERROR) << "Fail to init g_task_control";
            delete c;
            return NULL;
        }
        p->store(c, butil::memory_order_release);
        return c;
	}
    ```

memory_order_consume的语义是后面依赖此原子变量的访存指令勿重排至此条指令之前，memory_order_release的语义是前面访存指令勿重排至此条指令之后。当此条指令的结果对其他线程可见后，之前的所有指令都可见
* 首先检测当前TaskControl是否已经存在，存在则返回，不存在则继续
* 使用锁线程多线程的同时访问
* 二次检测TaskControl是否已经存在，存在则返回，不存在则继续
* new TaskControl,并使用需要创建的pthread个数初始化它，默认个数是9，支持可配
* 初始化成功则赋值到全局唯一实例g_task_control

TaskControl类的功能主要分为两类，一类是创建并管理TaskGroup,一类是统计TaskGroup相关的数据
<h5>管理TaskGroup</h5>

如下代码所示，在init函数中创建pthread

	```c++
    int TaskControl::init(int concurrency) {
        if (_concurrency != 0) {
            LOG(ERROR) << "Already initialized";
            return -1;
        }
        if (concurrency <= 0) {
            LOG(ERROR) << "Invalid concurrency=" << concurrency;
            return -1;
        }
        _concurrency = concurrency;

        // Make sure TimerThread is ready.
        if (get_or_create_global_timer_thread() == NULL) {
            LOG(ERROR) << "Fail to get global_timer_thread";
            return -1;
        }

        _workers.resize(_concurrency);   
        for (int i = 0; i < _concurrency; ++i) {
            const int rc = pthread_create(&_workers[i], NULL, worker_thread, this);
            if (rc) {
                LOG(ERROR) << "Fail to create _workers[" << i << "], " << berror(rc);
                return -1;
            }
        }

        // Wait for at least one group is added so that choose_one_group()
        // never returns NULL.
        // TODO: Handle the case that worker quits before add_group
        while (_ngroup == 0) {
            usleep(100);  // TODO: Elaborate
        }
        return 0;
	}
    ```
在worker_thread中创建TaskGroup并加起加入到TaskControl的成员变量_groups中，如果创建成功则调用TaskGroup的run_main_task函数，TaskGroup进入等待任务的循环中
	```c++
    void* TaskControl::worker_thread(void* arg) {
            TaskControl* c = static_cast<TaskControl*>(arg);
            TaskGroup* g = c->create_group();
            TaskStatistics stat;
            if (NULL == g) {
                LOG(ERROR) << "Fail to create TaskGroup in pthread=" << pthread_self();
                return NULL;
            }
            BT_VLOG << "Created worker=" << pthread_self()
                    << " bthread=" << g->main_tid();

            tls_task_group = g;
            c->_nworkers << 1;
            g->run_main_task();

            stat = g->main_stat();
            BT_VLOG << "Destroying worker=" << pthread_self() << " bthread="
                    << g->main_tid() << " idle=" << stat.cputime_ns / 1000000.0
                    << "ms uptime=" << g->current_uptime_ns() / 1000000.0 << "ms";
            tls_task_group = NULL;
            g->destroy_self();
            c->_nworkers << -1;
            return NULL;
    }

    TaskGroup* TaskControl::create_group() {
        TaskGroup* g = new (std::nothrow) TaskGroup(this);
        if (NULL == g) {
            LOG(FATAL) << "Fail to new TaskGroup";
            return NULL;
        }
        if (g->init(FLAGS_task_group_runqueue_capacity) != 0) {
            LOG(ERROR) << "Fail to init TaskGroup";
            delete g;
            return NULL;
        }
        if (_add_group(g) != 0) {
            delete g;
            return NULL;
        }
        return g;
    }
    ```
tls_task_group的定义是TaskGroup* tls_task_group;使用了tls，每个跟pthread唯一对应的TaskGroup的指针都存储在tls_task_group中

<h5>线程库的使用</h5>

使用线程库主要有两个函数，bthread_start_urgent和bthread_start_background，前者对应比较紧急的调度，后则则是不那么紧急的调度，换句话说，前者执行的时间快过后者
	```c++
    int bthread_start_urgent(bthread_t* __restrict tid,
                         const bthread_attr_t* __restrict attr,
                         void * (*fn)(void*),
                         void* __restrict arg) {
        bthread::TaskGroup* g = bthread::tls_task_group;
        if (g) {
            // start from worker
            return bthread::TaskGroup::start_foreground(&g, tid, attr, fn, arg);
        }
        return bthread::start_from_non_worker(tid, attr, fn, arg);
    }

    int bthread_start_background(bthread_t* __restrict tid,
                                 const bthread_attr_t* __restrict attr,
                                 void * (*fn)(void*),
                                 void* __restrict arg) {
        bthread::TaskGroup* g = bthread::tls_task_group;
        if (g) {
            // start from worker
            return g->start_background<false>(tid, attr, fn, arg);
        }
        return bthread::start_from_non_worker(tid, attr, fn, arg);
    }
	```
从上面实现可以看到，首先取tls中存储的tls_task_group判断当前线程是否存在TaskGroup，如果存在则调用该taskGroup的相关接口，不存在则统一调用start_from_non_worker。所以这里可以看出，如果当前不存在tls_task_group，则不管是紧急还是不紧急，都是统一的流程，而主线程是没有tls_task_group的，所以主线程不管是哪个接口都是不紧急的。
start_from_non_workder会调用choose_one_group函数选择一个TaskGroup。

	```c++
    TaskGroup* TaskControl::choose_one_group() {
        const size_t ngroup = _ngroup.load(butil::memory_order_acquire);
        if (ngroup != 0) {
            return _groups[butil::fast_rand_less_than(ngroup)];
        }
        CHECK(false) << "Impossible: ngroup is 0";
        return NULL;
	}
    ```
这里用了一种什么鬼的方式随机选择了一个taskGroup.选取之后调用start_background<true>函数

	```c++
    template <bool REMOTE>
	int TaskGroup::start_background(bthread_t* __restrict th,
                                const bthread_attr_t* __restrict attr,
                                void * (*fn)(void*),
                                void* __restrict arg) {
        if (__builtin_expect(!fn, 0)) {
            return EINVAL;
        }
        const int64_t start_ns = butil::cpuwide_time_ns();
        const bthread_attr_t using_attr = (attr ? *attr : BTHREAD_ATTR_NORMAL);
        butil::ResourceId<TaskMeta> slot;
        TaskMeta* m = butil::get_resource(&slot);
        if (__builtin_expect(!m, 0)) {
            return ENOMEM;
        }
        CHECK(m->current_waiter.load(butil::memory_order_relaxed) == NULL);
        m->stop = false;
        m->interrupted = false;
        m->about_to_quit = false;
        m->fn = fn;
        m->arg = arg;
        CHECK(m->stack == NULL);
        m->attr = using_attr;
        m->local_storage = LOCAL_STORAGE_INIT;
        m->cpuwide_start_ns = start_ns;
        m->stat = EMPTY_STAT;
        m->tid = make_tid(*m->version_butex, slot);
        *th = m->tid;
        if (using_attr.flags & BTHREAD_LOG_START_AND_FINISH) {
            LOG(INFO) << "Started bthread " << m->tid;
        }
        _control->_nbthreads << 1;
        if (REMOTE) {
            ready_to_run_remote(m->tid, (using_attr.flags & BTHREAD_NOSIGNAL));
        } else {
            ready_to_run(m->tid, (using_attr.flags & BTHREAD_NOSIGNAL));
        }
        return 0;
	}
    ```
* __builtin_expect是可以编译器优化的if指令，使用__builtin_expect比使用if快一些
* 获取当前时间，ns级别
* get_resource管理全局的内存，这里获取分配一个TaskMeta数据
* 

TaskMeta
<h5>统计TaskGroup</h5>
