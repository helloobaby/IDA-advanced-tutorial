```
//
// 一般来说LIST_ENTRY可以在结构体的任何位置
// 这个宏通过LIST_ENTRY的地址获得整个结构体的首地址
//

#define CONTAINING_RECORD(address, type, field) ((type *)( \
                                                  (PCHAR)(address) - \
                                                  (ULONG_PTR)(&((type *)0)->field)))



__int64 __fastcall ExpScanGeneralLookasideList(LIST_ENTRY *ExNPagedLookasideListHead, KSPIN_LOCK *a2)
{
  KIRQL v4; // al
  __int64 Flink; // r8
  KIRQL v6; // si
  int v7; // edx
  unsigned int v8; // ecx
  unsigned __int16 v9; // r10
  int v10; // er9
  int v11; // er9
  __int64 v12; // rdx
  __int64 result; // rax
  unsigned int v14; // eax
  unsigned int v15; // edx
  struct _KPRCB *CurrentPrcb; // rcx

  v4 = KeAcquireSpinLockRaiseToDpc(a2);
  Flink = ExNPagedLookasideListHead->Flink;
  v6 = v4;
  if ( ExNPagedLookasideListHead->Flink != ExNPagedLookasideListHead )
  {
    while ( 1 )
    {
      v7 = *(Flink - 40) - *(Flink + 20);
      *(Flink + 20) = *(Flink - 40);
      v8 = *(Flink - 44) - *(Flink + 16);
      v9 = *(Flink - 46);
      *(Flink + 16) = *(Flink - 44);
      if ( v9 != 0xFFFF )
        break;

```

上面的Flink就是所需结构体的LIST_ENTRY成员，但是无法将他将他转化为一个结构体的LIST_ENTRY字段。



IDA目前提供两种结构体相关的方法

1.Convert to struct*

2.Create new struct type



第一种方法是将一个变量转化为结构体的指针(注意不是结构体本身,如果要转化结构体的话,'Y'),很明显Flink并不是指向一个结构体的指针。



第二种方法是创建一个新的结构体，这也是要基于Flink是结构体开始的情况。





```
loc_FFFFF807254AF980:   ; r8 = _NPAGED_LOOKASIDE_LIST.LIST_ENTRY LIST_ENTRY字段往前移0x28 为AllocateMisses
mov     ecx, [r8-28h]
mov     edx, ecx
sub     edx, [r8+14h]   ; Misses = AllocateMisses - LastAllocateMisses
mov     [r8+14h], ecx   ; LastAllocateMisses = AllocateMisses
mov     eax, [r8-2Ch]
mov     ecx, eax
sub     ecx, [r8+10h]
movzx   r10d, word ptr [r8-2Eh]
```



把FLINK转化为ULONG_PTR或者整数类型，不要转化为他本身的LIST_ENTRY。然后根据相对的偏移如下图的0x28,0x14索引具体的字段。如果没有PDB的话，字段名称只能自己命名。

      Misses = *(_DWORD *)(Flink - 0x28) - *(_DWORD *)(Flink + 0x14);// Misses = Lookaside->L.AllocateMisses - Lookaside->L.LastAllocateMisses;
      *(_DWORD *)(Flink + 0x14) = *(_DWORD *)(Flink - 0x28);// Lookaside->L.LastAllocateMisses = Lookaside->L.AllocateMisses;
      v8 = *(_DWORD *)(Flink - 0x2C) - *(_DWORD *)(Flink + 16);// v8 = TotalAllocates - LastTotalAllocates
      v9 = *(_WORD *)(Flink - 0x2E);            // v9 = MaximumDepth
      *(_DWORD *)(Flink + 16) = *(_DWORD *)(Flink - 0x2C);// LastTotalAllocates = TotalAllocates
      if ( v9 != 0xFFFF )
        break;