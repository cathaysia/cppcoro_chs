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

协程类 ``shared_task<T>`` 以异步的方式产生单个值。

它也是 **延迟计算** ，仅当有协程 await 它的时候才开始执行计算。

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

一个 ``gengrator`` 用于产生一系列类型为 T 的值。值的产生是延迟计算和异步的。

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

async_generator<T>
========================================

异步类型
****************************************


single_consumer_event
========================================

single_consumer_async_auto_reset_event
========================================

async_mutex
========================================

async_manual_reset_event
========================================

async_auto_reset_event
========================================

async_latch
========================================

sequence_barrier
========================================

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
========================================

网络
****************************************


socket
========================================
ip_address, ipv4_address, ipv6_address
========================================
ip_endpoint, ipv4_endpoint, ipv6_endpoint
========================================

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


Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
