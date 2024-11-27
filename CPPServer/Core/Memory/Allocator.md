# Allocator

메모리 관리의 효율성과 안정성 유지 보수성을 위해 여러가지 할당자를 제작했습니다.

## StompAllocator
dangling pointer와 같은 메모리 오염 문제를 방지하기 위해 StompAllocator를 제작했습니다. OS에 직접 할당과 해제를 요청하여 해제된 메모리에 대해 접근할 시 바로 알 수 있게 됩니다.
또한 가상메모리의 PAGE단위 끝에 할당하여 Overflow와 같은 문제도 찾을 수 있게 하였습니다.  
물론 Context Switching과 같은 비용 문제로 인해 전처리기에서 일반 할당과 구별하여 사용할 수 있게 만들었습니다.

```c++
void* StompAllocator::Alloc(int32 size)
{
    const int64 pageCount = (size + PAGE_SIZE - 1) / PAGE_SIZE;
    const int64 dataOffset = pageCount * PAGE_SIZE - size;

    void* baseAddress = ::VirtualAlloc(NULL, pageCount * PAGE_SIZE, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE);
    return static_cast<void*>(static_cast<int8*>(baseAddress) + dataOffset);
}

void StompAllocator::Release(void* ptr)
{
    const int64 address = reinterpret_cast<int64>(ptr);
    const int64 baseAddress = address - (address % PAGE_SIZE);
    ::VirtualFree(reinterpret_cast<void*>(baseAddress), 0, MEM_RELEASE);
}
```

## PoolAllocator

Memory Pooling을 위한 Allocator입니다.
자세한 내용은 다음을 참고 해주세요 [MemoryPool](https://github.com/sonsungmin98/Portfolio/blob/main/CPPServer/Core/Memory/MemoryPool.md)
```c++
void* PoolAllocator::Alloc(int32 size)
{
    return GMemory->Allocate(size);
}

void PoolAllocator::Release(void* ptr)
{
    GMemory->Release(ptr);
}
```
