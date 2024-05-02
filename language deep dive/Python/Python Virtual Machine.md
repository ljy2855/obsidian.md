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

### Script Run

```python
i = 3
print(i)
```

다음과 같은 Script를 실행시킬 때, 아래의 step대로 코드를 실행한다.

1. **parsing, syntax analysis** : code string parsing, syntax analysis 진행 
2. **byte code complier** : `PVM`이 실행가능한 형태의 byte code(IR)로 컴파일을 진행한다. 주로 pyc 확장자 파일로 생성되며 `__pycache__` 폴더에 저장된다. 만약 같은 script를 실행시킬 때, 기존의 컴파일된 byte code를 실행시키도록 cache시킨다.
3. **execution** : 바이트 코드는 파이썬 가상 머신(`PVM`)에 의해 실행된다. `PVM`은 스택 기반의 인터프리터로, 바이트 코드를 한 줄씩 읽어 실행하며 필요한 연산을 수행된다. 여기서 실제 프로그램의 로직이 수행된다.

### PVM (Python Virtual Machine)


