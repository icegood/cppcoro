# CppCoro - A coroutine library for C++

The 'cppcoro' library provides a set of general-purpose primitives for making use of the coroutines TS proposal described in [N4628](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/n4628.pdf).

These include:
* Coroutine Types
  * `task<T>`
  * `lazy_task<T>`
  * `shared_task<T>`
  * `shared_lazy_task<T>`
  * `generator<T>`
  * `recursive_generator<T>`
  * `async_generator<T>`
* Awaitable Types
  * `single_consumer_event`
  * `async_mutex`
  * `async_manual_reset_event` (coming)
* Functions
  * `when_all()` (coming)
* Cancellation
  * `cancellation_token`
  * `cancellation_source`
  * `cancellation_registration`
* Schedulers and I/O
  * `io_service`
  * `io_work_scope`
  * `file`, `readable_file`, `writable_file`
  * `read_only_file`, `write_only_file`, `read_write_file`

This library is an experimental library that is exploring the space of high-performance,
scalable asynchronous programming abstractions that can be built on top of the C++ coroutines
proposal.

It has been open-sourced in the hope that others will find it useful and that the C++ community
can provide feedback on it and ways to improve it.

# Class Details

## `task<T>`

The `task<T>` type represents a computation that completes asynchronously,
yielding either a result of type `T` or an exception.

API Overview:
```c++
// <cppcoro/task.hpp>
namespace cppcoro
{
  template<typename T = void>
  class task
  {
  public:
    using promise_type = <unspecified>;

    // Construct to a detached task.
    task() noexcept;

    task(task&& other) noexcept;
    task& operator=(task&&) noexcept;

    // Task must either be detached or ready.
    ~task();

    // task is move-only
    task(const task&) = delete;
    task& operator=(const task&) = delete;

    // Query if the task result is ready yet.
    bool is_ready() const noexcept;

    // Detach the task from the coroutine.
    //
    // You will not be able to retrieve the result of a task
    // once it has been detached.
    void detach() noexcept;

    // Result of 'co_await task' has type:
    // - void if T is void
    // - T if T is a reference type
    // - T& if T is not a reference and task is an l-value reference
    // - T&& if T is not a reference and task is an r-value reference
    //
    // Either returns the result of the task or rethrows the
    // uncaught exception if the coroutine terminated due to
    // an unhandled exception.
    // Attempting to await a detached task results in the
    // cppcoro::broken_promsise exception being thrown.
    <unspecified> operator co_await() const & noexcept;
    <unspecified> operator co_await() const && noexcept;

    // Await this instead of awaiting directly on the task if
    // you just want to synchronise with the task and don't
    // need the result.
    //
    // Result of 'co_await t.when_ready()' expression has type
    // 'void' and is guaranteed not to throw an exception.
    <unspecified> when_ready() noexcept;
  };
}
```

Example:
```c++
#include <cppcoro/task.hpp>

cppcoro::task<std::string> get_name(int id)
{
  auto database = co_await open_database();
  auto record = co_await database.load_record_by_id(id);
  co_return record.name;
}

cppcoro::task<> usage_example()
{
  // Calling get_name() creates a new task and starts it immediately.
  cppcoro::task<std::string> nameTask = get_name(123);

  // The get_name() coroutine is now potentially executing concurrently
  // with the current coroutine.

  // We can later co_await the task to suspend the current coroutine
  // until the task completes. The result of the co_await expression
  // is the return value of the get_name() function.
  std::string name = co_await nameTask;
}
```

You create a `task<T>` object by calling a coroutine function that returns
a `task<T>`.

When a coroutine that returns a `task<T>` is called the coroutine starts
executing immediately and continues until the coroutine reaches the first
suspend point or runs to completion. Execution then returns to the caller
and a `task<T>` value representing the asynchronous computation is returned
from the function call.

Note that while execution returns to the caller on the current thread,
the coroutine/task should be considered to be still executing concurrently
with the current thread.

You must either `co_await` the task or otherwise ensure it has run to
completion or you must call `task.detach()` to detach the task from
the asynchronous computation before the returned task object is destroyed.
Failure to do so will result in the task<T> destructor calling std::terminate().

If the task is not yet ready when it is co_await'ed then the awaiting coroutine
will be suspended and will later be resumed on the thread that completes
execution of the coroutine.

## `lazy_task<T>`

A lazy_task represents an asynchronous computation that is executed lazily in
that the execution of the coroutine does not start until the task is awaited.

A lazy_task<T> has lower overhead than task<T> as it does not need to use
atomic operations to synchronise between consumer and producer coroutines
since the consumer coroutine suspends before the producer coroutine starts.

