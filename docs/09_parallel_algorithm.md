## [구현 전략（execution policy）](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t)

* C++17은 표준 라이브러리 알고리즘의 병렬 버전을 오버로드하지만 실행 전략을 지정하는 추가 파라미터가 있다는 차이점이 있다
  
```cpp
std::vector<int> v;
std::sort(std::execution::par, v.begin(), v.end());
```
  
* [std::execution::par](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag )는 알고리즘의 다중 스레드 병렬 실행이 허용됨을 나타내며, 이는 요구 사항이 아닌 권한임을 유의하자. 요구 사항이 없는 경우에도 알고리즘은 여전히 단일 스레드로 실행될 수 있다.
* 또한 실행 정책이 지정되어 있으면 알고리즘 복잡성 요구 사항이 더 완화되는데, 병렬 알고리즘은 일반적으로 시스템의 병렬성을 활용하기 위해 더 많은 작업을 수행해야 하기 때문이다. 예를 들어, 작업을 100개의 프로세서로 나누면 총 작업이 두 배로 늘어나더라도 50배의 성능을 얻을 수 있다.
* 다음 실행 정책 클래스는 [\<execution\>](https://en.cppreference.com/w/cpp/header/execution )에 지정되어 있다.
  
```cpp
std::execution::sequenced_policy
std::execution::parallel_policy
std::execution::parallel_unsequenced_policy
std::execution::unsequenced_policy  // C++20
```

* 호출하고 해당 전역 객체를 지정

```cpp
std::execution::seq
std::execution::par
std::execution::par_unseq
std::execution::unseq  // C++20
```
  
* 실행 정책을 사용하는 경우 알고리즘의 동작은 알고리즘 복잡성, 예외 발생 시 동작, 알고리즘 단계가 실행되는 위치, 방법 및 시기 측면에서 실행 정책의 영향을 받는다.
* 병렬 실행의 스케줄링 오버헤드를 관리하는 것 외에도 많은 병렬 알고리즘은 더 많은 핵심 연산(스와핑, 비교, 함수 객체 사용 등)을 수행하므로 실제 소요되는 총 시간이 줄어들어 전반적인 성능이 향상된다. 이것이 알고리즘 복잡성이 영향을 받는 이유이며, 알고리즘마다 구체적인 변경 사항은 다음과 같다.
* 실행 정책이 지정되지 않은 경우 알고리즘에 대한 다음 호출은 전파되는 예외를 던진다.
  
```cpp
std::for_each(v.begin(), v.end(), [](auto x) { throw my_exception(); });
```
  
* 而指定执行策略时，如果算法执行期间抛出异常，则行为结果由执行策略决定。如果有任何未捕获的异常，执行策略将调用 [std::terminate](https://en.cppreference.com/w/cpp/error/terminate) 终止程序，唯一可能抛出异常的情况是，内部操作不能获取足够的内存资源时抛出 [std::bad_alloc](https://en.cppreference.com/w/cpp/memory/new/bad_alloc)。如下操作将调用 [std::terminate](https://en.cppreference.com/w/cpp/error/terminate) 终止程序   
* 실행 정책이 지정된 경우 알고리즘 실행 중에 예외가 발생하면 실행 정책에 따라 동작 결과가 결정된다. 잡히지 않은 예외가 있는 경우 실행 정책은 [std::terminate](https://en.cppreference.com/w/cpp/error/terminate)를 호출하여 프로그램을 종료한다. 예외가 발생할 수 있는 유일한 경우는 내부 연산이 충분한 메모리 자원을 가져오는 데 실패하여 [std:: bad_alloc](https://en.cppreference.com/w/cpp/error/terminate)을 throw 하는 경우이다. 다음은 [std::terminate](https://en.cppreference.com/w/cpp/error/terminate)를 호출하여 프로그램을 종료한다.  

```cpp
std::for_each(std::execution::seq, v.begin(), v.end(),
              [](auto x) { throw my_exception(); });
```
  
* 실행 전략은 서로 다른 방식으로 실행된다. 실행 정책은 알고리즘 단계를 실행하는 에이전트를 지정하며 일반 스레드, 벡터 스트림, GPU 스레드 등 무엇이든 될 수 있습니다. 실행 정책은 알고리즘 단계를 특정 순서로 실행할지 여부, 서로 다른 알고리즘 단계의 일부를 서로 인터리빙하거나 병렬로 실행할 수 있는지 여부 등 알고리즘 단계가 실행되는 순서에 대한 제한 사항도 지정한다. 다양한 실행 정책은 아래에 자세히 설명되어 있다.    
  
### [std::execution::sequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t)  
  
* [std::execution::sequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t ) 이 정책은 병렬 실행이 없을 수 있고(없을 수도 있음) 모든 연산이 단일 스레드에서 실행될 것을 요구한다. 단일 스레드에서 실행될 것을 요구한다. 그러나 이 정책도 실행 정책이므로 다른 실행 정책과 동일한 방식으로 알고리즘 복잡성 및 예외 동작에 영향을 미친다.  
* 스레드에서 실행되는 모든 작업은 일정한 순서로 실행되어야 하므로 서로 인터리빙할 수 없다. 그러나 특정 순서가 지정되어 있지 않으므로 함수 호출에 따라 다른 순서를 지정할 수 있다.

```cpp
std::vector<int> v(1000);
int n = 0;
// 컨테이너에 1~1000개를 넣는다, 넣는 순서는 순차적이거나 무질서할 수 있다
std::for_each(std::execution::seq, v.begin(), v.end(),
              [&](int& x) { x = ++n; });
```  
  
* 따라서 [std::execution::sequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책은 알고리즘이 이터레이터, 값 및 호출 가능한 객체를 사용하는 경우가 거의 없다. 동기화 메커니즘을 자유롭게 사용할 수 있으며, 동일한 스레드에서 호출되는 연산에 의존할 수 있지만, 해당 연산 순서는 아니다    
  
### [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t)
  
* [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책은 여러 스레드에 걸쳐 기본적인 병렬 실행을 제공하며 연산은 해당 알고리즘을 호출한 스레드 또는 라이브러리에서 생성된 스레드에서 실행될 수 있다. 알고리즘을 호출한 스레드 또는 라이브러리에서 생성한 스레드에서 실행할 수 있다. 주어진 스레드의 작업은 결정론적 순서로 실행되어야 하며 서로 인터리빙되어서는 안 된다. 다시 말하지만 이 순서는 지정되지 않으며 호출마다 다를 수 있다. 주어진 연산은 고정 스레드에서 전체 주기 동안 실행된다.  
* 따라서 [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책은 이터레이터, 값 및 호출 가능한 객체의 사용에 대해 특정 요구 사항을 부과한다. 이들은 병렬로 호출될 때 데이터 경합을 일으키지 않아야 하며, 같은 스레드의 다른 연산이나 같은 스레드에서 실행되지 않는 다른 연산에 의존하지 않아야 한다.
* 대부분의 경우 [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책을 사용할 수 있다.  
  
```cpp
std::for_each(std::execution::par, v.begin(), v.end(), [](auto& x) { ++x; });
```
 
* 요소 간에 특정 순서가 있거나 공유 데이터에 대한 액세스가 동기화되지 않은 경우에만 문제가 된다  
  
```cpp
std::vector<int> v(1000);
int n = 0;
std::for_each(std::execution::par, v.begin(), v.end(), [&](int& x) {
  x = ++n;
});  // 둘 이상의 스레드가 람다를 실행하는 경우 n에 대한 데이터 경합이 발생한다
```
  
* 따라서 [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책을 사용하려면 정의되지 않은 동작 가능성을 고려해야 한다. 뮤텍스나 원자 변수를 사용하여 경합 문제를 해결할 수도 있지만 이는 동시성을 손상시킨다. 그러나 이 예제는 상황을 설명하기 위한 것이며 일반적으로 [std::execution::parallel_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책을 사용하여 공유 데이터에 대한 동기식 동기식 액세스를 허용하는 데 사용된다.  
    

### [std::execution::parallel_unsequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t)  
  
* [std::execution::parallel_unsequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 이 정책은 가능한 최대 병렬화를 제공하지만 다음과 같은 비용이 든다. 알고리즘에서 사용하는 이터레이터, 값 및 호출 가능한 객체에 대한 가장 엄격한 요구 사항을 제공한다.
* [std::execution::parallel_unsequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책을 사용하는 알고리즘은 순서 없이 실행할 수 있다. 지정되지 않은 모든 스레드에서 실행되고 각 스레드에서 서로에 대해 순서가 지정되지 않은 방식으로 실행될 수 있다. 즉, 단일 스레드에서 연산을 서로 인터리빙할 수 있고, 동일한 스레드의 두 번째 연산이 첫 번째 연산이 완료되기 전에 시작될 수 있으며 스레드 간에 이동할 수 있고, 주어진 연산이 한 스레드에서 시작하여 다른 스레드에서 실행되고 세 번째 스레드에서 완료될 수 있다.
* [std::execution::parallel_unsequenced_policy](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t) 정책을 사용할 때 알고리즘에 제공되는 이터레이터, 값, 호출 가능한 객체에 대한 연산은 어떤 형태의 동기화도 사용할 수 없으며, 다른 코드와 동기화된 함수를 호출할 수도 없다. 즉, 연산은 관련 요소 또는 해당 요소를 기반으로 액세스 가능한 데이터에만 작동할 수 있으며 스레드나 요소 간에 공유되는 데이터를 수정할 수 없다.  
  

## 표준 라이브러리 병렬 알고리즘
  
* [\<algorithm\>](https://en.cppreference.com/w/cpp/algorithm) 및 [\<numberic\>](https://en.cppreference.com/w/cpp/header/numeric)의 대부분의 알고리즘은 병렬 버전에 과부하가 걸린다. [std::accumulate](https://en.cppreference.com/w/cpp/algorithm/accumulate)에는 병렬 버전이 없지만, C++17은 [std::reduce](https://en.cppreference.com/w/cpp/algorithm/reduce)를 제공한다.   
  
```cpp
std::accumulate(v.begin(), v.end(), 0);
std::reduce(std::execution::par, v.begin(), v.end());
```
  
* 정규 알고리즘에 병렬 버전에 대한 과부하가 있는 경우 병렬 버전에는 정규 알고리즘의 모든 원래 과부하에 대한 해당 과부하 버전이 있다  

```cpp
template <class RandomIt>
void sort(RandomIt first, RandomIt last);

template <class RandomIt, class Compare>
void sort(RandomIt first, RandomIt last, Compare comp);

// 병렬 버전은 두 가지 과부하에 해당한다
template <class ExecutionPolicy, class RandomIt>
void sort(ExecutionPolicy&& policy, RandomIt first, RandomIt last);

template <class ExecutionPolicy, class RandomIt, class Compare>
void sort(ExecutionPolicy&& policy, RandomIt first, RandomIt last,
          Compare comp);
```
  
* 그러나 병렬 버전의 오버로딩은 일부 알고리즘에 약간의 차이가 있다. 일반 버전이 입력 이터레이터(input iterator) 또는 출력 이터레이터(output iterator)를 사용하는 경우 병렬 버전의 오버로딩은 정방향 이터레이터(forward iterator)를 사용한다
  
```cpp
template <class InputIt, class OutputIt>
OutputIt copy(InputIt first, InputIt last, OutputIt d_first);

template <class ExecutionPolicy, class ForwardIt1, class ForwardIt2>
ForwardIt2 copy(ExecutionPolicy&& policy, ForwardIt1 first, ForwardIt1 last,
                ForwardIt2 d_first);
```
  
* 입력 이터레이터는 가리키는 값을 읽는 데만 사용할 수 있고, 이터레이터는 자체 증가하며 더 이상 이전에 가리킨 값에 액세스할 수 없으며 일반적으로 콘솔이나 네트워크에서 입력하거나 마찬가지로 출력 이터레이터는 일반적으로 파일로 출력하거나 컨테이너에 값을 추가하는 데 사용되며 단방향으로 사용된다(예: [std::ostream_iterator](https://en.cppreference.com/w/cpp/iterator/ostream_iterator )).
* 정방향 이터레이터는 요소에 대한 참조를 반환하므로 읽기 및 쓰기에 사용할 수 있으며 한 방향으로만 전달할 수도 있다 [std::forward_list](https://en.cppreference.com/w/cpp/container/forward_list )의 이터레이터는 정방향 이터레이터이며 이전에 가리킨 값으로 돌아갈 수는 없지만 이전 요소를 가리키는 복사본을 저장하여 재사용할 수 있다(예: [std::forward_list::begin](https://en.cppreference.com/w/cpp/container/forward_list/begin )). 병렬 처리의 경우 반복자를 재사용할 수 있는 것이 중요하다. 또한 순방향 이터레이터의 자체 증가는 이터레이터의 다른 복사본을 무효화하지 않으므로 다른 스레드의 이터레이터가 영향을 받는 것에 대해 걱정할 필요가 없다. 입력 이터레이터를 사용하는 경우 모든 스레드는 병렬 처리가 불가능한 하나의 이터레이터만 공유할 수 있다.
* 구현에서 요구 사항을 더 잘 충족하는 비표준 전략을 제공하지 않는 한, [std::execution::par](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag )가 가장 일반적으로 사용되는 전략이다. 경우에 따라 [std::execution::par_unseq](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag)를 사용할 수도 있는데 이는 더 나은 동시성을 보장하지는 않지만 라이브러리에서 작업을 재배열하고 인터리빙하여 성능을 개선할 수 있는 기능을 제공한다. 작업을 재배치하고 인터리빙하여 성능을 향상시킬 수 있지만 알고리즘 자체가 여러 스레드가 동일한 요소에 액세스하는 것을 허용하지 않고 다른 스레드의 데이터 액세스를 방지하기 위해 알고리즘 호출에 외부 동기화를 사용하는 경우에만 스레드 안전할 수 있는 동기화 메커니즘을 사용할 수 없다는 대가를 치뤄야 한다.
* 내부 동기화 메커니즘은 [std::execution::par](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag )와 함께만 사용할 수 있으며, [std::execution::par_unseq ](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag )를 사용하면 정의되지 않은 동작이 발생한다


```cpp
#include <algorithm>
#include <mutex>
#include <vector>

class A {
 public:
  int get() const {
    std::lock_guard<std::mutex> l(m_);
    return n_;
  }

  void inc() {
    std::lock_guard<std::mutex> l(m_);
    ++n_;
  }

 private:
  mutable std::mutex m_;
  int n_ = 0;
};

void f(std::vector<A>& v) {
  std::for_each(std::execution::par, v.begin(), v.end(), [](A& x) { x.inc(); });
}
```

* [std::execution::par_unseq](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag )를 사용하는 경우 동기화 메커니즘은 외부에서 사용해야 한다.

```cpp
#include <algorithm>
#include <mutex>
#include <vector>

class A {
 public:
  int get() const { return n_; }
  void inc() { ++n_; }

 private:
  int n_ = 0;
};

class B {
 public:
  void lock() { m_.lock(); }
  void unlock() { m_.unlock(); }
  std::vector<A>& get() { return v_; }

 private:
  std::mutex m_;
  std::vector<A> v_;
};

void f(B& x) {
  std::lock_guard<std::mutex> l(x);
  auto& v = x.get();
  std::for_each(std::execution::par_unseq, v.begin(), v.end(),
                [](A& x) { x.inc(); });
}
```
  
* 좀 더 실용적인 예가 있다. 수백만 개의 접속 로그가 있는 웹사이트가 있고 데이터를 쉽게 보기 위해 로그를 처리해야 한다고 가정해 보겠다. 로그의 각 줄을 처리하는 것은 별도의 작업이며 병렬 알고리즘을 사용하는 데 적합하다.  

```cpp
struct Log {
  std::string page;
  time_t visit_time;
  // any other fields
};

extern Log parse(const std::string& line);

using Map = std::unordered_map<std::string, unsigned long long>;

Map f(const std::vector<std::string>& v) {
  struct Combine {
    // log、Map 두 매개 변수에는 네 가지 조합이 있으므로 네 가지 과부하가 필요하다
    Map operator()(Map lhs, Map rhs) const {
      if (lhs.size() < rhs.size()) {
        std::swap(lhs, rhs);
      }
      for (const auto& x : rhs) {
        lhs[x.first] += x.second;
      }
      return lhs;
    }

    Map operator()(Log l, Map m) const {
      ++m[l.page];
      return m;
    }

    Map operator()(Map m, Log l) const {
      ++m[l.page];
      return m;
    }

    Map operator()(Log lhs, Log rhs) const {
      Map m;
      ++m[lhs.page];
      ++m[rhs.page];
      return m;
    }
  };

  return std::transform_reduce(std::execution::par, v.begin(), v.end(),
                               Map{},      // 초기값, 빈 map
                               Combine{},  // 두 요소를 결합하는 이진 연산
                               parse);  // 각 요소에 대해 수행되는 단항 연산
}
```
