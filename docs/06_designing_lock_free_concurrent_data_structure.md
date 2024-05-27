## 비블록킹 데이터 구조
  
* 블로킹 알고리즘과 데이터 구조는 뮤텍스, 조건 변수, 기간 값을 사용하여 데이터를 동기화하지만 비 블로킹은 락 프리와 동일하지 않다(예: 스핀 락은 블로킹 함수 호출을 사용하지 않으며 비 블로킹이지만 락 프리는 아니다).
* 비차단 데이터 구조는 느슨한 것부터 엄격한 것까지 세 가지 클래스로 분류할 수 있다: obstruction-free, lock-free, wait-free
  * obstruction-free: 다른 모든 스레드가 일시 중단된 경우 특정 스레드가 유한한 수의 단계로 연산을 완료한다. 위의 예시에서는 이런 경우가 발생하지만 드물기 때문에 이 조건을 충족하면 잠금 없는 구현에 실패한 것으로 간주된다.
  * lock-free: 여러 스레드가 동일한 데이터 구조에서 작업하는 경우, 그 중 하나가 유한한 수의 단계로 작업을 완료한다. 잠금 없는 상태를 만족하려면 obstruction-free 상태를 만족해야 한다.
  * wait-free: 여러 스레드가 동일한 데이터 구조에서 작업하는 경우 각 스레드가 유한한 수의 단계로 작업을 완료한다. wait-free을 만족하려면 lock-free을 만족해야 하지만, 한정된 단계 수를 보장하기 위해서는 연산을 한 번만 통과해야 하고, 한 단계의 실행으로 인해 다른 스레드가 연산에 실패하지 않아야 하기 때문에 wait-free을 구현하기는 어렵다.
* lock-free 데이터 구조는 여러 스레드의 동시 액세스를 허용해야 하지만 동일한 작업을 수행해서는 안 된다(예: lock-free 큐에서는 한 스레드는 push 하고 다른 스레드는 pop 할 수 있지만 두 스레드가 동시에 push 해서는 안 된다). 또한 lock-free 데이터 구조에 액세스하는 스레드가 중단되면 다른 스레드는 중단된 스레드를 기다리지 않고 작업을 완료할 수 있어야 한다.
* lock-free 데이터 구조를 사용하는 가장 큰 이유는 차단 없이 동시 액세스를 최대화하기 위해서이다. 두 번째 이유는 견고성이다. 잠금을 유지하는 동안 스레드가 죽으면 데이터 구조가 영구적으로 손상되는 반면, lock-free 데이터 구조에서는 죽은 스레드의 데이터를 제외하고는 데이터가 손실되지 않는다. lock-free 데이터 구조에는 잠금이 없으므로 데드락이 없어야 한다.  
* lock-free에는 락이 없으므로 교착 상태가 없어야 한다. 그러나 lock-free는 더 많은 오버헤드를 유발할 수 있으며, lock-free에 사용되는 원자 연산은 비원자 연산보다 훨씬 느리고, 일반적으로 락 기반 데이터 구조보다 lock-free 데이터 구조에 더 많은 원자 연산이 있으며 스레드 간에 데이터를 동기화하기 위해 하드웨어가 동일한 원자 변수에 액세스할 수 있어야 한다. lock-free든 락 기반이든 성능 검사(최악의 대기 시간, 평균 대기 시간, 전체 실행 시간 등)는 매우 중요하다!
  
  
## lock-free thread-safe stack
  