Example:
```c++
#include <cppcoro/lazy_task.hpp>

cppcoro::lazy_task<int> count_lines(std::string path)
{
  auto file = co_await open_file_async(path);

  int lineCount = 0;

  char buffer[1024];
  size_t bytesRead;
  do
  {
    bytesRead = co_await file.read_async(buffer, sizeof(buffer));
    lineCount += std::count(buffer, buffer + bytesRead, '\n');
  } while (bytesRead > 0);
  
  co_return lineCount;
}

cppcoro::task<> usage_example()
{
  // Calling function creates a new lazy_task but doesn't start
  // executing the coroutine yet.
  cppcoro::lazy_task<int> countTask = count_lines("foo.txt");
  
  // ...
  
  // Coroutine is only started when we later co_await the task.
  int lineCount = co_await countTask;

  std::cout << "line count = " << lineCount << std::endl;
}
```

API Overview:
```c++
// <cppcoro/lazy_task.hpp>
namespace cppcoro
{
  template<typename T>
  class lazy_task
  {
  public:
    using promise_type = <unspecified>;
    lazy_task() noexcept;
    lazy_task(lazy_task&& other) noexcept;
    lazy_task(const lazy_task& other) = delete;
    lazy_task& operator=(lazy_task&& other);
    lazy_task& operator=(const lazy_task& other) = delete;
    bool is_ready() const noexcept;
    <unspecified> operator co_await() const & noexcept;
    <unspecified> operator co_await() const && noexcept;
    <unspecified> when_ready() const noexcept;
  };
}
```

Something to be aware of with `lazy_task<T>` is that if the coroutine
completes synchronously then the awaiting coroutine is resumed
from within the call to `await_suspend()`. If your compiler is not
able to guarantee tail-call optimisations for the `await_suspend()`
and `coroutine_handle<>::resume()` calls then this can result in
consumption of extra stack-space for each `co_await` of a `lazy_task`
that completes synchronously which can lead to stack-overflow if
performed in a loop.

Using `task<T>` is safer than `lazy_task<T>` with regards to potential
stack-overflow as it starts executing the task immediately on calling
the coroutine function and unwinds the stack back to the caller before
it can be awaited. The awaiting coroutine will continue execution without
suspending if the coroutine completed synchronously.

## `shared_task<T>`

The `shared_task<T>` class is a coroutine type that yields a single value
asynchronously.

The task value can be copied, allowing multiple references
to the result of the task to be created. It also allows multiple coroutines
to concurrently await the result.

API Summary
```c++
namespace cppcoro
{
  template<typename T = void>
  class shared_task
  {
  public:
    shared_task() noexcept;
    shared_task(const shared_task& other) noexcept;
    shared_task(shared_task&& other) noexcept;
    shared_task& operator=(const shared_task& other) noexcept;
    shared_task& operator=(shared_task&& other) noexcept;

    void swap(shared_task& other) noexcept;

    // Query if the task has completed and the result is ready.
    bool is_ready() const noexcept;

    // Returns an operation that when awaited will suspend the
    // current coroutine until the task completes and the result
    // is available.
    //
    // The type of the result of the 'co_await someTask' expression
    // is an l-value reference to the task's result value (unless T
    // is void in which case the expression has type 'void').
    // If the task completed with an unhandled exception then the
    // exception will be rethrown by the co_await expression.
    <unspecified> operator co_await() const noexcept;

    // Returns an operation that when awaited will suspend the
    // calling coroutine until the task completes and the result
    // is available.
    //
    // The result is not returned from the co_await expression.
    // This can be used to synchronise with the task without the
    // possibility of the co_await expression throwing an exception.
    <unspecified> when_ready() const noexcept;

  };

  template<typename T>
  bool operator==(const shared_task<T>& a, const shared_task<T>& b) noexcept;
  template<typename T>
  bool operator!=(const shared_task<T>& a, const shared_task<T>& b) noexcept;

  template<typename T>
  void swap(shared_task<T>& a, shared_task<T>& b) noexcept;

  // Wrap a task in a shared_task to allow multiple coroutines to concurrently
  // await the result.
  template<typename T>
  shared_task<T> make_shared_task(task<T> task);
}
```

All const-methods on `shared_task<T>` are safe to call concurrently with other const-methods on the same instance from multiple threads.
It is not safe to call non-const methods of `shared_task<T>` concurrently with any other method on the same instance of a `shared_task<T>`.

### Comparison to `task<T>`

The `shared_task<T>` class is similar to `task<T>` in that the task starts execution
immediately upon the coroutine function being called.

It differs from `task<T>` in that the resulting task object can
be copied, allowing multiple task objects to reference the same
asynchronous result. It also supports multiple coroutines concurrently
awaiting the result of the task.

The trade-off is that the result is always an l-value reference to the
result, never an r-value reference (since the result may be shared) which
may limit ability to move-construct the result into a local variable.
It also has a slightly higher run-time cost due to the need to maintain
a reference count and support multiple awaiters.

## `shared_lazy_task<T>`

