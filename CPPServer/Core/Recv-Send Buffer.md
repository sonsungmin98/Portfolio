# Recv Buffer
Recv Buffer의 경우에는 다음과 같은 인터페이스를 제공합니다. Recv Buffer의 경우에는 Send Buffer와 달리 멀티쓰레드 환경을 고려하지 않아도 됩니다. 
```c++
class RecvBuffer
{
	enum { BUFFER_COUNT = 10, };
public:
	RecvBuffer(int32 bufferSize);
	~RecvBuffer();

	void	      Clean();
	bool	      OnRead(int32 numOfBytes);
	bool	      OnWrite(int32 numOfBytes);

	BYTE*	      ReadPos() { return &_buffer[_readPos]; }
	BYTE*	      WritePos() { return &_buffer[_writePos]; }
	int32	      DataSize() { return _writePos - _readPos; }
	int32	      FreeSize() { return _capacity - _writePos; }

private:
	int32	      _capacity = 0;
	int32	      _bufferSize = 0;
	int32	      _readPos = 0;
	int32	      _writePos = 0;
	Vector<BYTE>    _buffer;
};
```
TCP의 경우에는 Boundary가 없기 때문에 Write와 Read의 Pos를 변수로 두어 현재 읽고 쓴 곳이 어디까지인지 표시해줍니다.  

# Send Buffer
Send의 경우에 같은 데이터를 여러명에게 보내줘야하는 일이 비일비재 할 것입니다. 따라서 데이터의 복사 비용등을 절약하기 위해 SendBuffer를 만들었습니다.
```c++
class SendBufferChunk;

class SendBuffer
{
public:
	SendBuffer(SendBufferChunkRef owner, BYTE* buffer, uint32 allocSize);
	~SendBuffer();

	BYTE* Buffer() { return _buffer; }
	uint32 AllocSize() { return _allocSize; }
	uint32 WriteSize() { return _writeSize; }
	void Close(uint32 writeSize);

private:
	BYTE* _buffer;
	uint32 _allocSize = 0;
	uint32 _writeSize = 0;
	SendBufferChunkRef _owner;
};

class SendBufferChunk : public enable_shared_from_this<SendBufferChunk>
{
	enum 
	{
		SEND_BUFFER_CHUNK_SIZE = 6000,
	};

public:
	SendBufferChunk();
	~SendBufferChunk();

	void Reset();
	SendBufferRef Open(uint32 allocSize);
	void Close(uint32 writeSize);

	bool IsOpen() { return _open; }
	BYTE* Buffer() { return &_buffer[_usedSize]; }
	uint32 FreeSize() { return static_cast<uint32>(_buffer.size()) - _usedSize; }

private:
	Array<BYTE, SEND_BUFFER_CHUNK_SIZE> _buffer = {};
	bool _open = false;
	uint32 _usedSize = 0;
};

class SendBufferManager
{
public:
	SendBufferRef Open(uint32 size);

private:
	SendBufferChunkRef Pop();
	void Push(SendBufferChunkRef buffer);

	static void PushGlobal(SendBufferChunk* buffer);

private:
	USE_LOCK;
	Vector<SendBufferChunkRef> _sendBufferChunks;
};
```

```c++
SendBuffer::SendBuffer(SendBufferChunkRef owner, BYTE* buffer, uint32 allocSize)
	:_owner(owner), _buffer(buffer), _allocSize(allocSize)
{
}

SendBuffer::~SendBuffer()
{
}

void SendBuffer::Close(uint32 writeSize)
{
	ASSERT_CRASH(_allocSize >= writeSize);
	_writeSize = writeSize;
	_owner->Close(writeSize);
}

SendBufferChunk::SendBufferChunk()
{
}

SendBufferChunk::~SendBufferChunk()
{
}

void SendBufferChunk::Reset()
{
	_open = false;
	_usedSize = 0;
}

SendBufferRef SendBufferChunk::Open(uint32 allocSize)
{
	ASSERT_CRASH(allocSize <= SEND_BUFFER_CHUNK_SIZE);
	ASSERT_CRASH(_open == false);

	if (allocSize > FreeSize())
		return nullptr;
	
	_open = true;
	return ObjectPool<SendBuffer>::MakeShared(shared_from_this(), Buffer(), allocSize);
}

void SendBufferChunk::Close(uint32 writeSize)
{
	ASSERT_CRASH(_open == true);
	_open = false;
	_usedSize += writeSize;
}

SendBufferRef SendBufferManager::Open(uint32 size)
{
	if (LSendBufferChunk == nullptr)
	{
		LSendBufferChunk = Pop();
		LSendBufferChunk->Reset();

	}

	ASSERT_CRASH(LSendBufferChunk->IsOpen() == false);
	if (LSendBufferChunk->FreeSize() < size)
	{
		LSendBufferChunk = Pop();
		LSendBufferChunk->Reset();
	}

	return LSendBufferChunk->Open(size);
}

SendBufferChunkRef SendBufferManager::Pop()
{
	{
		WRITE_LOCK;
		if (_sendBufferChunks.empty() == false)
		{
			SendBufferChunkRef sendBufferChunk = _sendBufferChunks.back();
			_sendBufferChunks.pop_back();
			return sendBufferChunk;
		}
	}

	return SendBufferChunkRef(xnew<SendBufferChunk>(), PushGlobal);
}

void SendBufferManager::Push(SendBufferChunkRef buffer)
{
	WRITE_LOCK;
	_sendBufferChunks.push_back(buffer);
}

void SendBufferManager::PushGlobal(SendBufferChunk* buffer)
{
	GSendBufferManager->Push(SendBufferChunkRef(buffer, PushGlobal));
}
```
SendBuffer는 크게 SendBuffer, SendBufferChunk, SendBufferManager로 구성되어 있습니다.
- SendBuffer: SendBuffer를 나타내며, Buffer의 시작 주소, 패킷의 Size를 가지고 있습니다.
- SendBufferChunk: SendBuffer를 사용할 때마다 만드는 것이 아닌, 미리 큰 크기의 영역을 할당하여 거기서 차례차례 할당하는 역활, TLS영역에 생성하여 Lock없이 접근이 가능합니다.
- SendBufferManager: SendBufferChunk를 관리하기 위한 부분입니다. SendBufferChunk의 할당과 교체 등을 담당합니다.

