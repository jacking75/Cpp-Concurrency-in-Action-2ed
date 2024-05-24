## 스레드 풀
  
* 스레드 풀은 일반적으로 스레드 수를 나타내는 매개 변수로 초기화되며 내부적으로 작업을 저장할 대기열이 필요하다. 다음은 가장 간단한 스레드 풀 구현 중 하나이다.
  
```cpp
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <utility>

class ThreadPool {
 public:
  explicit ThreadPool(std::size_t n) {
    for (std::size_t i = 0; i < n; ++i) {
      std::thread{[this] {
        std::unique_lock<std::mutex> l(m_);
        while (true) {
          if (!q_.empty()) {
            auto task = std::move(q_.front());
            q_.pop();
            l.unlock();
            task();
            l.lock();
          } else if (done_) {
            break;
          } else {
            cv_.wait(l);
          }
        }
      }}.detach();
    }
  }

  ~ThreadPool() {
    {
      std::lock_guard<std::mutex> l(m_);
      done_ = true;  // cv_.wait는 done_ 판단을 사용하므로 잠금이 추가된다
    }
    cv_.notify_all();
  }

  template <typename F>
  void submit(F&& f) {
    {
      std::lock_guard<std::mutex> l(m_);
      q_.emplace(std::forward<F>(f));
    }
    cv_.notify_one();
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  bool done_ = false;
  std::queue<std::function<void()>> q_;
};
```

* 제출된 작업을 매개 변수로 만들려면 훨씬 더 번거로울 것이다  
  
```cpp
template <class F, class... Args>
auto ThreadPool::submit(F&& f, Args&&... args) {
  using RT = std::invoke_result_t<F, Args...>;
  // std::packaged_task는 복사 구문을 허용하지 않으며 람다에 직접 전달할 수 없다
  // 따라서 std::shared_ptr을 사용해야 한다
  auto task = std::make_shared<std::packaged_task<RT()>>(
      std::bind(std::forward<F>(f), std::forward<Args>(args)...));
  // 그러나 std::bind는 실제 매개변수를 값으로 복사하므로 이 구현에서는 작업의 실제 매개변수가 이동 전용 유형일 수 없다
  {
    std::lock_guard<std::mutex> l(m_);
    q_.emplace([task]() { (*task)(); });  // std::packaged_task로 전달할 포인터 캡처
  }
  cv_.notify_one();
  return task->get_future();
}
```