The `shared_lazy_task<T>` class is a coroutine type that yields a single value
asynchronously.

It is 'lazy' in that execution of the task does not start until it is awaited by some
coroutine.

It is 'shared' in that the task value can be copied, allowing multiple references to
the result of the task to be created. It also allows multiple coroutines to
concurrently await the result.

API Summary
```c++
namespace cppcoro
{
  template<typename T = void>
  class shared_lazy_task
  {
  public:
    shared_lazy_task() noexcept;
    shared_lazy_task(const shared_lazy_task& other) noexcept;
    shared_lazy_task(shared_lazy_task&& other) noexcept;
    shared_lazy_task& operator=(const shared_lazy_task& other) noexcept;
    shared_lazy_task& operator=(shared_lazy_task&& other) noexcept;

    void swap(shared_lazy_task& other) noexcept;

    // Query if the task has completed and the result is ready.
    bool is_ready() const noexcept;

    // Returns an operation that when awaited will suspend the
    // current coroutine until the task completes and the result
    // is available.
    //
    // The type of the result of the 'co_await someTask' expression
    // is an l-value reference to the task's result value (unless T
    // is void in which case the expression has type 'void').
    // If the task completed with an unhandled exception then the
    // exception will be rethrown by the co_await expression.
    <unspecified> operator co_await() const noexcept;

    // Returns an operation that when awaited will suspend the
    // calling coroutine until the task completes and the result
    // is available.
    //
    // The result is not returned from the co_await expression.
    // This can be used to synchronise with the task without the
    // possibility of the co_await expression throwing an exception.
    <unspecified> when_ready() const noexcept;

  };

  template<typename T>
  bool operator==(const shared_lazy_task<T>& a, const shared_lazy_task<T>& b) noexcept;
  template<typename T>
  bool operator!=(const shared_lazy_task<T>& a, const shared_lazy_task<T>& b) noexcept;

  template<typename T>
  void swap(shared_lazy_task<T>& a, shared_lazy_task<T>& b) noexcept;

  // Wrap a lazy_task in a shared_lazy_task to allow multiple coroutines to concurrently
  // await the result.
  template<typename T>
  shared_lazy_task<T> make_shared_task(lazy_task<T> task);
}
```

All const-methods on `shared_lazy_task<T>` are safe to call concurrently with other const-methods on the same instance from multiple threads.
It is not safe to call non-const methods of `shared_lazy_task<T>` concurrently with any other method on the same instance of a `shared_lazy_task<T>`.

## `generator<T>`

A `generator` represents a coroutine type that produces a sequence of values of type, `T`, where values are produced lazily and synchronously.

The coroutine body is able to yield values of type `T` using the `co_yield` keyword.
Note, however, that the coroutine body is not able to use the `co_await` keyword; values must be produced synchronously.

For example:
```c++
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
```

When a coroutine function returning a `generator<T>` is called the coroutine is created initially suspended.
Execution of the coroutine enters the coroutine body when the `generator<T>::begin()` method is called and continues until
either the first `co_yield` statement is reached or the coroutine runs to completion.

If the returned iterator is not equal to the `end()` iterator then dereferencing the iterator will return a reference to the value passed to the `co_yield` statement.

Calling `operator++()` on the iterator will resume execution of the coroutine and continue until either the next `co_yield` point is reached or the coroutine runs to completion().

Any unhandled exceptions thrown by the coroutine will propagate out of the `begin()` or `operator++()` calls to the caller.

API Summary:
```c++
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

            // If the generator coroutine throws an unhandled exception before producing
            // the next element then the exception will propagate out of this call.
            iterator& operator++();

            reference operator*() const noexcept;
            pointer operator->() const noexcept;

            bool operator==(const iterator& other) const noexcept;
            bool operator!=(const iterator& other) const noexcept;
        };

        // Constructs to the empty sequence.
        generator() noexcept;

        generator(generator&& other) noexcept;
        generator& operator=(generator&& other) noexcept;
        
        generator(const generator& other) = delete;
        generator& operator=(const generator&) = delete;

        ~generator();

        // Starts executing the generator coroutine which runs until either a value is yielded
        // or the coroutine runs to completion or an unhandled exception propagates out of the
        // the coroutine.
        iterator begin();

        iterator end() noexcept;

        // Swap the contents of two generators.
        void swap(generator& other) noexcept;

    };

    template<typename T>
    void swap(generator<T>& a, generator<T>&b) noexcept;
}
```

## `recursive_generator<T>`

A `recursive_generator` is similar to a `generator` except that it is designed to more efficiently
support yielding the elements of a nested sequence as elements of an outer sequence.

In addition to being able to `co_yield` a value of type `T` you can also `co_yield` a value of type `recursive_generator<T>`.

