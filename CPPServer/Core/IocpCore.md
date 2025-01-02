# IocpCore

```c++
class IocpCore
{
public:
	IocpCore();
	~IocpCore();

	HANDLE			GetHandle() { return _iocpHandle; }

	bool			Register(IocpObjectRef iocpObject);
	bool			Dispatch(uint32 timeoutMs = INFINITE);

private:
	HANDLE			_iocpHandle;
};
```
IocpCore란 Iocp를 GetQueuedCompletionStatus를 실행해주는 곳으로 IO Completion Queue에서 작업거리를 꺼내 IO 완료에 따른 처리를 수행해줍니다.

```c++
IocpCore::IocpCore()
{
	_iocpHandle = ::CreateIoCompletionPort(INVALID_HANDLE_VALUE, 0, 0, 0);
	ASSERT_CRASH(_iocpHandle != INVALID_HANDLE_VALUE);
}

IocpCore::~IocpCore()
{
	::CloseHandle(_iocpHandle);
}

bool IocpCore::Register(IocpObjectRef iocpObject)
{
	return ::CreateIoCompletionPort(iocpObject->GetHandle(), _iocpHandle, /*key*/ 0, 0);
}

bool IocpCore::Dispatch(uint32 timeoutMs)
{
	DWORD numOfBytes = 0;
	ULONG_PTR key = 0;
	IocpObject* iocpObject = nullptr;
	IocpEvent* iocpEvent = nullptr;

	if (::GetQueuedCompletionStatus(_iocpHandle, OUT & numOfBytes, OUT &key, OUT reinterpret_cast<LPOVERLAPPED*>(&iocpEvent), timeoutMs))
	{
		IocpObjectRef iocpObject = iocpEvent->owner;
		iocpObject->Dispatch(iocpEvent, numOfBytes);
	}
	else
	{
		int32 errorCode = ::WSAGetLastError();
		switch (errorCode)
		{
		case WAIT_TIMEOUT:
			return false;
		default:
			// TODO : 로그 찍기
			IocpObjectRef iocpObject = iocpEvent->owner;
			iocpObject->Dispatch(iocpEvent, numOfBytes);
			break;
		}
	}
	return true;
}
```
- Dispatch에서 해당 작업을 실행하며 [IocpEvent의 Owner(IocpObject)](https://github.com/sonsungmin98/Portfolio/blob/main/CPPServer/Core/Session-Service.md)의 Dispatch를 통해 알맞은 작업을 실행합니다.  
- 만약 TimeOut이라면 해당 쓰레드는 다른 [GlobalJObQueue](https://github.com/sonsungmin98/Portfolio/blob/main/CPPServer/Core/Job.md)의 일감을 처리해줍니다.(쓰레드의 일감을 적절히 분배를 위해)
