CppCoro - 一个 C++ 协程库
########################################

**cppcoro** 提供了大量通用原语以便于使用 TS `N4680 <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf>`_ 提案描述的协程

这包括：

.. contents::

这个库是一个实验性的库，它在 C++ 协程的基础上探索高性能、可伸缩的异步编程抽象。

这个库是开源的，并希望其他人能够发现它的优点并提供反馈以及改进它的方法

它要求编译器支持协程：

- Windows + Visual Studio 2017 |Windows 构建状态|

.. |Windows 构建状态| image:: https://ci.appveyor.com/api/projects/status/github/lewissbaker/cppcoro?branch=master&svg=true&passingText=master%20-%20OK&failingText=master%20-%20Failing&pendingText=master%20-%20Pending
   :target:  https://ci.appveyor.com/project/lewissbaker/cppcoro/branch/master

- Linux + Clang 5.0/6.0 + libc++ |Linux 构建状态|

.. |Linux 构建状态| image:: https://travis-ci.org/lewissbaker/cppcoro.svg?branch=master
   :target: https://travis-ci.org/lewissbaker/cppcoro

Linux 除了 ``io_context`` 和 文件 I/O 相关的类没有实现外，其余功能都是可用的。（详情请查询  `#15 <https://github.com/lewissbaker/cppcoro/issues/15>`_ ）

协程类型
****************************************

task<T>
========================================

一个 task 代表一个异步的延迟计算，因为它直到被等待时才会开始运行（而不是创建时）。

比如：

.. code-block:: cpp

   #include <cppcoro/read_only_file.hpp>
   #include <cppcoro/task.hpp>

   cppcoro::task<int> count_lines(std::string path)
   {
   auto file = co_await cppcoro::read_only_file::open(path);

   int lineCount = 0;

   char buffer[1024];
   size_t bytesRead;
   std::uint64_t offset = 0;
   do
   {
      bytesRead = co_await file.read(offset, buffer, sizeof(buffer));
      lineCount += std::count(buffer, buffer + bytesRead, '\n');
      offset += bytesRead;
   } while (bytesRead > 0);

   co_return lineCount;
   }

   cppcoro::task<> usage_example()
   {
   // 调用函数创建一个新的 task ，但是 task 这时候并没有开始运行
   // executing the coroutine yet.
   cppcoro::task<int> countTask = count_lines("foo.txt");

   // ...

   // 协程仅在被 co_await 后才开始运行
   int lineCount = co_await countTask;

   std::cout << "line count = " << lineCount << std::endl;
   }

API 概览：

.. code-block:: cpp

   // <cppcoro/task.hpp>
   namespace cppcoro
   {
   template<typename T>
   class task
   {
   public:

      using promise_type = <unspecified>;
      using value_type = T;

      task() noexcept;

      task(task&& other) noexcept;
      task& operator=(task&& other);

      // task 是一个只能被移动的类型
      task(const task& other) = delete;
      task& operator=(const task& other) = delete;

      // 查询 task 是否已经准备好了
      bool is_ready() const noexcept;

      // 等待 task 运行完毕
      // 如果 task 执行时出现了未捕获的异常，那么将其重新抛出
      // 
      // 如果任务还没有准备好，那么挂起直到 task 完成，如果 task is_ready() ，那么直接返回异步计算的结果
      Awaiter<T&> operator co_await() const & noexcept;
      Awaiter<T&&> operator co_await() const && noexcept;

      // 返回一个 awaitable 对象，以便于 co_await 暂停协程直至 task 完成
      //
      // 与表达式 ``co_await t`` 不同的是，``co_await t.when_ready()`` 中的 when_ready() 是同步的，而且不会返回计算结果，或者是重新抛出异常
      Awaitable<void> when_ready() const noexcept;
   };

   template<typename T>
   void swap(task<T>& a, task<T>& b);

   // Creates a task that yields the result of co_await'ing the specified awaitable.
   //
   // This can be used as a form of type-erasure of the concrete awaitable, allowing
   // different awaitables that return the same await-result type to be stored in
   // the same task<RESULT> type.
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t>
   task<RESULT> make_task(AWAITABLE awaitable);
   }

你可以通过调用返回值为 ``task<T>`` 的函数来产生 ``task<T>`` 对象。

协程必须包含 ``co_await`` 或 ``co_return`` 。

.. note:: 

   ``task<T>`` 也许不使用 ``co_yield`` 关键字

当一个返回值为 ``task<T>`` 的协程被调用时，如果需要，将会获得一个协程帧。协程的参数在协程帧内完成捕获。然后协程将会在函数起始处被暂停，并返回一个用于表示异步计算结果的 ``task<T>`` 。