When you `co_yield` a `recursive_generator<T>` value the all elements of the yielded generator are yielded as elements of the current generator.
The current coroutine is suspended until the consumer has finished consuming all elements of the nested generator, after which point execution
of the current coroutine will resume execution to produce the next element.

The benefit of `recursive_generator<T>` over `generator<T>` for iterating over recursive data-structures is that the `iterator::operator++()`
is able to directly resume the leaf-most coroutine to produce the next element, rather than having to resume/suspend O(depth) coroutines for each element.
The down-side is that there is additional overhead 

For example:
```c++
// Lists the immediate contents of a directory.
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
```

## `async_generator<T>`

An `async_generator` represents a coroutine type that produces a sequence of values of type, `T`, where values are produced lazily and values may be produced asynchronously.

The coroutine body is able to use both `co_await` and `co_yield` expressions.

Consumers of the generator can use a `for co_await` range-based for-loop to consume the values.

Example
```c++
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
  auto sequence = ticker(tp);
  for co_await(std::uint32_t i : sequence)
  {
    std::cout << "Tick " << i << std::endl;
  }
}
```

API Summary
```c++
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

      // Resumes the generator coroutine if suspended
      // Returns an operation object that must be awaited to wait
      // for the increment operation to complete.
      // If the coroutine runs to completion then the iterator
      // will subsequently become equal to the end() iterator.
      // If the coroutine completes with an unhandled exception then
      // that exception will be rethrown from the co_await expression.
      <unspecified> operator++() noexcept;

      // Dereference the iterator.
      pointer operator->() const noexcept;
      reference operator*() const noexcept;

      bool operator==(const iterator& other) const noexcept;
      bool operator!=(const iterator& other) const noexcept;
    };

    // Construct to the empty sequence.
    async_generator() noexcept;
    async_generator(const async_generator&) = delete;
    async_generator(async_generator&& other) noexcept;
    ~async_generator();

    async_generator& operator=(const async_generator&) = delete;
    async_generator& operator=(async_generator&& other) noexcept;

    void swap(async_generator& other) noexcept;

    // Starts execution of the coroutine and returns an operation object
    // that must be awaited to wait for the first value to become available.
    // The result of co_await'ing the returned object is an iterator that
    // can be used to advance to subsequent elements of the sequence.
    //
    // This method is not valid to be called once the coroutine has
    // run to completion.
    <unspecified> begin() noexcept;
    iterator end() noexcept;

  };
}
```

### Early termination of an async_generator

When the `async_generator` object is destructed it requests cancellation of the underlying coroutine.
If the coroutine has already run to completion or is currently suspended in a `co_yield` expression
then the coroutine is destroyed immediately. Otherwise, the coroutine will continue execution until
it either runs to completion or reaches the next `co_yield` expression.

When the coroutine frame is destroyed the destructors of all variables in scope at that point will be
executed to ensure the resources of the generator are cleaned up.

Note that the caller must ensure that the `async_generator` object must not be destroyed while a
consumer coroutine is executing a `co_await` expression waiting for the next item to be produced.

## `single_consumer_event`

This is a simple manual-reset event type that supports only a single
coroutine awaiting it at a time.
This can be used to 

API Summary:
```c++
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
    <unspecified> operator co_await() const noexcept;
  };
}
```

Example:
```c++
#include <cppcoro/single_consumer_event.hpp>

cppcoro::single_consumer_event event;
std::string value;

cppcoro::task<> consumer()
{
  co_await event;
  std::cout << value << std::endl;
}

void producer()
{
  value = "foo";
  event.set();
}
```

## async_mutex

Provides a simple mutual exclusion abstraction that allows the caller to 'co_await' the mutex
from within a coroutine to suspend the coroutine until the mutex lock is acquired.

The implementation is lock-free in that a coroutine that awaits the mutex will not
block the thread but will instead suspend

API Summary:
```c++
// <cppcoro/async_mutex.hpp>
namespace cppcoro
{
  class async_mutex_lock_operation;

  class async_mutex
  {
  public:
    async_mutex() noexcept;
    ~async_mutex();

    async_mutex(const async_mutex&) = delete;
    async_mutex& operator(const async_mutex&) = delete;

    bool try_lock() noexcept;
    async_mutex_lock_operation lock_async() noexcept;
    void unlock();
  };

  using async_mutex_lock_result = <implementation-defined>;

  class async_mutex_lock_operation
  {
  public:
    bool await_ready() const noexcept;
    bool await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
    async_mutex_lock_result await_resume() const noexcept;
  };

  class async_mutex_lock
  {
  public:
    // Takes ownership of the lock.
    async_mutex_lock(async_mutex_lock_result lockResult) noexcept;
    async_mutex_lock(async_mutex& mutex, std::adopt_lock_t) noexcept;

    async_mutex_lock(const async_mutex_lock&) = delete;
    async_mutex_lock& operator=(const async_mutex_lock&) = delete;

    // Releases the lock by calling unlock() on the mutex.
    ~async_mutex_lock();
  };
}
```

