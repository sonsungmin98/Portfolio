# Object Pool

기존의 MemoryPool에서는 풀을 사용하는 객체를 사용하는 곳에서 메모리 오염을 시킨 후 다시 풀에 반환을 했을 경우가 있습니다. 이때 다른 곳에서 이 오염된 메모리를 사용 시 크래쉬가 날 수도 있습니다.  
이 경우에는 어디가 문제인지 찾기가 굉장히 힘듭니다. 이를 해결하기 위해 같은 클래스 끼리는 같이 풀링 하자는 것에서 아이디어를 얻어 오브젝트 풀링을 제작하게 되었습니다.

```c++
#pragma once
#include "Types.h"
#include "MemoryPool.h"

template<typename Type>
class ObjectPool
{
public:
	template<typename... Args>
	static Type* Pop(Args&&... args)
	{
#ifdef  _STOMP
		MemoryHeader* ptr = reinterpret_cast<MemoryHeader*>(StompAllocator::Alloc(s_allocSize));
		Type* memory = static_cast<Type*>(MemoryHeader::AttachHeader(ptr, s_allocSize));
#else
		Type* memory = static_cast<Type*>(MemoryHeader::AttachHeader(s_pool.Pop(), s_allocSize));
#endif 

		new(memory)Type(std::forward<Args>(args)...); // placement new, constructor
		return memory;
	}

	static void Push(Type* obj)
	{
		obj->Type();
#ifdef  _STOMP
		StompAllocator::Release(MemoryHeader::DetachHeader(obj));
#else
		s_pool.Push(MemoryHeader::DetachHeader(obj));
#endif
	}

	static shared_ptr<Type> MakeShared()
	{
		shared_ptr<Type> ptr = { Pop(), Push };
		return ptr;
	}

private:
	static int32		s_allocSize;
	static MemoryPool	s_pool;
};

template<typename Type>
int32 ObjectPool<Type>::s_allocSize = sizeof(Type) + sizeof(MemoryHeader);

template<typename Type>
MemoryPool ObjectPool<Type>::s_pool{ s_allocSize };
```

전체적으로 비슷하지만 같은 class는 같이 관리를 하게 됩니다.