在 ``task<T>`` 值被 ``co_await`` 后，协程将开始执行计算。然后等待的协成将被挂起，然后执行与 ``task<T>`` 相关联的协程。挂起的协程将在其关联的 ``task<T>`` 被 ``co_await`` 后唤醒。此线程要么 ``co_return``，要么跑出异常并被终止。

如果 task 已经完成，那么再次等待它将获得已经计算的结果，而不会重新计算。

如果 ``task`` 对象在被 co_await 之前就被销毁了，那么协程永远不会被执行。析构函数只会简单地释放协程帧内由于捕获参数而分配的内存。

shared_task<T>
========================================

协程类 ``shared_task<T>`` 以异步、惰性的方式产生单个值。

所谓 **惰性**，就是指仅当有协程 await 它的时候才开始执行计算。

它是 **共享** 的：task 允许被拷贝； task 的返回值可以被多次引用； task 可以被多个协程 await。

它在第一次被 co_await 时执行，其余 await 的协程要么挂起进入等待队列，要么直接拿到已经计算的结果。

如果协程由于 await task 被挂起，那么其将会在 task 完成计算后被恢复。task 要么 ``co_return`` 一个值，要么抛出一个未捕获的异常。

API 摘要:

.. code-block:: cpp

   namespace cppcoro
   {
   template<typename T = void>
   class shared_task
   {
   public:

      using promise_type = <unspecified>;
      using value_type = T;

      shared_task() noexcept;
      shared_task(const shared_task& other) noexcept;
      shared_task(shared_task&& other) noexcept;
      shared_task& operator=(const shared_task& other) noexcept;
      shared_task& operator=(shared_task&& other) noexcept;

      void swap(shared_task& other) noexcept;

      // 查询 task 是否已经完成，而且计算结果已经可用
      bool is_ready() const noexcept;

      // 返回一个 operation，其将会在被 await 时挂起当前协程，直到 task 完成而且计算结果可用。
      //
      // 表达式 ``co_await someTask`` 的结果是一个指向 task 计算结果的左值引用（除非 T 的类
      // 型是 void，此时这个表达式的结果类型为 void）
      // 未捕获异常将被 co_await 表达式重新抛出
      Awaiter<T&> operator co_await() const noexcept;
      // 返回一个 operation，其将会在被 await 时挂起当前协程，直到 task 完成而且计算结果可用
      // 此 co_await 表达式不会返回任何值。
      // 此表达式可用于与 task 进行同步而不用担心抛出异常。
      Awaiter<void> when_ready() const noexcept;

   };

   template<typename T>
   bool operator==(const shared_task<T>& a, const shared_task<T>& b) noexcept;
   template<typename T>
   bool operator!=(const shared_task<T>& a, const shared_task<T>& b) noexcept;

   template<typename T>
   void swap(shared_task<T>& a, shared_task<T>& b) noexcept;

   // 包装一个可 await 的值，以允许多个协程同时等待它
   template<
      typename AWAITABLE,
      typename RESULT = typename awaitable_traits<AWAITABLE>::await_result_t>
   shared_task<RESULT> make_shared_task(AWAITABLE awaitable);
   }

const 限定的函数可以安全地在多个线程中调用，是线程安全的，但是非 const 限定的函数则不然。

.. note:: 

   与 ``task<T>`` 相比而言：

   - 都是延迟计算：计算只在被 co_await 后才开始。
   - task<T> 的结果不允许被拷贝，是仅移动的。而 shared_task 可以被拷贝和移动
   - 由于可能被共享，shared_task 的结果总是左值，这可能导致局部变量无法进行“移动构造”，而且由于需要维护引用计数，其运行时成本略高。

gengrator<T>
========================================

一个 :abbr:`生成器 (Gengrator)` 用于产生一系列类型为 T 的值。值的产生是惰性和异步的。

协程可以使用 ``co_yield`` 来产生一个类型为 T 的值。但是协程内无法使用 co_await 关键字。值的产生必须是同步的。

.. code-block:: cpp

   cppcoro::generator<const std::uint64_t> fibonacci()
   {
   std::uint64_t a = 0, b = 1;
   while (true)
   {
      co_yield b;
      auto tmp = a;
      a = b;
      b += tmp;
   }
   }

   void usage()
   {
   for (auto i : fibonacci())
   {
      if (i > 1'000'000) break;
      std::cout << i << std::endl;
   }
   }

