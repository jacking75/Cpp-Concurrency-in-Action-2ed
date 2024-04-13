fork 한 [저장소](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed )을 글을 번역하였다.  
   
* C++11은 표준 스레딩 라이브러리로 Boost 스레딩 라이브러리를 도입했고, 저자 Anthony Williams는 2012년에 *[C++ Concurrency in Action](https://book.douban.com/subject/4130141/)*을 출간하여 그 기능을 소개했습니다. C++17에 대응하여 2019년 2월에 [2판](https://book.douban.com/subject/27036085/)을 출간했습니다. *[C++ Concurrency in Action 2ed](https://learning.oreilly.com/library/view/c-concurrency-in/9781617294693/)* 첫 5장에서는 [스레드 지원 라이브러리](https://en)를 소개합니다. cppreference.com/w/cpp/thread), 마지막 여섯 장에서는 실용적인 관점에서 동시 프로그래밍의 설계 아이디어를 소개하고, [std::scoped_lock](https://en.cppreference.com/w/cpp/. thread/scoped_lock](https://en.cppreference.com/w/cpp/thread/shared_mutex), [std::shared_mutex](https://en.cppreference.com/w/cpp/thread/shared_mutex), [C++17 표준 라이브러리 병렬 알고리즘](https://)에 대한 추가 장(10장)을 포함합니다. en.cppreference.com/w/cpp/header/execution), [std::jthread](https://en.cppreference.com/w/cpp/thread/jthread)와 같은 C++20 관련 기능을 개인적으로 추가했습니다. , [std::counting_semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore), [std::barriers](https://en.cppreference.com)를 추가했습니다. /w/cpp/thread/barrier), [std::latch](https://en.cppreference.com/w/cpp/thread/latch) 등이 있습니다. 이 책을 읽기 전에 [*현대 운영체제*](https://book.douban.com/)에 대한 [Andrew S. Tanenbaum](https://en.wikipedia.org/wiki/Andrew_S._Tanenbaum)을 참고하세요. subject/25864553/)에서 운영체제의 기초([프로세스와 스레드](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/reference/ processes_and_threads.md), [교착 상태](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/reference/deadlocks.md), [메모리 관리](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/reference/memory_management.md), [파일 시스템](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/reference/file_systems.md), [I/O](https://github. com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/reference/IO.md) 등). 이는 참고용 개인 메모이며 자세한 내용은 [원서](https://learning.oreilly.com/library/view/c-concurrency-in/9781617294693/)를 참조하세요.  
  
  
## [스레딩 지원 라이브러리](https://en.cppreference.com/w/cpp/thread)
1. [스레드 관리](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/01_managing_thread.md): [\< thread\>](https://en.cppreference.com/w/cpp/header/thread)  
2. [스레드 간 데이터 공유](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/02_sharing_data_) between_thread.md): [\<mutex\>](https://en.cppreference.com/w/cpp/header/mutex), [\<shared_mutex\>](https://en.cppreference.com/w/) CPP/헤더/공유_뮤텍스)  
3. [동시 작업 동기화](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/03_ synchronising_concurrent_operation.md): [\<조건_변수\>](https://en.cppreference.com/w/cpp/header/condition_variable), [\< 세마포어\>](https://en.cppreference.com/w/cpp/header/semaphore), [\<barrier\>](https://en.cppreference.com/w/cpp/header/barrier), [ \<latch\>](https://en.cppreference.com/w/cpp/header/latch), [\<future\>](https://en.cppreference.com/w/cpp/header/future), [\<chrono \>](https://en.cppreference.com/w/cpp/header/chrono), [\<비율\>](https://en.cppreference.com/w/cpp/header/ratio)  
4. [C++ 메모리 모델과 원자형에 대한 연산](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/) master/docs/04_the_cpp_memory_model_and_operations_on_atomic_type.md): [\<atomic\>](https://en.cppreference.com/w/cpp/header/atomic )  
  
  
## 동시 프로그래밍 실습
  
5. [잠금 기반 동시 데이터 구조 설계하기] (https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/) master/docs/05_designing_lock_based_concurrent_data_structure.md)  
6. [잠금 없는 동시 데이터 구조 설계](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/ docs/06_designing_lock_free_concurrent_data_structure.md)  
7. [동시 코드 설계](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/07_designing_ concurrent_code.md)  
8. [고급 스레드 관리](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/08_advanced_thread _management.md)  
9. [병렬 알고리즘 (병렬 알고리즘)](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/master/docs/09_parallel_algorithm.md): [. \<실행\>](https://en.cppreference.com/w/cpp/header/execution)    
10. [멀티 스레드 애플리케이션 테스트 및 디버깅 (多线程应用的测试与调试)](https://github.com/downdemo/Cpp-Concurrency-in-Action-2ed/blob/ master/docs/10_testing_and_debugging_multithreaded_application.md)  
  
  
## 표준 라이브러리 관련 헤더 파일

|헤더|설명|
|:-:|:-:|
|[\<thread\>](https://en.cppreference.com/w/cpp/header/thread), [\<stop_token\>](https://en.cppreference.com/w/cpp/header/stop_ 토큰)|스레드|
|[\<mutex\>](https://en.cppreference.com/w/cpp/header/mutex), [\<shared_mutex\>](https://en.cppreference.com/w/cpp/header/shared_ mutex)|잠금|
|[\<조건_변수\>](https://en.cppreference.com/w/cpp/header/condition_variable)|조건_변수|
|[\<세마포어\>](https://en.cppreference.com/w/cpp/header/semaphore)|신호 수량|
|[\<barrier\>](https://en.cppreference.com/w/cpp/header/barrier), [\<latch\>](https://en.cppreference.com/w/cpp/header/latch)|barrier|
|[\<미래\>](https://en.cppreference.com/w/cpp/header/future)|비동기 처리 결과|
|[\<chrono\>](https://en.cppreference.com/w/cpp/header/chrono)|clock|
|[\<비율\>](https://en.cppreference.com/w/cpp/header/ratio)|컴파일 기간 유리수 산술|
|[\<원자\>](https://en.cppreference.com/w/cpp/header/atomic)|원자형과 원자 연산|
|[\<실행\>](https://en.cppreference.com/w/cpp/header/execution)|표준 라이브러리 알고리즘 실행 정책|  
    
  
## 동시성 라이브러리 비교

### [C++11 Thread](https://en.cppreference.com/w/cpp/thread)

|특성|API|
|:-:|:-:|
|thread|[std::thread](https://en.cppreference.com/w/cpp/thread/thread)|
|mutex|[std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)、[std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)、[std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock)|
|condition variable|[std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)、[std::condition_variable_any](https://en.cppreference.com/w/cpp/thread/condition_variable_any)|
|atomic|[std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)、[std::atomic_thread_fence](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence)|
|future|[std::future](https://en.cppreference.com/w/cpp/thread/future)、[std::shared_future](https://en.cppreference.com/w/cpp/thread/shared_future)|
|interruption|없음|

### [Boost Thread](https://www.boost.org/doc/libs/1_82_0/doc/html/thread.html)

|특성|API|
|:-:|:-:|
|thread|[boost::thread](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/thread_management.html#thread.thread_management.thread)|
|mutex|[boost::mutex](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.mutex_types.mutex)、[boost::lock_guard](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.lock_guard.lock_guard)、[boost::unique_lock](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.locks.unique_lock)|
|condition variable|[boost::condition_variable](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.condvar_ref.condition_variable)、[boost::condition_variable_any](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.condvar_ref.condition_variable_any)|
|atomic|없음|
|future|[boost::future](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.futures.reference.unique_future)、[boost::shared_future](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/synchronization.html#thread.synchronization.futures.reference.shared_future)|
|interruption|[thread::interrupt](https://www.boost.org/doc/libs/1_82_0/doc/html/thread/thread_management.html#thread.thread_management.thread.interrupt)|

### [POSIX Thread](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)

|특성|API|
|:-:|:-:|
|thread|[pthread_create](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_create.html)、[pthread_detach](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_detach.html#)、[pthread_join](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_join.html#)|
|mutex|[pthread_mutex_lock、pthread_mutex_unlock](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_mutex_lock.html)|
|condition variable|[pthread_cond_wait](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_wait.html)、[pthread_cond_signal](https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cond_signal.html)|
|atomic|없음|
|future|없음|
|interruption|[pthread_cancel](http://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_cancel.html)|

### [Java Thread](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html)

|특성|API|
|:-:|:-:|
|thread|[java.lang.Thread](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html)|
|mutex|[synchronized blocks](http://tutorials.jenkov.com/java-concurrency/synchronized.html)|
|condition variable|[java.lang.Object.wait](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Object.html#wait())、[java.lang.Object.notify](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Object.html#notify())|
|atomic|volatile、[java.util.concurrent.atomic](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)|
|future|[java.util.concurrent.Future](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/Future.html)|
|interruption|[java.lang.Thread.interrupt](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/Thread.html#interrupt())|
|스레드 안전 컨테이너|[java.util.concurrent](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/package-summary.html) 컨테이너|
|스레드 풀|[java.util.concurrent.ThreadPoolExecutor](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)|