Example usage:
```c++
#include <cppcoro/async_mutex.hpp>
#include <cppcoro/task.hpp>
#include <set>
#include <string>

cppcoro::async_mutex mutex;
std::set<std::string> values;

cppcoro::task<> add_item(std::string value)
{
  cppcoro::async_mutex_lock lock = co_await mutex;
  values.insert(std::move(value));
}
```

## `cancellation_token`

A `cancellation_token` is a value that can be passed to a function that allows the caller to subsequently communicate a request to cancel the operation to that function.

To obtain a `cancellation_token` that is able to be cancelled you must first create a `cancellation_source` object.
The `cancellation_source::token()` method can be used to manufacture new `cancellation_token` values that are linked to that `cancellation_source` object.

When you want to later request cancellation of an operation you have passed a `cancellation_token` to
you can call `cancellation_source::request_cancellation()` on an associated `cancellation_source` object.

Functions can respond to a request for cancellation in one of two ways:
1. Poll for cancellation at regular intervals by calling either `cancellation_token::is_cancellation_requested()` or `cancellation_token::throw_if_cancellation_requested()`.
2. Register a callback to be executed when cancellation is requested using the `cancellation_registration` class.

API Summary:
```c++
namespace cppcoro
{
  class cancellation_source
  {
  public:
    // Construct a new, independently cancellable cancellation source.
    cancellation_source();

    // Construct a new reference to the same cancellation state.
    cancellation_source(const cancellation_source& other) noexcept;
    cancellation_source(cancellation_source&& other) noexcept;

    ~cancellation_source();

    cancellation_source& operator=(const cancellation_source& other) noexcept;
    cancellation_source& operator=(cancellation_source&& other) noexcept;

    bool is_cancellation_requested() const noexcept;
    bool can_be_cancelled() const noexcept;
    void request_cancellation();

    cancellation_token token() const noexcept;
  };

  class cancellation_token
  {
  public:
    // Construct a token that can't be cancelled.
    cancellation_token() noexcept;

    cancellation_token(const cancellation_token& other) noexcept;
    cancellation_token(cancellation_token&& other) noexcept;

    ~cancellation_token();

    cancellation_token& operator=(const cancellation_token& other) noexcept;
    cancellation_token& operator=(cancellation_token&& other) noexcept;

    bool is_cancellation_requested() const noexcept;
    void throw_if_cancellation_requested() const;

    // Query if this token can ever have cancellation requested.
    // Code can use this to take a more efficient code-path in cases
    // that the operation does not need to handle cancellation.
    bool can_be_cancelled() const noexcept;
  };

  // RAII class for registering a callback to be executed if cancellation
  // is requested on a particular cancellation token.
  class cancellation_registration
  {
  public:

    // Register a callback to be executed if cancellation is requested.
    // Callback will be called with no arguments on the thread that calls
    // request_cancellation() if cancellation is not yet requested, or
    // called immediately if cancellation has already been requested.
    // Callback must not throw an unhandled exception when called.
    template<typename CALLBACK>
    cancellation_registration(cancellation_token token, CALLBACK&& callback);

    cancellation_registration(const cancellation_registration& other) = delete;

    ~cancellation_registration();
  };

  class operation_cancelled : public std::exception
  {
  public:
    operation_cancelled();
    const char* what() const override;
  };
}
```

Example: Polling Approach
```c++
cppcoro::task<> do_something_async(cppcoro::cancellation_token token)
{
  // Explicitly define cancellation points within the function
  // by calling throw_if_cancellation_requested().
  token.throw_if_cancellation_requested();

  co_await do_step_1();

  token.throw_if_cancellation_requested();

  do_step_2();

  // Alternatively, you can query if cancellation has been
  // requested to allow yourself to do some cleanup before
  // returning.
  if (token.is_cancellation_requested())
  {
    display_message_to_user("Cancelling operation...");
    do_cleanup();
    throw cppcoro::operation_cancelled{};
  }

  do_final_step();
}
```

Example: Callback Approach
```c++
// Say we already have a timer abstraction that supports being
// cancelled but it doesn't support cancellation_tokens natively.
// You can use a cancellation_registration to register a callback
// that calls the existing cancellation API. e.g.
cppcoro::task<> cancellable_timer_wait(cppcoro::cancellation_token token)
{
  auto timer = create_timer(10s);

  cppcoro::cancellation_registration registration(token, [&]
  {
    // Call existing timer cancellation API.
    timer.cancel();
  });

  co_await timer;
}
```

## `io_service`

The `io_service` class provides an abstraction for processing I/O completion events
from asynchronous I/O operations.