当一个返回值为``generator<T>`` 的协程函数被调用后，其会被立即挂起。直到 ``generator<T>::begin()`` 函数被调用。在 ``co_yield`` 达到终点或者协程完成后不在产生值。

如果返回的迭代器与 ``end()`` 不相等，那么对迭代器进行解引用将会返回“传递给 ``co_yield`` ”的值。

调用 ``operator()++`` 将会恢复协程的运行，直至协程结束或 co_yield 不再产生新的值。

API 摘要:

.. code-block:: cpp

   namespace cppcoro
   {
      template<typename T>
      class generator
      {
      public:

         using promise_type = <unspecified>;

         class iterator
         {
         public:
               using iterator_category = std::input_iterator_tag;
               using value_type = std::remove_reference_t<T>;
               using reference = value_type&;
               using pointer = value_type*;
               using difference_type = std::size_t;

               iterator(const iterator& other) noexcept;
               iterator& operator=(const iterator& other) noexcept;

               // 如果异常在 co_yield 之前参数，异常将被重新抛出。
               iterator& operator++();

               reference operator*() const noexcept;
               pointer operator->() const noexcept;

               bool operator==(const iterator& other) const noexcept;
               bool operator!=(const iterator& other) const noexcept;
         };

         // 构造一个空的序列
         generator() noexcept;

         generator(generator&& other) noexcept;
         generator& operator=(generator&& other) noexcept;

         generator(const generator& other) = delete;
         generator& operator=(const generator&) = delete;

         ~generator();

         // 开始执行协成，直至 co_yield 不再产生新的值或协程结束或未捕获异常被抛出
         iterator begin();

         iterator end() noexcept;

         // 交换两个生成器
         void swap(generator& other) noexcept;

      };

      template<typename T>
      void swap(generator<T>& a, generator<T>& b) noexcept;

      // 以 source 为基础，对其每个元素调用一次 func 来产生一个新的序列。
      template<typename FUNC, typename T>
      generator<std::invoke_result_t<FUNC, T&>> fmap(FUNC func, generator<T> source);
   }

recursive_generator<T>
========================================

相比生成器而言， :abbr:`递归生成器 (Recursive Generator)` 能够生成嵌套在外部元素的序列。

``co_yield`` 除了可以生成类型 T 的元素外，还能生成一个元素为 T 的递归生成器。

当你 ``co_yield`` 一个递归生成器时，其将被作为当前元素的子元素。当前线程将被挂起，直至递归生成器的所有元素被生成。然后被恢复，等待请求下一个元素。

相比普通生成器而言，在迭代嵌套数据结构时，递归生成器能够通过 ``iterator::operator++()`` 直接恢复边缘协程以产生下一个元素，而不必未每个元素都暂停/恢复一个 O(depth) 的协程。缺点是有额外开销。

例如：

.. code-block:: cpp

   // 列出当前目录的内容
   cppcoro::generator<dir_entry> list_directory(std::filesystem::path path);

   cppcoro::recursive_generator<dir_entry> list_directory_recursive(std::filesystem::path path)
   {
   for (auto& entry : list_directory(path))
   {
      co_yield entry;
      if (entry.is_directory())
      {
         co_yield list_directory_recursive(entry.path());
      }
   }
   }

.. important::

   对 ``recursive_generator<T>`` 应用 ``fmap()`` 操作时，将产生 ``generator<U>`` 类型，而不是 ``recursive_generator<U>`` 类型。这是因为通常在递归上下文中不使用 ``fmap`` 操作，我们避免递归生成器的格外开销。

async_generator<T>
========================================

:abbr:`异步生成器 (Async Generator)` 用于产生类型为 T 的序列。值是惰性、异步产生的。

此协程体内既可以使用 ``co_wait`` 也可以使用 ``co_yield``

可以通过基于 ``for co_await`` 来处理数据序列。

比如：

.. code-block:: cpp

   cppcoro::async_generator<int> ticker(int count, threadpool& tp)
   {
   for (int i = 0; i < count; ++i)
   {
      co_await tp.delay(std::chrono::seconds(1));
      co_yield i;
   }
   }

   cppcoro::task<> consumer(threadpool& tp)
   {
   auto sequence = ticker(10, tp);
   for co_await(std::uint32_t i : sequence)
   {
      std::cout << "Tick " << i << std::endl;
   }
   }

API 摘要:

