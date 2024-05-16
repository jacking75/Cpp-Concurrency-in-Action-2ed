* 동시 데이터 구조의 설계는 두 가지를 고려해야 하는데 하나는 스레드 안전 액세스를 보장하는 것이고 다른 하나는 동시성 수준을 높이는 것이다.
  * 기본적인 스레드 안전 요구 사항은 다음과 같다.
    * 데이터 구조의 불변성이 스레드에 의해 손상되었을 때 스레드에 표시되지 않도록 한다.
    * 데이터 구조의 인터페이스에 내재된 경합 조건을 피하기 위해 작동적으로 완전한 함수를 제공한다.
    * 예외 발생 시 데이터 구조의 동작에 주의를 기울여 불변성이 파괴되지 않도록 한다.
    * 중첩 잠금이 발생하지 않도록 잠금 범위를 제한하고 교착 상태의 가능성을 최소화한다.
  * 데이터 구조 설계자는 데이터 구조의 동시성을 개선하기 위해 다음과 같은 관점을 고려할 수 있다.
    * 일부 연산이 잠금 범위 밖에서 실행될 수 있는지 여부
    * 데이터 구조의 다른 부분이 다른 뮤텍스에 의해 보호되는지 여부
    * 모든 연산을 동일한 수준에서 보호해야 하는지 여부
    * 연산의 의미론에 영향을 주지 않고 동시성을 향상시키기 위해 데이터 구조를 간단하게 수정할 수 있는지 여부.
  * 공유 데이터에 번갈아 액세스하는 스레드를 최소화하고 실제 동시성을 최대화하는 등 한 가지로 요약할 수 있다.
  
  
## thread-safe queue