When an asynchronous I/O operation completes, the coroutine that was awaiting
that operation will be resumed on an I/O thread inside a call to one of the
event-processing methods: `process_events()`, `process_pending_events()`,
`process_one_event()` or `process_one_pending_event()`.

The `io_service` class does not manage any I/O threads.
You must ensure that some thread calls one of the event-processing methods for coroutines awaiting I/O
completion events to be dispatched. This can either be a dedicated thread that calls `process_events()`
or mixed in with some other event loop (e.g. a UI event loop) by periodically polling for new events
via a call to `process_pending_events()` or `process_one_pending_event()`.

This allows integration of the `io_service` event-loop with other event loops, such as a user-interface event loop.

You can multiplex processing of events across multiple threads by having multiple threads call
`process_events()`. You can specify a hint as to the maximum number of threads to have actively
processing events via an optional `io_service` constructor parameter.

On Windows, the implementation makes use of the Windows I/O Completion Port facility to dispatch
events to I/O threads in a scalable manner.

API Summary:
```c++
namespace cppcoro
{
  class io_service
  {
  public:

    class schedule_operation;
    class timed_schedule_operation;

    io_service();
    io_service(std::uint32_t concurrencyHint);

    io_service(io_service&&) = delete;
    io_service(const io_service&) = delete;
    io_service& operator=(io_service&&) = delete;
    io_service& operator=(const io_service&) = delete;

    ~io_service();

    // Scheduler methods

    [[nodiscard]]
    schedule_operation schedule() noexcept;

    template<typename REP, typename RATIO>
    [[nodiscard]]
    timed_schedule_operation schedule_after(
      std::chrono::duration<REP, RATIO> delay,
      cppcoro::cancellation_token cancellationToken = {}) noexcept;

    // Event-loop methods
    //
    // I/O threads must call these to process I/O events and execute
    // scheduled coroutines.

    std::uint64_t process_events();
    std::uint64_t process_pending_events();
    std::uint64_t process_one_event();
    std::uint64_t process_one_pending_event();

    // Request that all threads processing events exit their event loops.
    void stop() noexcept;

    // Query if some thread has called stop()
    bool is_stop_requested() const noexcept;

    // Reset the event-loop after a call to stop() so that threads can
    // start processing events again.
    void reset();

    // Reference-counting methods for tracking outstanding references
    // to the io_service.
    //
    // The io_service::stop() method will be called when the last work
    // reference is decremented.
    //
    // Use the io_work_scope RAII class to manage calling these methods on
    // entry-to and exit-from a scope.
    void notify_work_started() noexcept;
    void notify_work_finished() noexcept;

  };

  class io_service::schedule_operation
  {
  public:
    schedule_operation(const schedule_operation&) noexcept;
    schedule_operation& operator=(const schedule_operation&) noexcept;

    bool await_ready() const noexcept;
    void await_suspend(std::experimental::coroutine_handle<> awaiter) noexcept;
    void await_resume() noexcept;
  };

  class io_service::timed_schedule_operation
  {
  public:
    timed_schedule_operation(timed_schedule_operation&&) noexcept;

    timed_schedule_operation(const timed_schedule_operation&) = delete;
    timed_schedule_operation& operator=(const timed_schedule_operation&) = delete;
    timed_schedule_operation& operator=(timed_schedule_operation&&) = delete;

    bool await_ready() const noexcept;
    void await_suspend(std::experimental::coroutine_handle<> awaiter);
    void await_resume();
  };

  class io_work_scope
  {
  public:

    io_work_scope(io_service& ioService) noexcept;

    io_work_scope(const io_work_scope& other) noexcept;
    io_work_scope(io_work_scope&& other) noexcept;

    ~io_work_scope();

    io_work_scope& operator=(const io_work_scope& other) noexcept;
    io_work_scope& operator=(io_work_scope&& other) noexcept;

    io_service& service() const noexcept;
  };

}
```

Example:
```c++
#include <cppcoro/task.hpp>
#include <cppcoro/lazy_task.hpp>
#include <cppcoro/io_service.hpp>
#include <cppcoro/read_only_file.hpp>

#include <experimental/filesystem>
#include <memory>
#include <algorithm>
#include <iostream>

namespace fs = std::experimental::filesystem;

cppcoro::lazy_task<std::uint64_t> count_lines(cppcoro::io_service& ioService, fs::path path)
{
  auto file = cppcoro::read_only_file::open(ioService, path);

  constexpr size_t bufferSize = 4096;
  auto buffer = std::make_unique<std::uint8_t[]>(bufferSize);

  std::uint64_t newlineCount = 0;

  for (std::uint64_t offset = 0, fileSize = file.size(); offset < fileSize;)
  {
    const auto bytesToRead = static_cast<size_t>(
      std::min<std::uint64_t>(bufferSize, fileSize - offset));

    const auto bytesRead = co_await file.read(offset, buffer.get(), bytesToRead);

    newlineCount += std::count(buffer.get(), buffer.get() + bytesRead, '\n');

    offset += bytesRead;
  }

  co_return newlineCount;
}

cppcoro::task<> run(cppcoro::io_service& ioService)
{
  cppcoro::io_work_scope ioScope(ioService);

  auto lineCount = co_await count_lines(ioService, fs::path{"foo.txt"});

  std::cout << "foo.txt has " << lineCount << " lines." << std::endl;;
}

int main()
{
  cppcoro::io_service ioService;

  auto t = run(ioService);

  // Run until all io_work_scope objects have destructed.
  ioService.process_events();

  return 0;
}
```

