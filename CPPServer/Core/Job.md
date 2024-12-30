# Job
서버에서는 패킷 처리 이외에도 몬스터의 AI나 NPC 등 여러가지 일을 처리 해야합니다. 또한 여러 쓰레드의 동등한 일감 처리를 위해 Job(Task)를 만들어 넘겨주고 이 Job을 각 쓰레드들이 처리를하게 하여 쓰레드 간의 동등한 일처리를 가능하게 만들었습니다.
  
## Job class
```c++
using CallbackType = std::function<void()>;

class Job
{
public:
	Job(CallbackType&& callback) : _callback(std::move(callback))
	{

	}

	template<typename T, typename Ret, typename... Args>
	Job(shared_ptr<T> owner, Ret(T::* memFunc)(Args...), Args&&... args)
	{
		_callback = [owner, memFunc, args...]()
		{
			(owner.get()->*memFunc)(args...);
		};
	}

	void Execute()
	{
		_callback();
	}

private:
	CallbackType _callback;
};
```
Job은 다음과 같이 functor와 같은 역활을 합니다. 람다 캡쳐를 통해 인자로 넘겨준 함수와 인자들을 저장해 클래스를 늘리지 않아도 사용이 가능하게 제작했습니다.

## JOb Queue

```c++
class JobQueue : public enable_shared_from_this<JobQueue>
{
public:
	void DoAsync(CallbackType&& callback)
	{
		Push(ObjectPool<Job>::MakeShared(std::move(callback)));
	}

	template<typename T, typename Ret, typename... Args>
	void DoAsync(Ret(T::*memFunc)(Args...), Args... args)
	{
		shared_ptr<T> owner = static_pointer_cast<T>(shared_from_this());
		Push(ObjectPool<Job>::MakeShared(owner, memFunc, std::forward<Args>(args)...));
	}

	void DoTimer(uint64 tickAfter, CallbackType&& callback)
	{
		JobRef job = ObjectPool<Job>::MakeShared(std::move(callback));
		GJobTimer->Reserve(tickAfter, shared_from_this(), job);
	}

	template<typename T, typename Ret, typename... Args>
	void DoTimer(uint64 tickAfter, Ret(T::* memFunc)(Args...), Args... args)
	{
		shared_ptr<T> owner = static_pointer_cast<T>(shared_from_this());
		JobRef job = ObjectPool<Job>::MakeShared(owner, memFunc, std::forward<Args>(args)...);
		GJobTimer->Reserve(tickAfter, shared_from_this(), job);
	}

	void				ClearJobs() { _jobs.Clear(); }

public:
	void				Push(JobRef job, bool pushOnly = false);
	void				Execute();

protected:
	LockQueue<JobRef>	_jobs;
	Atomic<int32>		_jobCount = 0;
};

void JobQueue::Push(JobRef job, bool pushOnly)
{
	const int32 prevCount = _jobCount.fetch_add(1);
	_jobs.Push(job); // WRITE_LOCK

	if (prevCount == 0)
	{
		if (LCurrentJobQueue == nullptr && pushOnly == false)
		{
			Execute();
		}
		else
		{
			GGlobalQueue->Push(shared_from_this());
		}
	}
}
```
Job들을 보관하고 처리를 담당하는 단위를 필요하다 판단해 제작했고 이것을 JobQueue라고 했습니다. Job을 추가하고 처리하는 인터페이스를 제공합니다.  
제일 처음 Push를 하는 쓰레드가 Job을 처리하게 됩니다. 하지만 이렇게 제작하면 최악의 경우 한 쓰레드만 일을 할 수도 있습니다. 따라서 현재 처리 중인 JobQueue가 있다면 GlobalJObQueue에 추가하여 글로벌로 선언된 JobQueue에서 처리하게 해줬습니다.  
또한 일정 시간 뒤에 처리해야하는 Job이 있을 수도 있으므로 이를 위해 JobTimer를 만들었습니다.



