## 조건 변수（condition variable）
    
* 동시 프로그래밍에서는 스레드가 작업을 계속하기 전에 다른 스레드가 특정 이벤트를 완료할 때까지 기다리는 것이 일반적인 요구 사항이다. 이 경우 표준 라이브러리에서는 [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)을 제공한다.  
  
```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

class A {
 public:
  void step1() {
    {
      std::lock_guard<std::mutex> l(m_);
      step1_done_ = true;
    }
    std::cout << 1;
    cv_.notify_one();
  }

  void step2() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return step1_done_; });
    step2_done_ = true;
    std::cout << 2;
    cv_.notify_one();
  }

  void step3() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return step2_done_; });
    std::cout << 3;
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  bool step1_done_ = false;
  bool step2_done_ = false;
};

int main() {
  A a;
  std::thread t1(&A::step1, &a);
  std::thread t2(&A::step2, &a);
  std::thread t3(&A::step3, &a);
  t1.join();
  t2.join();
  t3.join();
}  // 123
```
  
* 깨울 수 있는 작업이 여러 개 있는 경우 [notify_one()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_one)은 무작위로 하나씩 깨운다.    
  
```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

class A {
 public:
  void wait1() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return done_; });
    std::cout << 1;
  }

  void wait2() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return done_; });
    std::cout << 2;
  }

  void signal() {
    {
      std::lock_guard<std::mutex> l(m_);
      done_ = true;
    }
    cv_.notify_all();
  }

 private:
  std::mutex m_;
  std::condition_variable cv_;
  bool done_ = false;
};

int main() {
  A a;
  std::thread t1(&A::wait1, &a);
  std::thread t2(&A::wait2, &a);
  std::thread t3(&A::signal, &a);
  t1.join();
  t2.join();
  t3.join();
}  // 12 or 21
```
  
* [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)는 [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock)과 함께 작동하며, 이 표준 라이브러리가 제공하는 [std::condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any)를 추가로 사용할 수 있다.  
  
```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <thread>

class Mutex {
 public:
  void lock() {}
  void unlock() {}
};

class A {
 public:
  void signal() {
    std::cout << 1;
    cv_.notify_one();
  }

  void wait() {
    Mutex m;
    cv_.wait(m);
    std::cout << 2;
  }

 private:
  std::condition_variable_any cv_;
};

int main() {
  A a;
  std::thread t1(&A::signal, &a);
  std::thread t2(&A::wait, &a);
  t1.join();
  t2.join();
}  // 12
```
    
