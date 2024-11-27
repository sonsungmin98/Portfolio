# MemoryPool

메모리의 할당과 해제에 따른 Context Switching 비용의 절감과, 메모리 단편화와 같은 문제를 해결하기 위해 Memory Pool을 제작했습니다. 풀에서 관리되는 객체의 경우 메모리 헤더를 부착해 관리를 위한 다양한 정보를 가지게 만들었습니다. 


## MemoryPool.h
```c++
#pragma once

enum 
{
	SLIST_ALIGMENT = 16
};

DECLSPEC_ALIGN(SLIST_ALIGMENT)
struct MemoryHeader : public SLIST_ENTRY
{
	MemoryHeader(int32 size) : allocSize(size) { }

	static void* AttachHeader(MemoryHeader* header, int32 size)
	{
		new(header)MemoryHeader(size);
		return reinterpret_cast<void*>(++header);
	}

	static MemoryHeader* DetachHeader(void* ptr)
	{
		MemoryHeader* header = reinterpret_cast<MemoryHeader*>(ptr) - 1;
		return header;
	}

	int32 allocSize;
	// TODO : 필요한 추가 정보
};

DECLSPEC_ALIGN(SLIST_ALIGMENT)
class MemoryPool
{
public:
	MemoryPool(int32 allocSize);
	~MemoryPool();

	void			Push(MemoryHeader* ptr);
	MemoryHeader*	Pop(void);

private:
	SLIST_HEADER	_header;
	int32			_allocSize = 0;
	atomic<int32>	_useCount = 0;
	atomic<int32>	_reserveCount = 0;
};
```
Memory Header에는 기본적으로 여러 정보들을 담게 되며 SLIST_ENTRY 구조체를 상속 받고 있습니다.
이 이유는 window에서 제공하는 InterlockedPushEntrySList등 lock free 함수를 사용하기 위함입니다.
  
> https://learn.microsoft.com/ko-kr/windows/win32/api/winnt/ns-winnt-slist_entry 에 따르면  
> 모든 목록 항목은 MEMORY_ALLOCATION_ALIGNMENT 경계에 맞춰야 합니다. 정렬되지 않은 항목으로 인해 예측할 수 없는 결과가 발생할 수 있습니다.

따라서 DECLSPEC_ALIGN(SLIST_ALIGMENT)를 사용하여 ALignment을 맞춰줬습니다.
  
## MemoryPool.cpp
```c++
#include "pch.h"
#include "MemoryPool.h"

MemoryPool::MemoryPool(int32 allocSize) : _allocSize(allocSize)
{
	::InitializeSListHead(&_header);
}

MemoryPool::~MemoryPool()
{
	while (MemoryHeader* memory = static_cast<MemoryHeader*>(::InterlockedPopEntrySList(&_header)))
	{
		::_aligned_free(memory);
	}
}

void MemoryPool::Push(MemoryHeader* ptr)
{
	ptr->allocSize = 0;

	::InterlockedPushEntrySList(&_header, static_cast<PSLIST_ENTRY>(ptr));

	_useCount.fetch_sub(1);
	_reserveCount.fetch_add(1);
}

MemoryHeader* MemoryPool::Pop(void)
{
	MemoryHeader* memory = static_cast<MemoryHeader*>(::InterlockedPopEntrySList(&_header));

	// 없으면 새로 만들기
	if (memory == nullptr)
	{
		memory = reinterpret_cast<MemoryHeader*>(::_aligned_malloc(_allocSize, SLIST_ALIGMENT));
	}
	else
	{
		ASSERT_CRASH(memory->allocSize == 0);
		_reserveCount.fetch_sub(1);
	}

	_useCount.fetch_add(1);

	return memory;
}
```  
위의 함수들이 Pooling에서 사용하는 기본적인 클래스 입니다.  

# Memory

