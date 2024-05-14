## 메모리 모델의 기본 사항

* 경합 조건을 피하려면 스레드가 실행될 순서를 지정해야 한다. 이를 위한 한 가지 방법은 뮤텍스를 사용하는 것인데 여기서 후자의 스레드는 전자의 스레드가 잠금 해제될 때까지 기다려야 한다. 두 번째 방법은 원자 연산을 사용하여 동일한 메모리 위치에 액세스하기 위한 경쟁을 피하는 것이다.
* 원자 연산은 완료되거나 완료되지 않는 분할 불가능한 연산으로, 반쯤 완료된 상태 같은 것은 존재하지 않는다. 객체의 값을 읽는 로드 연산이 원자 연산이면 객체에 대한 모든 수정 연산도 원자 연산이며, 일부 수정이 완료된 후 초기 값이나 저장된 값을 읽는다. 따라서 원자 연산은 수정 중에 다른 스레드가 값을 볼 수 있는 상황이 발생하지 않으며 경합의 위험을 피할 수 있다.
* 각 객체는 초기화부터 모든 스레드에서 객체에 대한 쓰기 작업으로 구성된 수정 순서를 갖는다. 일반적으로 이 순서는 런타임에 변경되지만, 특정 프로그램 실행에서 시스템의 모든 스레드는 다음 순서를 따라야 한다.
* 객체가 원자형이 아닌 경우, 동기화를 사용하여 스레드가 각 변수가 수정되는 순서를 따르도록 한다. 변수가 스레드마다 다른 값 순서를 나타내면 데이터 경합과 정의되지 않은 동작이 발생할 수 있다. 원자 연산을 사용하면 동기화 책임이 컴파일러에 넘어간다.
  
  
## 원자 연산과 원자 타입

### 표준 원자 타입