* 之前实现过的 thread-safe stack 和 queue 都是用一把锁定保护整个数据结构，这限制了并发性，多线程在成员函数中阻塞时，同一时间只有一个线程能工作。这种限制主要是因为内部实现使用的是 [std::queue](https://en.cppreference.com/w/cpp/container/queue)，为了支持更高的并发，需要更换内部的实现方式，使用细粒度的（fine-grained）锁。最简单的实现方式是包含头尾指针的单链表，不考虑并发的单链表实现如下
* 이전에 구현된 스레드 세이프한 스택과 큐는 모두 소수의 잠금으로 전체 데이터 구조를 보호하므로 멤버 함수에서 여러 스레드가 차단되는 경우 한 번에 하나의 스레드만 작동할 수 있다는 점에서 동시성을 제한한다. 이러한 제한은 주로 내부 구현이 [std::queue](https://en.cppreference.com/w/cpp/container/queue )를 사용하기 때문이며, 더 높은 동시성을 지원하려면 내부 구현을 세분화된 잠금으로 대체해야 한다. 가장 간단한 구현은 헤드와 테일 포인터를 포함하는 단일 링크된 테이블이다. 동시성을 고려하지 않은 단일 링크된 테이블 구현은 다음과 같다.    
  
```cpp
#include <memory>
#include <utility>

template <typename T>
class Queue {
 public:
  Queue() = default;

  Queue(const Queue&) = delete;

  Queue& operator=(const Queue&) = delete;

  void push(T x) {
    auto new_node = std::make_unique<Node>(std::move(x));
    Node* new_tail_node = new_node.get();
    if (tail_) {
      tail_->next = std::move(new_node);
    } else {
      head_ = std::move(new_node);
    }
    tail_ = new_tail_node;
  }

  std::shared_ptr<T> try_pop() {
    if (!head_) {
      return nullptr;
    }
    auto res = std::make_shared<T>(std::move(head_->v));
    std::unique_ptr<Node> head_node = std::move(head_);
    head_ = std::move(head_node->next);
    return res;
  }

 private:
  struct Node {
    explicit Node(T x) : v(std::move(x)) {}
    T v;
    std::unique_ptr<Node> next;
  };

  std::unique_ptr<Node> head_;
  Node* tail_ = nullptr;
};
```  
  
* 즉, 두 개의 뮤텍스를 사용하여 헤드와 테일 포인터를 보호하는 이 구현은 멀티스레딩에서도 명백한 문제가 있다. push는 헤드와 테일 포인터를 동시에 수정할 수 있고, 두 뮤텍스를 잠그며 요소가 하나만 있을 때 헤드와 테일 포인터가 같으면 push가 쓰고 try_pop이 읽는 다음 노드는 동일한 객체이므로 경쟁이 발생하고 잠금도 동일한 뮤텍스에 있다.  
* 이 문제는 더미 노드를 헤드 노드보다 먼저 초기화하면 쉽게 해결할 수 있으므로 push는 테일 노드에만 접근하고 헤드 노드에 대한 try_pop과 경쟁하지 않는다.  
  
```cpp
#include <memory>
#include <utility>

template <typename T>
class Queue {
 public:
  Queue() : head_(new Node), tail_(head_.get()) {}

  Queue(const Queue&) = delete;

  Queue& operator=(const Queue&) = delete;

  void push(T x) {
    auto new_val = std::make_shared<T>(std::move(x));
    auto new_node = std::make_unique<Node>();
    Node* new_tail_node = new_node.get();
    tail_->v = new_val;
    tail_->next = std::move(new_node);
    tail_ = new_tail_node;
  }

  std::shared_ptr<T> try_pop() {
    if (head_.get() == tail_) {
      return nullptr;
    }
    std::shared_ptr<T> res = head->v;
    std::unique_ptr<Node> head_node = std::move(head_);
    head_ = std::move(head_node->next);
    return res;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    std::unique_ptr<Node> next;
  };

  std::unique_ptr<Node> head_;
  Node* tail_ = nullptr;
};
```
  
* 그런 다음 가능한 한 작게 lock을 추가한다  
  
```cpp
#include <memory>
#include <mutex>
#include <utility>

template <typename T>
class ConcurrentQueue {
 public:
  ConcurrentQueue() : head_(new Node), tail_(head_.get()) {}

  ConcurrentQueue(const ConcurrentQueue&) = delete;

  ConcurrentQueue& operator=(const ConcurrentQueue&) = delete;

  void push(T x) {
    auto new_val = std::make_shared<T>(std::move(x));
    auto new_node = std::make_unique<Node>();
    Node* new_tail_node = new_node.get();

    std::lock_guard<std::mutex> l(tail_mutex_);
    tail_->v = new_val;
    tail_->next = std::move(new_node);
    tail_ = new_tail_node;
  }

  std::shared_ptr<T> try_pop() {
    std::unique_ptr<Node> head_node = pop_head();
    return head_node ? head_node->v : nullptr;
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    std::unique_ptr<Node> next;
  };

 private:
  std::unique_ptr<Node> pop_head() {
    std::lock_guard<std::mutex> l(head_mutex_);
    if (head_.get() == get_tail()) {
      return nullptr;
    }
    std::unique_ptr<Node> head_node = std::move(head_);
    head_ = std::move(head_node->next);
    return head_node;
  }

  Node* get_tail() {
    std::lock_guard<std::mutex> l(tail_mutex_);
    return tail_;
  }

 private:
  std::unique_ptr<Node> head_;
  Node* tail_ = nullptr;
  std::mutex head_mutex_;
  std::mutex tail_mutex_;
};
```
  
* push는 새 값과 잠금 해제된 새 노드를 모두 생성하며 여러 스레드를 사용하여 새 값과 새 노드를 동시에 생성할 수 있다. 하나의 스레드만 동시에 새 노드를 추가할 수 있지만 포인터 할당 연산만 필요하고 꼬리 노드를 잠그는 시간이 매우 짧다. try_pop은 꼬리 노드를 비교하는데만 사용되며 꼬리 노드를 보유하는 시간도 매우 짧으므로 try_pop과 push를 거의 동시에 호출할 수 있다. try_pop은 포인터 할당 연산만 하는 헤드 노드를 잠그기 때문에 오버헤드가 높다. 즉, 하나의 스레드만 동시에 pop_head를 호출할 수 있지만 여러 스레드가 노드를 삭제하고 데이터를 반환할 수 있으므로 try_pop에 대한 동시 호출 횟수가 증가한다.
* 마지막으로 [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)과 함께 wait_and_pop을 구현하면 이전과 동일한 인터페이스를 제공하지만 동시 스레드 수가 더 많아진니다

```cpp
#include <condition_variable>
#include <memory>
#include <mutex>
#include <utility>

template <typename T>
class ConcurrentQueue {
 public:
  ConcurrentQueue() : head_(new Node), tail_(head_.get()) {}

  ConcurrentQueue(const ConcurrentQueue&) = delete;

  ConcurrentQueue& operator=(const ConcurrentQueue&) = delete;

  void push(T x) {
    auto new_val = std::make_shared<T>(std::move(x));
    auto new_node = std::make_unique<Node>();
    Node* new_tail_node = new_node.get();
    {
      std::lock_guard<std::mutex> l(tail_mutex_);
      tail_->v = new_val;
      tail_->next = std::move(new_node);
      tail_ = new_tail_node;
    }
    cv_.notify_one();
  }

  std::shared_ptr<T> try_pop() {
    std::unique_ptr<Node> head_node = try_pop_head();
    return head_node ? head_node->v : nullptr;
  }

  bool try_pop(T& res) {
    std::unique_ptr<Node> head_node = try_pop_head(res);
    return head_node != nullptr;
  }

  std::shared_ptr<T> wait_and_pop() {
    std::unique_ptr<Node> head_node = wait_pop_head();
    return head_node->v;
  }

  void wait_and_pop(T& res) { wait_pop_head(res); }

  bool empty() const {
    std::lock_guard<std::mutex> l(head_mutex_);
    return head_.get() == get_tail();
  }

 private:
  struct Node {
    std::shared_ptr<T> v;
    std::unique_ptr<Node> next;
  };

 private:
  std::unique_ptr<Node> try_pop_head() {
    std::lock_guard<std::mutex> l(head_mutex_);
    if (head_.get() == get_tail()) {
      return nullptr;
    }
    return pop_head();
  }

  std::unique_ptr<Node> try_pop_head(T& res) {
    std::lock_guard<std::mutex> l(head_mutex_);
    if (head_.get() == get_tail()) {
      return nullptr;
    }
    res = std::move(*head_->v);
    return pop_head();
  }

  std::unique_ptr<Node> wait_pop_head() {
    std::unique_lock<std::mutex> l(wait_for_data());
    return pop_head();
  }

  std::unique_ptr<Node> wait_pop_head(T& res) {
    std::unique_lock<std::mutex> l(wait_for_data());
    res = std::move(*head_->v);
    return pop_head();
  }

  std::unique_lock<std::mutex> wait_for_data() {
    std::unique_lock<std::mutex> l(head_mutex_);
    cv_.wait(l, [this] { return head_.get() != get_tail(); });
    return l;
  }

  std::unique_ptr<Node> pop_head() {
    std::unique_ptr<Node> head_node = std::move(head_);
    head_ = std::move(head_node->next);
    return head_node;
  }

  Node* get_tail() {
    std::lock_guard<std::mutex> l(tail_mutex_);
    return tail_;
  }

 private:
  std::unique_ptr<Node> head_;
  Node* tail_ = nullptr;
  std::mutex head_mutex_;
  mutable std::mutex tail_mutex_;
  std::condition_variable cv_;
};
```

## thread-safe map
    
* [std::map](https://en.cppreference.com/w/cpp/container/map) 및 [std::unordered_map](https://en.cppreference.com/w/cpp/container/unordered_map)에 대한 동시 액세스. 인터페이스는 요소를 삭제하는 다른 스레드에 의해 무효화되는 이터레이터에 문제가 있으므로 스레드 세이프한 맵 인터페이스는 이터레이터를 건너뛰도록 설계되었다.
* 세분화된 잠금을 사용하려면 표준 라이브러리 컨테이너를 사용해서는 안 된다. 연관 컨테이너를 위한 세 가지 대체 데이터 구조가 있다. 하나는 이진 트리(예: 빨간색-검정색 트리)이지만 각 조회 수정은 루트 노드에 액세스하는 것으로 시작되므로 루트 노드를 잠가야 하며 트리 아래쪽 노드에 액세스하면 잠금이 해제되더라도 전체 데이터 구조를 포괄하는 단일 잠금보다 낫지 않다.
* 두 번째 방법은 정렬된 배열로 주어진 값이 어디에 위치해야 하는지 미리 알 수 없기 때문에 전체 배열을 커버하는 잠금도 필요하므로 이진 트리보다 나쁘다다.
* 세 번째 방법은 해시 테이블이다. 고정된 수의 버킷이 있고 키의 속성과 해시 함수에 따라 키가 속한 버킷이 달라지는 경우, 각 버킷을 개별적으로 안전하게 잠글 수 있다는 의미이다. 읽기/쓰기 잠금을 사용하는 경우 버킷 개수의 배수만큼 동시성을 높일 수 있다.  
    
```cpp
#include <algorithm>
#include <functional>
#include <list>
#include <map>
#include <memory>
#include <mutex>
#include <shared_mutex>
#include <utility>
#include <vector>

template <typename K, typename V, typename Hash = std::hash<K>>
class ConcurrentMap {
 public:
  // 버킷 수는 기본값이 19개이다(일반적으로 버킷 수의 x %가 x에 대한 버킷의 인덱스로 사용된다. 버킷 수가 소수이면 버킷이 균등하게 분포된다).
  ConcurrentMap(std::size_t n = 19, const Hash& h = Hash{})
      : buckets_(n), hasher_(h) {
    for (auto& x : buckets_) {
      x.reset(new Bucket);
    }
  }

  ConcurrentMap(const ConcurrentMap&) = delete;

  ConcurrentMap& operator=(const ConcurrentMap&) = delete;

  V get(const K& k, const V& default_value = V{}) const {
    return get_bucket(k).get(k, default_value);
  }

  void set(const K& k, const V& v) { get_bucket(k).set(k, v); }

  void erase(const K& k) { get_bucket(k).erase(k); }

  // 사용하기 쉽도록 std::map에 매핑을 제공
  std::map<K, V> to_map() const {
    std::vector<std::unique_lock<std::shared_mutex>> locks;
    for (auto& x : buckets_) {
      locks.emplace_back(std::unique_lock<std::shared_mutex>(x->m));
    }
    std::map<K, V> res;
    for (auto& x : buckets_) {
      for (auto& y : x->data) {
        res.emplace(y);
      }
    }
    return res;
  }

 private:
  struct Bucket {
    std::list<std::pair<K, V>> data;
    mutable std::shared_mutex m;  // 모든 배럴은 이 잠금 장치로 보호

    V get(const K& k, const V& default_value) const {
      // 값이 수정되지 않아 비정상적으로 안전함
      std::shared_lock<std::shared_mutex> l(m);  // 읽기 전용 잠금, 공유 가능
      auto it = std::find_if(data.begin(), data.end(),
                             [&](auto& x) { return x.first == k; });
      return it == data.end() ? default_value : it->second;
    }

    void set(const K& k, const V& v) {
      std::unique_lock<std::shared_mutex> l(m);  // 쓰기, 단독 점유 
      auto it = std::find_if(data.begin(), data.end(),
                             [&](auto& x) { return x.first == k; });
      if (it == data.end()) {
        data.emplace_back(k, v);  // emplace_back 
      } else {
        it->second = v;  // 할당은 예외를 발생시킬 수 있지만 값은 사용자가 제공하므로 사용자가 안전하게 처리할 수 있다
      }
    }

    void erase(const K& k) {
      std::unique_lock<std::shared_mutex> l(m);  // 쓰기, 단독 점유
      auto it = std::find_if(data.begin(), data.end(),
                             [&](auto& x) { return x.first == k; });
      if (it != data.end()) {
        data.erase(it);
      }
    }
  };

  Bucket& get_bucket(const K& k) const {  // 버킷 수는 고정되어 있으므로 잠금 없이 호출할 수 있다
    return *buckets_[hasher_(k) % buckets_.size()];
  }

 private:
  std::vector<std::unique_ptr<Bucket>> buckets_;
  Hash hasher_;
};
```

## thread-safe list

```cpp
#include <memory>
#include <mutex>
#include <utility>

template <typename T>
class ConcurrentList {
 public:
  ConcurrentList() = default;

  ~ConcurrentList() {
    remove_if([](const Node&) { return true; });
  }

  ConcurrentList(const ConcurrentList&) = delete;

  ConcurrentList& operator=(const ConcurrentList&) = delete;

  void push_front(const T& x) {
    std::unique_ptr<Node> t(new Node(x));
    std::lock_guard<std::mutex> head_lock(head_.m);
    t->next = std::move(head_.next);
    head_.next = std::move(t);
  }

  template <typename F>
  void for_each(F f) {
    Node* cur = &head_;
    std::unique_lock<std::mutex> head_lock(head_.m);
    while (Node* const next = cur->next.get()) {
      std::unique_lock<std::mutex> next_lock(next->m);
      head_lock.unlock();  // 이전 노드의 잠금을 해제할 수 있도록 다음 노드를 잠근다
      f(*next->data);
      cur = next;                        // 현재 노드가 다음 노드를 가리킴
      head_lock = std::move(next_lock);  // 위의 과정을 반복하면서 다음 노드의 잠금 소유권을 이전한다
    }
  }

  template <typename F>
  std::shared_ptr<T> find_first_if(F f) {
    Node* cur = &head_;
    std::unique_lock<std::mutex> head_lock(head_.m);
    while (Node* const next = cur->next.get()) {
      std::unique_lock<std::mutex> next_lock(next->m);
      head_lock.unlock();
      if (f(*next->data)) {
        return next->data;  // 추가 검색 없이 목표 값을 반환
      }
      cur = next;
      head_lock = std::move(next_lock);
    }
    return nullptr;
  }

  template <typename F>
  void remove_if(F f) {
    Node* cur = &head_;
    std::unique_lock<std::mutex> head_lock(head_.m);
    while (Node* const next = cur->next.get()) {
      std::unique_lock<std::mutex> next_lock(next->m);
      if (f(*next->data)) {  // true 이면 다음 노드를 제거
        std::unique_ptr<Node> old_next = std::move(cur->next);
        cur->next = std::move(next->next);  // 다음 노드를 다음 노드로 설정
        next_lock.unlock();
      } else {  // 그렇지 않으면 다음 노드로 계속 진행
        head_lock.unlock();
        cur = next;
        head_lock = std::move(next_lock);
      }
    }
  }

 private:
  struct Node {
    std::mutex m;
    std::shared_ptr<T> data;
    std::unique_ptr<Node> next;
    Node() = default;
    Node(const T& x) : data(std::make_shared<T>(x)) {}
  };

  Node head_;
};
```