.. code-block:: cpp

   // <cppcoro/async_generator.hpp>
   namespace cppcoro
   {
   template<typename T>
   class async_generator
   {
   public:

      class iterator
      {
      public:
         using iterator_tag = std::forward_iterator_tag;
         using difference_type = std::size_t;
         using value_type = std::remove_reference_t<T>;
         using reference = value_type&;
         using pointer = value_type*;

         iterator(const iterator& other) noexcept;
         iterator& operator=(const iterator& other) noexcept;

         // 如果协程被挂起，则恢复它
         // 返回一个 operation ，其必须被 await 至自增操作结束
         // 最后返回的迭代器与 end() 相等
         // 若有未捕获异常，则将其抛出
         Awaitable<iterator&> operator++() noexcept;

         // 对迭代器解引用
         pointer operator->() const noexcept;
         reference operator*() const noexcept;

         bool operator==(const iterator& other) const noexcept;
         bool operator!=(const iterator& other) const noexcept;
      };

      // 构造一个空的序列
      async_generator() noexcept;
      async_generator(const async_generator&) = delete;
      async_generator(async_generator&& other) noexcept;
      ~async_generator();

      async_generator& operator=(const async_generator&) = delete;
      async_generator& operator=(async_generator&& other) noexcept;

      void swap(async_generator& other) noexcept;

      // 开始执行协程并返回起一个 operation，其必须被 await 至第一个元素可用
      // co_wait 获得的是一个迭代器对象，并且可用其来推动协程的执行
      // 在协程执行结束后，调用此函数是非法的
      Awaitable<iterator> begin() noexcept;
      iterator end() noexcept;

   };

   template<typename T>
   void swap(async_generator<T>& a, async_generator<T>& b);

   // 以 source 为基础，对其每个元素调用一次 func 来产生一个新的序列。
   template<typename FUNC, typename T>
   async_generator<std::invoke_result_t<FUNC, T&>> fmap(FUNC func, async_generator<T> source);
   }

.. important:: 

   异步迭代器的提前终止：

   当异步生成器被析构时，它将请求取消协程。如果协程已经运行结束，或者在 ``co_yield`` 表达式中挂起，那么协程立即被销毁。否则协程将继续执行，直到它运行结束或到达下一个 ``co_yield`` 表达式。

   在协程被销毁时，其作用于内的所有变量也将被销毁，以确保完全清理资源。

   在协程使用 ``co_await`` 等待下一个元素时，调用者必须确保此时异步生成器不被销毁。

异步类型
****************************************

single_consumer_event
========================================

这是一个简单的手动重置事件类型。其在同一时间内只能被一个协程等待。

API 摘要：

.. code-block:: cpp

   // <cppcoro/single_consumer_event.hpp>
   namespace cppcoro
   {
   class single_consumer_event
   {
   public:
      single_consumer_event(bool initiallySet = false) noexcept;
      bool is_set() const noexcept;
      void set();
      void reset() noexcept;
      Awaiter<void> operator co_await() const noexcept;
   };
   }

例子：

.. code-block:: cpp

   #include <cppcoro/single_consumer_event.hpp>

   cppcoro::single_consumer_event event;
   std::string value;

   cppcoro::task<> consumer()
   {
   // 协程将会在此处挂起，直至有线程调用 event.set()
   // 比如下面的 producer() 函数
   co_await event;

   std::cout << value << std::endl;
   }

   void producer()
   {
   value = "foo";

   // This will resume the consumer() coroutine inside the call to set()
   // if it is currently suspended.
   event.set();
   }

single_consumer_async_auto_reset_event
========================================

async_mutex
========================================

提供了一个简单的互斥抽象，允许调用者在协程中 ``co_await`` 互斥锁，挂起协程，直到获得互斥锁。

这个实现是无锁的，因为等待互斥锁的协程不会阻塞线程，而是挂起协程，然后前一个锁持有者通过调用 unlock() 来恢复它。

API 摘要：

.. code-block:: cpp

   // <cppcoro/async_mutex.hpp>
   namespace cppcoro
   {
   class async_mutex_lock;
   class async_mutex_lock_operation;
   class async_mutex_scoped_lock_operation;

   class async_mutex
   {
   public:
      async_mutex() noexcept;
      ~async_mutex();

      async_mutex(const async_mutex&) = delete;
      async_mutex& operator(const async_mutex&) = delete;

      bool try_lock() noexcept;
      async_mutex_lock_operation lock_async() noexcept;
      async_mutex_scoped_lock_operation scoped_lock_async() noexcept;
      void unlock();
   };

   class async_mutex_lock_operation
   {
   public:
      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      void await_resume() const noexcept;
   };

   class async_mutex_scoped_lock_operation
   {
   public:
      bool await_ready() const noexcept;
      bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
      [[nodiscard]] async_mutex_lock await_resume() const noexcept;
   };

   class async_mutex_lock
   {
   public:
      // 获得锁的所有权
      async_mutex_lock(async_mutex& mutex, std::adopt_lock_t) noexcept;

      // 移交锁的所有权
      async_mutex_lock(async_mutex_lock&& other) noexcept;

      async_mutex_lock(const async_mutex_lock&) = delete;
      async_mutex_lock& operator=(const async_mutex_lock&) = delete;

      // 通过调用 unlock() 来解锁
      ~async_mutex_lock();
   };
   }