* 표준 원자형은 [\<atomic\>](https://en.cppreference.com/w/cpp/header/atomic)에 정의되어 있다. 뮤텍스로 원자 연산을 시뮬레이션하는 것도 가능하며, 실제로 표준 원자 형은 그렇게 구현될 수 있다. 둘 다 [is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free) 함수를 가지고 있는데, 이 함수는 참을 반환하여 모두 [is_lock_free]() 함수가 있는데, 이 함수는 원자 연산이 락이 없고 원자 명령어를 사용함을 나타내는 참을 반환하고, 락을 사용함을 나타내는 거짓을 반환한다.

```cpp
struct A {
  int a[100];
};

struct B {
  int x, y;
};

assert(!std::atomic<A>{}.is_lock_free());
assert(std::atomic<B>{}.is_lock_free());
```

* atomic 오퍼레이션의 주요 용도는 동기화를 위해 뮤텍스를 대체하는 것이다. 뮤텍스를 사용하여 내부적으로 원자 연산을 구현하는 경우, 성능 향상을 기대할 수 없으며 뮤텍스와 직접 동기화하는 것이 좋다. C++17의 각 원자 타입에는 [is_always_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_always_lock_free ) 멤버 변수가 있으며, 참이면 해당 원자 타입이 이 플랫폼에서 lock이 해제되어 있음을 의미한다.

```cpp
assert(std::atomic<int>{}.is_always_lock_free);
```

* C++17 이전에는 각 원자 유형에 대해 표준 라이브러리에 정의된 [ATOMIC_xxx_LOCK_FREE](https://en.cppreference.com/w/c/atomic/ATOMIC_LOCK_FREE_consts) 매크로를 사용하여 해당 유형의 잠금 여부를 확인할 수 있으며, 0 값은 해당 원자 타입이 잠기지 않았음을 나타낸다. 값이 0이면 타입이 잠겨 있음을, 2이면 잠겨 있지 않음을 1이면 런타임에만 확인할 수 있음을 의미한다.  
  
```cpp
// LOCK-FREE PROPERTY
#define ATOMIC_BOOL_LOCK_FREE 2
#define ATOMIC_CHAR_LOCK_FREE 2
#ifdef __cpp_lib_char8_t
#define ATOMIC_CHAR8_T_LOCK_FREE 2
#endif // __cpp_lib_char8_t
#define ATOMIC_CHAR16_T_LOCK_FREE 2
#define ATOMIC_CHAR32_T_LOCK_FREE 2
#define ATOMIC_WCHAR_T_LOCK_FREE  2
#define ATOMIC_SHORT_LOCK_FREE  2
#define ATOMIC_INT_LOCK_FREE    2
#define ATOMIC_LONG_LOCK_FREE   2
#define ATOMIC_LLONG_LOCK_FREE  2
#define ATOMIC_POINTER_LOCK_FREE  2
```
  
* 모든 연산에 대해 무잠금을 보장하는 단순 부울 플래그인 [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag) 만이 is_lock_free를 제공하지 않는다. [atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag)를 사용하면 간단한 잠금을 구현하고 다른 기본 원자 타입을 구현할 수 있다. 나머지 원자 타입은 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)을 특수화하여 구현할 수 있으며 더 완전한 기능을 가질 수 있지만 잠금이 없는 것이 보장되지는 않는다!
* 타입 별칭은 내장 타입의 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 전문화를 위한 표준 라이브러리에 정의되어 있다.  

```cpp
namespace std {
using atomic_bool = atomic<bool>;
using atomic_char = std::atomic<char>;
}  // namespace std
```

* 'std::atomic<T>' 타입의 일반적인 별칭은 `atomic_T` 이며, 다음과 같은 예외가 있다: signed는 s, unsigned는 u, longlong은 llong으로 축약
  
```cpp
namespace std {
using atomic_schar = std::atomic<signed char>;
using atomic_uchar = std::atomic<unsigned char>;
using atomic_uint = std::atomic<unsigned>;
using atomic_ushort = std::atomic<unsigned short>;
using atomic_ulong = std::atomic<unsigned long>;
using atomic_llong = std::atomic<long long>;
using atomic_ullong = std::atomic<unsigned long long>;
}  // namespace std
```

* 원자 타입은 다른 원자 타입으로 복사 및 할당할 수 없다. 할당 복사 시 두 개의 객체가 호출되고 연산의 원자성이 파괴되므로 원자형은 복사 및 할당이 허용되지 않는다. 그러나 해당 기본 제공 타입을 사용할 수는 있다

```cpp
T operator=(T desired) noexcept;
T operator=(T desired) volatile noexcept;
atomic& operator=(const atomic&) = delete;
atomic& operator=(const atomic&) volatile = delete;
```

* [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 은 할당을 지원하는 멤버 함수를 제공한다.  

```cpp
std::atomic<T>::store     // 현재 값을 바꾼다
std::atomic<T>::load      // 현재 값을 반환한다
std::atomic<T>::exchange  // 값을 바꾸고, 바꾸기 전의 값을 반환한다

// 원하는 값과 비교, 같지 않으면 원하는 값을 원자 값으로 설정하고 거짓을 반환한다.
// 같으면 원자 값을 목표 값으로 설정하고 참을 반환한다.
// CAS(비교-교환) 명령어가 없는 머신에서는 약한 버전이 같을 때 대체하지 못하고 거짓을 반환할 수 있다.
// 따라서 약한 버전은 일반적으로 루프가 필요한 반면, 강한 버전은 불평등을 보장하기 위해 거짓을 반환한다.
std::atomic<T>::compare_exchange_weak
std::atomic<T>::compare_exchange_strong

std::atomic<T>::fetch_add        // 더하기 전 값을 반환
std::atomic<T>::fetch_sub        // 빼기 전 값을 반환
std::atomic<T>::fetch_and
std::atomic<T>::fetch_or
std::atomic<T>::fetch_xor
std::atomic<T>::operator++       // 사전 자기 증가는 fetch_add(1) + 1 와 같다
std::atomic<T>::operator++(int)  // 사후 증가는 fetch_add(1) 와 같다
std::atomic<T>::operator--       // 사전 뺄셈은 fetch_sub(1) - 1 에 해당한다
std::atomic<T>::operator--(int)  // fetch_sub(1) 과 동등한 사후 감산
std::atomic<T>::operator+=       // fetch_add(x) + x
std::atomic<T>::operator-=       // fetch_sub(x) - x
std::atomic<T>::operator&=       // fetch_and(x) & x
std::atomic<T>::operator|=       // fetch_or(x) | x
std::atomic<T>::operator^=       // fetch_xor(x) ^ x
```

* 멤버 함수에는 메모리 순서를 지정하는 데 사용되는 매개 변수 [std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) 가 있으며 이것은 다음에 설명한다  
  
```cpp
typedef enum memory_order {
  memory_order_relaxed,
  memory_order_consume,
  memory_order_acquire,
  memory_order_release,
  memory_order_acq_rel,
  memory_order_seq_cst
} memory_order;

void store(T desired, std::memory_order order = std::memory_order_seq_cst);
// store 메모리 순서는 다음과 같다
// memory_order_relaxed、memory_order_release、memory_order_seq_cst
T load(std::memory_order order = std::memory_order_seq_cst);
// load 메모리 순서는 다음과 같다
// memory_order_relaxed、memory_order_consume、memory_order_acquire、memory_order_seq_cst
```

### [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag)

* [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag) 는 atomic bool 이며 잠금이 없는 것이 보장되는 유일한 atomic 타입으로[ATOMIC_FLAG_INIT](https://en.cppreference.com/w/cpp/atomic/ATOMIC_FLAG_INIT) 로 false으로 초기화해야 한다

```cpp
std::atomic_flag x = ATOMIC_FLAG_INIT;

x.clear(std::memory_order_release);  // 상태를 false 으로 설정
// 연산 의미를 읽을 수 없다：memory_order_consume、memory_order_acquire、memory_order_acq_rel

bool y = x.test_and_set();  // 상태를 true로 설정하고 이전 값을 반환
```

* [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag) 로 스핀락 구현하기  

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

class Spinlock {
 public:
  void lock() {
    while (flag_.test_and_set(std::memory_order_acquire)) {
    }
  }

  void unlock() { flag_.clear(std::memory_order_release); }

 private:
  std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
};

Spinlock m;

void f(int n) {
  for (int i = 0; i < 100; ++i) {
    m.lock();
    std::cout << "Output from thread " << n << '\n';
    m.unlock();
  }
}

int main() {
  std::vector<std::thread> v;
  for (int i = 0; i < 10; ++i) {
    v.emplace_back(f, i);
  }
  for (auto& x : v) {
    x.join();
  }
}
```

### 기타 atomic 타입

* [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag)는 사용하기 쉽고 잠금을 보장하지 않는 `std::atomic<bool>`에 비해 bool로도 사용하기에는 너무 제한적이다. 없는 경우 [is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free)를 사용하여 현재 플랫폼에서 lock-free 여부를 확인할 수 있다.  
  
```cpp
std::atomic<bool> x(true);
x = false;
bool y = x.load(std::memory_order_acquire);  
x.store(true);                               
y = x.exchange(false,
               std::memory_order_acq_rel);  // x는 false로 대체되고 이전 값은 y로 반환
bool expected = false;                      
// 같지 않음은 예상 값을 x로 설정하고 거짓을 반환하고, 같음은 x를 대상 값 참으로 설정하고 참을 반환한다.
// 약한 버전은 같을 때 대체에 실패하고 거짓을 반환할 수도 있으므로 일반적으로 루프에서 사용된다.
while (!x.compare_exchange_weak(expected, true) && !expected) {
}
// 값이 두 개만 있는 std::atomic<bool>의 경우 약간 번거롭다
// 하지만 다른 atomic 타입의 경우 큰 영향을 미친다.
```

* `std::atomic<T*>` 도 지원한다, [is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free)、[load](https://en.cppreference.com/w/cpp/atomic/atomic/load)、[store](https://en.cppreference.com/w/cpp/atomic/atomic/store)、[exchange](https://en.cppreference.com/w/cpp/atomic/atomic/exchange)、[compare_exchange_weak、compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange), `std::atomic<bool>` 읽기 및 반환 유형이 bool 대신 'T*'이고 포인터 원자 유형이 산술 연산도 지원한다는 점을 제외하면 의미는 동일：[fetch_add](https://en.cppreference.com/w/cpp/atomic/atomic/fetch_add)、[fetch_sub](https://en.cppreference.com/w/cpp/atomic/atomic/fetch_sub)、[++、--](https://en.cppreference.com/w/cpp/atomic/atomic/operator_arith)、[+=、-=](https://en.cppreference.com/w/cpp/atomic/atomic/operator_arith2)  

```cpp
class A {};
A a[5];
std::atomic<A*> p(a);   
A* x = p.fetch_add(2);  // p에 대해 &a[2]를 대체하고, 원래 값 a[0]을 반환
assert(x == a);
assert(p.load() == &a[2]);
x = (p -= 1);  // p는 &a[1]이며 x로 반환되며, 이는 x = p.fetch_sub(1) - 1과 동일
assert(x == &a[1]);
assert(p.load() == &a[1]);
```

* 정수 atomic 타입(예 `std::atomic<int>`) 은 위의 연산 외에도 [fetch_or](https://en.cppreference.com/w/cpp/atomic/atomic/fetch_or)、[fetch_and](https://en.cppreference.com/w/cpp/atomic/atomic/fetch_and)、[fetch_xor](https://en.cppreference.com/w/cpp/atomic/atomic/fetch_xor)、[\|=、&=、^=](https://en.cppreference.com/w/cpp/atomic/atomic/operator_arith2) 를 지원한다

```cpp
std::atomic<int> i(5);
int j = i.fetch_and(3);  // 101 & 011 = 001
assert(i == 1);
assert(j == 5);
```

* 정수 atomic 타입으로 Spinlock 구현하기

```cpp
#include <atomic>

class Spinlock {
 public:
  void lock() {
    int expected = 0;
    while (!flag_.compare_exchange_weak(expected, 1, std::memory_order_release,
                                        std::memory_order_relaxed)) {
      expected = 0;
    }
  }

  void unlock() { flag_.store(0, std::memory_order_release); }

 private:
  std::atomic<int> flag_ = 0;
};
```

* 정수 atomic 타입으로 SharedSpinlock 구현하기

```cpp
#include <atomic>

class SharedSpinlock {
 public:
  void lock() {
    int expected = 0;
    while (!flag_.compare_exchange_weak(expected, 1, std::memory_order_release,
                                        std::memory_order_relaxed)) {
      expected = 0;
    }
  }

  void unlock() { flag_.store(0, std::memory_order_release); }

  void lock_shared() {
    int expected = 0;
    while (!flag_.compare_exchange_weak(expected, 2, std::memory_order_release,
                                        std::memory_order_acquire) &&
           expected == 2) {
      expected = 0;
    }
    count_.fetch_add(1, std::memory_order_release);
  }

  void unlock_shared() {
    if (count_.fetch_sub(1, std::memory_order_release) == 1) {
      flag_.store(0, std::memory_order_release);
    }
  }

 private:
  std::atomic<int> flag_ = 0;
  std::atomic<int> count_ = 0;
};
```

* 정수 atomic 타입으로 Barrier 구현하기

```cpp
#include <atomic>
#include <thread>

class Barrier {
 public:
  explicit Barrier(unsigned n) : count_(n), spaces_(n), generation_(0) {}

  void wait() {
    unsigned gen = generation_.load();
    if (--spaces_ == 0) {
      spaces_ = count_.load();
      ++generation_;
      return;
    }
    while (generation_.load() == gen) {
      std::this_thread::yield();
    }
  }

  void arrive() {
    --count_;
    if (--spaces_ == 0) {
      spaces_ = count_.load();
      ++generation_;
    }
  }

 private:
  std::atomic<unsigned> count_;       // 동기화할 스레드 수
  std::atomic<unsigned> spaces_;      // Barrier 에 도달하기 위해 남은 스레드 수
  std::atomic<unsigned> generation_;  // 모든 스레드가 Barrier 에 도달한 총 횟수
};
```

* 원자 유형이 사용자 정의 유형인 경우, 사용자 정의 유형은 [trivially copyable](https://en.cppreference.com/w/cpp/named_req/TriviallyCopyable), 즉 가상 함수나 가상 베이스 클래스를 가질 수 없는 유형이어야 한다. 이는 [is_trivially_copyable](https://en.cppreference.com/w/cpp/types/is_trivially_copyable)로 확인할 수 있다.

```cpp
class A {
 public:
  virtual void f() {}
};

assert(!std::is_trivially_copyable_v<A>);
std::atomic<A> a;                 // 오류：A 는 trivially copyable 
std::atomic<std::vector<int>> v;  // 오류
std::atomic<std::string> s;       // 오류
```

* 自定义类型的原子类型不允许运算操作，只允许 [is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free)、[load](https://en.cppreference.com/w/cpp/atomic/atomic/load)、[store](https://en.cppreference.com/w/cpp/atomic/atomic/store)、[exchange](https://en.cppreference.com/w/cpp/atomic/atomic/exchange)、[compare_exchange_weak、compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)，以及赋值操作和向自定义类型转换的操作
* 除了每个类型各自的成员函数，[原子操作库](https://en.cppreference.com/w/cpp/atomic)还提供了通用的自由函数，只不过函数名多了一个 `atomic_` 前缀，参数变为指针类型

```cpp
std::atomic<int> i(42);
int j = std::atomic_load(&i);  // 等价于 i.load()
```

* 除 [std::atomic_is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic_is_lock_free) 外，每个自由函数有一个 `_explicit` 后缀版本，`_explicit` 自由函数额外接受一个内存序参数

```cpp
std::atomic<int> i(42);
std::atomic_load_explicit(&i, std::memory_order_acquire);  // i.load(std::memory_order_acquire)
```

* 自由函数的设计主要考虑的是 C 语言没有引用而只能使用指针，[compare_exchange_weak、compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange) 的第一个参数是引用，因此 [std::atomic_compare_exchange_weak、std::atomic_compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic_compare_exchange) 的参数用的是指针

```cpp
bool compare_exchange_weak(T& expected, T desired, std::memory_order success,
                           std::memory_order failure);

template <class T>
bool atomic_compare_exchange_weak(std::atomic<T>* obj,
                                  typename std::atomic<T>::value_type* expected,
                                  typename std::atomic<T>::value_type desired);

template <class T>
bool atomic_compare_exchange_weak_explicit(
    std::atomic<T>* obj, typename std::atomic<T>::value_type* expected,
    typename std::atomic<T>::value_type desired, std::memory_order succ,
    std::memory_order fail);
```

* [std::atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag)는 `atomic_`가 아닌 `atomic_flag_`가 접두사로 붙은 자유 함수에 해당하지만 메모리 순서 인자와 `_ 명시적` 접미사가 붙은 버전도 허용한다.    

```cpp
std::atomic_flag x = ATOMIC_FLAG_INIT;
bool y = std::atomic_flag_test_and_set_explicit(&x, std::memory_order_acquire);
std::atomic_flag_clear_explicit(&x, std::memory_order_release);
```

* C++20 은 [std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic) 을 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)의 모듈 참조로 허용한다

```cpp
std::atomic<std::shared_ptr<int>> x;
```

## 同步操作和强制排序（enforced ordering）

* 两个线程分别读写数据，为了避免竞争，设置一个标记

```cpp
std::vector<int> data;
std::atomic<bool> data_ready(false);

void read_thread() {
  while (!data_ready.load()) {  // 1 happens-before 2
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
  }
  std::cout << data[0];  // 2
}

void write_thread() {
  data.emplace_back(42);  // 3 happens-before 4
  data_ready = true;      // 4 inter-thread happens-before 1
}
```

* `std::atomic<bool>` 上的操作要求强制排序，该顺序由内存模型关系 happens-before 和 synchronizes-with 提供
* happens-before 保证了 1 在 2 之前发生，3 在 4 之前发生，而 1 要求 4，所以 4 在 1 之前发生，最终顺序确定为 3412

![](images/4-1.png)

* 如果没有强制排序，CPU 可能会调整指令顺序，如果顺序是 4123，读操作就会因为越界而出错

### synchronizes-with

* synchronizes-with 关系只存在于原子类型操作上，如果一个数据结构包含原子类型，这个数据结构上的操作（比如加锁）也可能提供 synchronizes-with 关系
* 变量 x 上，标记了内存序的原子写操作 W，和标记了内存序的原子读操作，如果两者存在 synchronizes-with 关系，表示读操作读取的是：W 写入的值，或 W 之后同一线程上原子写操作写入 x 的值，或任意线程上对 x 的一系列原子读改写操作（比如 fetch_add、compare_exchange_weak）的值
* 简单来说，如果线程 A 写入一个值，线程 B 读取该值，则 A synchronizes-with B

### [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before)

* [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 和 [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) 关系是程序操作顺序的基本构建块，它指定某个操作可以看到其他操作的结果。对单线程来说很简单，如果一个操作在另一个之前，就可以说前一个操作 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before)（且 [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before)） 后一个操作
* 如果操作发生在同一语句中，一般不存在 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 关系，因为它们是无序的

```cpp
#include <iostream>

void f(int x, int y) { std::cout << x << y; }

int g() {
  static int i = 0;
  return ++i;
}

int main() {
  f(g(), g());  // 无序调用 g，可能是 21 也可能是 12
  // 一般 C++ 默认使用 __cdecl 调用模式，参数从右往左入栈，就是21
}
```

* 前一条语句中的所有操作都 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 下一条语句中的所有操作

### [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before)

* 如果一个线程中的操作 A [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 另一个线程中的操作 B，则 A [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) B
* A [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) B 包括以下情况
  * A synchronizes-with B
  * A [dependency-ordered-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Dependency-ordered_before) B
  * A [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) X，X [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) B
  * A [sequenced-before](https://en.cppreference.com/w/cpp/language/eval_order) X，X [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) B
  * A synchronizes-with X，X [sequenced-before](https://en.cppreference.com/w/cpp/language/eval_order) B
  
### [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before)

* [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) 关系大多数情况下和 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 一样，A [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) B 包括以下情况
  * A synchronizes-with B
  * A [sequenced-before](https://en.cppreference.com/w/cpp/language/eval_order) X，X [inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) B
  * A [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) X，X [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) B
* 略微不同的是，[inter-thread happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Inter-thread_happens-before) 关系可以用 memory_order_consume 标记，而 [strongly-happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Strongly_happens-before) 不行。但大多数代码不应该使用 memory_order_consume，所以这点实际上影响不大

### [std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order)

```cpp
typedef enum memory_order {
  memory_order_relaxed,  // 无同步或顺序限制，只保证当前操作原子性
  memory_order_consume,  // 标记读操作，依赖于该值的读写不能重排到此操作前
  memory_order_acquire,  // 标记读操作，之后的读写不能重排到此操作前
  memory_order_release,  // 标记写操作，之前的读写不能重排到此操作后
  memory_order_acq_rel,  // 仅标记读改写操作，读操作相当于 acquire，写操作相当于 release
  memory_order_seq_cst   // sequential consistency：顺序一致性，不允许重排，所有原子操作的默认选项
} memory_order;
```

### [Relaxed ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering)

* 标记为 memory_order_relaxed 的原子操作不是同步操作，不强制要求并发内存的访问顺序，只保证原子性和修改顺序一致性

```cpp
#include <atomic>
#include <thread>

std::atomic<int> x = 0;
std::atomic<int> y = 0;

void f() {
  int i = y.load(std::memory_order_relaxed);  // 1
  x.store(i, std::memory_order_relaxed);      // 2
}

void g() {
  int j = x.load(std::memory_order_relaxed);  // 3
  y.store(42, std::memory_order_relaxed);     // 4
}

int main() {
  std::thread t1(f);
  std::thread t2(g);
  t1.join();
  t2.join();
  // 可能执行顺序为 4123，结果 i == 42, j == 42
}
```

* [Relaxed ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering) 不允许循环依赖

```cpp
#include <atomic>
#include <thread>

std::atomic<int> x = 0;
std::atomic<int> y = 0;

void f() {
  i = y.load(std::memory_order_relaxed);  // 1
  if (i == 42) {
    x.store(i, std::memory_order_relaxed);  // 2
  }
}

void g() {
  j = x.load(std::memory_order_relaxed);  // 3
  if (j == 42) {
    y.store(42, std::memory_order_relaxed);  // 4
  }
}

int main() {
  std::thread t1(f);
  std::thread t2(g);
  t1.join();
  t2.join();
  // 结果不允许为i == 42, j == 42
  // 因为要产生这个结果，1 依赖 4，4 依赖 3，3 依赖 2，2 依赖 1
}
```

* 典型使用场景是自增计数器，比如 [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) 的引用计数器，它只要求原子性，不要求顺序和同步

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

std::atomic<int> x = 0;

void f() {
  for (int i = 0; i < 1000; ++i) {
    x.fetch_add(1, std::memory_order_relaxed);
  }
}

int main() {
  std::vector<std::thread> v;
  for (int i = 0; i < 10; ++i) {
    v.emplace_back(f);
  }
  for (auto& x : v) {
    x.join();
  }
  std::cout << x;  // 10000
}
```

### [Release-Consume ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Consume_ordering)

* 对于标记为 memory_order_consume 原子变量 x 的读操作 R，当前线程中依赖于 x 的读写不允许重排到 R 之前，其他线程中对依赖于 x 的变量写操作对当前线程可见
* 如果线程 A 对一个原子变量x的写操作为 memory_order_release，线程 B 对同一原子变量的读操作为 memory_order_consume，带来的副作用是，线程 A 中所有 [dependency-ordered-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Dependency-ordered_before) 该写操作的其他写操作（non-atomic和relaxed atomic），在线程 B 的其他依赖于该变量的读操作中可见
* 典型使用场景是访问很少进行写操作的数据结构（比如路由表），以及以指针为中介的 publisher-subscriber 场景，即生产者发布一个指针给消费者访问信息，但生产者写入内存的其他内容不需要对消费者可见，这个场景的一个例子是 RCU（Read-Copy Update）。该顺序的规范正在修订中，并且暂时不鼓励使用 memory_order_consume

```cpp
#include <atomic>
#include <cassert>
#include <thread>

std::atomic<int*> x;
int i;

void producer() {
  int* p = new int(42);
  i = 42;
  x.store(p, std::memory_order_release);
}

void consumer() {
  int* q;
  while (!(q = x.load(std::memory_order_consume))) {
  }
  assert(*q == 42);  // 一定不出错：*q 带有 x 的依赖
  assert(i == 42);   // 可能出错也可能不出错：i 不依赖于 x
}

int main() {
  std::thread t1(producer);
  std::thread t2(consumer);
  t1.join();
  t2.join();
}
```

### [Release-Acquire ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering)

* 对于标记为 memory_order_acquire 的读操作 R，当前线程的其他读写操作不允许重排到 R 之前，其他线程中在同一原子变量上所有的写操作在当前线程可见
* 如果线程 A 对一个原子变量的写操作 W 为 memory_order_release，线程 B 对同一原子变量的读操作为 memory_order_acquire，带来的副作用是，线程 A 中所有 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) W 的写操作（non-atomic 和 relaxed atomic）都在线程 B 中可见
* 典型使用场景是互斥锁，线程 A 的释放后被线程 B 获取，则 A 中释放锁之前发生在 critical section 的所有内容都在 B 中可见

```cpp
#include <atomic>
#include <cassert>
#include <thread>

std::atomic<int*> x;
int i;

void producer() {
  int* p = new int(42);
  i = 42;
  x.store(p, std::memory_order_release);
}

void consumer() {
  int* q;
  while (!(q = x.load(std::memory_order_acquire))) {
  }
  assert(*q == 42);  // 一定不出错
  assert(i == 42);   // 一定不出错
}

int main() {
  std::thread t1(producer);
  std::thread t2(consumer);
  t1.join();
  t2.join();
}
```

* 对于标记为 memory_order_release 的写操作 W，当前线程中的其他读写操作不允许重排到W之后，若其他线程 acquire 该原子变量，则当前线程所有 [happens-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Happens-before) 的写操作在其他线程中可见，若其他线程 consume 该原子变量，则当前线程所有 [dependency-ordered-before](https://en.cppreference.com/w/cpp/atomic/memory_order#Dependency-ordered_before) W 的其他写操作在其他线程中可见
* 对于标记为 memory_order_acq_rel 的读改写（read-modify-write）操作，相当于写操作是 memory_order_release，读操作是 memory_order_acquire，当前线程的读写不允许重排到这个写操作之前或之后，其他线程中 release 该原子变量的写操作在修改前可见，并且此修改对其他 acquire 该原子变量的线程可见
* [Release-Acquire ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering) 并不表示 total ordering

```cpp
#include <atomic>
#include <thread>

std::atomic<bool> x = false;
std::atomic<bool> y = false;
std::atomic<int> z = 0;

void write_x() {
  x.store(true,
          std::memory_order_release);  // 1 happens-before 3（由于 3 的循环）
}

void write_y() {
  y.store(true,
          std::memory_order_release);  // 2 happens-before 5（由于 5 的循环）
}

void read_x_then_y() {
  while (!x.load(std::memory_order_acquire)) {  // 3 happens-before 4
  }
  if (y.load(std::memory_order_acquire)) {  // 4
    ++z;
  }
}

void read_y_then_x() {
  while (!y.load(std::memory_order_acquire)) {  // 5 happens-before 6
  }
  if (x.load(std::memory_order_acquire)) {  // 6
    ++z;
  }
}

int main() {
  std::thread t1(write_x);
  std::thread t2(write_y);
  std::thread t3(read_x_then_y);
  std::thread t4(read_y_then_x);
  t1.join();
  t2.join();
  t3.join();
  t4.join();
  // z 可能为 0，134 y 为 false，256 x 为 false，但 12 之间没有关系
}
```

![](images/4-2.png)

* 为了使两个写操作有序，将其放到一个线程里

```cpp
#include <atomic>
#include <cassert>
#include <thread>

std::atomic<bool> x = false;
std::atomic<bool> y = false;
std::atomic<int> z = 0;

void write_x_then_y() {
  x.store(true, std::memory_order_relaxed);  // 1 happens-before 2
  y.store(true,
          std::memory_order_release);  // 2 happens-before 3（由于 3 的循环）
}

void read_y_then_x() {
  while (!y.load(std::memory_order_acquire)) {  // 3 happens-before 4
  }
  if (x.load(std::memory_order_relaxed)) {  // 4
    ++z;
  }
}

int main() {
  std::thread t1(write_x_then_y);
  std::thread t2(read_y_then_x);
  t1.join();
  t2.join();
  assert(z.load() != 0);  // 顺序一定为 1234，z 一定不为 0
}
```

* 利用 [Release-Acquire ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Release-Acquire_ordering) 可以传递同步

```cpp
#include <atomic>
#include <cassert>

std::atomic<bool> x = false;
std::atomic<bool> y = false;
std::atomic<int> v[2];

void f() {
  // v[0]、v[1] 的设置没有先后顺序，但都 happens-before 1
  v[0].store(1, std::memory_order_relaxed);
  v[1].store(2, std::memory_order_relaxed);
  x.store(true,
          std::memory_order_release);  // 1 happens-before 2（由于 2 的循环）
}

void g() {
  while (!x.load(std::memory_order_acquire)) {  // 2：happens-before 3
  }
  y.store(true,
          std::memory_order_release);  // 3 happens-before 4（由于 4 的循环）
}

void h() {
  while (!y.load(
      std::memory_order_acquire)) {  // 4 happens-before v[0]、v[1] 的读取
  }
  assert(v[0].load(std::memory_order_relaxed) == 1);
  assert(v[1].load(std::memory_order_relaxed) == 2);
}
```

* 使用读改写操作可以将上面的两个标记合并为一个

```cpp
#include <atomic>
#include <cassert>

std::atomic<int> x = 0;
std::atomic<int> v[2];

void f() {
  v[0].store(1, std::memory_order_relaxed);
  v[1].store(2, std::memory_order_relaxed);
  x.store(1, std::memory_order_release);  // 1 happens-before 2（由于 2 的循环）
}

void g() {
  int i = 1;
  while (!x.compare_exchange_strong(
      i, 2,
      std::memory_order_acq_rel)) {  // 2 happens-before 3（由于 3 的循环）
    // x 为 1 时，将 x 替换为 2，返回 true
    // x 为 0 时，将 i 替换为 x，返回 false
    i = 1;  // 返回 false 时，x 未被替换，i 被替换为 0，因此将 i 重新设为 1
  }
}

void h() {
  while (x.load(std::memory_order_acquire) < 2) {  // 3
  }
  assert(v[0].load(std::memory_order_relaxed) == 1);
  assert(v[1].load(std::memory_order_relaxed) == 2);
}
```

### [Sequentially-consistent ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Sequentially-consistent_ordering)

* memory_order_seq_cst 是所有原子操作的默认选项，可以省略不写。对于标记为 memory_order_seq_cst 的操作，读操作相当于 memory_order_acquire，写操作相当于 memory_order_release，读改写操作相当于 memory_order_acq_rel，此外还附加一个单独的 total ordering，即所有线程对同一操作看到的顺序也是相同的。这是最简单直观的顺序，但由于要求全局的线程同步，因此也是开销最大的

```cpp
#include <atomic>
#include <cassert>
#include <thread>

std::atomic<bool> x = false;
std::atomic<bool> y = false;
std::atomic<int> z = 0;

// 要么 1 happens-before 2，要么 2 happens-before 1
void write_x() {
  x.store(true);  // 1 happens-before 3（由于 3 的循环）
}

void write_y() {
  y.store(true);  // 2 happens-before 5（由于 5 的循环）
}

void read_x_then_y() {
  while (!x.load()) {  // 3 happens-before 4
  }
  if (y.load()) {  // 4 为 false 则 1 happens-before 2
    ++z;
  }
}

void read_y_then_x() {
  while (!y.load()) {  // 5 happens-before 6
  }
  if (x.load()) {  // 6 如果返回 false 则一定是 2 happens-before 1
    ++z;
  }
}

int main() {
  std::thread t1(write_x);
  std::thread t2(write_y);
  std::thread t3(read_x_then_y);
  std::thread t4(read_y_then_x);
  t1.join();
  t2.join();
  t3.join();
  t4.join();
  assert(z.load() != 0);  // z 一定不为 0
  // z 可能为 1 或 2，12 之间必定存在 happens-before 关系
}
```

![](images/4-3.png)

### [std::atomic_thread_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)

```cpp
#include <atomic>
#include <cassert>
#include <thread>

std::atomic<bool> x, y;
std::atomic<int> z;

void f() {
  x.store(true, std::memory_order_relaxed);             // 1 happens-before 2
  std::atomic_thread_fence(std::memory_order_release);  // 2 synchronizes-with 3
  y.store(true, std::memory_order_relaxed);
}

void g() {
  while (!y.load(std::memory_order_relaxed)) {
  }
  std::atomic_thread_fence(std::memory_order_acquire);  // 3 happens-before 4
  if (x.load(std::memory_order_relaxed)) {              // 4
    ++z;
  }
}

int main() {
  x = false;
  y = false;
  z = 0;
  std::thread t1(f);
  std::thread t2(g);
  t1.join();
  t2.join();
  assert(z.load() != 0);  // 1 happens-before 4
}
```

* x를 atomic 이 아닌 bool 타입으로 대체하면 동일한 동작이 발생한다.   

```cpp
#include <atomic>
#include <cassert>
#include <thread>

bool x = false;
std::atomic<bool> y;
std::atomic<int> z;

void f() {
  x = true;                                             // 1 happens-before 2
  std::atomic_thread_fence(std::memory_order_release);  // 2 synchronizes-with 3
  y.store(true, std::memory_order_relaxed);
}

void g() {
  while (!y.load(std::memory_order_relaxed)) {
  }
  std::atomic_thread_fence(std::memory_order_acquire);  // 3 happens-before 4
  if (x) {                                              // 4
    ++z;
  }
}

int main() {
  x = false;
  y = false;
  z = 0;
  std::thread t1(f);
  std::thread t2(g);
  t1.join();
  t2.join();
  assert(z.load() != 0);  // 1 happens-before 4
}
```
