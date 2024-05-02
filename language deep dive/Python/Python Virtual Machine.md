---
aliases: 
tags:
  - Python
---

### Interpreter

파이썬은 [[Interpreter]]를 통해 실행된다. 

```bash
❯ python
Python 3.9.13 | packaged by conda-forge | (main, May 27 2022, 17:00:33)
[Clang 13.0.1 ] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

`python`을 실행하면 interactive shell이 실행되는데, 한줄 씩 코드를 입력하고 독립적으로 코드를 실행하고 결과를 출력한다.

C, Java 기반의 컴파일 언어보다 유연하게 코드 작성 및 실행이 가능하지만, 성능적인 부분에서 trade off가 발생한다.


> [!NOTE] 왜 C, Java보다 성능이 떨어질까?
>**Overhead 발생**
> 파이썬은  line마다 byte code로 컴파일되어, runtime(PVM)에서 실행된다. 바로 binary file을 실행하는 imperative Language보다 한단계를 더 거치에 여기서 overhead가 발생한다.
> 
>**하드웨어 최적화의 제한**
> CPU가 이해할 수 있는 instruction으로 컴파일되는 `C`와 `Java`는 `pipeling`, `branch optimization`등 하드웨어 내의 최적화가 가능하지만, `Python`은 PVM내에서 Complier 단계의 최적화만 가능하다.
> 
>**컴파일 최적화**
> Java의 경우, 자바 가상 머신(JVM)은 Just-In-Time(JIT) 컴파일러를 통해 실행 시점에 바이트 코드를 기계어로 컴파일하고 최적화한다. 이 방법은 초기 실행 속도는 느릴 수 있지만, 장기적으로는 실행 속도를 크게 향상 가능하다. Python도 PyPy와 같은 대안적인 구현체에서 JIT 컴파일을 지원하여 성능을 개선하고 있으나, 기본적인 CPython 구현체는 이러한 기능을 제공하지 않는다.


### Script Run

```python
i = 3
print(i)
```

다음과 같은 Script를 실행시킬 때, 아래의 step대로 코드를 실행한다.

1. **parsing, syntax analysis** : code string parsing, syntax analysis 진행 
2. **byte code complier** : `PVM`이 실행가능한 형태의 byte code(IR)로 컴파일을 진행한다. 주로 pyc 확장자 파일로 생성되며 `__pycache__` 폴더에 저장된다. 만약 같은 script를 실행시킬 때, 기존의 컴파일된 byte code를 실행시키도록 cache한다.
3. **execution** : 바이트 코드는 파이썬 가상 머신(`PVM`)에 의해 실행된다. `PVM`은 스택 기반의 인터프리터로, 바이트 코드를 한 줄씩 읽어 실행하며 필요한 연산을 수행된다. 여기서 실제 프로그램의 로직이 수행된다.

### PVM (Python Virtual Machine)


