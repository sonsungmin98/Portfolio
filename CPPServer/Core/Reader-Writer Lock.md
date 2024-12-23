# Reader-Writer Lock

기존 표준 뮤텍스로 작업을 하면 다음과 같은 아쉬운 점이 있기 때문에 ReaderLock 과 WriterLock을 제작했습니다.
 1. 재귀적으로 lock을 잡고 싶은 경우가 있습니다.
 2. 아주 특수한 상황에서만 상호 배타적인 동작을 필요로 하는경우가 있습니다.

단순 읽기(Read)의 경우에는 일관성을 해치지 않기 때문에 동시에 같이 Read를 해도 동기화 문제가 발생하지 않습니다. 따라서 Read끼리는 lock을 하지 않는 다면 자원을 조금 더 효율적으로 사용할 수 있게 됩니다.

## Lock
```c++
class Lock
{
	enum : uint32
	{
		ACQUIRE_TIMEOUT_TICK = 10000,
		MAX_SPIN_COUNT = 5000,
		WRITE_THREAD_MASK = 0xFFFF'0000,
		READ_COUNT_MASK = 0x0000'FFFF,
		EMPTY_FLAG = 0x0000'0000,
	};

public:
	void WriteLock(const char* name);
	void WriteUnlock(const char* name);
	void ReadLock(const char* name);
	void ReadUnlock(const char* name);

private:
	Atomic<uint32>	_lockFlag		= EMPTY_FLAG;
	uint16			_writeCount		= 0;
};
```
enum의 경우에는 위에서 부터

| 열거자  | 설명 |
| ------------- | ------------- |
| ACQUIRE_TIMEOUT_TICK  | Lock을 잡기 위한 최대 대기 시간 |
| MAX_SPIN_COUNT  | 최대 경합 횟수 (Spin Lock의 반복 횟수)  |
| WRITE_THREAD_MASK  | Write의 정보를 담고 있는 FLAG의 값을 얻기 위한 MASK |
| READ_COUNT_MASK  | READ의 정보를 담고 있는 FLAG의 값을 얻기 위한 MASK |
| EMPTY_FLAG  | 비교를 위한 빈 FLAG |

## Read Lock

```c++
void Lock::ReadLock(const char* name)
{
#if _DEBUG
	GDeadLockProfiler->PushLock(name);
#endif

	const uint32 lockThreadId = (_lockFlag.load() & WRITE_THREAD_MASK) >> 16;
	if (LThreadId == lockThreadId)
	{
		_lockFlag.fetch_add(1);
		return;
	}
	
	const int64 beginTick = ::GetTickCount64();
	while(true)
	{
		for (uint32 spinCount = 0; spinCount < MAX_SPIN_COUNT; spinCount++)
		{
			uint32 expected = (_lockFlag.load() & READ_COUNT_MASK);
			if (_lockFlag.compare_exchange_strong(OUT expected, expected + 1))
			{
				return;
			}
		}

		if (::GetTickCount64() - beginTick >= ACQUIRE_TIMEOUT_TICK)
		{
			CRASH("LOCK_TIMEOUT");
		}

		this_thread::yield();
	}
}
```
Read Lock의 경우에는 Write Lock과 동일한 쓰레드인지 확인하고 동일한 쓰레드면 바로 성공을 시킵니다.  
  
만약 아무도 소유하고 있지 않거나 다른 쓰레드가 점유하고 있다면 경합을 해서 소유권을 얻고 공유 카운트를 올립니다.

```c++
void Lock::ReadUnlock(const char* name)
{
#if _DEBUG
	GDeadLockProfiler->PopLock(name);
#endif

	if ((_lockFlag.fetch_sub(1) & READ_COUNT_MASK) == 0)
	{
		CRASH("MULTIPLE_UNLOCK");
	}
}
```
unlock은 간단하게 공유 카운트를 하나 줄입니다. 마지막으로 Read Count가 0이라면 CRASH를 내줍니다.

## Write Lock

```c++
void Lock::WriteLock(const char* name)
{
#if _DEBUG
	GDeadLockProfiler->PushLock(name);
#endif

	const uint32 lockThreadId = (_lockFlag.load() & WRITE_THREAD_MASK) >> 16;
	if (LThreadId == lockThreadId)
	{
		_writeCount++;
		return;
	}

	const int64 beginTick = ::GetTickCount64();
	const uint32 desired = ((LThreadId << 16) & WRITE_THREAD_MASK);
	while (true)
	{
		for (uint32 spinCount = 0; spinCount < MAX_SPIN_COUNT; spinCount++)
		{
			uint32 expected = EMPTY_FLAG;
			if (_lockFlag.compare_exchange_strong(OUT expected, desired))
			{
				_writeCount++;
				return;
			}
		}

		if (::GetTickCount64() - beginTick >= ACQUIRE_TIMEOUT_TICK)
		{
			CRASH("LOCK_TIMEOUT");
		}

		this_thread::yield();
	}
}
```

마찬가지로 동일 쓰레드가 Lock을 소유하고 있다면 바로 성공을 시키고 만약 소유하고 있지않다면 EMPTY_FLAG가 될때까지 경합합니다. (아무도 소유하고 있지 않을 때만 접근해야하기 때문에)

```c++
void Lock::WriteUnlock(const char* name)
{
#if _DEBUG
	GDeadLockProfiler->PopLock(name);
#endif

	if ((_lockFlag.load() & READ_COUNT_MASK) != 0)
	{
		CRASH("INVALID_UNLOCK_ORDERED");
	}

	const int32 lockCount = --_writeCount;
	if (lockCount == 0)
	{
		_lockFlag.store(EMPTY_FLAG);
	}
}
```
unlock의 경우에는 WriteLock count가 0이 되면 FLAG를 EMPTY_FLAG로 돌려놓습니다. (어짜피 동일 쓰레드에서 일어나기 때문에 Atomic하게 제작하지 않았습니다 -> --_writeCount;)