## Memory.h
```c++
#pragma once
#include "Allocator.h"

class MemoryPool;

/*-------------
	Memory
-------------*/
class Memory
{
	enum
	{
		POOL_COUNT = (1024 / 32) + (1024 / 128) + (2048 / 256),
		MAX_ALLOC_SIZE = 4096,
	};

public:
	Memory();
	~Memory();

	void* Allocate(int32 size);
	void Release(void* ptr);

private:
	vector<MemoryPool*> _pools;

	//메모리 크기 <-> 메모리 풀
	// O(1) 빠르게 찾기 위한 테이블
	MemoryPool* _poolTable[MAX_ALLOC_SIZE + 1];
};
```

## Memory.cpp
```c++
#include "pch.h"
#include "Memory.h"
#include "MemoryPool.h"

Memory::Memory()
{
	int size = 0;
	int32 tableIndex = 0;

	for (size = 32; size < 1024; size += 32)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}

	for (; size < 2048; size += 128)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}

	for (; size <= 4096; size += 256)
	{
		MemoryPool* pool = new MemoryPool(size);
		_pools.push_back(pool);

		while (tableIndex <= size)
		{
			_poolTable[tableIndex] = pool;
			tableIndex++;
		}
	}
}

Memory::~Memory()
{
	for (MemoryPool* pool : _pools)
	{
		delete pool;
	}

	_pools.clear();
}

void* Memory::Allocate(int32 size)
{
	MemoryHeader* header = nullptr;
	const int32 allocSize = size + sizeof(MemoryHeader);

#ifdef _STOMP
	header = reinterpret_cast<MemoryHeader*>(StompAllocator::Alloc(allocSize));
#else
	if (allocSize > MAX_ALLOC_SIZE)
	{
		// 메모리 풀링 최대 크기를 벗어나면 일반 할당
		header = reinterpret_cast<MemoryHeader*>(::_aligned_malloc(allocSize, SLIST_ALIGMENT));
	}
	else
	{
		// 메모리 풀에서 꺼내온다.
		header = _poolTable[allocSize]->Pop();
	}
#endif
	

	return MemoryHeader::AttachHeader(header, allocSize);
}

void Memory::Release(void* ptr)
{
	MemoryHeader* header = MemoryHeader::DetachHeader(ptr);

	const int32 allocSize = header->allocSize;
	ASSERT_CRASH(allocSize > 0);

#ifdef _STOMP
	StompAllocator::Release(header);
#else
	if (allocSize > MAX_ALLOC_SIZE)
	{
		// 메모리 풀링 최대 크기를 벗어나면 일반 해제
		::_aligned_free(header);
	}
	else
	{
		// 메모리 풀에 반납
		_poolTable[allocSize]->Push(header);
	}
#endif
}
```

Memory에서는 각 memory size에 맞게 MemoryPool이 나눠지게 만들었고 각 size에 맞는 MemoryPool에서 메모리 풀링이 이루어지게 만들었습니다.  
 - 1024까지 32단위
 - 2048까지 128단위
 - 4096까지 256단위로 
  
> 만약 엄청나게 큰 size의 객체(4096)가 들어오면 일반 해제를 하게 만들었습니다.  
> 그정도로 큰 size의 데이터의 경우에는 Pooling에 따른 메리트가 없을 것 같다고 판단했기 때문입니다.


**new**
```c++
template<typename Type, typename... Args>
Type* xnew(Args&&... args)
{
	Type* memory = static_cast<Type*>(PoolAllocator::Alloc(sizeof(Type)));
	new(memory)Type(std::forward<Args>(args)...); // placement new, constructor
	return memory;
}

template<typename Type>
void xdelete(Type* obj)
{
	obj->~Type(); // destructor
	PoolAllocator::Release(obj);
}

template<typename Type>
shared_ptr<Type> MakeShared()
{
	return shared_ptr<Type>{ xnew<Type>(), xdelete<Type> };
}
```
그리고 xnew와 xdelete를 통해 MemoryPooling을 쉽게 사용할 수 있게 만들었습니다. 

사용 예시
```c++
class A
{
public:
	A() {}
	~A() {}
	A(int a) : _a(a) {}
private:
	int _a;
};
int main(void)
{
	A* AClass = xnew<A>(1);
	xdelete(AClass);
}
```