例如：

.. code-block:: cpp

   #include <cppcoro/async_mutex.hpp>
   #include <cppcoro/task.hpp>
   #include <set>
   #include <string>

   cppcoro::async_mutex mutex;
   std::set<std::string> values;

   cppcoro::task<> add_item(std::string value)
   {
   cppcoro::async_mutex_lock lock = co_await mutex.scoped_lock_async();
   values.insert(std::move(value));
   }

async_manual_reset_event
========================================

async_auto_reset_event
========================================

async_latch
========================================

:abbr:`异步锁存器 (Async Latch)` 是一个同步原语，用于异步等待一个计数器递减为零。

此锁存器是一次性的。一旦由于计数器变为零导致锁存器进入 ready 状态，其将保持此状态直至销毁。

API 摘要：

.. code-block:: cpp

   // <cppcoro/async_latch.hpp>
   namespace cppcoro
   {
   class async_latch
   {
   public:

      // 用指定的计数初始化此锁存器
      async_latch(std::ptrdiff_t initialCount) noexcept;

      // 查询计数是否已经变为零
      bool is_ready() const noexcept;

      // 将计数减少 n
      // 当此函数的调用导致计数为零时，所有等待的协程将被恢复
      // 计数器减到负值是未定义行为
      void count_down(std::ptrdiff_t n = 1) noexcept;

      // 等待锁存器状态变为 ready
      // 如果计数没有变为零，则所有等待的协程将被挂起，直至由于调用 count_down() 导致计数变为零。
      // 如果计数已经变为零，则不会被挂起
      Awaiter<void> operator co_await() const noexcept;

   };
   }

sequence_barrier
========================================

:abbr:`顺序墙 (Sequence Barrier)` 是一个同步原语，允许一个生产者和多个消费者之间通过一个单调递增的数字序列来协作。

生产者通过发布一组单调递增的数来推进序列，消费者则可以查询生产者最后发布的数，并可以等待至特定的数被发布。

顺序墙可以充当线程安全的生产-消费环形缓存区的游标。

有关更多信息，参见 LMAX Disruptor 模式：https://lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf

API 摘要：

.. code-block:: cpp

   namespace cppcoro
   {
   template<typename SEQUENCE = std::size_t,
            typename TRAITS = sequence_traits<SEQUENCE>>
   class sequence_barrier
   {
   public:
      sequence_barrier(SEQUENCE initialSequence = TRAITS::initial_sequence) noexcept;
      ~sequence_barrier();

      SEQUENCE last_published() const noexcept;

      // 等待至序列号 targetSequence 被发布
      //
      // 如果操作没有同步地完成，则等待的协程将被特定的 scheduler 唤醒。否则协程将直接顺序执行而无需等待
      //
      // co_await 表达式将在 last_published() 后被唤醒，最后发布的数就是其 targetSequence
      template<typename SCHEDULER>
      [[nodiscard]]
      Awaitable<SEQUENCE> wait_until_published(SEQUENCE targetSequence,
                                                SCHEDULER& scheduler) const noexcept;

      void publish(SEQUENCE sequence) noexcept;
   };
   }

multi_producer_sequencer
========================================

single_producer_sequencer
========================================

函数
****************************************

sync_wait()
========================================
when_all()
========================================
when_all_ready()
========================================
fmap()
========================================
schedule_on()
========================================
resume_on()
========================================

撤销
****************************************


cancellation_token
========================================
cancellation_source
========================================
cancellation_registration
========================================

I/O 调度
****************************************


static_thread_pool
========================================
io_service and io_work_scope
========================================
file, readable_file, writable_file
========================================

read_only_file, write_only_file, read_write_file
==================================================

网络
****************************************


socket
========================================
ip_address, ipv4_address, ipv6_address
========================================
ip_endpoint, ipv4_endpoint, ipv6_endpoint
==========================================

元函数
****************************************


is_awaitable<T>
========================================
awaitable_traits<T>
========================================

概念
****************************************


Awaitable<T>
========================================

Awaiter<T>
========================================
Scheduler
========================================
DelayedScheduler
========================================

Indices and tables
****************************************


* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
