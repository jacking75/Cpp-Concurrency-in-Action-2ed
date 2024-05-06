## 스레드 간 데이터 공유 관련 문제
  
* 불변성: 특정 데이터 구조에 대해 항상 참인 문장으로, `양방향 체인 테이블에서 인접한 두 노드 A와 B는 A의 뒤쪽 포인터는 B를 가리키고 B의 앞쪽 포인터는 A를 가리켜야 한다` 와 같이 특정 데이터 구조에 대해 항상 참인 문장을 말한다. 프로그램이 편의상 불변성을 일시적으로 파괴하는 경우가 있는데, 이는 복잡한 데이터 구조를 업데이트할 때 주로 발생하며, 예를 들어 양방향 체인 테이블에서 노드 N을 삭제할 때 먼저 N의 이전 노드가 N의 다음 노드를 가리키게 한 다음(불변성이 파괴됨), N의 다음 노드가 이전 노드를 가리키고 마지막으로 N을 삭제하는 방식으로(이 시점에서 불변성이 다시 복원됨) 이루어진다.
* 불변성의 파괴는 한 스레드가 공유 데이터를 수정할 때 발생하며, 이때 다른 스레드가 액세스하면 불변성이 영구적으로 파괴될 수 있으며 이것이 바로 race condition 이다.
* 스레드가 실행되는 순서가 결과에 영향을 미치지 않는다면 이는 신경 쓸 필요가 없는 선의의 경쟁이다. 우려되는 것은 불변성이 파괴될 때 발생하는 race condition 이다.
* C++ 표준은 단일 객체가 동시에 수정되는 특정 종류의 경쟁 조건을 지칭하는 데이터 경쟁 개념을 정의한다. data race는 정의되지 않은 동작을 유발할 수 있다.
* race condition은 다른 스레드가 동일한 데이터 블록에 액세스하는 동안 한 스레드가 계속 진행해야 하므로 문제가 발생하면 재현하기 어렵고, 프로그래밍은 race condition을 피하기 위해 복잡한 연산을 많이 사용해야 한다.
  
  
  
