## IocpEvent
```c++
class IocpEvent : public OVERLAPPED
{
public:
	IocpEvent(EventType type);
	void Init();
public:
	EventType eventType;
	IocpObjectRef	owner;
};
```
IocpEvent는 Overlapped모델을 상속받아 사용합니다. 상속을 통해 넘겨주기 때문에 IO Completion Queue에 넘겨줄 떄 알맞은 IocpEvent를 넘겨줍니다. 따라서 GetQueuedCompletionStatus를 통해 Overlapped모델을 받을 때 넘겨준 IocpEvent를 받을 수 있습니다. 물론 reinterpret_cast로 캐스팅 하기 때문에 IocpEvent는 virtual 함수를 가지면 안됩니다. IocpEvent를는 IocpObject를 owner로 가집니다.
## IocpObject
```c++
class IocpObject : public enable_shared_from_this<IocpObject>
{
public:
	virtual HANDLE GetHandle() abstract;
	virtual void Dispatch(class IocpEvent* iocpEvent, int32 numOfBytes = 0) abstract;
};
```
먼저 최상위 클래스로 IocpObject를 만들었습니다. GetHandle과 Dispatch 인터페이스를 제공합니다. IocpEvent에서 owner를 통해 Dispatch를 실행하여 재정의 한 원하는 동작을 하게 해줍니다.
## Session
클라이언트와 서버, 서버와 서버 간의 통신 등 여러 상황이 생길 수 있습니다. 현재 연결되어있는 서버 or 클라이언트를 세션이라는 단위로 묶어서 관리를 했습니다.
```c++
class Session : public IocpObject
{
	friend class Listener;
	friend class IocpCore;
	friend class Service;

	enum 
	{
		BUFFER_SIZE = 0x10000, // 64KB
	};
public:
	Session();
	virtual ~Session();

public:
	void Send(SendBufferRef sendBuffer);
	bool Connect();
	void Disconnect(const WCHAR* cause);

	shared_ptr<Service>	GetService() { return _service.lock(); }
	void SetService(shared_ptr<Service> service) { _service = service; }

public:
	void SetNetAddress(NetAddress address) { _netAddress = address; }
	NetAddress GetAddress() { return _netAddress; }
	SOCKET GetSocket() { return _socket; }
	bool IsConnected() { return _connected; }
	SessionRef GetSessionRef() { return static_pointer_cast<Session>(shared_from_this()); }

private:
	virtual HANDLE GetHandle() override;
	virtual void Dispatch(class IocpEvent* iocpEvent, int32 numOfBytes = 0) override;

private:
	bool RegisterConnect();
	bool RegisterDisconnect();
	void RegisterRecv();
	void RegisterSend();

	void ProcessConnect();
	void ProcessDisconnect();
	void ProcessRecv(int32 numOfBytes);
	void ProcessSend(int32 numOfBytes);

	void HandleError(int32 errorCode);

protected:
	virtual void OnConnected() { }
	virtual int32 OnRecv(BYTE* buffer, int32 len) { return len; }
	virtual void OnSend(int32 len) { }
	virtual void OnDisconnected() { }

private:
	weak_ptr<Service>_service;
	SOCKET _socket = INVALID_SOCKET;
	NetAddress _netAddress = {};
	Atomic<bool> _connected = false;

private:
	USE_LOCK;

	RecvBuffer _recvBuffer;
	Queue<SendBufferRef> _sendQueue;
	Atomic<bool> _sendRegistered = false;

private:
	ConnectEvent _connectEvent;
	DisconnectEvent _disconnectEvent;
	RecvEvent _recvEvent;
	SendEvent _sendEvent;
};
```
Session은 IocpObjet를 상속 받고 다음과 같은 인터페이스를 제공합니다.

```c++
void Session::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)
{
	switch (iocpEvent->eventType)
	{
	case EventType::Connect:
		ProcessConnect();
		break;
	case EventType::Disconnect:
		ProcessDisconnect();
		break;
	case EventType::Recv:
		ProcessRecv(numOfBytes);
		break;
	case EventType::Send:
		ProcessSend(numOfBytes);
		break;
	default:
		break;
	}
}
```
Dispath를 통해 해당 EventType에 따라 알맞은 함수를 실행하게 됩니다.

## Service
Service는 이러한 Session을 관리하기 위해 만들었습니다.

```c++
enum class ServiceType : uint8 
{
	Server,
	Client,
};

using SessionFactory = function<SessionRef(void)>;

class Service : public enable_shared_from_this<Service>
{
public:
	Service(ServiceType type, NetAddress address, IocpCoreRef core, SessionFactory factory, int32 maxSessionCount = 1);
	virtual ~Service();

	virtual bool Start() abstract;
	bool CanStart() { return _sessionFactory != nullptr; }

	virtual void CloseService();
	void SetSessionFactory(SessionFactory func) { _sessionFactory = func; }
	
	void Broadcast(SendBufferRef sendBuffer);
	SessionRef CreateSession();
	void AddSession(SessionRef session);
	void ReleaseSession(SessionRef session);
	int32 GetCurrentSessionCount() { return _sessionCount; }
	int32 GetMaxSessionCount() { return _maxSessionCount; }

public:
	ServiceType GetServiceType() { return _type; }
	NetAddress GetNetAddress() { return _netAddress; }
	IocpCoreRef& GetIocpCore() { return _iocpCore; }

protected:
	USE_LOCK;

	ServiceType _type;
	NetAddress _netAddress = {};
	IocpCoreRef _iocpCore;

	Set<SessionRef> _sessions;
	int32 _sessionCount = 0;
	int32 _maxSessionCount = 0;
	SessionFactory _sessionFactory;
};
```

현재 service가 관리하는 Session의 관리를 위한 인테페이스로 구성되어 있습니다.
```c++
ServerServiceRef service = MakeShared<ServerService>(
  NetAddress(L"127.0.0.1", 7777), // 임시
  MakeShared<IocpCore>(),
  MakeShared<GameSession>,
  100);
```
다음과 같이 NetAddress, IocpCore, Session의 Factory Functor를 넘겨줘 네트워크에 연결 되는 기능들을 wrapping해 사용자가 사용에 편리하게 제작했습니다.