### `io_service` as a scheduler

An `io_sevice` class implements the interfaces for the `Scheduler` and `DelayedScheduler` concepts.

This allows a coroutine to suspend execution on the current thread and schedule itself for resumption
on an I/O thread associated with a particular `io_service` object.

Example:
```c++
cppcoro::task<> do_something(cppcoro::io_service& ioService)
{
  // Coroutine starts execution on the thread of the caller.

  // A coroutine can transfer execution to an I/O thread by awaiting the
  // result of io_service::schedule().
  co_await ioService.schedule();

  // At this point, the coroutine is now executing on an I/O thread.

  // A coroutine can also perform a delayed-schedule that will suspend
  // the coroutine for a specified duration of time before scheduling
  // it for resumption on an I/O thread
  co_await ioService.schedule_after(100ms);

  // At this point, the coroutine is executing on a potentially different I/O thread.
}
```

## `file`, `readable_file`, `writable_file`

These types are abstract base-classes for performing concrete file I/O.

API Summary:
```c++
namespace cppcoro
{
  class file_read_operation;
  class file_write_operation;

  class file
  {
  public:

    virtual ~file();

    std::uint64_t size() const;

  protected:

    file(file&& other) noexcept;

  };

  class readable_file : public virtual file
  {
  public:

    [[nodiscard]]
    file_read_operation read(
      std::uint64_t offset,
      void* buffer,
      std::size_t byteCount,
      cancellation_token ct = {}) const noexcept;

  };

  class writable_file : public virtual file
  {
  public:

    void set_size(std::uint64_t fileSize);

    [[nodiscard]]
    file_write_operation write(
      std::uint64_t offset,
      const void* buffer,
      std::size_t byteCount,
      cancellation_token ct = {}) noexcept;

  };

  class file_read_operation
  {
  public:

    file_read_operation(file_read_operation&& other) noexcept;

    bool await_ready() const noexcept;
    bool await_suspend(std::experimental::coroutine_handle<> awaiter);
    std::size_t await_resume();

  };

  class file_write_operation
  {
  public:

    file_write_operation(file_write_operation&& other) noexcept;

    bool await_ready() const noexcept;
    bool await_suspend(std::experimental::coroutine_handle<> awaiter);
    std::size_t await_resume();

  };
}
```

## `read_only_file`, `write_only_file`, `read_write_file`

These types represent concrete file I/O classes.

API Summary:
```c++
namespace cppcoro
{
  class read_only_file : public readable_file
  {
  public:

    [[nodiscard]]
    static read_only_file open(
      io_service& ioService,
      const std::experimental::filesystem::path& path,
      file_share_mode shareMode = file_share_mode::read,
      file_buffering_mode bufferingMode = file_buffering_mode::default_);

  };

  class write_only_file : public writable_file
  {
  public:

    [[nodiscard]]
    static write_only_file open(
      io_service& ioService,
      const std::experimental::filesystem::path& path,
      file_open_mode openMode = file_open_mode::create_or_open,
      file_share_mode shareMode = file_share_mode::none,
      file_buffering_mode bufferingMode = file_buffering_mode::default_);

  };

  class read_write_file : public readable_file, public writable_file
  {
  public:

    [[nodiscard]]
    static read_write_file open(
      io_service& ioService,
      const std::experimental::filesystem::path& path,
      file_open_mode openMode = file_open_mode::create_or_open,
      file_share_mode shareMode = file_share_mode::none,
      file_buffering_mode bufferingMode = file_buffering_mode::default_);

  };
}
```

All `open()` functions throw `std::system_error` on failure.

# Concepts

## `Scheduler` concept

A `Scheduler` is a concept that allows scheduling execution of coroutines within some execution context.

```c++
concept Scheduler
{
  <awaitable-type> schedule();
}
```

Given a type, `S`, that implements the `Scheduler` concept, and an instance, `s`, of type `S`:
* The `s.schedule()` method returns an awaitable-type such that `co_await s.schedule()`
  will unconditionally suspend the current coroutine and schedule it for resumption on the
  execution context associated with the scheduler, `s`.
* The result of the `co_await s.schedule()` expression has type `void`.

