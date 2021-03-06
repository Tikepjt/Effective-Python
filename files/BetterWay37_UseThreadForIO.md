# Better Way 37. 스레드를 블로킹 I/O 용으로 사용하고
# 병렬화용으로는 사용하지 말자.

#### 178쪽.
#### 2017/01/24 작성

## Part 1.
파이썬의 표준 구현을 **CPython**이라고 한다. CPython은 파이썬 프로그램을 두 단계로 실행한다.

1. 소스 텍스트를 바이트코드(bytecode)로 파생하고 컴파일한다.  
2. 스택 기반 인터프리터(PVM, Python Virtual Machine)으로 바이트코드를 실행한다.  

바이트코드 인터프리터는 파이썬 프로그램이 실행되는 동안 지속되고, 일관성 있는 상태를 유지한다.  
파이썬은 그 무시무시한 **GIL**(Global Interpreter Lock)으로 일관성을 유지한다.  

본질적으로 GIL은 상호 배제 잠금(mutax)이며 CPython이 선점형 멀티스레딩의 영향을 받지 않게 막는다.  
선점형 멀티스레딩(preemptive multithreading)은 한 스레드가 다른 스레드를 인터럽트해서  
프로그램의 제어를 얻는 것을 말한다.  

GIL은 이런 인터럽트를 막아주며 모든 바이트코드가 올바르게 작동함을 보장한다.  
쉽게 말해 파이썬은 스레드가 본질적으로 하나만 작동하기 때문에 다른 언어처럼 병렬화의 용도로 쓰기 약하다.  

GIL은 중요한 부작용을 갖고 있는데 자바 같은 언어로 작성한 프로그램에서 여러 스레드를 실행하는 건  
프로그램이 동시에 여러 CPU 코어를 사용하는 것을 의미한다.  

파이썬도 멀티스레드를 지원하지만 **GIL은 한 번에 한 스레드만 실행한다.**  
다시 말해 스레드가 병렬 연산을 해야 하거나 파이썬 프로그램의 속도를 높여야 하는 상황이라면 실망하게 된다.

```python
from time import time

def factorize(number):
    for i in range(1, number + 1):
        if number % i == 0:
            yield i
            
numbers = [1921931, 144872, 384539, 345983]
start = time()
for number in numbers:
    list(factorize(number))
end = time()

print('It took', end - start)

>>> It took 4.128361463546753
```

예를 살펴보자. 소인수 분해를 여러 숫자에 사용했다. 4초가 걸린다.  
다른 언어에서는 당연히 이런 연산에 멀티스레드를 이용한다.  
멀티스레드를 이용하면 컴퓨터의 모든 CPU를 최대한 활용할 수 있기 때문이다.  
이 작업을 파이썬으로 해보자. 같은 연산을 스레드를 사용한다.  

```python
from threading import Thread

class FactorizeThread(Thread):
    def __init__(self, number):
        super().__init__()  # 스레드의 기본을 초기화
        self.number = number
        
    def run(self):  # thread의 start 메소드를 사용하면 run이 trigger된다.
        self.factors = list(factorize(self.number))
        
start = time()
threads = []
for number in numbers:
    thread = FactorizeThread(number)
    thread.start()
    threads.append(thread)
    
for thread in threads:
    thread.join()  # 모든 스레드가 끝나기를 기다린다. 그 다음 end를 계산한다.
end = time()

print('It took', end - start)    


>>> It took 4.197026014328003
```

황당하게도 시간이 그냥 할 때보다 더 걸렸다.  
숫자별로 스레드 하나를 사용하면 스레드를 생성하고 실행 순서를 조율하는 부담을 감안할 때  
4배 미만의 속도 향상을 기대했을 것이다. 이 코드를 듀얼코어머신에서 실행하면 2배 정도의 속도를 기대했을 것이다.  
이로써 GIL이 표준 CPython 인터프리터에서 실행하는 프로그램에 미치는 영향을 알 수 있다.  

CPython이 멀티코어를 활용하게 하는 방법은 여러 가지지만, 표준 Thread에는 작동하지 않기 때문에  
노력이 필요하다. _multiprocess_ 모듈, _concurrent.futures_ 등.  

<br>
<br>


## Part 2.

그러면 애초에 파이썬 스레드는 존재 자체가 무슨 의미일까? 여기에는 두 가지 이유가 있을 수 있다.

1. 파이썬 멀티스레드를 사용하면 동시에 여러 작업을 하는 것처럼 보이게 할 수 있다.
  > 동시에 동작하는 태스크를 관리하는 코드를 직접 구현하는 것은 어렵다.
  > 스레드를 이용하면 함수를 마치 병렬로 시행하는 것처럼 할 수 있다.
  > 비록 한 번에 한 스레드만 진행하지만, CPython은 스레드가 어느 정도 공평하게 실행됨을 보장한다.  

