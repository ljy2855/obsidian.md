
async io는 python 내에서 thread 기반 병렬 처리를 구현하는 라이브러리

파이썬은 기본적으로 single proccess, single thread이기 때문에 multi thread 작업을 위해서는 async io 사용


```python
import asyncio

loop = asyncio.get_event_loop()
result = await loop.run_in_executor(None,fn,arg)
```



### GPU job은 async io로 보아야 하는가?