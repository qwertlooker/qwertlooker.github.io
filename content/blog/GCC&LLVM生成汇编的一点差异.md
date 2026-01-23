---
title: GCC&LLVM生成汇编的一点差异
summary: GCC&LLVM生成汇编的一点差异
date: 2026-01-23
draft: true
authors:
  - qwertlooker
tags:
  - GCC
  - LLVM
  - c&c++
---

# 背景
前面在项目中切换编译器的时候遇到一点问题。最终定位下来是由于编译器生成的汇编代码有关系，在此总结下

# 栈平衡
在linux下，函数调用约定都是cdecl，由调用者负责参数空间的准备和清理。
gcc和llvm的策略，存在一线差异。
比如下面的简单代码，使用-m32 -O0在x86下编译
```
int callee(int a, int b, int c)
{
    return a + b + c;
}

 
int caller() {
    int i;
    return callee(1,2,i);
}
```
详细的反汇编见后面
总体来说GCC调用使用下面的策略
```
push    ebp                          # 保存ebp
mov     ebp, esp                     # 当前栈顶复制给ebp
sub     esp, 16                      # 分配局部变量栈空间，最小目前看分配了16字节，实际只需要4字节。

push    DWORD PTR [ebp-4]            # 准备参数3,压栈
push    2                            # 准备参数2,压栈
push    1                            # 准备参数1,压栈
call    callee                       # 调用callee
sub     esp, 12                      # 清理3个参数栈空间

leave                                # mov esp,ebp; pop ebp
ret                                  # 返回上一层
```
llvm使用下面的策略
```
push    ebp                          # 保存ebp
mov     ebp, esp                     # 当前栈顶复制给ebp

sub     esp, 20                      # 一次性分配局部变量+参数栈空间
mov     dword ptr [esp], 1           # 准备参数1,直接搬移
mov     dword ptr [esp + 4], 2       # 准备参数2,直接搬移
mov     dword ptr [esp + 8], eax     # 准备参数3,直接搬移
call    callee(int, int, int)        # 调用callee
add     esp, 20                      # 清理栈空间

pop     ebp                          # 恢复ebp
ret                                  # 返回上一层
```

### GCC
```
"callee(int, int, int)":
        push    ebp
        mov     ebp, esp
        mov     edx, DWORD PTR [ebp+8]
        mov     eax, DWORD PTR [ebp+12]
        add     edx, eax
        mov     eax, DWORD PTR [ebp+16]
        add     eax, edx
        pop     ebp
        ret
"caller()":
        push    ebp
        mov     ebp, esp
        sub     esp, 16
        mov     DWORD PTR [ebp-4], 1
        push    DWORD PTR [ebp-4]
        push    2
        push    1
        call    "callee(int, int, int)"
        add     esp, 12
        leave
        ret
```

### LLVM
```
callee(int, int, int):
        push    ebp
        mov     ebp, esp
        mov     eax, dword ptr [ebp + 16]
        mov     eax, dword ptr [ebp + 12]
        mov     eax, dword ptr [ebp + 8]
        mov     eax, dword ptr [ebp + 8]
        add     eax, dword ptr [ebp + 12]
        add     eax, dword ptr [ebp + 16]
        pop     ebp
        ret

caller():
        push    ebp
        mov     ebp, esp
        push    ebx
        sub     esp, 20
        call    .L1$pb
.L1$pb:
        pop     ebx
.Ltmp3:
        add     ebx, offset _GLOBAL_OFFSET_TABLE_+(.Ltmp3-.L1$pb)
        mov     eax, dword ptr [ebp - 8]
        mov     dword ptr [esp], 1
        mov     dword ptr [esp + 4], 2
        mov     dword ptr [esp + 8], eax
        call    callee(int, int, int)
        add     esp, 20
        pop     ebx
        pop     ebp
        ret
```