2. **특정 유형의 시스템 호출(System Call)할 때 일어나는 블로킹 I/O를 다루기 위해서다.**
  > **시스템 콜**이란 **파이썬 프로그램이 외부 환경과 상호 작용하도록 운영체제 커널에 요청하는 것**을 의미한다.
  > 블로킹 I/O로는 파일 읽기/쓰기, 네트워크와의 상호작용, 디스플레이 같은 장치와의 통신이 있다.
  > 즉 파이썬 _open_으로 파일을 여는 것은 바로 `커널을 통해 파일을 여는 것이다.`
  > 스레드는 운영체제가 이런 요청에 응답하는 데 드는 시간을 프로그램과 분리하므로 블로킹 I/O 처리에 유용하다.

  
다음 예가 이상하기는 한데..  

원격 제어가 가능한 헬리콥터에 직렬포트로 신호를 보내고 싶다고 하자.  
이번 예제는 느림 시스템 호출을 담당하는 `select` 모듈을 사용할 것이다. async I/O를 담당한다.  

이 함수는 동기식 직렬 포트를 사용할 때 일어나는 상황과 비슷하게 하려고 운영체제에 0.1초간 블록한 후,  
제어를 프로그램에 돌려달라고 요청한다.


```python
import select

def slow_systemcall():
    select.select([], [], [], 0.1)   # 자세한 내용은 select 모듈을 찾아보기 바란다.
    
# 이 시스템 호출을 연속해서 실행하면 시간이 선형으로 증가한다.
start = time()
for _ in range(5):
    slow_systemcall()
end = time()

print('It took', end - start)


>>> It took 0.5013508796691895
```

0.1초의 통신을 5번 해서 0.5초가 걸린 것을 알 수 있다.  

이 방법의 문제는 *slow_systemcall* 함수가 실행되는 동안 프로그램이 다른 일을 할 수 없다는 점이다.  
프로그램의 메인 스레드는 시스템 호출 _select_ 때문에 실행이 막혀 있다.  
신호를 헬리콥터에 보내는 동안 헬리콥터의 다른 이동을 계산해야 한다.  
그렇지 않으면 헬리콥터가 충돌할 것이다. 블로킹 I/O를 사용하며 동시에 연산도 해야 한다면  
시스템 호출을 스레드로 옮기는 방법을 고려해야 한다.  

다음 코드는 *slow_systemcall* 함수를 별도의 스레드에서 여러 번 호출하여 실행한다.  
이렇게 하면 동시에 여러 직렬 포트(및 헬리콥터)와 통신할 수 있게 되고,  
메인 스레드는 필요한 계산이 무엇이든 수행하도록 남겨둘 수 있다.

```python
start = time()
threads = []
for _ in range(5):
    thread = Thread(target=slow_systemcall)
    thread.start()
    threads.append(thread)
    
for thread in threads:
    thread.join()
end = time()

print('It took', end - start)

>>> It took 0.10623431205749512
```

만약 지금까지의 내용을 이해했다면 나를 개새끼라고 불러야 한다.  
스레드는 한 번에 하나의 스레드만을 사용한다고 했기 때문에 아까처럼 0.5초가 나왔어야 하지 않는가!  
즉 **이번에는 스레드의 병렬처리가 성공했다.**  

마지막이다. 조금만 더.  
정리하면, GIL은 파이썬 코드가 병렬로 실행하지 못하도록 한다.  
**하지만 시스템 호출에서는 이런 부정적인 영향이 없다.**  
**이는 파이썬 스레드가 시스템 호출을 만들기 전에 GIL을 풀고 시스템 호출의 작업이 끝나는 대로 GIL을 다시 얻기 때문.**  

스레드 이외에도 내장 모듈 asyncio처럼 블로킹 I/O를 다루는 다양한 수단이 있고,  
이런 대체 수단에는 중요한 이점이 있다.  
하지만 이런 옵션을 선택하면 실행 모델에 맞춰 코드를 재작성해야 하는 추가 작업이 필요하다.  

스레드를 이용하는 방법은 프로그램의 수정을 최소화하면서도 블로킹 I/O를 병렬로 수행하는 가장 간단한 방법.


## 핵심정리
1. 파이썬 스레드는 GIL 때문에 여러 CPU 코어에서 병렬로 바이트코드를 실행할 수 없다.
2. GIL에도 불구하고 파이썬 스레드는 동시에 여러 작업을 하는 것처럼 보여주기 쉽게 해주므로 여전히 유용하다.
3. 여러 시스템 호출을 병렬로 수행하려면 파이썬 스레드를 사용하자. 블로킹 I/O를 수행할 수 있다.



