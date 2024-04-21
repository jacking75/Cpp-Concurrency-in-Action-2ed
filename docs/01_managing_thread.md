## [std::thread](https://en.cppreference.com/w/cpp/thread/thread)

* 각 응용 프로그램에는 main() 함수를 실행하는 메인 스레드가 있다. 다른 스레드를 시작하려면 [std::thread] (https://en.cppreference.com/w/cpp/thread/thread) 에 함수를 인수로 추가하면 두 스레드가 동시에 실행된다.  
  
```cpp
#include <iostream>
#include <thread>

void f() { std::cout << "hello world"; }

int main() {
  std::thread t{f};
  t.join();  // 等待新起的线程退出
}
```

* [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 는 함수 객체 또는 lambda

```cpp
#include <iostream>
#include <thread>

struct A {
  void operator()() const { std::cout << 1; }
};

int main() {
  A a;
  std::thread t1(a);  // A에 대한 복사 생성자를 호출한다
  std::thread t2(A());  // 가장 까다로운 구문 분석으로 A 타입의 매개 변수를 가진 t2를 선언한다
  std::thread t3{A()};
  std::thread t4((A()));
  std::thread t5{[] { std::cout << 1; }};
  t1.join();
  t3.join();
  t4.join();
  t5.join();
}
```
  
* [join](https://en.cppreference.com/w/cpp/thread/thread/join )을 호출하여 스레드가 종료될 때까지 기다리거나 [detach](https://en.cppreference.com/w/cpp/thread/ )를 호출하여 스레드를 분리한 후 소멸시킨다. 그렇지 않으면 [std::thread](https://en.cppreference.com/w/cpp/thread/thread )의 소멸자가 [std::terminate](https://en.cppreference.com/w/cpp/error/terminate )를 호출하여 스레드를 종료한다.  분리된 스레드는 널 참조를 가질 수 있다는 점에 유의한다.  
  
```cpp
#include <iostream>
#include <thread>

class A {
 public:
  A(int& x) : x_(x) {}

  void operator()() const {
    for (int i = 0; i < 1000000; ++i) {
      call(x_);  // 오브젝트 디스트럭처링 후 참조 무효화의 숨겨진 위험이 있습니다
    }
  }

 private:
  void call(int& x) {}

 private:
  int& x_;
};

void f() {
  int x = 0;
  A a{x};
  std::thread t{a};
  t.detach();  // t가 끝날 때까지 기다리지 않는다
}  // 함수가 완료되고 x가 소멸되었을 때 t가 여전히 실행 중일 수 있으며, a.x_는 널 댕글링 참조한다

int main() {
  std::thread t{f};  // 원인 댕글링 참조
  t.join();
}
```
  
* [join](https://en.cppreference.com/w/cpp/thread/thread/join )은 [std::thread](https://en.cppreference.com/w/cpp/thread/thread )를 정리하여 완료된 스레드와 더 이상 연결되지 않도록 하므로 [join](https://en.cppreference.com/w/cpp/thread/thread/join )은 스레드에 대해 한 번만 수행할 수 있다.  
    
```cpp
#include <thread>

int main() {
  std::thread t([] {});
  t.join();
  t.join();  // 오류
}
```
  
* 스레드가 실행되는 동안 예외가 발생하면 후속 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 은 무시되므로 예외를 잡아서 [join](https://en.cppreference.com/w/cpp/thread/thread/join)

```cpp
#include <thread>

int main() {
  std::thread t([] {});
  try {
    throw 0;
  } catch (int x) {
    t.join();  
    throw x;   
  }
  t.join();  // 이전에 예외를 던진 경우 여기서는 예외가 실행되지 않는다
}
```  
  
* C++20 지원 [std::jthread](https://en.cppreference.com/w/cpp/thread/jthread)，소멸자 함수에 스레드를 [join](https://en.cppreference.com/w/cpp/thread/thread/join) 한다  
  
```cpp
#include <thread>

int main() {
  std::jthread t([] {});
}
```

* [detach](https://en.cppreference.com/w/cpp/thread/thread/detach) 스레드를 분리하면 백그라운드에서 실행 중인 스레드가 남게 되며, 백그라운드에서 실행 중인 스레드를 일반적으로 데몬 스레드라고 하며, 이는 메인 스레드와 직접 상호 작용할 수 없다. [join](https://en.cppreference.com/w/cpp/thread/thread/join) 할 수 없다.  
  
```cpp
std::thread t([] {});
t.detach();
assert(!t.joinable());
```
   
* 예를 들어 문서 처리 애플리케이션이 있는 경우 여러 문서를 동시에 편집하기 위해 새 문서를 열 때마다 해당 데몬 스레드를 열 수 있는 등 데몬 스레드 생성은 일반적으로 오랜 기간 동안 수행된다.  
  
```cpp
void edit_document(const std::string& filename) {
  open_document_and_display_gui(filename);

  while (!done_editing()) {
    user_command cmd = get_user_input();
    
    if (cmd.type == open_new_document) {
      const std::string new_name = get_filename_from_user();
      std::thread t(edit_document, new_name);
      t.detach();
    } else {
      process_user_input(cmd);
    }
  }
}
```  
  
## 인자가 있는 함수에 대한 스레드 만들기

* 인자가 있는 함수는 [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 로 전달할 수도 있으며, 인자의 기본 실수 매개 변수는 무시된다.  
  
```cpp
#include <thread>

void f(int i = 1) {}

int main() {
  std::thread t{f, 42};  // std::thread t{f} 는 기본 실제 매개 변수가 무시되므로 오류가 발생한다. f 함수에서 파라미터 i를 사용하지 않고 있다
  t.join();
}
```

* 매개 변수의 참조 유형도 무시되며, 이를 위해서는 [std::ref](https://en.cppreference.com/w/cpp/utility/functional/ref)

```cpp
#include <cassert>
#include <thread>

void f(int& i) { ++i; }

int main() {
  int i = 1;
  std::thread t{f, std::ref(i)};
  t.join();
  assert(i == 2);
}
```

* 인스턴스의 비정적 멤버 함수에 대해 스레드가 생성되는 경우, 첫 번째 인자는 멤버 함수 포인터 유형이고, 두 번째 인자는 인스턴스 포인터 유형이며, 후속 인자는 함수 매개 변수입니다.  
  
```cpp
#include <iostream>
#include <thread>

class A {
 public:
  void f(int i) { std::cout << i; }
};

int main() {
  A a;
  std::thread t1{&A::f, &a, 42};  // 호출 a->f(42)
  std::thread t2{&A::f, a, 42};   // 복사 구조체 tmp_a 를 만든 다음 tmp_a.f(42) 를 호출한다
  t1.join();
  t2.join();
}
```
  
* 인자 유형이 move-only 인 함수에 대한 스레드를 만들려면 [std::move](https://en.cppreference.com/w/cpp/utility/move )를 사용하여 인자를 전달해야 한다.  
 
```cpp
#include <iostream>
#include <thread>
#include <utility>

void f(std::unique_ptr<int> p) { std::cout << *p; }

int main() {
  std::unique_ptr<int> p(new int(42));
  std::thread t{f, std::move(p)};
  t.join();
}
```
  
## 스레드 소유권 이전
* [std::thread](https://en.cppreference.com/w/cpp/thread/thread )는 이동 전용 유형으로 복사할 수 없으며, 이동을 통해서만 소유권을 이전할 수 있고 조인 가능한 스레드에는 소유권을 이전할 수 없다.   
  
```cpp
#include <thread>
#include <utility>

void f() {}
void g() {}

int main() {
  std::thread a{f};
  std::thread b = std::move(a);
  assert(!a.joinable());
  assert(b.joinable());
  a = std::thread{g};
  assert(a.joinable());
  assert(b.joinable());
  // a = std::move(b);  // 오류, joinable한 스레드로 소유권을 이전할 수 없다.
  a.join();
  a = std::move(b);
  assert(a.joinable());
  assert(!b.joinable());
  a.join();
}
```

* 이동을 지원하는 컨테이너에도 이동 작업이 적용된다  
  
```cpp
#include <algorithm>
#include <thread>
#include <vector>

int main() {
  std::vector<std::thread> v;
  for (int i = 0; i < 10; ++i) {
    v.emplace_back([] {});
  }
  std::for_each(std::begin(v), std::end(v), std::mem_fn(&std::thread::join));
}
```

* [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 함수 반환값으로 사용 가능

```cpp
#include <thread>

std::thread f() {
  return std::thread{[] {}};
}

int main() {
  std::thread t{f()};
  t.join();
}
```

* [std::thread](https://en.cppreference.com/w/cpp/thread/thread) 를 함수 매개변수로 사용할 수도 있다

```cpp
#include <thread>
#include <utility>

void f(std::thread t) { t.join(); }

int main() {
  f(std::thread([] {}));
  std::thread t([] {});
  f(std::move(t));
}
```  
  
* [std::thread](https://en.cppreference.com/w/cpp/thread/thread ) 로 직접 생성할 수 있는 스레드를 자동으로 정리하는 클래스를 구현한다  
  
```cpp
#include <stdexcept>
#include <thread>
#include <utility>

class scoped_thread {
 public:
  explicit scoped_thread(std::thread x) : t_(std::move(x)) {
    if (!t_.joinable()) {
      throw std::logic_error("no thread");
    }
  }
  ~scoped_thread() { t_.join(); }
  scoped_thread(const scoped_thread&) = delete;
  scoped_thread& operator=(const scoped_thread&) = delete;

 private:
  std::thread t_;
};

int main() {
  scoped_thread t{std::thread{[] {}}};
}
```

* [std::jthread](https://en.cppreference.com/w/cpp/thread/jthread) 와 비슷한 클래스  

```cpp
#include <thread>

class Jthread {
 public:
  Jthread() noexcept = default;

  template <typename T, typename... Ts>
  explicit Jthread(T&& f, Ts&&... args)
      : t_(std::forward<T>(f), std::forward<Ts>(args)...) {}

  explicit Jthread(std::thread x) noexcept : t_(std::move(x)) {}
  Jthread(Jthread&& rhs) noexcept : t_(std::move(rhs.t_)) {}

  Jthread& operator=(Jthread&& rhs) noexcept {
    if (joinable()) {
      join();
    }
    t_ = std::move(rhs.t_);
    return *this;
  }

  Jthread& operator=(std::thread t) noexcept {
    if (joinable()) {
      join();
    }
    t_ = std::move(t);
    return *this;
  }

  ~Jthread() noexcept {
    if (joinable()) {
      join();
    }
  }

  void swap(Jthread&& rhs) noexcept { t_.swap(rhs.t_); }
  std::thread::id get_id() const noexcept { return t_.get_id(); }
  bool joinable() const noexcept { return t_.joinable(); }
  void join() { t_.join(); }
  void detach() { t_.detach(); }
  std::thread& as_thread() noexcept { return t_; }
  const std::thread& as_thread() const noexcept { return t_; }

 private:
  std::thread t_;
};

int main() {
  Jthread t{[] {}};
}
```

## 하드웨어에서 지원하는 스레드 수 보기

* [hardware_concurrency](https://en.cppreference.com/w/cpp/thread/thread/hardware_concurrency) 하드웨어가 지원하는 동시 스레드 수를 반환한다  

```cpp
#include <iostream>
#include <thread>

int main() {
  unsigned int n = std::thread::hardware_concurrency();
  std::cout << n << " concurrent threads are supported.\n";
}
```

* 병렬 버전 [std::accumulate](https://en.cppreference.com/w/cpp/algorithm/accumulate)

```cpp
#include <algorithm>
#include <cassert>
#include <functional>
#include <iterator>
#include <numeric>
#include <thread>
#include <vector>

template <typename Iterator, typename T>
struct accumulate_block {
  void operator()(Iterator first, Iterator last, T& res) {
    res = std::accumulate(first, last, res);
  }
};

template <typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init) {
  long len = std::distance(first, last);
  if (!len) {
    return init;
  }
  long min_per_thread = 25;
  long max_threads = (len + min_per_thread - 1) / min_per_thread;
  long hardware_threads = std::thread::hardware_concurrency();
  long num_threads =  // 스레드 수
      std::min(hardware_threads == 0 ? 2 : hardware_threads, max_threads);
  long block_size = len / num_threads;  // 각 스레드의 데이터 양
  std::vector<T> res(num_threads);
  std::vector<std::thread> threads(num_threads - 1);  // 이미 메인 스레드가 있으므로 
  Iterator block_start = first;
  for (long i = 0; i < num_threads - 1; ++i) {
    Iterator block_end = block_start;
    std::advance(block_end, block_size);  // block_end 현재 블록의 끝을 가리킨다
    threads[i] = std::thread{accumulate_block<Iterator, T>{}, block_start,
                             block_end, std::ref(res[i])};
    block_start = block_end;
  }
  accumulate_block<Iterator, T>()(block_start, last, res[num_threads - 1]);
  std::for_each(threads.begin(), threads.end(),
                std::mem_fn(&std::thread::join));
  return std::accumulate(res.begin(), res.end(), init);
}

int main() {
  std::vector<long> v(1000000);
  std::iota(std::begin(v), std::end(v), 0);
  long res = std::accumulate(std::begin(v), std::end(v), 0);
  assert(parallel_accumulate(std::begin(v), std::end(v), 0) == res);
}
```
  
## 스레드 번호
* 스레드 인스턴스에서 멤버 함수 [get_id](https://en.cppreference.com/w/cpp/thread/thread/get_id ) 또는 현재 스레드에서 [std::this_thread::get_id](https://en.cppreference.com/w/cpp/thread/get_id )를 호출하여 얻을 수 있다. [thread_number](https://en.cppreference.com/w/cpp/thread/thread/id )를 얻을 수 있으며, 이는 본질적으로 복사 및 비교를 허용하는 부호 없는 정수를 감싸는 래퍼이므로 컨테이너로 사용할 수 있다. 두 스레드의 스레드 번호가 같으면 동일한 스레드이거나 둘 다 빈 스레드이다(일반적으로 빈 스레드의 스레드 번호는 0 이다).  
  
```cpp
#include <iostream>
#include <thread>

#ifdef _WIN32
#include <Windows.h>
#elif defined __GNUC__
#include <syscall.h>
#include <unistd.h>

#endif

void print_current_thread_id() {
#ifdef _WIN32
  std::cout << std::this_thread::get_id() << std::endl;       // 19840
  std::cout << GetCurrentThreadId() << std::endl;             // 19840
  std::cout << GetThreadId(GetCurrentThread()) << std::endl;  // 19840
#elif defined __GNUC__
  std::cout << std::this_thread::get_id() << std::endl;  // 1
  std::cout << pthread_self() << std::endl;              // 140250646003520
  std::cout << getpid() << std::endl;  // 1502109，ps aux 이 표시 pid
  std::cout << syscall(SYS_gettid) << std::endl;  // 1502109
#endif
}

std::thread::id master_thread_id = std::this_thread::get_id();

void f() {
  if (std::this_thread::get_id() == master_thread_id) {
    // do_master_thread_work();
  }
  // do_common_work();
}

int main() {
  print_current_thread_id();
  f();
  std::thread t{f};
  t.join();
}
```

## CPU 친화성（affinity）

* 스레드를 특정 CPU 코어에 바인딩하여 멀티코어 CPU 컨텍스트 스위치 및 캐시 누락으로 인한 오버헤드를 방지한다  
  
```cpp
#ifdef _WIN32
#include <Windows.h>
#elif defined __GNUC__
#include <pthread.h>
#include <sched.h>
#include <string.h>
#endif

#include <iostream>
#include <thread>

void affinity_cpu(std::thread::native_handle_type t, int cpu_id) {
#ifdef _WIN32
  if (!SetThreadAffinityMask(t, 1ll << cpu_id)) {
    std::cerr << "fail to affinity" << GetLastError() << std::endl;
  }
#elif defined __GNUC__
  cpu_set_t cpu_set;
  CPU_ZERO(&cpu_set);
  CPU_SET(cpu_id, &cpu_set);
  int res = pthread_setaffinity_np(t, sizeof(cpu_set), &cpu_set);
  if (res != 0) {
    errno = res;
    std::cerr << "fail to affinity" << strerror(errno) << std::endl;
  }
#endif
}

void affinity_cpu_on_current_thread(int cpu_id) {
#ifdef _WIN32
  if (!SetThreadAffinityMask(GetCurrentThread(), 1ll << cpu_id)) {
    std::cerr << "fail to affinity" << GetLastError() << std::endl;
  }
#elif defined __GNUC__
  cpu_set_t cpu_set;
  CPU_ZERO(&cpu_set);
  CPU_SET(cpu_id, &cpu_set);
  int res = pthread_setaffinity_np(pthread_self(), sizeof(cpu_set), &cpu_set);
  if (res != 0) {
    errno = res;
    std::cerr << "fail to affinity" << strerror(errno) << std::endl;
  }
#endif
}

void f() { affinity_cpu_on_current_thread(0); }

int main() {
  std::thread t1{[] {}};
  affinity_cpu(t1.native_handle(), 1);
  std::thread t2{f};
  t1.join();
  t2.join();
}
```