```c++
cppcoro::task<> f(Scheduler& scheduler)
{
  // Execution of the coroutine is initially on the caller's execution context.

  // Suspends execution of the coroutine and schedules it for resumption on
  // the scheduler's execution context.
  co_await scheduler.schedule();

  // At this point the coroutine is now executing on the scheduler's
  // execution context.
}
```

## `DelayedScheduler` concept

A `DelayedScheduler` is a concept that allows a coroutine to schedule itself for execution on
the scheduler's execution context after a specified duration of time has elapsed.

```c++
concept DelayedScheduler : Scheduler
{
  template<typename REP, typename RATIO>
  <awaitable-type> schedule_after(std::chrono::duration<REP, RATIO> delay);

  template<typename REP, typename RATIO>
  <awaitable-type> schedule_after(
    std::chrono::duration<REP, RATIO> delay,
    cppcoro::cancellation_token cancellationToken);
}
```

Given a type, `S`, that implements the `DelayedScheduler` and an instance, `s` of type `S`:
* The `s.schedule_after(delay)` method returns an object that can be awaited
  such that `co_await s.schedule_after(delay)` suspends the current coroutine
  for a duration of `delay` before scheduling the coroutine for resumption on
  the execution context associated with the scheduler, `s`.
* The `co_await s.schedule_after(delay)` expression has type `void`.

# Building

This library makes use of the [Cake build system](https://github.com/lewissbaker/cake) (no, not the [C# one](http://cakebuild.net/)).

This library currently requires Visual Studio 2017 or later and the Windows 10 SDK.

Support for Clang ([#3](https://github.com/lewissbaker/cppcoro/issues/3)) and Linux ([#15](https://github.com/lewissbaker/cppcoro/issues/15)) is planned.

## Prerequisites

The Cake build-system is implemented in Python and requires Python 2.7 to be installed.

Ensure Python 2.7 interpreter is in your PATH and available as 'python'.

Ensure Visual Studio 2017 is installed (preferably the latest update).

You can also use an experimental version of the Visual Studio compiler by downloading a NuGet package from https://vcppdogfooding.azurewebsites.net/ and unzipping the .nuget file to a directory.
Just update the `config.cake` file to point at the unzipped location by modifying and uncommenting the following line:
```python
nugetPath = None # r'C:\Path\To\VisualCppTools.14.0.25224-Pre'
```

Ensure that you have the Windows 10 SDK installed.
It will use the latest Windows 10 SDK and Universal C Runtime version by default.

## Cloning the repository

The cppcoro repository makes use of git submodules to pull in the source for the Cake build system.

This means you need to pass the `--recursive` flag to the `git clone` command. eg.
```
c:\Code> git clone --recursive https://github.com/lewissbaker/cppcoro
```

If you have already cloned cppcoro, then you should update the submodules after pulling changes.
```
c:\Code\cppcoro> git submodule update --init --recursive
```

## Building from the command-line

To build from the command-line just run 'cake.bat' in the workspace root.

eg.
```
C:\cppcoro> cake.bat
Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
Compiling test\main.cpp
Compiling test\main.cpp
Compiling test\main.cpp
Compiling test\main.cpp
Linking build\windows_x86_msvc14.10_debug\test\run.exe
Linking build\windows_x64_msvc14.10_optimised\test\run.exe
Linking build\windows_x86_msvc14.10_optimised\test\run.exe
Linking build\windows_x64_msvc14.10_debug\test\run.exe
Generating code
Finished generating code
Generating code
Finished generating code
Build succeeded.
Build took 0:00:02.419.
```

By default this will build all projects with all build variants and execute the unit-tests.
You can narrow what is built by passing additional command-line arguments.
eg.
```
c:\cppcoro> cake.bat release=debug architecture=x64 lib/build.cake
Building with C:\Users\Lewis\Code\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
Archiving build\windows_x64_msvc14.10_debug\lib\cppcoro.lib
Build succeeded.
Build took 0:00:00.321.
```

You can run `cake --help` to list available command-line options.

## Building Visual Studio project files

To develop from within Visual Studio you can build .vcproj/.sln files by running `cake -p`.

eg.
```
c:\cppcoro> cake.bat -p
Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='debug', platform='windows', architecture='x64', compilerFamily='msvc', compiler='msvc14.10')
Building with C:\cppcoro\config.cake - Variant(release='optimised', platform='windows', architecture='x86', compilerFamily='msvc', compiler='msvc14.10')
Generating Solution build/project/cppcoro.sln
Generating Project build/project/cppcoro_tests.vcxproj
Generating Filters build/project/cppcoro_tests.vcxproj.filters
Generating Project build/project/cppcoro.vcxproj
Generating Filters build/project/cppcoro.vcxproj.filters
Build succeeded.
Build took 0:00:00.247.
```

When you build these projects from within Visual Studio it will call out to cake to perform the compilation.