* [std::stack](https://en.cppreference.com/w/cpp/container/stack )과 마찬가지로, [std::queue](https://en.cppreference.com/w/cpp/container/queue )는 front과 pop에 대한 경쟁 조건이 있다. front과 pop을 try_pop 함수로 결합하고, [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable )을 사용하여 구현한다. 조건 변수를 사용하여 팝업할 요소가 없을 때 팝업할 요소가 있을 때까지 차단하는 wait_and_pop 인터페이스를 구현한다.  
  
```cpp
#include <condition_variable>
#include <memory>
#include <mutex>
#include <queue>

template <typename T>
class ConcurrentQueue {
 public:
  ConcurrentQueue() = default;

  ConcurrentQueue(const ConcurrentQueue& rhs) {
    std::lock_guard<std::mutex> l(rhs.m_);
    q_ = rhs.q_;
  }

  void push(T x) {
    std::lock_guard<std::mutex> l(m_);
    q_.push(std::move(x));
    cv_.notify_one();
  }

  void wait_and_pop(T& res) {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return !q_.empty(); });
    res = std::move(q_.front());
    q_.pop();
  }

  std::shared_ptr<T> wait_and_pop() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return !q_.empty(); });
    auto res = std::make_shared<T>(std::move(q_.front()));
    q_.pop();
    return res;
  }

  bool try_pop(T& res) {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return false;
    }
    res = std::move(q_.front());
    q_.pop();
    return true;
  }

  std::shared_ptr<T> try_pop() {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return nullptr;
    }
    auto res = std::make_shared<T>(std::move(q_.front()));
    q_.pop();
    return res;
  }

  bool empty() const {
    std::lock_guard<std::mutex> l(m_);
    // 다른 스레드가 이 객체(복사 구조체)를 가질 수 있으므로 lock 한다
    return q_.empty();
  }

 private:
  mutable std::mutex m_;
  std::condition_variable cv_;
  std::queue<T> q_;
};
```  
  
* 这个实现有一个异常安全问题，[notify_one()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_one) 只会唤醒一个线程，如果多线程等待时，被唤醒线程 wait_and_pop 中抛出异常（如构造 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 对象时可能抛异常），其余线程将永远不被唤醒。用 [notify_all()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all) 可解决此问题，但会有不必要的唤醒，抛出异常时再调用 [notify_one()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_one) 更好一些。对于此场景，最好的做法是将内部的 `std::queue<T>` 改为 `std::queue<std::shared_ptr<T>>`，[std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 对象只在 push 中构造，这样 wait_and_pop 就不会抛异常
* 이 구현에는 예외 안전 문제가 있다. [notify_one()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_one )은 하나의 스레드만 깨우며, 여러 스레드가 대기 중일 때 깨어난 스레드에서 wait_and_pop이 예외를 던지면 나머지 스레드는 절대 깨어나지 않는다. [notify_all()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all)을 사용하면 이 문제가 해결되지만 불필요한 웨이크업이 발생하므로 예외가 발생하면 [notify_one()](https://en.cppreference.com/w/cpp/thread/condition_variable/notify_all )을 호출하는 것이 좋다. 이 시나리오의 경우 가장 좋은 방법은 내부 `std::queue<T>`를 `std::queue<std::shared_ptr<T>>`로 변경하고, [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr ) 객체는 push로만 구성되므로 wait_and_pop이 예외를 던지지 않는다.   
  
```cpp
#include <condition_variable>
#include <memory>
#include <mutex>
#include <queue>
#include <utility>

template <typename T>
class ConcurrentQueue {
 public:
  ConcurrentQueue() = default;

  ConcurrentQueue(const ConcurrentQueue& rhs) {
    std::lock_guard<std::mutex> l(rhs.m_);
    q_ = rhs.q_;
  }

  void push(T x) {
    auto data = std::make_shared<T>(std::move(x));
    std::lock_guard<std::mutex> l(m_);
    q_.push(data);
    cv_.notify_one();
  }

  void wait_and_pop(T& res) {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return !q_.empty(); });
    res = std::move(*q_.front());
    q_.pop();
  }

  std::shared_ptr<T> wait_and_pop() {
    std::unique_lock<std::mutex> l(m_);
    cv_.wait(l, [this] { return !q_.empty(); });
    auto res = q_.front();
    q_.pop();
    return res;
  }

  bool try_pop(T& res) {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return false;
    }
    res = std::move(*q_.front());
    q_.pop();
    return true;
  }

  std::shared_ptr<T> try_pop() {
    std::lock_guard<std::mutex> l(m_);
    if (q_.empty()) {
      return nullptr;
    }
    auto res = q_.front();
    q_.pop();
    return res;
  }

  bool empty() const {
    std::lock_guard<std::mutex> l(m_);
    return q_.empty();
  }

 private:
  mutable std::mutex m_;
  std::condition_variable cv_;
  std::queue<std::shared_ptr<T>> q_;
};
```

## semaphore
  
* semaphore는 여러 스레드 간에 지정된 횟수의 이벤트 알림을 구현하는 데 사용된다. P 연산은 시그널을 1씩 감소시키고, V 연산은 시그널을 1씩 증가시키며, P 연산으로 인해 시그널이 0보다 작아지면 감소될 때까지 블록한다. C++20에서는 [std::counting_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore )를 제공하는데, 템플릿 파라미터를 통해 세마포어의 최대값을 설정하고 함수 파라미터인 [acquire()](https://en.cppreference.com/w/cpp/thread/counting_semaphore/acquire ) 즉, P 연산은 세마포어의 값이 0보다 크면 세마포어를 1만큼 줄이고, 그렇지 않으면 1만큼 줄일 수 있을 때까지 블록한다. [release()](https://en.cppreference.com/w/cpp/thread/counting_semaphore/release )은 세마포어에 지정된 값(지정되지 않은 경우 1)을 더하고 [acquire()](https://en.cppreference.com/w/cpp/thread/counting_semaphore/acquire )는 차단된 세마포어를 지정된 개수만큼 깨우는 V 연산이다.
  
```cpp
#include <iostream>
#include <semaphore>
#include <thread>

class A {
 public:
  void wait1() {
    sem_.acquire();
    std::cout << 1;
  }

  void wait2() {
    sem_.acquire();
    std::cout << 2;
  }

  void signal() { sem_.release(2); }

 private:
  std::counting_semaphore<2> sem_{0};  // 초기값 0，최대값 2
};

int main() {
  A a;
  std::thread t1(&A::wait1, &a);
  std::thread t2(&A::wait2, &a);
  std::thread t3(&A::signal, &a);
  t1.join();
  t2.join();
  t3.join();
}  // 12 or 21
```

* [std::binary_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore) 는 최대값이 1인 세마포어이다. 템플릿 파라미터 1이 있는 [std::counting_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore) 의 별칭이다.

```cpp
#include <iostream>
#include <semaphore>
#include <thread>

class A {
 public:
  void wait() {
    sem_.acquire();
    std::cout << 2;
  }

  void signal() {
    std::cout << 1;
    sem_.release();
  }

 private:
  std::binary_semaphore sem_{0};
};

int main() {
  A a;
  std::thread t1(&A::wait, &a);
  std::thread t2(&A::signal, &a);
  t1.join();
  t2.join();
}  // 12
```

## barrier
   
* C++20은 대기할 스레드 수로 값을 구성하는 [std::barrier](https://en.cppreference.com/w/cpp/thread/barrier)를 제공하며, [std::barrier::arrive_and_wait]( https://en.cppreference.com/w/cpp/thread/barrier/arrive_and_wait )는 모든 스레드가 작업을 완료할 때까지 차단하고(따라서 배리어라는 이름이 붙음), 마지막 스레드가 작업을 완료하면 모든 스레드가 해제되고 배리어가 재설정된다. 모든 스레드가 차단 지점에 도달하면 스레드 중 하나에서 실행되는 noexcept 함수를 사용하여 [std::barrier](https://en.cppreference.com/w/cpp/thread/barrier) 구문을 추가적으로 설정할 수 있다. 스레드 집합에서 스레드를 제거하려면 해당 스레드의 배리어에서 [std::barrier::arrive_and_drop](https://en.cppreference.com/w/cpp/thread/barrier/arrive_and_drop )을 호출한다.
  
```cpp
#include <barrier>
#include <cassert>
#include <iostream>
#include <thread>

class A {
 public:
  void f() {
    std::barrier sync_point{3, [&]() noexcept { ++i_; }};
    for (auto& x : tasks_) {
      x = std::thread([&] {
        std::cout << 1;
        sync_point.arrive_and_wait();
        assert(i_ == 1);
        std::cout << 2;
        sync_point.arrive_and_wait();
        assert(i_ == 2);
        std::cout << 3;
      });
    }
    for (auto& x : tasks_) {
      x.join();  // 배리어를 파괴하기 전에 배리어를 사용하는 모든 스레드에 합류한다 
    }  // 스레드가 배리어를 소멸할 때 배리어의 멤버 함수를 호출하는 것은 정의되지 않은 동작이다 
  }

 private:
  std::thread tasks_[3] = {};
  int i_ = 0;
};

int main() {
  A a;
  a.f();
}
```  
  
* C++20은 값을 카운터의 초기값으로 구성한 일회성 배리어로 [std::latch](https://en.cppreference.com/w/cpp/thread/latch )를 제공하고, [std::latch::count_down](https://en.cppreference.com/w/cpp/thread/latch/count_down )은 카운터를 1씩 감소시키고, [std::latch::wait](https://en.cppreference.com/w/cpp/thread/latch/wait )는 다음과 같은 값이 될 때까지 차단한다. 카운터를 1씩 감소시키고 0으로 차단하려면 [std::latch::arrive_and_wait](https://en.cppreference.com/w/cpp/thread/latch/arrive_and_wait )를 호출하면 된다.   

```cpp
#include <iostream>
#include <latch>
#include <string>
#include <thread>

class A {
 public:
  void f() {
    for (auto& x : data_) {
      x.t = std::jthread([&] {
        x.s += x.s;
        done_.count_down();
      });
    }
    done_.wait();
    for (auto& x : data_) {
      std::cout << x.s << std::endl;
    }
  }

 private:
  struct {
    std::string s;
    std::jthread t;
  } data_[3] = {
      {"hello"},
      {"down"},
      {"demo"},
  };

  std::latch done_{3};
};

int main() {
  A a;
  a.f();
}
```

## future
  
* 표준 라이브러리는 [std::thread](https://en.cppreference.com/w/cpp/thread/thread)가 실행하는 함수와 함수의 반환 결과를 연관시키기 위해 [std::future](https://en.cppreference.com/w/cpp/thread/future )를 제공하여 스레드가 실행하는 함수와 함수의 반환 결과를 연관시킨다. [std::future](https://en.cppreference.com/w/cpp/thread/future )를 사용하여 스레드가 실행 중인 함수와 함수 결과를 연관시킬 수 있으며, 이 결과를 가져오는 방법은 비동기식이다. [std::async()](https://en.cppreference.com/w/cpp/thread/async )를 통해 비동기 태스크의 [std::future](https://en.cppreference.com/w/cpp/thread/future )와 동일한 매개변수를 전달하여 태스크를 생성하고, [std::async](https://en.cppreference.com/w/cpp/thread/async )는 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 와 동일한 방식으로 전달된다.  
  
```cpp
#include <future>
#include <iostream>

class A {
 public:
  int f(int i) { return i; }
};

int main() {
  A a;
  std::future<int> res = std::async(&A::f, &a, 1);
  std::cout << res.get();  // 1，스레드가 결과를 반환할 때까지 차단
}
```
  
* [std::future](https://en.cppreference.com/w/cpp/thread/future )는 [get()](https://en.cppreference.com/w/cpp/thread/future/get) 을 한번만 사용할 수 있다  
  
```cpp
#include <future>
#include <iostream>

int main() {
  std::future<void> res = std::async([] {});
  res.get();
  try {
    res.get();
  } catch (const std::future_error& e) {
    std::cout << e.what() << std::endl;  // no state
  }
}
```
    
* [std::async](https://en.cppreference.com/w/cpp/thread/async)의 첫 번째 인수는 열거형 [std::launch](https://en.cppreference.com/w/cpp/thread/launch)의 값으로 지정할 수 있으며, 이는 태스크의 런타임 정책을 설정하는 데 사용된다.
  
```cpp
namespace std {
enum class launch { // names for launch options passed to async
    async    = 0x1, // 새 스레드를 실행하여 작업을 실행한다
    deferred = 0x2  // 비활성 요청, 결과가 요청될 때만 작업을 실행한다
};
}

// std::async 작업은 기본적으로 두 가지를 모두 사용하여 만들어진다
std::async([] {}); // std::async(std::launch::async | std::launch::deferred, [] {})
```
  
* 비동기 작업은 [std::packaged_task](https://en.cppreference.com/w/cpp/thread/packaged_task)로 캡슐화할 수도 있는데, 이는 두 스레드 간에 작업을 전달하는 데 사용할 수 있다(예: 한 스레드가 비동기 작업을 큐에 추가하고 스레드는 큐에서 작업을 계속 가져와 실행한다.
    
```cpp
#include <future>
#include <iostream>

int main() {
  std::packaged_task<int(int)> task([](int i) { return i; });
  task(1);  // 계산 결과를 요청하면 내부 future가 결과 값을 설정한다
  std::future<int> res = task.get_future();
  std::cout << res.get();  // 1
}
```  
  
* 더 간단한 시나리오는 고정된 반환값을 갖는 것인데 이 경우 [std::promise](https://en.cppreference.com/w/cpp/thread/promise)로 충분할 것이다.  

```cpp
#include <future>
#include <iostream>

int main() {
  std::promise<int> ps;
  ps.set_value(1);  // 내부적으로 future가 결과 값을 설정한다
  std::future<int> res = ps.get_future();
  std::cout << res.get();  // 1
}
```
  
* [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 는 이벤트 알림을 허용한다
  
```cpp
#include <chrono>
#include <future>
#include <iostream>

class A {
 public:
  void signal() {
    std::cout << 1;
    ps_.set_value();
  }

  void wait() {
    std::future<void> res = ps_.get_future();
    res.wait();
    std::cout << 2;
  }

 private:
  std::promise<void> ps_;
};

int main() {
  A a;
  std::thread t1{&A::signal, &a};
  std::thread t2{&A::wait, &a};
  t1.join();
  t2.join();
}
```
  
* [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable )과 달리 [std::promise](https://en. cppreference.com/w/cpp/thread/promise )는 한 번만 알림을 받을 수 있으므로 일반적으로 일시 중지된 상태의 스레드를 생성하는 데 사용된다.  

```cpp
#include <chrono>
#include <future>
#include <iostream>

class A {
 public:
  void task() { std::cout << 1; }
  void wait_for_task() {
    ps_.get_future().wait();
    task();
  }
  void signal() { ps_.set_value(); }

 private:
  std::promise<void> ps_;
};

void task() { std::cout << 1; }

int main() {
  A a;
  std::thread t(&A::wait_for_task, &a);
  a.signal();
  t.join();
}
```

* [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 는 하나의 [std::future](https://en.cppreference.com/w/cpp/thread/future) 와만 연결할 수 있다

```cpp
#include <future>
#include <iostream>

int main() {
  std::promise<void> ps;
  auto a = ps.get_future();
  try {
    auto b = ps.get_future();
  } catch (const std::future_error& e) {
    std::cout << e.what() << std::endl;  // future already retrieved
  }
}
```

* [std::future](https://en.cppreference.com/w/cpp/thread/future) 는 태스크에 예외를 저장할 수 있다

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int main() {
  std::future<void> res = std::async([] { throw std::logic_error("error"); });
  try {
    res.get();
  } catch (const std::exception& e) {
    std::cout << e.what() << std::endl;
  }
}
```

* [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 수동 예외 저장 필요

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int main() {
  std::promise<void> ps;
  try {
    throw std::logic_error("error");
  } catch (...) {
    ps.set_exception(std::current_exception());
  }
  auto res = ps.get_future();
  try {
    res.get();
  } catch (const std::exception& e) {
    std::cout << e.what() << std::endl;
  }
}
```

* [set_value()](https://en.cppreference.com/w/cpp/thread/promise/set_value) 의 예외는 future에서 설정 되지 않는다는 점에 유의

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int main() {
  std::promise<int> ps;
  try {
    ps.set_value([] {
      throw std::logic_error("error");
      return 0;
    }());
  } catch (const std::exception& e) {
    std::cout << e.what() << std::endl;
  }
  ps.set_value(1);
  auto res = ps.get_future();
  std::cout << res.get();  // 1
}
```

* [std::packaged_task](https://en.cppreference.com/w/cpp/thread/packaged_task) 및 [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 는 설정 되지 않은 값을 직접 계산하고, [std::future::get()](https://en.cppreference.com/w/cpp/thread/future/get) 는 정상적으로 작동한다.  
  
```cpp
#include <future>
#include <iostream>

int main() {
  std::future<void> ft1;
  std::future<void> ft2;
  {
    std::packaged_task<void()> task([] {});
    std::promise<void> ps;
    ft1 = task.get_future();
    ft2 = ps.get_future();
  }
  try {
    ft1.get();
  } catch (const std::future_error& e) {
    std::cout << e.what() << std::endl;  // broken promise
  }
  try {
    ft2.get();
  } catch (const std::future_error& e) {
    std::cout << e.what() << std::endl;  // broken promise
  }
}
```

* [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 는 결과를 여러 번 얻을 수 있으며 [std::future](https://en.cppreference.com/w/cpp/thread/future) 로도 얻을 수 있더. [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 객체에서 반환된 결과는 동기화 되지 않는다. [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 에서 접근하려면 race condition 을 방지하기 위해 잠금이 필요하므로 더 나은 접근 방식은 [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 객체를 각 스레드에 복사하여 잠금 없이 안전하게 접근할 수 있다  
  
```cpp
#include <future>

int main() {
  std::promise<void> ps;
  std::future<void> ft = ps.get_future();
  std::shared_future<void> sf(std::move(ft));
  // 또는 직접 std::shared_future<void> sf{ps.get_future()};
  ps.set_value();
  sf.get();
  sf.get();
}
```

* [std::future::share()](https://en.cppreference.com/w/cpp/thread/future/share) 로 직접 [std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future) 생성할 수 있다  
  
```cpp
#include <future>

int main() {
  std::promise<void> ps;
  auto sf = ps.get_future().share();
  ps.set_value();
  sf.get();
  sf.get();
}
```

## 시계

* 对于标准库来说，时钟是提供了四种信息的类
  * 当前时间，如 [std::chrono::system_clock::now()](https://en.cppreference.com/w/cpp/chrono/system_clock/now)
  * 表示时间值的类型，如 [std::chrono::time_point](https://en.cppreference.com/w/cpp/chrono/time_point)
  * 时钟节拍（一个 tick 的周期），一般一秒有 25 个 tick，一个周期则为 [std::ratio\<1, 25\>](https://en.cppreference.com/w/cpp/numeric/ratio/ratio)
  * 通过时钟节拍确定时钟是否稳定（steady，匀速），如 [std::chrono::steady_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/steady_clock)（稳定时钟，代表系统时钟的真实时间）、[std::chrono::system_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/system_clock)（一般因为时钟可调节而不稳定，即使这是为了考虑本地时钟偏差的自动调节）、[high_resolution_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/high_resolution_clock)（最小节拍最高精度的时钟）
* 获取当前 UNIX 时间戳，单位为纳秒
* 표준 라이브러리에서 시계는 네 가지 유형의 정보를 제공하는 클래스이다.
    * 현재 시간, 예: [std::chrono::system_clock::now()](https://en.cppreference.com/w/cpp/chrono/system_clock/now)
    * 시간 값을 나타내는 타입, 예: [std::chrono::time_point](https://en.cppreference.com/w/cpp/chrono/time_point)
    * 시계 비트(틱의 주기), 일반적으로 초당 25 틱이 있으며 기간은 [std::ratio\<1, 25\>](https://en.cppreference.com/w/cpp/numeric/ratio/ratio) 이다.
    * 시계 비트에 따라 시계가 안정적인지 (안정적, 균일한지) 결정합니다, 예: [std::chrono::steady_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/steady_clock) (안정된 시계는 시스템 시계의 실제 시간을 나타냅니다), [std::chrono::system_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/system_clock) (일반적으로 시계가 조정 가능하기 때문에 불안정하다) (이 작업을 수행하더라도 로컬 시계 편차의 자동 조정을 고려), [high_resolution_clock::is_steady()](https://en.cppreference.com/w/cpp/chrono/high_resolution_clock) (가장 작은 비트에 대해 가장 정밀도가 높은 시계).
* 현재 UNIX 타임스탬프를 나노초 단위로 가져온다.  
  
```cpp
#ifdef _WIN32
#include <chrono>
#elif defined __GNUC__
#include <time.h>
#endif

long long now_in_ns() {
#ifdef _WIN32
  return std::chrono::duration_cast<std::chrono::nanoseconds>(
             std::chrono::system_clock::now().time_since_epoch())
      .count();
#elif defined __GNUC__
  struct timespec t;
  clockid_t clk_id = CLOCK_REALTIME;
  clock_gettime(clk_id, &t);
  return t.tv_sec * 1e9 + t.tv_nsec;
#endif
}
```

* [std::put_time](https://en.cppreference.com/w/cpp/io/manip/put_time) 으로 인쇄 시간 형식 지정  
  
```cpp
#include <chrono>
#include <iomanip>
#include <iostream>

int main() {
  std::chrono::system_clock::time_point t = std::chrono::system_clock::now();
  std::time_t c = std::chrono::system_clock::to_time_t(t);  // UNIX 타임스탬프，초
  //  %F는 %Y-%m-%d， %T는 %H:%M:%S로 2011-11-11 11:11:11
  std::cout << std::put_time(std::localtime(&c), "%F %T");
}
```

* [std::chrono::duration](https://en.cppreference.com/w/cpp/chrono/duration) 은 시간 간격을 나타낸다  
  
```cpp
namespace std {
namespace chrono {
using nanoseconds  = duration<long long, nano>;
using microseconds = duration<long long, micro>;
using milliseconds = duration<long long, milli>;
using seconds      = duration<long long>;
using minutes      = duration<int, ratio<60>>;
using hours        = duration<int, ratio<3600>>;
// C++20
using days   = duration<int, ratio_multiply<ratio<24>, hours::period>>;
using weeks  = duration<int, ratio_multiply<ratio<7>, days::period>>;
using years  = duration<int, ratio_multiply<ratio<146097, 400>, days::period>>;
using months = duration<int, ratio_divide<years::period, ratio<12>>>;
}  // namespace chrono
}  // namespace std
```

* C++14 는 [std::literals::chrono_literals](https://en.cppreference.com/w/cpp/symbol_index/chrono_literals) 에서 시간을 나타내는 접미사를 제공한다  
  
```cpp
#include <cassert>
#include <chrono>

using namespace std::literals::chrono_literals;

int main() {
  auto a = 45min;
  assert(a.count() == 45);
  auto b = std::chrono::duration_cast<std::chrono::seconds>(a);
  assert(b.count() == 2700);
  auto c = std::chrono::duration_cast<std::chrono::hours>(a);
  assert(c.count() == 0);  // 변환 잘라내기
}
```

* duration 연산에 대한 기간 지원
  
```cpp
#include <cassert>
#include <chrono>

using namespace std::literals::chrono_literals;

int main() {
  assert((1h - 2 * 15min).count() == 30);
  assert((0.5h + 2 * 15min + 60s).count() == 3660);
}
```

* duration 을 사용하여 대기 시간을 설정한다  

```cpp
#include <chrono>
#include <future>
#include <iostream>
#include <thread>

int f() {
  std::this_thread::sleep_for(std::chrono::seconds(1));
  return 1;
}

int main() {
  auto res = std::async(f);
  if (res.wait_for(std::chrono::seconds(5)) == std::future_status::ready) {
    std::cout << res.get();
  }
}
```

* [std::chrono::time_point](https://en.cppreference.com/w/cpp/chrono/time_point) 是表示时间的类型，值为从某个时间点开始计时的时间长度

```cpp
// 첫 번째 템플릿 매개 변수는 시작 시점의 시계 유형이고 두 번째 매개 변수는 시간 단위이다
std::chrono::time_point<std::chrono::system_clock, std::chrono::seconds>
```

* [std::chrono::time_point](https://en.cppreference.com/w/cpp/chrono/time_point) 는 duration 에서 더하거나 빼거나, 자체에서 뺄 수 있다  
  
```cpp
#include <cassert>
#include <chrono>

int main() {
  std::chrono::system_clock::time_point a = std::chrono::system_clock::now();
  std::chrono::system_clock::time_point b = a + std::chrono::hours(1);
  long long diff =
      std::chrono::duration_cast<std::chrono::seconds>(b - a).count();
  assert(diff == 3600);
}
```

* 다음 기능은 타임아웃 설정을 지원하며, 최대 시간이 만료될 때까지 기능을 차단한다
  * [std::this_thread::sleep_for](https://en.cppreference.com/w/cpp/thread/sleep_for)
  * [std::this_thread::sleep_until](https://en.cppreference.com/w/cpp/thread/sleep_until)
  * [std::condition_variable::wait_for](https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for)
  * [std::condition_variable::wait_until](https://en.cppreference.com/w/cpp/thread/condition_variable/wait_until)
  * [std::condition_variable_any::wait_for](https://en.cppreference.com/w/cpp/thread/condition_variable_any/wait_for)
  * [std::condition_variable_any::wait_until](https://en.cppreference.com/w/cpp/thread/condition_variable_any/wait_until)
  * [std::timed_mutex::try_lock_for](https://en.cppreference.com/w/cpp/thread/timed_mutex/try_lock_for)
  * [std::timed_mutex::try_lock_until](https://en.cppreference.com/w/cpp/thread/timed_mutex/try_lock_until)
  * [std::recursive_timed_mutex::try_lock_for](https://en.cppreference.com/w/cpp/thread/recursive_timed_mutex/try_lock_for)
  * [std::recursive_timed_mutex::try_lock_until](https://en.cppreference.com/w/cpp/thread/recursive_timed_mutex/try_lock_until)
  * [std::unique_lock::try_lock_for](https://en.cppreference.com/w/cpp/thread/unique_lock/try_lock_for)
  * [std::unique_lock::try_lock_until](https://en.cppreference.com/w/cpp/thread/unique_lock/try_lock_until)
  * [std::future::wait_for](https://en.cppreference.com/w/cpp/thread/future/wait_for)
  * [std::future::wait_until](https://en.cppreference.com/w/cpp/thread/future/wait_until)
  * [std::shared_future::wait_for](https://en.cppreference.com/w/cpp/thread/shared_future/wait_for)
  * [std::shared_future::wait_until](https://en.cppreference.com/w/cpp/thread/shared_future/wait_until)
  * [std::counting_semaphore::try_acquire_for](https://en.cppreference.com/w/cpp/thread/counting_semaphore/try_acquire_for)
  * [std::counting_semaphore::try_acquire_until](https://en.cppreference.com/w/cpp/thread/counting_semaphore/try_acquire_until)
  
* 머신마다 CPU 주파수가 다르기 때문에 보다 정확한 성능 테스트를 수행하기 위해 일반적으로 시간을 직접 사용하지 않고 rdtsc 명령어를 사용하여 CPU 사이클을 얻고, rdtsc는 낮은 32 비트의 tsc를 EAX에 저장하고 높은 32 비트는 EDX에 저장, 다른 CPU의 tsc가 동기화되지 않을 수 있으며 constant_tsc의 플래그가 활성화되면 (`cat /proc/cpuinfo | grep constant_tsc`로 확인), 다른 CPU의 다른 코어의 tsc가 모두 동기화된다. constant_tsc 플래그가 활성화된 경우(`cat /proc/cpuinfo | grep constant_tsc`로 확인), 서로 다른 CPU의 서로 다른 코어의 tsc가 동기화된다. constant_tsc 플래그가 없는 인텔 CPU인 경우, 같은 프로세서의 다른 코어의 tsc는 동기화되고 다른 CPU의 다른 코어의 tsc는 동기화되지 않는다. CPU가 rdtsc 이후에 명령어를 재순서할 수 있는 경우, tsc를 읽은 후 메모리 배리어를 추가해야 한다. CPU가 rdtsc 보다 먼저 명령어를 스크램블할 수 있는 경우에는 rdtsc 대신 rdtscp를 사용할 수 있다. 오버헤드는 약 10 클럭 사이클이 더 발생하지만, CPU가 지원해야 하는 rdtsc를 사용하고 미리 메모리 배리어를 설정하는 것보다 적으며 `cat /proc/cpuinfo | grep rdtscp`로 확인할 수 있다  

```cpp
#include <cstdint>

#ifdef _WIN32
#include <intrin.h>
#endif

static inline std::uint64_t read_tsc() {
#ifdef _WIN32
  return __rdtsc();
#elif defined __GNUC__
  std::uint64_t res;
  __asm__ __volatile__(
      "rdtsc;"
      "shl $32, %%rdx;"
      "or %%rdx, %%rax"
      : "=a"(res)
      :
      : "%rcx", "%rdx");
  return res;
#endif
}

static inline std::uint64_t read_tscp() {
#ifdef _WIN32
  std::uint32_t t;
  return __rdtscp(&t);
#elif defined __GNUC__
  std::uint64_t res;
  __asm__ __volatile__(
      "rdtscp;"
      "shl $32, %%rdx;"
      "or %%rdx, %%rax"
      : "=a"(res)
      :
      : "%rcx", "%rdx");
  return res;
#endif
}

static inline void fence() {
#ifdef _WIN32
  __faststorefence();
#elif defined __GNUC__
  __asm__ __volatile__("mfence" : : : "memory");
#endif
}

inline std::uint64_t tsc_begin() {
  std::uint64_t res = read_tsc();
  fence();
  return res;
}

inline std::uint64_t tsc_mid() {
  std::uint64_t res = read_tscp();
  fence();
  return res;
}

inline std::uint64_t tsc_end() { return read_tscp(); }
```

## functional programming

* C++

```cpp
#include <algorithm>
#include <iostream>
#include <list>
#include <utility>

template <typename T>
std::list<T> quick_sort(std::list<T> v) {
  if (v.empty()) {
    return v;
  }
  std::list<T> res;
  res.splice(res.begin(), v, v.begin());  
  // 조건에 따라 v를 두 부분으로 나누고 조건 요소를 만족하지 않는 첫 번째 이터레이터를 반환
  auto it = std::partition(v.begin(), v.end(),
                           [&](const T& x) { return x < res.front(); });
  std::list<T> low;
  low.splice(low.end(), v, v.begin(), it);  // 왼쪽 절반을 low 로 이동
  auto l(quick_sort(std::move(low)));       // 왼쪽 절반에 재귀적 빠른 정렬
  auto r(quick_sort(std::move(v)));         // 재귀적으로 오른쪽 절반을 빠르게 정렬
  res.splice(res.end(), r);                 // 결과 뒤에 오른쪽 절반을 이동
  res.splice(res.begin(), l);               // 왼쪽 절반을 결과 앞에 이동
  return res;
}

int main() {
  for (auto& x : quick_sort(std::list<int>{1, 3, 2, 4, 5})) {
    std::cout << x;  // 12345
  }
}
```

* 병렬 버전을 구현하려면 [std::future](https://en.cppreference.com/w/cpp/thread/future) 를 사용한다

```cpp
#include <algorithm>
#include <future>
#include <iostream>
#include <list>
#include <utility>

template <typename T>
std::list<T> quick_sort(std::list<T> v) {
  if (v.empty()) {
    return v;
  }
  std::list<T> res;
  res.splice(res.begin(), v, v.begin());
  auto it = std::partition(v.begin(), v.end(),
                           [&](const T& x) { return x < res.front(); });
  std::list<T> low;
  low.splice(low.end(), v, v.begin(), it);
  // 다른 스레드로 왼쪽 절반 정렬하기
  std::future<std::list<T>> l(std::async(&quick_sort<T>, std::move(low)));
  auto r(quick_sort(std::move(v)));
  res.splice(res.end(), r);
  res.splice(res.begin(), l.get());
  return res;
}

int main() {
  for (auto& x : quick_sort(std::list<int>{1, 3, 2, 4, 5})) {
    std::cout << x;  // 12345
  }
}
```

## Reactive

* 链式调用是函数式编程中经常使用的形式，常见于 [ReactiveX](https://reactivex.io/intro.html)，比如 [RxJS](https://github.com/ReactiveX/rxjs)，当上游产生数据时交给下游处理，将复杂的异步逻辑拆散成了多个小的操作，只需要关注每一步操作并逐步转换到目标结果即可。C++20 的 [ranges](https://en.cppreference.com/w/cpp/ranges) 使用的 [range-v3](https://github.com/ericniebler/range-v3) 就脱胎自 [RxCpp](https://github.com/ReactiveX/RxCpp)

```ts
import { interval } from 'rxjs';
import { withLatestFrom } from 'rxjs/operators';

const source1$ = interval(500);
const source2$ = interval(1000);
source1$.pipe(withLatestFrom(source2$, (x, y) => `${x}${y}`));  // 10 20 31 41 52 62---
```

* [Concurrency TS](https://en.cppreference.com/w/cpp/experimental/concurrency) 는 [std::experimental::promise](https://en.cppreference.com/w/cpp/experimental/concurrency/promise) 및 [std::experimental::packaged_task](https://en.cppreference.com/w/cpp/experimental/concurrency/packaged_task) 표준 라이브러리와 유일한 차이점은 [std::experimental::future](https://en.cppreference.com/w/cpp/experimental/future)，[std::experimental::future::then()](https://en.cppreference.com/w/cpp/experimental/future/then) 를 체인으로 연결하여 호출할 수 있다는 것이다

```cpp
int f(std::experimental::future<int>);

std::experimental::future<int> eft;
auto ft1 = eft(); // 자체 생성자에 의해 생성된 std::experimental::future 
// std::async 와 달리 f 를 전달할 수 없다
// 매개변수가 이미 라이브러리에 준비된 기간 값으로 정의되어 있기 때문에 전달할 수 없다
// 여기서 f는 int를 반환하므로 인수는 std::experimental::future<int> 이다
auto ft2 = ft1.then(f);
// then 그러면 원래의 기간 값이 무효화 된다
assert(!ft1.valid());
assert(ft2.valid());
```

* [std::async](https://en.cppreference.com/w/cpp/thread/async) 는 [std::future](https://en.cppreference.com/w/cpp/thread/future) 만 반환할 수 있으며 [std::experimental::future](https://en.cppreference.com/w/cpp/experimental/future) 를 반환하려면 새 비동기를 수동으로 구현해야 한다  
  
```cpp
template <typename F>
std::experimental::future<decltype(std::declval<F>()())> new_async(F&& func) {
  std::experimental::promise<decltype(std::declval<F>()())> p;
  auto ft = p.get_future();
  std::thread t([p = std::move(p), f = std::decay_t<F>(func)]() mutable {
    try {
      p.set_value_at_thread_exit(f());
    } catch (...) {
      p.set_exception_at_thread_exit(std::current_exception());
    }
  });
  t.detach();
  return ft;
}
```

* 유효성 검사를 위해 사용자 이름과 비밀번호를 백엔드로 전송한 다음 사용자 정보로 디스플레이 인터페이스를 업데이트 하는 로그인 로직을 구현하려는 경우 직렬 구현은 다음과 같다

```cpp
void process_login(const std::string& username, const std::string& password) {
  try {
    const user_id id = backend.authenticate_user(username, password);
    const user_data info_to_display = backend.request_current_info(id);
    update_display(info_to_display);
  } catch (std::exception& e) {
    display_error(e);
  }
}
```

* UI 스레드를 차단하지 않기 위해 비동기 구현

```cpp
std::future<void> process_login(const std::string& username,
                                const std::string& password) {
  return std::async(std::launch::async, [=]() {
    try {
      const user_id id = backend.authenticate_user(username, password);
      const user_data info_to_display = backend.request_current_info(id);
      update_display(info_to_display);
    } catch (std::exception& e) {
      display_error(e);
    }
  });
}
```

* 그러나 이 구현은 각 작업이 완료되면 이전 작업에 연결되는 연쇄 호출 메커니즘이 필요한 UI 스레드를 여전히 차단한다

```cpp
std::experimental::future<void> process_login(const std::string& username,
                                              const std::string& password) {
  return new_async(
             [=]() { return backend.authenticate_user(username, password); })
      .then([](std::experimental::future<user_id> id) {
        return backend.request_current_info(id.get());
      })
      .then([](std::experimental::future<user_data> info_to_display) {
        try {
          update_display(info_to_display.get());
        } catch (std::exception& e) {
          display_error(e);
        }
      });
}
```

* 如果调用后台函数内部阻塞，可能是因为需要等待消息通过网络或者完成一个数据库操作，而这些还没有完成。即使把任务划分为多个独立部分，也仍会阻塞调用，得到阻塞的线程。这时后台调用真正需要的是，在数据准备好时返回就绪的期值，而不阻塞任何线程，所以这里用返回 `std::experimental::future<user_id>` 的 `backend.async_authenticate_user` 替代返回 `user_id` 的 `backend.authenticate_user`
* 백그라운드 함수에 대한 호출이 내부적으로 차단되는 경우는 네트워크를 통해 메시지가 전달될 때까지 기다리거나 아직 완료되지 않은 데이터베이스 작업을 완료해야 하기 때문일 수 있다. 작업이 여러 부분으로 나뉘어져 있어도 호출은 여전히 차단되고 차단 스레드를 받게 된다. 이 시점에서 백엔드 호출에 실제로 필요한 것은 스레드를 차단하지 않고 데이터가 준비되었을 때 준비 기간 값을 반환하는 것이므로 여기서는 `user_id`를 반환하는 `backend.authenticate_user` 대신 `std::experimental::future<user_id>`를 반환하는 `backend.async_authenticate_user`가 사용된다.   
  
```cpp
std::experimental::future<void> process_login(const std::string& username,
                                              const std::string& password) {
  return backend.async_authenticate_user(username, password)
      .then([](std::experimental::future<user_id> id) {
        return backend.async_request_current_info(id.get());
      })
      .then([](std::experimental::future<user_data> info_to_display) {
        try {
          update_display(info_to_display.get());
        } catch (std::exception& e) {
          display_error(e);
        }
      });
}
```

* 마지막으로 일반 lambda를 사용하여 코드를 간소화할 수 있습니다  
  
```cpp
std::experimental::future<void> process_login(const std::string& username,
                                              const std::string& password) {
  return backend.async_authenticate_user(username, password)
      .then(
          [](auto id) { return backend.async_request_current_info(id.get()); })
      .then([](auto info_to_display) {
        try {
          update_display(info_to_display.get());
        } catch (std::exception& e) {
          display_error(e);
        }
      });
}
```

* [std::experimental::future](https://en.cppreference.com/w/cpp/experimental/future) 외에도 [std::experimental::shared_future](https://en.cppreference.com/w/cpp/experimental/shared_future) 에서 체인 호출을 지원한다

```cpp
auto ft1 = new_async(some_function).share();
auto ft2 = ft1.then(
    [](std::experimental::shared_future<some_data> data) { do_stuff(data); });
auto ft3 = ft1.then([](std::experimental::shared_future<some_data> data) {
  return do_other_stuff(data);
});
```

* 여러 기간의 결과를 가져오기 위해 [std::async](https://en.cppreference.com/w/cpp/thread/async) 를 반복되는 웨이크업으로 인해 오버헤드가 발생한다  
  
```cpp
std::future<FinalResult> process_data(std::vector<MyData>& vec) {
  const size_t chunk_size = whatever;
  std::vector<std::future<ChunkResult>> res;
  for (auto begin = vec.begin(), end = vec.end(); beg ! = end;) {
    const size_t remaining_size = end - begin;
    const size_t this_chunk_size = std::min(remaining_size, chunk_size);
    res.push_back(std::async(process_chunk, begin, begin + this_chunk_size));
    begin += this_chunk_size;
  }
  return std::async([all_results = std::move(res)]() {
    std::vector<ChunkResult> v;
    v.reserve(all_results.size());
    for (auto& f : all_results) {
      v.push_back(f.get());  // 반복적인 깨우기가 발생하여 많은 오버헤드가 추가된다
    }
    return gather_results(v);
  });
}
```

* [std::experimental::when_all](https://en.cppreference.com/w/cpp/experimental/when_all)을 사용하면 반복되는 웨이크업으로 인한 오버헤드를 피할 수 있다. 전달된 모든 기간 값이 준비되면 반환된 기간 값은 준비된 것이다.  
  
```cpp
std::experimental::future<FinalResult> process_data(std::vector<MyData>& vec) {
  const size_t chunk_size = whatever;
  std::vector<std::experimental::future<ChunkResult>> res;
  for (auto begin = vec.begin(), end = vec.end(); beg ! = end;) {
    const size_t remaining_size = end - begin;
    const size_t this_chunk_size = std::min(remaining_size, chunk_size);
    res.push_back(new_async(process_chunk, begin, begin + this_chunk_size));
    begin += this_chunk_size;
  }
  return std::experimental::when_all(res.begin(), res.end())
      .then([](std::future<std::vector<std::experimental::future<ChunkResult>>>
                   ready_results) {
        std::vector<std::experimental::future<ChunkResult>> all_results =
            ready_results.get();
        std::vector<ChunkResult> v;
        v.reserve(all_results.size());
        for (auto& f : all_results) {
          v.push_back(f.get());
        }
        return gather_results(v);
      });
}
```

* 들어오는 기간 값 중 하나가 준비되면 [std::experimental::when_any](https://en.cppreference.com/w/cpp/experimental/when_any)에서 반환한 기간 값이 준비된 것이다.  
 
```cpp
std::experimental::future<FinalResult> find_and_process_value(
    std::vector<MyData>& data) {
  const unsigned concurrency = std::thread::hardware_concurrency();
  const unsigned num_tasks = (concurrency > 0) ? concurrency : 2;
  std::vector<std::experimental::future<MyData*>> res;
  const auto chunk_size = (data.size() + num_tasks - 1) / num_tasks;
  auto chunk_begin = data.begin();
  std::shared_ptr<std::atomic<bool>> done_flag =
      std::make_shared<std::atomic<bool>>(false);
  for (unsigned i = 0; i < num_tasks; ++i) {  // 비동기 작업을 res 로 생성
    auto chunk_end =
        (i < (num_tasks - 1)) ? chunk_begin + chunk_size : data.end();
    res.push_back(new_async([=] {
      for (auto entry = chunk_begin; !*done_flag && (entry != chunk_end);
           ++entry) {
        if (matches_find_criteria(*entry)) {
          *done_flag = true;
          return &*entry;
        }
      }
      return (MyData**)nullptr;
    }));
    chunk_begin = chunk_end;
  }
  std::shared_ptr<std::experimental::promise<FinalResult>> final_result =
      std::make_shared<std::experimental::promise<FinalResult>>();

  struct DoneCheck {
    std::shared_ptr<std::experimental::promise<FinalResult>> final_result;

    DoneCheck(
        std::shared_ptr<std::experimental::promise<FinalResult>> final_result_)
        : final_result(std::move(final_result_)) {}

    void operator()(
        std::experimental::future<std::experimental::when_any_result<
            std::vector<std::experimental::future<MyData*>>>>
            res_param) {
      auto res = res_param.get();
      MyData* const ready_result =
          res.futures[res.index].get();  // 준비 기간 값에서 값 가져오기
      // 조건과 일치하는 값이 발견되면 결과가 처리되고 set_value가 사용된다
      if (ready_result) {
        final_result->set_value(process_found_value(*ready_result));
      } else {
        res.futures.erase(res.futures.begin() + res.index);  // 그렇지 않으면 값을 버린다
        if (!res.futures.empty()) {  // 여전히 확인해야 할 값이 있으면 when_any를 다시 호출한다.
          std::experimental::when_any(res.futures.begin(), res.futures.end())
              .then(std::move(*this));
        } else {  // 기간에 다른 값이 없는 경우 promise에서 예외를 설정
          final_result->set_exception(
              std::make_exception_ptr(std::runtime_error("Not found")));
        }
      }
    }
  };
  std::experimental::when_any(res.begin(), res.end())
      .then(DoneCheck(final_result));
  return final_result->get_future();
}
```

* when_all은 한 쌍의 반복자 외에도 기간 값을 직접 받을 수 있다

```cpp
std::experimental::future<int> ft1 = new_async(f1);
std::experimental::future<std::string> ft2 = new_async(f2);
std::experimental::future<double> ft3 = new_async(f3);
std::experimental::future<std::tuple<std::experimental::future<int>,
                                     std::experimental::future<std::string>,
                                     std::experimental::future<double>>>
    res = std::experimental::when_all(std::move(ft1), std::move(ft2),
                                      std::move(ft3));
```

## CSP（Communicating Sequential Processes）

* CSP 是一种描述并发系统交互的编程模型，线程理论上是分开的，没有共享数据，每个线程可以完全独立地思考，消息通过 communication channel 在不同线程间传递，线程行为取决于收到的消息，因此每个线程实际上是一个状态机，收到一条消息时就以某种方式更新状态，并且还可能发送消息给其他线程。Erlang 采用了这种编程模型，并用于 [MPI](https://en.wikipedia.org/wiki/Message_Passing_Interface) 做 C 和 C++ 的高性能计算。真正的 CSP 没有共享数据，所有通信通过消息队列传递，但由于 C++ 线程共享地址空间，无法强制实现这个要求，所以需要应用或者库的作者来确保线程间不会共享数据
* 考虑实现一个 ATM 应用，它需要处理取钱时和银行的交互，并控制物理机器对银行卡的反应。一个处理方法是分三个线程，分别处理物理机器、ATM 逻辑、与银行的交互，线程间通过消息通讯而非共享数据，比如插卡时机器线程发送消息给逻辑线程，逻辑线程返回一条消息通知机器线程可以给多少钱
* 一个简单的 ATM 逻辑的状态机建模如下
* CSP는 동시 시스템의 상호작용을 설명하는 프로그래밍 모델로 이론적으로 스레드가 분리되어 있고 공유 데이터가 없으며 각 스레드가 완전히 독립적으로 생각할 수 있다. 메시지는 통신 채널을 통해 스레드 간에 전달되며 스레드의 동작은 수신하는 메시지에 따라 달라지므로 각 스레드는 사실상 상태 머신으로 메시지를 수신할 때 어떤 방식으로든 상태를 업데이트하고 다른 스레드로 메시지를 보낼 수도 있다. Erlang은 이 프로그래밍 모델을 채택하고 [MPI](https://en.wikipedia.org/wiki/Message_Passing_Interface)에서 이를 사용하여 C와 C++에서 고성능 컴퓨팅을 수행한다. 진정한 CSP는 데이터를 공유하지 않으며 모든 통신은 메시지 큐를 통해 전달되지만 C++ 스레드는 주소 공간을 공유하기 때문에 이를 강제할 수 없으므로 애플리케이션 또는 라이브러리 작성자가 스레드 간에 데이터가 공유되지 않도록 보장해야 한다.
* 돈을 인출할 때 은행과의 상호 작용을 처리하고 은행 카드에 대한 물리적 기계의 응답을 제어해야 하는 ATM 애플리케이션을 구현한다고 가정해 보겠다. 이를 처리하는 한 가지 방법은 물리적 기계, ATM 로직, 은행과의 상호작용을 처리하는 세 개의 스레드를 사용하는 것이다. 스레드는 데이터를 공유하는 대신 메시지를 통해 서로 통신한다(예: 카드를 삽입하면 기계 스레드가 로직 스레드에 메시지를 보내고, 로직 스레드는 기계 스레드에 얼마의 돈을 줄 수 있는지 알려주는 메시지를 반환하는 식).
* 간단한 ATM 로직 상태 머신은 다음과 같이 모델링된다.    
  
![](images/3-1.png)

* 这个 ATM 逻辑的状态机与系统的其他部分各自运行在独立的线程上，不需要考虑同步和并发的问题，只要考虑在某个点接受和发送的消息，这种设计方式称为 actor model，系统中有多个独立的 actor，actor 之间可以互相发送消息但不会共享状态，这种方式可以极大简化并发系统的设计。完整的代码实现[见此](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/src/atm.cpp)
* 이 ATM 로직의 상태 머신과 나머지 시스템은 각각 별도의 스레드에서 실행되며 특정 지점에서 수신 및 전송되는 메시지를 고려하는 한 동기화 및 동시성을 고려할 필요가 없으며 이 디자인을 액터 모델이라고 하며 시스템에는 여러 독립 액터가 있으며 액터는 서로 메시지를 보낼 수 있지만 상태를 공유하지 않으며 이 접근 방식은 동시 시스템 설계를 크게 단순화한다. 전체 코드 구현 [여기 참조](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/src/atm.cpp )  
  