* 스레드 풀의 책 구현은 모두 데드 루프에서 시간 조각을 전송하기 위해 [std::this_thread::yield](https://en.cppreference.com/w/cpp/thread/yield)를 사용한다
  
```cpp
#include <atomic>
#include <functional>
#include <thread>
#include <vector>

#include "concurrent_queue.hpp"

class ThreadPool {
 public:
  ThreadPool() {
    std::size_t n = std::thread::hardware_concurrency();
    try {
      for (std::size_t i = 0; i < n; ++i) {
        threads_.emplace_back(&ThreadPool::worker_thread, this);
      }
    } catch (...) {
      done_ = true;
      for (auto& x : threads_) {
        if (x.joinable()) {
          x.join();
        }
      }
      throw;
    }
  }

  ~ThreadPool() {
    done_ = true;
    for (auto& x : threads_) {
      if (x.joinable()) {
        x.join();
      }
    }
  }

  template <typename F>
  void submit(F f) {
    q_.push(std::function<void()>(f));
  }

 private:
  void worker_thread() {
    while (!done_) {
      std::function<void()> task;
      if (q_.try_pop(task)) {
        task();
      } else {
        std::this_thread::yield();
      }
    }
  }

 private:
  std::atomic<bool> done_ = false;
  ConcurrentQueue<std::function<void()>> q_;
  std::vector<std::thread> threads_;   
};
```
  
* 이 경우의 문제점은 스레드 풀이 유휴 상태이면 타임 슬라이스를 무한정 전송하여 다음 책에 있는 스레드 풀의 CPU 사용량 테스트에서 볼 수 있듯이 100% CPU 사용량을 초래한다는 것이다
  
![](images/8-1.png)

* 동일한 작업에 대해 이전에 구현된 스레드 풀을 사용한 테스트 결과

![](images/8-2.png)
  
* 다음은 여전히 책에 나와 있는 내용이다.
* 이 스레드 풀은 매개변수가 없고 반환값이 없는 함수만 실행할 수 있으며, 데드락이 발생할 수 있으므로 매개변수는 없지만 반환값이 있는 함수를 실행할 수 있도록 하려면 다음과 같이 해야 한다. 반환값을 얻으려면 함수를 [std::packaged_task](https://en.cppreference.com/w/cpp/thread/packaged_task)에 전달한 다음 대기열에 추가하고 [std::packaged_task](https://en.cppreference.com/w/cpp/thread/packaged_task)를 [std::future](https://en.cppreference.com/w/cpp/thread/future)에 반환한다. [std::packaged_task](https://en.cppreference.com/w/cpp/thread/packaged_task)는 이동 전용 유형이고, [std::function](https://en.cppreference.com/w/cpp/utility/functional/function)는 저장된 함수 인스턴스를 복사-구성할 수 있어야 하므로 이동 전용 타입을 지원하는 함수 래퍼 클래스, 즉 호출 연산이 있는 타입 삭제 클래스를 구현해야 한다.

```cpp
#include <memory>
#include <utility>

class FunctionWrapper {
 public:
  FunctionWrapper() = default;

  FunctionWrapper(const FunctionWrapper&) = delete;

  FunctionWrapper& operator=(const FunctionWrapper&) = delete;

  FunctionWrapper(FunctionWrapper&& rhs) noexcept
      : impl_(std::move(rhs.impl_)) {}

  FunctionWrapper& operator=(FunctionWrapper&& rhs) noexcept {
    impl_ = std::move(rhs.impl_);
    return *this;
  }

  template <typename F>
  FunctionWrapper(F&& f) : impl_(new ImplType<F>(std::move(f))) {}

  void operator()() const { impl_->call(); }

 private:
  struct ImplBase {
    virtual void call() = 0;
    virtual ~ImplBase() = default;
  };

  template <typename F>
  struct ImplType : ImplBase {
    ImplType(F&& f) noexcept : f_(std::move(f)) {}
    void call() override { f_(); }

    F f_;
  };

 private:
  std::unique_ptr<ImplBase> impl_;
};
```
  
* `std::function<void()>`를 이 래퍼 클래스로 바꾼다
  
```cpp
#include <atomic>
#include <future>
#include <thread>
#include <type_traits>
#include <vector>

#include "concurrent_queue.hpp"
#include "function_wrapper.hpp"

class ThreadPool {
 public:
  ThreadPool() {
    std::size_t n = std::thread::hardware_concurrency();
    try {
      for (std::size_t i = 0; i < n; ++i) {
        threads_.emplace_back(&ThreadPool::worker_thread, this);
      }
    } catch (...) {
      done_ = true;
      for (auto& x : threads_) {
        if (x.joinable()) {
          x.join();
        }
      }
      throw;
    }
  }

  ~ThreadPool() {
    done_ = true;
    for (auto& x : threads_) {
      if (x.joinable()) {
        x.join();
      }
    }
  }

  template <typename F>
  std::future<std::invoke_result_t<F>> submit(F f) {
    std::packaged_task<std::invoke_result_t<F>()> task(std::move(f));
    std::future<std::invoke_result_t<F>> res(task.get_future());
    q_.push(std::move(task));
    return res;
  }

 private:
  void worker_thread() {
    while (!done_) {
      FunctionWrapper task;
      if (q_.try_pop(task)) {
        task();
      } else {
        std::this_thread::yield();
      }
    }
  }

 private:
  std::atomic<bool> done_ = false;
  ConcurrentQueue<FunctionWrapper> q_;
  std::vector<std::thread> threads_;  
};
```
  
* 스레드 풀에 작업을 추가하면 작업 대기열의 경쟁이 증가하는데 이는 잠금 없는 대기열을 사용하면 피할 수 있지만 핑퐁 캐싱 문제가 있다. 이렇게 하려면 작업 대기열을 스레드에 독립적인 로컬 대기열과 전역 대기열로 분할하여 스레드 대기열이 비어 있으면 전역 대기열에서 작업을 가져올 수 있도록 해야 한다.
  
```cpp
#include <atomic>
#include <future>
#include <memory>
#include <queue>
#include <thread>
#include <type_traits>
#include <vector>

#include "concurrent_queue.hpp"
#include "function_wrapper.hpp"

class ThreadPool {
 public:
  ThreadPool() {
    std::size_t n = std::thread::hardware_concurrency();
    try {
      for (std::size_t i = 0; i < n; ++i) {
        threads_.emplace_back(&ThreadPool::worker_thread, this);
      }
    } catch (...) {
      done_ = true;
      for (auto& x : threads_) {
        if (x.joinable()) {
          x.join();
        }
      }
      throw;
    }
  }

  ~ThreadPool() {
    done_ = true;
    for (auto& x : threads_) {
      if (x.joinable()) {
        x.join();
      }
    }
  }

  template <typename F>
  std::future<std::invoke_result_t<F>> submit(F f) {
    std::packaged_task<std::invoke_result_t<F>()> task(std::move(f));
    std::future<std::invoke_result_t<F>> res(task.get_future());
    if (local_queue_) {
      local_queue_->push(std::move(task));
    } else {
      pool_queue_.push(std::move(task));
    }
    return res;
  }

 private:
  void worker_thread() {
    local_queue_.reset(new std::queue<FunctionWrapper>);
    while (!done_) {
      FunctionWrapper task;
      if (local_queue_ && !local_queue_->empty()) {
        task = std::move(local_queue_->front());
        local_queue_->pop();
        task();
      } else if (pool_queue_.try_pop(task)) {
        task();
      } else {
        std::this_thread::yield();
      }
    }
  }

 private:
  std::atomic<bool> done_ = false;
  ConcurrentQueue<FunctionWrapper> pool_queue_;
  inline static thread_local std::unique_ptr<std::queue<FunctionWrapper>>
      local_queue_;
  std::vector<std::thread> threads_;
};
```
  
* 이렇게 하면 데이터 경합을 피할 수 있지만 작업이 고르지 않게 분산되면 다른 스레드가 할 일이 없는 동안 스레드가 로컬 대기열에 많은 작업을 보유하게 될 수 있으므로 작동하지 않는 스레드가 다른 스레드에서 작업을 가져올 수 있어야 한다.
    
```cpp
#include <atomic>
#include <deque>
#include <future>
#include <memory>
#include <mutex>
#include <thread>
#include <type_traits>
#include <vector>

#include "concurrent_queue.hpp"
#include "function_wrapper.hpp"

class WorkStealingQueue {
 public:
  WorkStealingQueue() = default;

  WorkStealingQueue(const WorkStealingQueue&) = delete;

  WorkStealingQueue& operator=(const WorkStealingQueue&) = delete;

  void push(FunctionWrapper f) {
    std::lock_guard<std::mutex> l(m_);
    q_.push_front(std::move(f));
  }

  bool empty() const {
    std::lock_guard<std::mutex> l(m_);
    return q_.empty();
  }

  bool try_pop(FunctionWrapper& res) {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return false;
    }
    res = std::move(q_.front());
    q_.pop_front();
    return true;
  }

  bool try_steal(FunctionWrapper& res) {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return false;
    }
    res = std::move(q_.back());
    q_.pop_back();
    return true;
  }

 private:
  std::deque<FunctionWrapper> q_;
  mutable std::mutex m_;
};

class ThreadPool {
 public:
  ThreadPool() {
    std::size_t n = std::thread::hardware_concurrency();
    try {
      for (std::size_t i = 0; i < n; ++i) {
        work_stealing_queue_.emplace_back(
            std::make_unique<WorkStealingQueue>());
        threads_.emplace_back(&ThreadPool::worker_thread, this, i);
      }
    } catch (...) {
      done_ = true;
      for (auto& x : threads_) {
        if (x.joinable()) {
          x.join();
        }
      }
      throw;
    }
  }

  ~ThreadPool() {
    done_ = true;
    for (auto& x : threads_) {
      if (x.joinable()) {
        x.join();
      }
    }
  }

  template <typename F>
  std::future<std::invoke_result_t<F>> submit(F f) {
    std::packaged_task<std::invoke_result_t<F>()> task(std::move(f));
    std::future<std::invoke_result_t<F>> res(task.get_future());
    if (local_queue_) {
      local_queue_->push(std::move(task));
    } else {
      pool_queue_.push(std::move(task));
    }
    return res;
  }

 private:
  bool pop_task_from_local_queue(FunctionWrapper& task) {
    return local_queue_ && local_queue_->try_pop(task);
  }

  bool pop_task_from_pool_queue(FunctionWrapper& task) {
    return pool_queue_.try_pop(task);
  }

  bool pop_task_from_other_thread_queue(FunctionWrapper& task) {
    for (std::size_t i = 0; i < work_stealing_queue_.size(); ++i) {
      std::size_t index = (index_ + i + 1) % work_stealing_queue_.size();
      if (work_stealing_queue_[index]->try_steal(task)) {
        return true;
      }
    }
    return false;
  }

  void worker_thread(std::size_t index) {
    index_ = index;
    local_queue_ = work_stealing_queue_[index_].get();
    while (!done_) {
      FunctionWrapper task;
      if (pop_task_from_local_queue(task) || pop_task_from_pool_queue(task) ||
          pop_task_from_other_thread_queue(task)) {
        task();
      } else {
        std::this_thread::yield();
      }
    }
  }

 private:
  std::atomic<bool> done_ = false;
  ConcurrentQueue<FunctionWrapper> pool_queue_;
  std::vector<std::unique_ptr<WorkStealingQueue>> work_stealing_queue_;
  std::vector<std::thread> threads_;

  static thread_local WorkStealingQueue* local_queue_;
  static thread_local std::size_t index_;
};

thread_local WorkStealingQueue* ThreadPool::local_queue_;
thread_local std::size_t ThreadPool::index_;
```
  

## 인터럽트
  
* 인터럽트 가능한 스레드의 간단한 구현
  
```cpp
class InterruptFlag {
 public:
  void set();
  bool is_set() const;
};

thread_local InterruptFlag this_thread_interrupt_flag;

class InterruptibleThread {
 public:
  template <typename F>
  InterruptibleThread(F f) {
    std::promise<InterruptFlag*> p;
    t = std::thread([f, &p] {
      p.set_value(&this_thread_interrupt_flag);
      f();
    });
    flag = p.get_future().get();
  }

  void interrupt() {
    if (flag) {
      flag->set();
    }
  }

 private:
  std::thread t;
  InterruptFlag* flag;
};

void interruption_point() {
  if (this_thread_interrupt_flag.is_set()) {
    throw thread_interrupted();
  }
}
```
  
* 함수에서 사용  
  
```cpp
void f() {
  while (!done) {
    interruption_point();
    process_next_item();
  }
}
```
  
* 루프에서 연속적으로 실행하는 대신 [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)을 사용하는 것이 더 좋은 방법이다  
  
```cpp
class InterruptFlag {
 public:
  void set() {
    b_.store(true, std::memory_order_relaxed);
    std::lock_guard<std::mutex> l(m_);
    if (cv_) {
      cv_->notify_all();
    }
  }

  bool is_set() const { return b_.load(std::memory_order_relaxed); }

  void set_condition_variable(std::condition_variable& cv) {
    std::lock_guard<std::mutex> l(m_);
    cv_ = &cv;
  }

  void clear_condition_variable() {
    std::lock_guard<std::mutex> l(m_);
    cv_ = nullptr;
  }

  struct ClearConditionVariableOnDestruct {
    ~ClearConditionVariableOnDestruct() {
      this_thread_interrupt_flag.clear_condition_variable();
    }
  };

 private:
  std::atomic<bool> b_;
  std::condition_variable* cv_ = nullptr;
  std::mutex m_;
};

void interruptible_wait(std::condition_variable& cv,
                        std::unique_lock<std::mutex>& l) {
  interruption_point();
  this_thread_interrupt_flag.set_condition_variable(cv);
  // 나중에 wait_for가 예외를 던질 수 있으므로 RAII는 플래그를 지우는 데 필요하다
  InterruptFlag::ClearConditionVariableOnDestruct guard;
  interruption_point();
  // 스레드가 인터럽트를 보기 전에 대기할 수 있는 시간 상한을 설정한다
  cv.wait_for(l, std::chrono::milliseconds(1));
  interruption_point();
}

template <typename Predicate>
void interruptible_wait(std::condition_variable& cv,
                        std::unique_lock<std::mutex>& l, Predicate pred) {
  interruption_point();
  this_thread_interrupt_flag.set_condition_variable(cv);
  InterruptFlag::ClearConditionVariableOnDestruct guard;
  while (!this_thread_interrupt_flag.is_set() && !pred()) {
    cv.wait_for(l, std::chrono::milliseconds(1));
  }
  interruption_point();
}
```

* 和 [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable) 不同的是，[std::condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any) 可以使用不限于 [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) 的任何类型的锁，这意味着可以使用自定义的锁类型  
* [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)과 달리 [std::condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any)는 [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) 즉 사용자 정의 잠금 유형을 사용할 수 있다  
  
```cpp
#include <atomic>
#include <condition_variable>
#include <mutex>

class InterruptFlag {
 public:
  void set() {
    b_.store(true, std::memory_order_relaxed);
    std::lock_guard<std::mutex> l(m_);
    if (cv_) {
      cv_->notify_all();
    } else if (cv_any_) {
      cv_any_->notify_all();
    }
  }

  template <typename Lockable>
  void wait(std::condition_variable_any& cv, Lockable& l) {
    class Mutex {
     public:
      Mutex(InterruptFlag* self, std::condition_variable_any& cv, Lockable& l)
          : self_(self), lock_(l) {
        self_->m_.lock();
        self_->cv_any_ = &cv;
      }

      ~Mutex() {
        self_->cv_any_ = nullptr;
        self_->m_.unlock();
      }

      void lock() { std::lock(self_->m_, lock_); }

      void unlock() {
        lock_.unlock();
        self_->m_.unlock();
      }

     private:
      InterruptFlag* self_;
      Lockable& lock_;
    };

    Mutex m(this, cv, l);
    interruption_point();
    cv.wait(m);
    interruption_point();
  }
  // rest as before

 private:
  std::atomic<bool> b_;
  std::condition_variable* cv_ = nullptr;
  std::condition_variable_any* cv_any_ = nullptr;
  std::mutex m_;
};

template <typename Lockable>
void interruptible_wait(std::condition_variable_any& cv, Lockable& l) {
  this_thread_interrupt_flag.wait(cv, l);
}
```
  
* 다른 차단 호출(예: 뮤텍스, 퓨처)을 중단하려면 일반적으로 [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)에서와 같이 시간 제한을 설정하면 된다. 내부 뮤텍스나 퓨처에 액세스하지 않으면 대기 조건을 충족하지 않고는 대기를 중단할 수 없기 때문이다  
  
```cpp
template <typename T>
void interruptible_wait(std::future<T>& ft) {
  while (!this_thread_interrupt_flag.is_set()) {
    if (ft.wait_for(std::chrono::milliseconds(1)) ==
        std::future_status::ready) {
      break;
    }
  }
  interruption_point();
}
```

* 인터럽트는 인터럽트된 스레드의 관점에서 볼 때 `thread_interrupted` 예외이다. 따라서 인터럽트가 감지되면 예외처럼 처리할 수 있다.
  
```cpp
internal_thread = std::thread{[f, &p] {
  p.set_value(&this_thread_interrupt_flag);
  try {
    f();
  } catch (const thread_interrupted&) {
    // std::terminate 는 예외가 std::thread 의 소멸자에게 전달될 때 호출된다
    // 예외를 잡아내어 프로그램 종료를 방지한다
  }
}};
```
  
* 데스크톱 검색 애플리케이션을 사용하는 경우 애플리케이션은 사용자와 상호 작용하는 것 외에도 파일시스템의 상태를 모니터링하여 변경 사항을 식별하고 색인을 업데이트해야 한다. GUI의 응답성에 영향을 주지 않기 위해 이 처리는 일반적으로 프로그램의 전체 수명 기간 동안 실행되어야 하는 백그라운드 스레드로 넘겨진다. 이러한 프로그램은 일반적으로 컴퓨터가 종료될 때만 종료되며, 다른 상황에서 프로그램을 종료하려면 백그라운드 스레드를 잘 정리된 방식으로 종료해야 하는데 종료하는 한 가지 방법은 백그라운드 스레드를 중단하는 것이다.   
    
```cpp
std::mutex config_mutex;
std::vector<InterruptibleThread> background_threads;

void background_thread(int disk_id) {
  while (true) {
    interruption_point();
    fs_change fsc = get_fs_changes(disk_id);
    if (fsc.has_changes()) {
      update_index(fsc);
    }
  }
}

void start_background_processing() {
  background_threads.emplace_back(background_thread, disk_1);
  background_threads.emplace_back(background_thread, disk_2);
}

int main() {
  start_background_processing();
  process_gui_until_exit();
  std::unique_lock<std::mutex> l(config_mutex);
  for (auto& x : background_threads) {
    x.interrupt();
  }
  // 모든 스레드를 중단한 다음 다시 join
  for (auto& x : background_threads) {
    if (x.joinable()) {
      x.join();
    }
  }
  // 루프에서 인터럽트하고 바로 join 하지 않는 이유는 동시성을 위해서이다
  // 인터럽트는 즉시 끝나지 않기 때문에 다음 인터럽트 지점으로 이동한 다음
  // 그리고 종료하기 전에 필요한 소멸자와 예외 처리 코드를 호출해야 한다
  // 모든 스레드가 중단되었다가 다시 join 하면, 중단된 스레드는 여전히 일부 작업을 수행할 수 있음에도 불구하고 대기한다
  // 다른 스레드를 중단하는 등 여전히 유용한 작업을 수행할 수 있음에도 불구하고
}
```
