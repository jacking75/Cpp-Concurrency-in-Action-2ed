* C++11은 표준 스레딩 라이브러리로 부스트 스레딩 라이브러리를 도입했고, 저자 Anthony Williams는 2012년에 *[C++ Concurrency in Action](https://book.douban.com/subject/4130141/)* 을 출간하여 그 기능을 소개했습니다. C++17에 대응하여 2019년 2월에 [2판] (https://book.douban.com/subject/27036085/) 을 출간했습니다. *[C++ Concurrency in Action 2ed](https://learning.oreilly.com/library/view/c-concurrency-in/9781617294693/)*  첫 5장에서는 [스레드 지원 라이브러리](https://en.cppreference.com/w/cpp/thread) 를 소개합니다. 마지막 여섯 장에서는 실용적인 관점에서 동시 프로그래밍의 설계 아이디어를 소개하고[std::scoped_lock](https://en.cppreference.com/w/cpp/thread/scoped_lock)、[std::shared_mutex](https://en.cppreference.com/w/cpp/thread/shared_mutex)，并多出一章（第十章）介绍 [C++17 표준 라이브러리 병렬 알고리즘](https://en.cppreference.com/w/cpp/header/execution)，또한 다음과 같이 적절한 경우 C++20 관련 기능을 추가할 예정입니다 [std::jthread](https://en.cppreference.com/w/cpp/thread/jthread)、[std::counting_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore)、[std::barrier](https://en.cppreference.com/w/cpp/thread/barrier)、[std::latch](https://en.cppreference.com/w/cpp/thread/latch) 등. 이 책을 읽기 전에 다음을 참조하세요. [Andrew S. Tanenbaum](https://en.wikipedia.org/wiki/Andrew_S._Tanenbaum)  [*Modern Operating Systems*](https://book.douban.com/subject/25864553/)，운영 체제의 기본 사항[프로세스 및 스레드](reference/processes_and_threads.html)、[데드락](reference/deadlocks.html)、[메모리 관리](reference/memory_management.html)、[파일 시스템](reference/file_systems.html)、[I/O](reference/IO.html) 등）。이것은 참고용 개인 메모이며, 자세한 내용은 [원서](https://learning.oreilly.com/library/view/c-concurrency-in/9781617294693/) 를 참조하세요. 
  
## [스레딩 지원 라이브러리](https://en.cppreference.com/w/cpp/thread)

1. [스레드 관리（Managing thread）](01_managing_thread.html)：[\<thread\>](https://en.cppreference.com/w/cpp/header/thread)
2. [스레드 간 데이터 공유（Sharing data between thread）](02_sharing_data_between_thread.html)：[\<mutex\>](https://en.cppreference.com/w/cpp/header/mutex)、[\<shared_mutex\>](https://en.cppreference.com/w/cpp/header/shared_mutex)
3. [동기식 동시 작업（Synchronizing concurrent operation）](03_synchronizing_concurrent_operation.html)：[\<condition_variable\>](https://en.cppreference.com/w/cpp/header/condition_variable)、[\<semaphore\>](https://en.cppreference.com/w/cpp/header/semaphore)、[\<barrier\>](https://en.cppreference.com/w/cpp/header/barrier)、[\<latch\>](https://en.cppreference.com/w/cpp/header/latch)、[\<future\>](https://en.cppreference.com/w/cpp/header/future)、[\<chrono\>](https://en.cppreference.com/w/cpp/header/chrono)、[\<ratio\>](https://en.cppreference.com/w/cpp/header/ratio)
4. [C++ 메모리 모델 및 원자 유형 기반 연산(The C++ memory model and operations on atomic type)](04_the_cpp_memory_model_and_operations_on_atomic_type.html)：[\<atomic\>](https://en.cppreference.com/w/cpp/header/atomic)

## 편집 및 실제 편집

5. [잠금 기반 및 데이터 구조 설계（Designing lock-based concurrent data structure）](05_designing_lock_based_concurrent_data_structure.html)
6. [잠금이 없는 동시 데이터 구조 설계하기（Designing lock-free concurrent data structure）](06_designing_lock_free_concurrent_data_structure.html)
7. [동시 코드 설계하기（Designing concurrent code）](07_designing_concurrent_code.html)
8. [고급 스레드 관리（Advanced thread management）](08_advanced_thread_management.html)
9. [병렬 알고리즘（Parallel algorithm）](09_parallel_algorithm.html)：[\<execution\>](https://en.cppreference.com/w/cpp/header/execution)
10. [멀티 스레드 응용 프로그램 테스트 및 디버깅（Testing and debugging multithreaded application）](10_testing_and_debugging_multithreaded_application.html)

## 표준 파일 관련 문서

|헤더|설명|
|:-:|:-:|
|[\<thread\>](https://en.cppreference.com/w/cpp/header/thread)、[\<stop_token\>](https://en.cppreference.com/w/cpp/header/stop_token)|线程|
|[\<mutex\>](https://en.cppreference.com/w/cpp/header/mutex)、[\<shared_mutex\>](https://en.cppreference.com/w/cpp/header/shared_mutex)|锁|
|[\<condition_variable\>](https://en.cppreference.com/w/cpp/header/condition_variable)|조건 변수|
|[\<semaphore\>](https://en.cppreference.com/w/cpp/header/semaphore)|세마포어|
|[\<barrier\>](https://en.cppreference.com/w/cpp/header/barrier)、[\<latch\>](https://en.cppreference.com/w/cpp/header/latch)|屏障|
|[\<future\>](https://en.cppreference.com/w/cpp/header/future)|비동기 처리 결과|
|[\<chrono\>](https://en.cppreference.com/w/cpp/header/chrono)|시계|
|[\<ratio\>](https://en.cppreference.com/w/cpp/header/ratio)|컴파일 기간 유리수 산술|
|[\<atomic\>](https://en.cppreference.com/w/cpp/header/atomic)|원자 유형과 원자 연산|
|[\<execution\>](https://en.cppreference.com/w/cpp/header/execution)|표준 라이브러리 알고리즘 실행 정책|

## 동시성 라이브러리 비교

### [C++11 Thread](https://en.cppreference.com/w/cpp/thread)

|특성|API|
|:-:|:-:|
|thread|[std::thread](https://en.cppreference.com/w/cpp/thread/thread)|
|mutex|[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)、[std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)、[std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock)|
|condition variable|[std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)、[std::condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any)|
|atomic|[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)、[std::atomic_thread_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)|
|future|[std::future](https://en.cppreference.com/w/cpp/thread/future)、[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)|
|interruption|없다|

### [Boost Thread](https://www.boost.org/doc/libs/1_82_0/doc/html/thread.html)

|특성|API|
|:-:|:-:|
|thread|[boost::thread](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/thread_management.html#thread.thread_management.thread)|
|mutex|[boost::mutex](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.mutex_types.mutex)、[boost::lock_guard](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.lock_guard.lock_guard)、[boost::unique_lock](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.locks.unique_lock)|
|condition variable|[boost::condition_variable](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.condvar_ref.condition_variable)、[boost::condition_variable_any](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.condvar_ref.condition_variable_any)|
|atomic|없다|
|future|[boost::future](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.futures.reference.unique_future)、[boost::shared_future](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.futures.reference.shared_future)|
|interruption|[thread::interrupt](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/thread_management.html#thread.thread_management.thread.interrupt)|

### [POSIX Thread](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)

|특성|API|
|:-:|:-:|
|thread|[pthread_create](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_create.html)、[pthread_detach](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_detach.html#)、[pthread_join](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_join.html#)|
|mutex|[pthread_mutex_lock、pthread_mutex_unlock](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_mutex_lock.html)|
|condition variable|[pthread_cond_wait](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_wait.html)、[pthread_cond_signal](https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_signal.html)|
|atomic|없다|
|future|없다|
|interruption|[pthread_cancel](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cancel.html)|

### [Java Thread](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html)

|특성|API|
|:-:|:-:|
|thread|[java.lang.Thread](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html)|
|mutex|[synchronized blocks](http://tutorials.jenkov.com/java-concurrency/synchronized.html)|
|condition variable|[java.lang.Object.wait](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Object.html#wait())、[java.lang.Object.notify](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Object.html#notify())|
|atomic|volatile 변형、[java.util.concurrent.atomic](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)|
|future|[java.util.concurrent.Future](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/Future.html)|
|interruption|[java.lang.Thread.interrupt](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html#interrupt())|
|스레드 안전 컨테이너|[java.util.concurrent](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/package-summary.html) 컨테이너|
|스레드 풀|[java.util.concurrent.ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)|