* 스택의 가장 간단한 구현은 헤드 노드에 대한 포인터의 연쇄 목록이다. push는 새 노드를 생성한 다음 새 노드의 다음 포인터가 현재 헤드를 가리키고 마지막으로 헤드가 새 노드로 설정되는 간단한 프로세스이다.
* 여기서 경쟁 조건은 두 스레드가 동시에 push 하고 새 노드의 다음 포인터가 현재 헤드를 가리키면 필연적으로 헤드가 새 노드 중 하나로 설정되고 다른 하나는 버려진다는 것이다.
* 해결책은 마지막에 헤드를 설정할 때 판단을 내리고, 현재 헤드가 새 노드의 다음 헤드와 같을 경우에만 헤드를 새 노드로 설정하고, 그렇지 않으면 다음 포인터가 현재 헤드를 가리키게 하고 새로 판단하는 것이다. 이 연산은 원자적 연산이어야 하므로 [compare_exchange_weak](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)이 아닌 [compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)을 사용해야 한다. [exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)이 아니라 [compare_exchange_weak](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)는 같을 때 대체에 실패할 수 있지만 대체 실패도 거짓을 반환하므로 루프에 배치하면 동일한 효과를 가져오는 반면, [compare_exchange_weak](https://en. .cppreference.com/w/cpp/atomic/atomic/compare_exchange)는 일부 머신 아키텍처에서 [compare_exchange_strong](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)보다 최적화된 코드를 제공한다.   
    
```cpp
#include <atomic>

template <typename T>
class LockFreeStack {
 public:
  void push(const T& x) {
    Node* t = new Node(x);
    t->next = head_.load();
    while (!head_.compare_exchange_weak(t->next, t)) {
    }
  }

 private:
  struct Node {
    T v;
    Node* next = nullptr;
    Node(const T& x) : v(x) {}
  };

 private:
  std::atomic<Node*> head_;
};
```
  
* pop 프로세스는 간단하다. 먼저 현재 헤더 노드에 대한 포인터를 저장한 다음 헤더 노드를 다음 노드로 설정하고 마지막으로 저장된 헤더 노드로 돌아가서 포인터를 삭제한다. 여기서 경합 조건은 두 개의 스레드가 동시에 pop-up 되는 경우 한 스레드가 이미 헤더 노드를 삭제한 경우 헤더 노드의 다음 노드를 읽는 다른 스레드가 빈 포인터에 액세스한다는 것이다.  
* 포인터 삭제 단계를 건너뛰고 이전 단계의 구현을 고려하자.
  
```cpp
template <typename T>
void LockFreeStack<T>::pop(T& res) {
  Node* t = head_.load();  
  while (!head_.compare_exchange_weak(t, t->next)) {
  }
  res = t->v;
}
```
  
* 참조를 전달하여 결과를 저장하는 이유는 값을 직접 반환하는 경우 반환하기 전에 요소를 제거해야 하며, 반환된 값을 복사할 때 예외가 발생하면 제거된 요소가 손실되기 때문이다. 그러나 참조 전달의 문제점은 다른 스레드가 노드를 제거하면 제거된 노드를 역참조할 수 없고 현재 스레드가 데이터를 안전하게 복사할 수 없다는 것이다. 따라서 값을 안전하게 반환하려면 스마트 포인터를 반환해야 한다.  
  
```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  void push(const T& x) {
    Node* t = new Node(x);
    t->next = head_.load();
    while (!head_.compare_exchange_weak(t->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {  
    Node* t = head_.load();
    while (t && !head_.compare_exchange_weak(t, t->next)) {
    }
    return t ? t->v : nullptr;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    Node* next = nullptr;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

 private:
  std::atomic<Node*> head_;
};
```

* 제거된 노드를 해제할 때 어려운 점은 스레드가 메모리를 해제할 때 다른 스레드가 해제할 포인터를 보유하고 있는지 여부를 알 수 없다는 것이다.
* 다른 스레드가 pop 을 호출하지 않는 한 해제해도 안전하므로 카운터를 사용하여 팝을 호출한 스레드 수를 추적하고 그 수가 1이 아닌 경우 삭제할 노드 목록에 노드를 추가하고 그 수가 1이면 안전하게 해제할 수 있다.
  
```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  void push(const T& x) {
    Node* t = new Node(x);
    t->next = head_.load();
    while (!head_.compare_exchange_weak(t->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {
    ++pop_cnt_;
    Node* t = head_.load();
    while (t && !head_.compare_exchange_weak(t, t->next)) {
    }
    std::shared_ptr<T> res;
    if (t) {
      res.swap(t->v);
    }
    try_delete(t);
    return res;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    Node* next = nullptr;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

 private:
  static void delete_list(Node* head) {
    while (head) {
      Node* t = head->next;
      delete head;
      head = t;
    }
  }

  void append_to_delete_list(Node* first, Node* last) {
    last->next = to_delete_list_;
    // last->next가 to_delete_list_인지 확인한 다음 첫 번째를 새 헤드 노드로 설정한다.
    while (!to_delete_list_.compare_exchange_weak(last->next, first)) {
    }
  }

  void append_to_delete_list(Node* head) {
    Node* last = head;
    while (Node* t = last->next) {
      last = t;
    }
    append_to_delete_list(head, last);
  }

  void try_delete(Node* head) {
    if (pop_cnt_ == 0) {
      return;
    }
    if (pop_cnt_ > 1) {
      append_to_delete_list(head, head);
      --pop_cnt_;
      return;
    }
    Node* t = to_delete_list_.exchange(nullptr);
    if (--pop_cnt_ == 0) {
      delete_list(t);
    } else if (t) {
      append_to_delete_list(t);
    }
    delete head;
  }

 private:
  std::atomic<Node*> head_;
  std::atomic<std::size_t> pop_cnt_;
  std::atomic<Node*> to_delete_list_;
};
```
  
* 모든 노드를 해제하려면 카운터가 0이 되는 순간이 있어야 한다. 부하가 높으면 이러한 순간이 존재하지 않는 경우가 많기 때문에 삭제할 노드 목록이 무한정 늘어난다.

### Hazard Pointer
  
* 해제에 대한 또 다른 아이디어는 스레드가 노드에 액세스할 때 스레드 ID와 노드를 보유하는 Hazard Pointer를 설정하는 것이다. 글로벌 배열을 사용하여 모든 스레드의 Hazard Pointer를 보관하고, 노드를 해제할 때 해당 노드가 포함된 배열에 Hazard Pointer가 없으면 바로 해제할 수 있고, 그렇지 않으면 삭제할 목록에 노드를 추가한다. Hazard Pointer는 다음과 같이 구현된다.
  
```cpp
#include <atomic>
#include <stdexcept>
#include <thread>

static constexpr std::size_t MaxSize = 100;

struct HazardPointer {
  std::atomic<std::thread::id> id;
  std::atomic<void*> p;
};

static HazardPointer HazardPointers[MaxSize];

class HazardPointerHelper {
 public:
  HazardPointerHelper() {
    for (auto& x : HazardPointers) {
      std::thread::id default_id;
      if (x.id.compare_exchange_strong(default_id,
                                       std::this_thread::get_id())) {
        hazard_pointer = &x;  // 설정 되지 않은 위험에 대한 포인터를 가져온다
        break;
      }
    }
    if (!hazard_pointer) {
      throw std::runtime_error("No hazard pointers available");
    }
  }

  ~HazardPointerHelper() {
    hazard_pointer->p.store(nullptr);
    hazard_pointer->id.store(std::thread::id{});
  }

  HazardPointerHelper(const HazardPointerHelper&) = delete;

  HazardPointerHelper operator=(const HazardPointerHelper&) = delete;

  std::atomic<void*>& get() { return hazard_pointer->p; }

 private:
  HazardPointer* hazard_pointer = nullptr;
};

std::atomic<void*>& hazard_pointer_for_this_thread() {
  static thread_local HazardPointerHelper t;
  return t.get();
}

bool is_existing(void* p) {
  for (auto& x : HazardPointers) {
    if (x.p.load() == p) {
      return true;
    }
  }
  return false;
}
```

* Hazard Pointer 사용

```cpp
#include <atomic>
#include <functional>
#include <memory>

#include "hazard_pointer.hpp"

template <typename T>
class LockFreeStack {
 public:
  void push(const T& x) {
    Node* t = new Node(x);
    t->next = head_.load();
    while (!head_.compare_exchange_weak(t->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {
    std::atomic<void*>& hazard_pointer = hazard_pointer_for_this_thread();
    Node* t = head_.load();
    do {  // 외부 루프는 t가 최신 헤드 노드가 되도록 하고, 루프가 끝날 때 헤드 노드를 다음 노드로 설정한다
      Node* t2;
      do {  // Hazard Pointer로 루프하여 현재 최신 헤더 노드를 저장한다
        t2 = t;
        hazard_pointer.store(t);
        t = head_.load();
      } while (t != t2);
    } while (t && !head_.compare_exchange_strong(t, t->next));
    hazard_pointer.store(nullptr);
    std::shared_ptr<T> res;
    if (t) {
      res.swap(t->v);
      if (is_existing(t)) {
        append_to_delete_list(new DataToDelete{t});
      } else {
        delete t;
      }
      try_delete();
    }
    return res;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    Node* next = nullptr;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

  struct DataToDelete {
    template <typename T>
    DataToDelete(T* p)
        : data(p), deleter([](void* p) { delete static_cast<T*>(p); }) {}

    ~DataToDelete() { deleter(data); }

    void* data = nullptr;
    std::function<void(void*)> deleter;
    DataToDelete* next = nullptr;
  };

 private:
  void append_to_delete_list(DataToDelete* t) {
    t->next = to_delete_list_.load();
    while (!to_delete_list_.compare_exchange_weak(t->next, t)) {
    }
  }

  void try_delete() {
    DataToDelete* cur = to_delete_list_.exchange(nullptr);
    while (cur) {
      DataToDelete* t = cur->next;
      if (!is_existing(cur->data)) {
        delete cur;
      } else {
        append_to_delete_list(new DataToDelete{cur});
      }
      cur = t;
    }
  }

 private:
  std::atomic<Node*> head_;
  std::atomic<std::size_t> pop_cnt_;
  std::atomic<DataToDelete*> to_delete_list_;
};
```  
  
* 리스크 포인터는 구현이 간단하고 안전한 해제라는 목표를 달성하지만, 배열을 탐색하고 각 노드 삭제 전후에 내부 포인터에 원자적으로 액세스하기 때문에 많은 오버헤드가 추가된다.
* lock-free 메모리 매립 기술 분야는 대기업에서 자체 특허를 출원하는 등 매우 활발하게 연구되고 있으며 IBM이 출원한 특허 출원에는 Hazard Pointer가 포함되어 있으며 GPL에 따라 무료로 사용할 수 있다.

### 참조 수

* 또 다른 옵션은 참조 계수를 사용하여 각 노드에 액세스하는 스레드 수를 추적하는 것이다. [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr)의 연산은 원자적이지만 잠금이 없는지 확인한다.  
  
```cpp
std::shared_ptr<int> p(new int(42));
assert(std::atomic_is_lock_free(&p));
```

* lock-free stack

```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  ~LockFreeStack() {
    while (pop()) {
    }
  }

  void push(const T& x) {
    auto t = std::make_shared<Node>(x);
    t->next = std::atomic_load(&head_);
    while (!std::atomic_compare_exchange_weak(&head_, &t->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {
    std::shared_ptr<Node> t = std::atomic_load(&head_);
    while (t && !std::atomic_compare_exchange_weak(&head_, &t, t->next)) {
    }
    if (t) {
      std::atomic_store(&t->next, nullptr);
      return t->v;
    }
    return nullptr;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    std::shared_ptr<Node> next;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

 private:
  std::shared_ptr<Node> head_;
};
```

* C++20 기능 [std::atomic\<std::shared_ptr\>](https://en.cppreference.com/w/cpp/memory/shared_ptr/atomic2)

```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  ~LockFreeStack() {
    while (pop()) {
    }
  }

  void push(const T& x) {
    auto t = std::make_shared<Node>(x);
    t->next = head_.load();
    while (!head_.compare_exchange_weak(t->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {
    std::shared_ptr<Node> t = head_.load();
    while (t && !head_.compare_exchange_weak(t, t->next.load())) {
    }
    if (t) {
      t->next = std::shared_ptr<Node>();
      return t->v;
    }
    return nullptr;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    std::atomic<std::shared_ptr<Node>> next;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

 private:
  std::atomic<std::shared_ptr<Node>> head_;
};
```

* 단 VS2022 버전에서는 [std::atomic\<std::shared_ptr\>](https://en.cppreference.com/w/cpp/memory/shared_ptr/atomic2) 비 lock-free 이다(VS2022 버전 업데이트에 따라 달라질 수도 있음)  
  
```cpp
assert(!std::atomic<std::shared_ptr<int>>{}.is_lock_free());
```
  
* 보다 일반적인 접근 방식은 각 노드에 대해 내부와 외부의 참조 카운트를 두 개씩 설정하고 그 합이 노드의 참조 카운트로서 기본적으로 외부 카운트는 1로 설정되며 오브젝트에 액세스하면 외부 카운트는 증가하고 내부 카운트는 감소하며 액세스가 끝나면 외부 카운트는 더 이상 필요하지 않고 외부 카운트에서 빼고 내부 카운트에 추가하여 수동으로 참조 카운트를 관리하는 방식이다.  
  
```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  ~LockFreeStack() {
    while (pop()) {
    }
  }

  void push(const T& x) {
    ReferenceCount t;
    t.p = new Node(x);
    t.external_cnt = 1;
    t.p->next = head_.load();
    while (!head_.compare_exchange_weak(t.p->next, t)) {
    }
  }

  std::shared_ptr<T> pop() {
    ReferenceCount t = head_.load();
    while (true) {
      increase_count(t);  // 외부 카운트가 증가하면 노드가 사용 중임을 나타낸다
      Node* p = t.p;      
      if (!p) {
        return nullptr;
      }
      if (head_.compare_exchange_strong(t, p->next)) {
        std::shared_ptr<T> res;
        res.swap(p->v);
        // 노드가 삭제되었으므로 외부 카운트에 내부 카운트에서 2를 뺀 2를 더하고, 
        // 노드가 삭제되었으므로 스레드가 이 노드에 다시 액세스할 수 없으므로 1을 뺀 1을 더한다
        const int cnt = t.external_cnt - 2;
        if (p->inner_cnt.fetch_add(cnt) == -cnt) {
          delete p;  // 내부 및 외부 카운트의 합계는 0
        }
        return res;
      }
      if (p->inner_cnt.fetch_sub(1) == 1) {
        delete p;  // 내부 카운트 합계는 0
      }
    }
  }

 private:
  struct Node;

  struct ReferenceCount {
    int external_cnt;
    Node* p;
  };

  struct Node {
    std::shared_ptr<T> v;
    std::atomic<int> inner_cnt = 0;
    ReferenceCount next;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

  void increase_count(ReferenceCount& old_cnt) {
    ReferenceCount new_cnt;
    do {
      new_cnt = old_cnt;
      ++new_cnt.external_cnt;  // head_에 액세스할 때 외부 카운트가 증가하면 노드가 사용 중임을 나타낸다
    } while (!head_.compare_exchange_strong(old_cnt, new_cnt));
    old_cnt.external_cnt = new_cnt.external_cnt;
  }

 private:
  std::atomic<ReferenceCount> head_;
};
```
  
* 메모리 순서를 지정하지 않으면 기본적으로 오버헤드가 가장 높은 `std::memory_order_seq_cst`가 사용되며, 연산 간의 종속성에 따라 가장 작은 메모리 순서로 최적화된다.  
  
```cpp
#include <atomic>
#include <memory>

template <typename T>
class LockFreeStack {
 public:
  ~LockFreeStack() {
    while (pop()) {
    }
  }

  void push(const T& x) {
    ReferenceCount t;
    t.p = new Node(x);
    t.external_cnt = 1;
    // 다음 비교에서 release는 모든 이전 문이 먼저 실행되도록 보장하므로 load에 여유를 두고 사용할 수 있다
    t.p->next = head_.load(std::memory_order_relaxed);
    while (!head_.compare_exchange_weak(t.p->next, t, std::memory_order_release,
                                        std::memory_order_relaxed)) {
    }
  }

  std::shared_ptr<T> pop() {
    ReferenceCount t = head_.load(std::memory_order_relaxed);
    while (true) {
      increase_count(t);  // acquire
      Node* p = t.p;
      if (!p) {
        return nullptr;
      }
      if (head_.compare_exchange_strong(t, p->next,
                                        std::memory_order_relaxed)) {
        std::shared_ptr<T> res;
        res.swap(p->v);
        // 노드가 삭제되었으므로 외부 카운트에 내부 카운트에서 2를 뺀 2를 더하고 
        // 노드가 삭제되었으므로 스레드가 이 노드에 다시 액세스할 수 없으므로 1을 뺀 1을 더한다
        const int cnt = t.external_cnt - 2;
        // 삭제하기 전에 swap 하므로 release를 사용한다
        if (p->inner_cnt.fetch_add(cnt, std::memory_order_release) == -cnt) {
          delete p;  // 내부 및 외부 카운트를 0으로 합산한다
        }
        return res;
      }
      if (p->inner_cnt.fetch_sub(1, std::memory_order_relaxed) == 1) {
        p->inner_cnt.load(std::memory_order_acquire);  // 그냥 acquire 동기화 한다
        // acquire는 delete 따르도록 보장한다
        delete p;  // 내부 카운트는 0
      }
    }
  }

 private:
  struct Node;

  struct ReferenceCount {
    int external_cnt;
    Node* p = nullptr;
  };

  struct Node {
    std::shared_ptr<T> v;
    std::atomic<int> inner_cnt = 0;
    ReferenceCount next;
    Node(const T& x) : v(std::make_shared<T>(x)) {}
  };

  void increase_count(ReferenceCount& old_cnt) {
    ReferenceCount new_cnt;
    do {  // 비교에 실패해도 현재 값은 변경되지 않으며 루프가 계속될 수 있으므로 느긋하게 선택할 수 있다
      new_cnt = old_cnt;
      ++new_cnt.external_cnt;  // head_에 액세스할 때 외부 카운트가 증가하면 노드가 사용 중임을 나타낸다
    } while (!head_.compare_exchange_strong(old_cnt, new_cnt,
                                            std::memory_order_acquire,
                                            std::memory_order_relaxed));
    old_cnt.external_cnt = new_cnt.external_cnt;
  }

 private:
  std::atomic<ReferenceCount> head_;
};
```