## mutex
* 뮤텍스를 사용하여 공유 데이터에 액세스하기 전에 잠그고 액세스가 완료되면 잠금을 해제한다. 특정 뮤텍스로 스레드를 잠그면 다른 스레드는 해당 스레드의 뮤텍스가 잠금 해제될 때까지 기다려야 공유 데이터에 액세스할 수 있다.  
* C++11은 뮤텍스를 생성하기 위해 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex )를 제공하며, [lock](https://en.cppreference.com/w/cpp/thread/mutex/lock)로 잠그고 [unlock](https://en.cppreference.com/w/cpp/thread/mutex/unlock )으로 잠금을 해제할 수 있다. 이 두 멤버 함수를 수동으로 사용하는 대신 [std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard )를 사용하여 잠금 및 잠금 해제를 자동으로 처리할 수 있는데, 이 함수는 생성 시 mutex를 수락하고 생성 시 mutex.lock()를 호출하고, 파괴 시에는 mutex.unlock()을 호출한다.  
  
```cpp
#include <iostream>
#include <mutex>

class A {
 public:
  void lock() { std::cout << "lock" << std::endl; }
  void unlock() { std::cout << "unlock" << std::endl; }
};

int main() {
  A a;
  {
    std::lock_guard<A> l(a);  // lock
  }                           // unlock
}
```
  
* C++17 提供了的 [std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock)，它可以接受任意数量的 mutex，并将这些 mutex 传给 [std::lock](https://en.cppreference.com/w/cpp/thread/lock) 来同时上锁，它会对其中一个 mutex 调用 lock()，对其他调用 try_lock()，若 try_lock() 返回 false 则对已经上锁的 mutex 调用 unlock()，然后重新进行下一轮上锁，标准未规定下一轮的上锁顺序，可能不一致，重复此过程直到所有 mutex 上锁，从而达到同时上锁的效果。C++17 支持类模板实参推断，可以省略模板参数  
* C++17은 뮤텍스를 원하는 수만큼 받아서 [std::lock](https://en.cppreference.com/w/cpp/thread/lock )에 전달하여 동시에 잠그는 [std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock ) 함수를 제공한다. 뮤텍스 중 하나에서는 lock(), 다른 뮤텍스에서는 try_lock(), try_lock()이 false를 반환하면 이미 잠긴 뮤텍스에서 unlock()을 호출하고 다음 단계의 잠금을 수행한다. try_lock()이 false를 반환하면 잠긴 뮤텍스에서 unlock()을 호출한 다음 다음 잠금 라운드로 진행하는데, 표준에서는 다음 라운드의 잠금 순서를 지정하지 않아 일관성이 없을 수 있으며 모든 뮤텍스가 잠길 때까지 이 과정을 반복하여 동시에 잠금 효과를 얻을 수 있다.  
  
```cpp
#include <iostream>
#include <mutex>

class A {
 public:
  void lock() { std::cout << 1; }
  void unlock() { std::cout << 2; }
  bool try_lock() {
    std::cout << 3;
    return true;
  }
};

class B {
 public:
  void lock() { std::cout << 4; }
  void unlock() { std::cout << 5; }
  bool try_lock() {
    std::cout << 6;
    return true;
  }
};

int main() {
  A a;
  B b;
  {
    std::scoped_lock l(a, b);  // 16
    std::cout << std::endl;
  }  // 25
}
```
  
* 일반적으로 뮤텍스는 보호할 데이터가 있는 클래스에 배치되며, 전역 변수가 아닌 비공개 데이터 멤버로 정의되므로 코드가 더 명확해진다. 그러나 멤버 함수가 데이터 멤버에 대한 포인터나 참조를 반환하는 경우 해당 포인터를 통한 액세스는 뮤텍스에 의해 제한되지 않으므로 뮤텍스가 데이터를 잠그도록 인터페이스 설정 방법에 주의를 기울여야 한다.  

```cpp
#include <mutex>

class A {
 public:
  void f() {}
};

class B {
 public:
  A* get_data() {
    std::lock_guard<std::mutex> l(m_);
    return &data_;
  }

 private:
  std::mutex m_;
  A data_;
};

int main() {
  B b;
  A* p = b.get_data();
  p->f();  // mutex 잠금 없이 데이터 엑세스
}
```
  
* 매우 간단한 인터페이스에서도 race condition이 발생할 수 있다.  
  
```cpp
std::stack<int> s；
if (!s.empty()) {
  int n = s.top();  // 다른 스레드에서 pop 하면 잘못된 top이 반환된다  
  s.pop();
}
```  
    
* 스택의 맨 위를 가져오기 전에 null이 아닌지 확인하는 위의 코드는 단일 스레드에서는 안전하지만, 멀티 스레드에서는 다른 스레드가 먼저 pop될 경우 현재 스레드의 맨 위가 잘못될 수 있다. 또 다른 잠재적 논쟁은 두 스레드 모두 pop 되지 않고 개별적으로 맨 위를 가져오는 경우 정의되지 않은 동작이 발생하지는 않지만 동일한 값이 두 번 처리되는 경우 오류가 없는 것처럼 보이고 버그를 찾기가 어렵기 때문에 더 나빠질 수 있다는 것이다.  
* 그렇다면 그냥 스택의 맨 위를 반환하도록 하면 어떨까? 그 이유는 반환값을 구성하는 과정에서 예외가 발생할 수 있고, 반환하지 않고 pop 하면 데이터가 손실될 수 있기 때문이다. 예를 들어, 벡터를 요소로 하는 스택이 있고 벡터를 복사하기 위해 힙에 메모리를 할당해야 하는데 시스템에 부하가 많거나 리소스가 제한되어 있는 경우(예: 벡터에 많은 수의 요소가 있는 경우) 벡터의 복사 생성자는 [std::bad_alloc](https://en.cppreference.com/w/cpp/memory/new/bad_alloc ) 예외를 발생시킨다. pop이 스택 최상위 요소의 값을 반환할 수 있다면, 반환은 실행된 마지막 문이어야 하고 스택은 반환하기 전에 이미 요소를 pop 했지만, 반환 값을 복사할 때 예외가 발생하면 pop 된 데이터가 손실된다(스택에서 제거되지만 복사는 실패한다). 그래서 [std::stack](https://en.cppreference.com/w/cpp/container/stack )의 디자이너는 이 연산을 top과 pop의 두 부분으로 나눈다.  
* top과 pop을 한 단계로 결합하는 몇 가지 방법을 생각해 보자. 가장 먼저 생각하기 쉬운 방법은 결과 값을 얻기 위해 참조를 전달하는 것이다. 이 접근법의 명백한 단점은 스택의 엘리먼트 타입 인스턴스를 구성해야 하는데, 이는 비실용적이며 결과를 얻기 위해 임시 객체를 구성하는 것은 비용 효율적이지 않으며 엘리먼트 타입이 (예를 들어 사용자 정의 타입의) 할당을 지원하지 않을 수 있고 생성자에 일부 인자가 필요할 수 있다는 점이다.  
  
```cpp
std::vector<int> res;
s.pop(res);
```
  
* pop은 프로시저가 예외를 던진다는 우려만 있는 값을 반환하므로, 두 번째 옵션은 예외를 던지지 않는 엘리먼트 타입에 대한 복사 또는 이동 생성자를 설정하는 것으로 [std::is_nothrow_copy_constructable](https://en.cppreference.com/w/cpp/types/is_copy_constructible ) 및 [std::is_nothrow_move_constructible](https://en.cppreference.com/w/cpp/types/is_move_constructible )을 사용할 수 있다. 그러나 이 방법은 너무 제한적이며 복사나 이동이 예외를 던지지 않는 타입만 지원한다.  
* 세 번째 옵션은 예외를 던지지 않고 자유롭게 복사할 수 있는 pop된 엘리먼트에 대한 포인터를 반환하는 것으로 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr )이 좋은 선택이지만, 이 옵션의 오버헤드가 너무 높다. 타입의 경우 예를 들어 int는 4바이트이고 `shared_ptr<int>`는 16바이트로 4배나 많다.  
* 네 번째 옵션은 옵션 1, 2 또는 3을 결합하는 것이다(예: 옵션 1 또는 3과 함께 스레드 안전 스택을 구현하는 경우).    
  
```cpp
#include <exception>
#include <memory>
#include <mutex>
#include <stack>
#include <utility>

struct EmptyStack : std::exception {
  const char* what() const noexcept { return "empty stack!"; }
};

template <typename T>
class ConcurrentStack {
 public:
  ConcurrentStack() = default;

  ConcurrentStack(const ConcurrentStack& rhs) {
    std::lock_guard<std::mutex> l(rhs.m_);
    s_ = rhs.s_;
  }

  ConcurrentStack& operator=(const ConcurrentStack&) = delete;

  void push(T x) {
    std::lock_guard<std::mutex> l(m_);
    s_.push(std::move(x));
  }

  bool empty() const {
    std::lock_guard<std::mutex> l(m_);
    return s_.empty();
  }

  std::shared_ptr<T> pop() {
    std::lock_guard<std::mutex> l(m_);
    if (s_.empty()) {
      throw EmptyStack();
    }
    auto res = std::make_shared<T>(std::move(s_.top()));
    s_.pop();
    return res;
  }

  void pop(T& res) {
    std::lock_guard<std::mutex> l(m_);
    if (s_.empty()) {
      throw EmptyStack();
    }
    res = std::move(s_.top());
    s_.pop();
  }

 private:
  mutable std::mutex m_;
  std::stack<T> s_;
};
```
  
* 잠금 세분성(잠금으로 보호되는 데이터의 크기)이 너무 작으면 보호 작업이 완전히 커버되지 않으며, 여기서는 세분성이 더 커서 많은 수의 작업을 커버한다.   그러나 세분성이 클수록 좋지만 잠금 세분성이 너무 커서 리소스를 놓고 경쟁을 요청하는 스레드가 너무 많으면 동시성 성능이 저하한다.  
* 주어진 연산에 여러 뮤텍스에 대한 잠금이 필요한 경우, 교착 상태라는 새로운 잠재적 문제가 발생한다.  
  
## 교착 상태
  
* 교착 상태에 필요한 네 가지 조건은 상호 배제, 소유 및 대기, 사용할 수 없음, 주기적 대기이다.
* 교착 상태를 피하려면 일반적으로 두 뮤텍스를 같은 순서로 잠그는 것이 좋지만, 항상 A를 B보다 먼저 잠그는 것이 좋지만 모든 경우에 적용되는 것은 아니다. [std::lock](https://en.cppreference.com/w/cpp/thread/lock )은 교착 상태의 위험 없이 동시에 여러 뮤텍스를 잠글 수 있으며, 예외를 던진 다음 잠그지 않을 수 있으므로 둘 다 또는 어느 쪽도 아니다!  
    
```cpp
#include <mutex>
#include <thread>

struct A {
  explicit A(int n) : n_(n) {}
  std::mutex m_;
  int n_;
};

void f(A &a, A &b, int n) {
  if (&a == &b) {
    return;  // 동일한 개체의 반복적인 잠금 방지
  }
  std::lock(a.m_, b.m_);  // 동시 잠금으로 교착 상태 방지
  // 다음 잠금은 고정된 순서로 추가되므로 교착 상태 문제는 없는 것처럼 보인다.
  // 하지만 std::lock이 동시에 잠기지 않은 상태에서 다른 스레드에서 f(b, a, n)이 실행되면 
  // 두 잠금의 순서가 뒤바뀌어서 교착 상태가 발생할 수 있다.
  std::lock_guard<std::mutex> lock1(a.m_, std::adopt_lock);
  std::lock_guard<std::mutex> lock2(b.m_, std::adopt_lock);

  // 동등한 구현, 처음에는 잠금 없이, 그다음에는 잠금과 동시에 구현하기
  //   std::unique_lock<std::mutex> lock1(a.m_, std::defer_lock);
  //   std::unique_lock<std::mutex> lock2(b.m_, std::defer_lock);
  //   std::lock(lock1, lock2);

  a.n_ -= n;
  b.n_ += n;
}

int main() {
  A x{70};
  A y{30};

  std::thread t1(f, std::ref(x), std::ref(y), 20);
  std::thread t2(f, std::ref(y), std::ref(x), 10);

  t1.join();
  t2.join();
}
```
  
* [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) mutex를 수락하고 생성 시 mutex.lock()，소멸 시에는 mutex.unlock()을 호출한다.  
  
```cpp
#include <iostream>
#include <mutex>

class A {
 public:
  void lock() { std::cout << "lock" << std::endl; }
  void unlock() { std::cout << "unlock" << std::endl; }
};

int main() {
  A a;
  {
    std::unique_lock l(a);  // lock
  }                         // unlock
}
```

* [std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard) 未提供任何接口且不支持拷贝和移动，而 [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) 多提供了一些接口，使用更灵活，占用的空间也多一点。一种要求灵活性的情况是转移锁的所有权到另一个作用域

```cpp
std::unique_lock<std::mutex> get_lock() {
  extern std::mutex m;
  std::unique_lock<std::mutex> l(m);
  prepare_data();
  return l;  // 不需要 std::move，编译器负责调用移动构造函数
}

void f() {
  std::unique_lock<std::mutex> l(get_lock());
  do_something();
}
```

* 对一些费时的操作上锁可能造成很多操作被阻塞，可以在面对这些操作时先解锁

```cpp
void process_file_data() {
  std::unique_lock<std::mutex> l(m);
  auto data = get_data();
  l.unlock();  // 费时操作没有必要持有锁，先解锁
  auto res = process(data);
  l.lock();  // 写入数据前上锁
  write_result(data, res);
}
```

* C++17 最优的同时上锁方法是使用 [std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock)
* 解决死锁并不简单，[std::lock](https://en.cppreference.com/w/cpp/thread/lock) 和 [std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock) 无法获取其中的锁，此时解决死锁更依赖于开发者的能力。避免死锁有四个建议
  * 第一个避免死锁的建议是，一个线程已经获取一个锁时就不要获取第二个。如果每个线程只有一个锁，锁上就不会产生死锁（但除了互斥锁，其他方面也可能造成死锁，比如即使无锁，线程间相互等待也可能造成死锁）
  * 第二个建议是，持有锁时避免调用用户提供的代码。用户提供的代码可能做任何时，包括获取锁，如果持有锁时调用用户代码获取锁，就会违反第一个建议，并造成死锁。但有时调用用户代码是无法避免的
  * 第三个建议是，按固定顺序获取锁。如果必须获取多个锁且不能用 [std::lock](https://en.cppreference.com/w/cpp/thread/lock) 同时获取，最好在每个线程上用固定顺序获取。上面的例子虽然是按固定顺序获取锁，但如果不同时加锁就会出现死锁，对于这种情况的建议是规定固定的调用顺序
  * 第四个建议是使用层级锁，如果一个锁被低层持有，就不允许在高层再上锁
* 层级锁实现如下

```cpp
#include <iostream>
#include <mutex>
#include <stdexcept>

class HierarchicalMutex {
 public:
  explicit HierarchicalMutex(int hierarchy_value)
      : cur_hierarchy_(hierarchy_value), prev_hierarchy_(0) {}

  void lock() {
    validate_hierarchy();  // 层级错误则抛异常
    m_.lock();
    update_hierarchy();
  }

  bool try_lock() {
    validate_hierarchy();
    if (!m_.try_lock()) {
      return false;
    }
    update_hierarchy();
    return true;
  }

  void unlock() {
    if (thread_hierarchy_ != cur_hierarchy_) {
      throw std::logic_error("mutex hierarchy violated");
    }
    thread_hierarchy_ = prev_hierarchy_;  // 恢复前一线程的层级值
    m_.unlock();
  }

 private:
  void validate_hierarchy() {
    if (thread_hierarchy_ <= cur_hierarchy_) {
      throw std::logic_error("mutex hierarchy violated");
    }
  }

  void update_hierarchy() {
    // 先存储当前线程的层级值（用于解锁时恢复）
    prev_hierarchy_ = thread_hierarchy_;
    // 再把其设为锁的层级值
    thread_hierarchy_ = cur_hierarchy_;
  }

 private:
  std::mutex m_;
  const int cur_hierarchy_;
  int prev_hierarchy_;
  static thread_local int thread_hierarchy_;  // 所在线程的层级值
};

// static thread_local 表示存活于一个线程周期
thread_local int HierarchicalMutex::thread_hierarchy_(INT_MAX);

HierarchicalMutex high(10000);
HierarchicalMutex mid(6000);
HierarchicalMutex low(5000);

void lf() {  // 最低层函数
  std::lock_guard<HierarchicalMutex> l(low);
  // 调用 low.lock()，thread_hierarchy_ 为 INT_MAX，
  // cur_hierarchy_ 为 5000，thread_hierarchy_ > cur_hierarchy_，
  // 通过检查，上锁，prev_hierarchy_ 更新为 INT_MAX，
  // thread_hierarchy_ 更新为 5000
}  // 调用 low.unlock()，thread_hierarchy_ == cur_hierarchy_，
// 通过检查，thread_hierarchy_ 恢复为 prev_hierarchy_ 保存的 INT_MAX，解锁

void hf() {
  std::lock_guard<HierarchicalMutex> l(high);  // high.cur_hierarchy_ 为 10000
  // thread_hierarchy_ 为 10000，可以调用低层函数
  lf();  // thread_hierarchy_ 从 10000 更新为 5000
  //  thread_hierarchy_ 恢复为 10000
}  //  thread_hierarchy_ 恢复为 INT_MAX

void mf() {
  std::lock_guard<HierarchicalMutex> l(mid);  // thread_hierarchy_ 为 6000
  hf();  // thread_hierarchy_ < high.cur_hierarchy_，违反了层级结构，抛异常
}

int main() {
  lf();
  hf();
  try {
    mf();
  } catch (std::logic_error& ex) {
    std::cout << ex.what();
  }
}
```

### 读写锁（reader-writer mutex）

* 有时会希望对一个数据上锁时，根据情况，对某些操作相当于不上锁，可以并发访问，对某些操作保持上锁，同时最多只允许一个线程访问。比如对于需要经常访问但很少更新的缓存数据，用 [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) 加锁会导致同时最多只有一个线程可以读数据，这就需要用上读写锁，读写锁允许多个线程并发读但仅一个线程写
* C++14 提供了 [std::shared_timed_mutex](https://en.cppreference.com/w/cpp/thread/shared_timed_mutex)，C++17 提供了接口更少性能更高的 [std::shared_mutex](https://en.cppreference.com/w/cpp/thread/shared_mutex)，如果多个线程调用 shared_mutex.lock_shared()，多个线程可以同时读，如果此时有一个写线程调用 shared_mutex.lock()，则读线程均会等待该写线程调用 shared_mutex.unlock()。C++11 没有提供读写锁，可使用 [boost::shared_mutex](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.mutex_types.shared_mutex)
* C++14 提供了 [std::shared_lock](https://en.cppreference.com/w/cpp/thread/shared_lock)，它在构造时接受一个 mutex，并会调用 mutex.lock_shared()，析构时会调用 mutex.unlock_shared()

```cpp
#include <iostream>
#include <shared_mutex>

class A {
 public:
  void lock_shared() { std::cout << "lock_shared" << std::endl; }
  void unlock_shared() { std::cout << "unlock_shared" << std::endl; }
};

int main() {
  A a;
  {
    std::shared_lock l(a);  // lock_shared
  }                         // unlock_shared
}
```

* 对于 [std::shared_mutex](https://en.cppreference.com/w/cpp/thread/shared_mutex)，通常在读线程中用 [std::shared_lock](https://en.cppreference.com/w/cpp/thread/shared_lock) 管理，在写线程中用 [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock) 管理

```cpp
class A {
 public:
  int read() const {
    std::shared_lock<std::shared_mutex> l(m_);
    return n_;
  }

  int write() {
    std::unique_lock<std::shared_mutex> l(m_);
    return ++n_;
  }

 private:
  mutable std::shared_mutex m_;
  int n_ = 0;
};
```

### 递归锁

* [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex) 是不可重入的，未释放前再次上锁是未定义行为

```cpp
#include <mutex>

class A {
 public:
  void f() {
    m_.lock();
    m_.unlock();
  }

  void g() {
    m_.lock();
    f();
    m_.unlock();
  }

 private:
  std::mutex m_;
};

int main() {
  A{}.g();  // Undefined Behavior
}
```

* 为此 C++ 提供了 [std::recursive_mutex](https://en.cppreference.com/w/cpp/thread/recursive_mutex)，它可以在一个线程上多次获取锁，但在其他线程获取锁之前必须释放所有的锁

```cpp
#include <mutex>

class A {
 public:
  void f() {
    m_.lock();
    m_.unlock();
  }

  void g() {
    m_.lock();
    f();
    m_.unlock();
  }

 private:
  std::recursive_mutex m_;
};

int main() {
  A{}.g();  // OK
}
```

* 多数情况下，如果需要递归锁，说明代码设计存在问题。比如一个类的每个成员函数都会上锁，一个成员函数调用另一个成员函数，就可能多次上锁，这种情况用递归锁就可以避免产生未定义行为。但显然这个设计本身是有问题的，更好的办法是提取其中一个函数作为 private 成员并且不上锁，其他成员先上锁再调用该函数

## 对并发初始化的保护

* 除了对并发访问共享数据的保护，另一种常见的情况是对并发初始化的保护

```cpp
#include <memory>
#include <mutex>
#include <thread>

class A {
 public:
  void f() {}
};

std::shared_ptr<A> p;
std::mutex m;

void init() {
  m.lock();
  if (!p) {
    p.reset(new A);
  }
  m.unlock();
  p->f();
}

int main() {
  std::thread t1{init};
  std::thread t2{init};

  t1.join();
  t2.join();
}
```

* 上锁只是为了保护初始化过程，会不必要地影响性能，一种容易想到的优化方式是双重检查锁模式，但这存在潜在的 race condition

```cpp
#include <memory>
#include <mutex>
#include <thread>

class A {
 public:
  void f() {}
};

std::shared_ptr<A> p;
std::mutex m;

void init() {
  if (!p) {  // 未上锁，其他线程可能在执行 #1，则此时 p 不为空
    std::lock_guard<std::mutex> l(m);
    if (!p) {
      p.reset(new A);  // 1
      // 先分配内存，再在内存上构造 A 的实例并返回内存的指针，最后让 p 指向它
      // 也可能先让 p 指向它，再在内存上构造 A 的实例
    }
  }
  p->f();  // p 可能指向一块还未构造实例的内存，从而崩溃
}

int main() {
  std::thread t1{init};
  std::thread t2{init};

  t1.join();
  t2.join();
}
```

* 为此，C++11 提供了 [std::once_flag](https://en.cppreference.com/w/cpp/thread/once_flag) 和 [std::call_once](https://en.cppreference.com/w/cpp/thread/call_once) 来保证对某个操作只执行一次

```cpp
#include <memory>
#include <mutex>
#include <thread>

class A {
 public:
  void f() {}
};

std::shared_ptr<A> p;
std::once_flag flag;

void init() {
  std::call_once(flag, [&] { p.reset(new A); });
  p->f();
}

int main() {
  std::thread t1{init};
  std::thread t2{init};

  t1.join();
  t2.join();
}
```

* [std::call_once](https://en.cppreference.com/w/cpp/thread/call_once) 也可以用在类中

```cpp
#include <iostream>
#include <mutex>
#include <thread>

class A {
 public:
  void f() {
    std::call_once(flag_, &A::print, this);
    std::cout << 2;
  }

 private:
  void print() { std::cout << 1; }

 private:
  std::once_flag flag_;
};

int main() {
  A a;
  std::thread t1{&A::f, &a};
  std::thread t2{&A::f, &a};
  t1.join();
  t2.join();
}  // 122
```

* static 局部变量在声明后就完成了初始化，这存在潜在的 race condition，如果多线程的控制流同时到达 static 局部变量的声明处，即使变量已在一个线程中初始化，其他线程并不知晓，仍会对其尝试初始化。为此，C++11 规定，如果 static 局部变量正在初始化，线程到达此处时，将等待其完成，从而避免了 race condition。只有一个全局实例时，可以直接用 static 而不需要 [std::call_once](https://en.cppreference.com/w/cpp/thread/call_once)

```cpp
template <typename T>
class Singleton {
 public:
  static T& Instance();
  Singleton(const Singleton&) = delete;
  Singleton& operator=(const Singleton&) = delete;

 private:
  Singleton() = default;
  ~Singleton() = default;
};

template <typename T>
T& Singleton<T>::Instance() {
  static T instance;
  return instance;
}
```
