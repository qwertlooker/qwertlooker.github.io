---
title: C&C++ mock原理解析
summary: 深入理解 C&C++ mock原理
date: 2025-12-24
authors:
  - qwertlooker
tags:
  - mock
  - c&c++
image:
  [image](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQYP2ElE_ndTJtFBls85ygxyoQQOS2IoSR4AA&s)
---

# mock原理
将原函数的入口处的指令修改为跳转指令，跳转目的地址为mock函数的地址。
当执行原函数时，直接跳转到mock函数执行，达到mock的目的。

# 简单例子
下面是一个说明mock原理的示例程序。
```cpp

#include <bits/stdc++.h>
#include <memoryapi.h>
#include <windows.h>

using namespace std;

int FuncA() {
    cout << "FuncA " << endl; // 不能删除，删除后FuncA函数指令太少，导致跳转指令覆盖下一个函数
    return 0;
}

int FuncA_Stub() {
    cout << "FuncA_Stub " << endl;
    return 100;
}

int main(int argc, char **argv) {
    //    x64 跳转指令
    uint8_t jmpInst[] = {0xFF, 0x25, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00};
    //     获取原函数地址和桩函数地址
    void *oldFuncAddr = (void *) &FuncA;
    void *newFuncAddr = (void *) &FuncA_Stub;
    cout << "ptr size " << sizeof(uintptr_t) << endl;

    *((uintptr_t *) (jmpInst + 6)) = (uintptr_t) newFuncAddr;

    /*电请写权限*/
    DWORD reSerued;
    // 计算页面边界
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    SIZE_T pageSize = si.dwPageSize;

    void *target = (void *) &FuncA;
    uintptr_t pageStart = (uintptr_t) target & ~(pageSize - 1);
    SIZE_T protectSize = pageSize; // 简单起见保护整个页面
    if (!VirtualProtect((LPVOID) pageStart, protectSize, PAGE_EXECUTE_READWRITE, &reSerued)) {

        printf("VirtualProtect fail\n");
        return -1;
    }
    memcpy(oldFuncAddr, jmpInst, sizeof(jmpInst));
    VirtualProtect((LPVOID) pageStart, protectSize, reSerued, &reSerued);

    printf("call FuncA=%d\n", FuncA());
    return 0;
}
```