另外一个复杂的例子：
[test](https://gcc.godbolt.org/#g:!((g:!((g:!((h:codeEditor,i:(filename:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,selection:(endColumn:2,endLineNumber:56,positionColumn:2,positionLineNumber:56,selectionStartColumn:2,selectionStartLineNumber:56,startColumn:2,startLineNumber:56),source:'%23include+%3Cstdint.h%3E%0A%23include+%3Cstring.h%3E%0A%0Atypedef+enum+DT_DataType+%7B%0A++++DT_eBASICTYPE+++%3D+0,%0A++++DT_eDOUBLETYPE++%3D+1,%0A++++DT_eCOMPLEXTYPE+%3D+2,%0A++++DT_eFLOATTYPE+%3D+3,%0A++++DT_eSTDCALL+%3D+4%0A%7DDT_DATATYPE%3B%0A%0Atemplate%3Ctypename+T%3E+DT_DATATYPE+DT_GetDataType(T(*pfnTest)(uint32_t,+uint32_t*),+uint64_t+*pReturnLen)%0A%7B%0A++++uint64_t+returnLen+%3D+sizeof(T)%3B%0A++++memcpy(pReturnLen,+%26returnLen,+sizeof(uint64_t))%3B%0A++++DT_DATATYPE+returnType+%3D+DT_eCOMPLEXTYPE%3B%0A%0A++++uint32_t+result+%3D+0%3B+++++//+%E9%92%88%E5%AF%B9X86%E7%9A%84%E6%9E%9A%E4%B8%BE%E5%80%BC%E7%89%B9%E6%AE%8A%E6%83%85%E5%86%B5%E5%88%A4%E6%96%AD%0A++++%0A%0A++++pfnTest(1,+%26result)%3B%0A%0A%0A++++return+((returnType+%3D%3D+DT_eCOMPLEXTYPE)+%26%26+(result+!!%3D+1))+%3F+DT_eBASICTYPE+:+returnType%3B%0A%0A%7D%0A%0Astruct+TimeJob+%7B%0A++++int+a%3B%0A++++int+b%3B%0A%7D%3B%0A%0A%0Avoid+JudgeComplexStruct(int+paraA,+int+paraB,+int+paraC)%0A%7B%0A++++int+*p%3B%0A++++if+(1+%3D%3D+paraA)%0A++++%7B%0A++++++++p+%3D+(int+*)paraB%3B%0A++++++++*p+%3D+0%3B%0A++++%7D+%0A++++else+if+(1+%3D%3D+paraB)%0A++++%7B%0A++++++++p+%3D+(int+*)paraC%3B%0A++++++++*p++%3D+1%3B%0A++++%7D+%0A++++return+%3B%0A%7D%0A%0Atypedef+TimeJob(*pfnTest)(uint32_t,+uint32_t*)%3B%0A%0Aint+test()+%7B%0A++++uint64_t+ReturnLen%3B%0A++++DT_DATATYPE+a+%3D+DT_GetDataType%3CTimeJob%3E((pfnTest)JudgeComplexStruct,+%26ReturnLen)%3B%0A++++return+(int)a%3B%0A%7D'),l:'5',n:'0',o:'C%2B%2B+source+%231',t:'0')),k:50.17596782302665,l:'4',n:'0',o:'',s:0,t:'0'),(g:!((h:compiler,i:(compiler:clang_trunk,filters:(b:'0',binary:'1',binaryObject:'1',commentOnly:'0',debugCalls:'1',demangle:'0',directives:'0',execute:'1',intel:'0',libraryCode:'0',trim:'1',verboseDemangling:'0'),flagsViewOpen:'1',fontScale:14,fontUsePx:'0',j:1,lang:c%2B%2B,libs:!(),options:'-m32+-O0',overrides:!(),selection:(endColumn:1,endLineNumber:1,positionColumn:1,positionLineNumber:1,selectionStartColumn:1,selectionStartLineNumber:1,startColumn:1,startLineNumber:1),source:1),l:'5',n:'0',o:'+x86-64+clang+(trunk)+(Editor+%231)',t:'0')),k:49.82403217697336,l:'4',n:'0',o:'',s:0,t:'0')),l:'2',n:'0',o:'',t:'0')),version:4)
