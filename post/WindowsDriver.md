<p align="right"><a href="https://doexercise.github.io">Table of Contents</a></p>  

# ***Introduction***
Windows Device Driver 작성에 대한 내용을 정리하였다.

<br />

# ***FileSystem Filter Driver***
미니필터 드라이버를 작성하여 파일에 접근하는 프로세스를 알아낼 수 있다.

## Basic Example

```C
typedef PCHAR(*GET_PROCESS_IMAGE_NAME)(PEPROCESS Process);
GET_PROCESS_IMAGE_NAME PsGetProcessImageFileName;

// 어떤 Major Functon 에 대한 콜백함수라 치자.
FLT_PREOP_CALLBACK_STATUS FsFilterPreOperation(
       _Inout_ PFLT_CALLBACK_DATA Data,
       _In_ PCFLT_RELATED_OBJECTS FltObjects,
       _Flt_CompletionContext_Outptr_ PVOID *CompletionContext)
{
       /********************************************************************
       * PsGetProcessImageFileName() 함수는 숨겨져 있다. 아래와 같이 찾아보자
       ********************************************************************/
       UNICODE_STRING funtion_str = RTL_CONSTANT_STRING(L"PsGetProcessImageFileName");
#pragma warning(push)
#pragma warning(disable : 4055) // type cast warning
       PsGetProcessImageFileName = (GET_PROCESS_IMAGE_NAME)MmGetSystemRoutineAddress(&funtion_str);
#pragma warning(pop)
       if (NULL != PsGetProcessImageFileName)
       {
              PCHAR pImageName = PsGetProcessImageFileName(IoThreadToProcess(Data->Thread));
              if (NULL != pImageName)
              {
                     DbgPrint("Process Image Name : %s\n", pImageName); // cmd.exe 이렇게 나온다.
              }
       }


       /********************************************************************
       * 접근하려 했던 파일 정보에 대해 찾아보자
       * 다만, 언제 이 콜백이 호출되었느냐에 따라 파일 정보를 볼 수 없기도 하다.
       * 게다가 BSOD 를 유발하기도 한다.
       ********************************************************************/
       PFLT_FILE_NAME_INFORMATION nameInfo;
       NTSTATUS status;
       status = FltGetFileNameInformation(Data, FLT_FILE_NAME_NORMALIZED, &nameInfo);
       if (NT_SUCCESS(status))
       {
              // 아래 함수로 파싱하지 않으면 보이지 않는 필드가 몇개 있다. 필수적인 사항은 아니다.
              FltParseFileNameInformation(nameInfo);
              // FLT_FILE_NAME_SHORT 일 때는 안 보이는가? 글쎄?
              if (nameInfo->Format != FLT_FILE_NAME_SHORT)
              {
                     DbgPrint("%wZ\n", nameInfo->Name);
                     DbgPrint("%wZ\n", nameInfo->Volume);    // same as FltObjects->Volume
                     DbgPrint("%wZ\n", nameInfo->Extension); // Only available after FltParseFileNameInformation()
              }
              FltReleaseFileNameInformation(nameInfo);
       }


       /********************************************************************
       * 위의 PsGetProcessImageFileName() 에서 이미 나왔지만...
       * Process 관련 정보를 찾는 함수들을 몇가지 살펴보자.
       * Mini-filter 의 경우 FltXXX() 이고 일반 Filter 는 IoGetXXX() 이렇게 되는 듯
       ********************************************************************/
       ULONG pid = FltGetRequestorProcessId(Data);
       PEPROCESS objProcess = IoThreadToProcess(Data->Thread);
       HANDLE hPid = PsGetProcessId(objProcess); // 이 return 값(HANDLE)과 FltGetRequestorProcessId() 의 return 값(ULONG)은 같다

       return FLT_PREOP_SUCCESS_WITH_CALLBACK;
}
```

* 아래 함수들도 참고해 보자.  
    - PsLookupProcessByProcessId();  
    - IoGetRequestorProcess();  
    - IoGetRequestorProcessId();  
    - IoGetRequestorSessionId();  
    - IoQueryFileDosDeviceName();  
    - RtlVolumeDeviceToDosName();  

<br />

## [**Table of Contents**](../README.md)