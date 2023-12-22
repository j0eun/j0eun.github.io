---
id: 0
title: "Buffer OverFlow? Beacon Object File!"
subtitle: ""
date: "2023.12.22"
tags: "Post Exploitation"
---

모의해킹/레드팀 관련 공부를 할 때 종종 참고하는 사이트가 있다. [레드라쿤](https://www.redraccoon.kr/)이라는 국내 사이버 보안 커뮤니티인데, 그 곳의 프로젝트 중 하나인 레드팀 플레이북에서 ['후속 공격의 진화'](https://www.레드팀.com/post-exploitation/evolution-of-postex)라는 글을 읽은 적이 있다. 공격자의 명령을 감염PC에서 어떤 방식으로 실행하는지 그간 역사적으로 사용됐던 기법들을 시간대 순으로 나열한 내용이었는데, 그 중 Beacon Object File이라는 방식이 인상적이어서 공부하게 됐다.

# BOF(Beacon Object File) 소개

지난 2020년 6월, Fortra 사는 Cobalt Strike 4.1 릴리즈에서 새로운 페이로드 생성 기능을 소개했다. 오브젝트 파일(COFF) 형태로 페이로드를 생성하는 기능으로, C2[^1]가 오브젝트 파일을 생성해 비컨[^2]에게 전달하고, 비컨은 이를 메모리 상에 직접 로딩하여 실행시키는 방식으로 동작했다. 이 때 생성하는 오브젝트 파일이 Beacon Object File(이하 BOF)이다. 파일을 메모리에 직접 로딩하여 실행시킨다는 점에서 BOF는 Reflective DLL Loading과 종종 비교된다. 그러나 DLL과는 다르게 BOF는 링킹을 거치지 않기 때문에 소스코드를 작성할 때 표준 또는 외부 라이브러리를 사용할 수 없다는 제약사항을 가진다. 그럼 모든 기능을 다 직접 작성해야 하는 걸까? 그건 아니다. 

```c
// 함수 이름을 <MODULE>$<FUNCTION> 형식으로 선언하는 것은 BOF와 비컨 사이의 암묵적인 규칙!
// 이 규칙을 Fortra 사는 Dynamic Function Resolution(이하 DFR)이라고 부른다.
__declspec(dllimport) void* WINAPI KERNEL32$VirtualAlloc(LPVOID lpAddress, 
                                                         SIZE_T dwSize, 
                                                         DWORD flAllocationType, 
                                                         DWORD flProtect);
```

라이브러리 함수는 외부 심볼을 참조하도록 C/C++에서는 `__declspec(dllimport)`로 선언하여 컴파일하면 된다. 그럼 비컨 측에서 심볼 테이블을 파싱해 함수의 실제 주소를 입력해 준다. 참고로 [bofdefs.h](https://github.com/trustedsec/CS-Situational-Awareness-BOF/blob/master/src/common/bofdefs.h)를 사용하면 외부 심볼을 선언할 때 번거로운 작업을 피할 수 있다. 범용적인 명령들은 C2 프레임워크에서 그에 맞는 BOF 페이로드를 생성할 수 있으나, 새로운 명령을 원하거나 기존 명령의 커스텀을 원한다면 직접 작성해야 한다. 이론 상으로는 외부 심볼을 참조할 수 있는 기능을 제공하는 프로그래밍 언어라면 모두 BOF를 작성할 수 있다. 

정리하면, BOF 페이로드가 실행되기 위해서 비컨은 아래와 같은 보조를 해줘야 한다.

1. 오브젝트 파일을 로딩할 메모리 공간 할당 
2. 시작 주소에 알맞게 재배치
3. 심볼 테이블을 파싱해 함수의 실제 주소 찾아서 입력
4. Entrypoint 실행
5. 할당했던 메모리 공간 해제

전체적으로 비컨의 역할이 Reflective DLL Loading에서의 PE 로더 역할과 일치하는 듯 보이지만 여기에는 두 가지 차이점이 있다.

**첫째, 비컨은 [BOF API](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/beacon-object-files_bof-c-api.htm)를 별도로 제공한다.** 위에서 3번은 PE 로더가 Import Address Table(이하 IAT)을 파싱하는 것과 비슷하지만, IAT를 순회하며 모든 심볼을 리졸빙하는 것과는 다르게 비컨은 심볼이 [비컨에 정의되어 있는 함수](https://github.com/trustedsec/COFFLoader/blob/main/beacon_compatibility.c#L28-L59)이냐 아니냐를 판별하고, 아니라면 그 때 리졸빙을 수행한다. 이 때 비컨에 정의되어 있는 함수를 internal function, 정의되어 있지 않아 실제 주소를 리졸빙해야 하는 함수를 external function으로 구분하기도 한다.

**둘째, 비컨은 안정적으로 동작해야 한다.** Reflective DLL Loading은 보안 제품의 감시를 피해 다음 스테이지의 페이로드를 실행하는 것이 주 목적이기 때문에 최종 페이로드의 실행이 끝난 뒤 프로세스와 스레드의 안정성을 보장하지 않는 경향이 있다. 그런데 BOF 페이로드는 비컨 프로세스에서 실행되므로 BOF가 비정상적으로 종료된다는 것은 비컨도 종료된다는 것을 의미한다. 공격자가 해당 PC에 대한 접근 권한을 잃지 않고 다음 명령을 전달하기 위해선 비컨 프로세스가 지속 가능해야 한다. 따라서 BOF의 로딩, 실행, 그 후 다시 비컨으로 실행 흐름이 돌아오기까지 프로세스의 컨텍스트를 오염시키지 않고 스레드가 예상치 못한 예외를 발생시키지 않도록 BOF를 잘 작성하는 것은 중요한 과제라고 볼 수 있다. 

BOF가 뭔지 알아봤으니 이제 강점과 약점을 알아보자.

# BOF의 강점 

**첫째, 페이로드가 BOF를 지원하는 여러 C2 프레임워크 간에 호환이 가능하다.** 이름을 대면 알만한 유명한 C2 프레임워크 중에서 상당 수가 BOF 호환을 지원한다는 설명이 있었다. 상용 제품들은 코드를 확인하기가 곤란해 오픈 소스 C2 프레임워크인 [Sliver](https://github.com/BishopFox/sliver)와 [Havoc](https://github.com/HavocFramework/Havoc)을 기준으로 어떻게 호환이 되고 있는지 살펴봤다.

- Sliver - [sliverarmory/COFFLoader](https://github.com/sliverarmory/COFFLoader)
- Havoc - [Cracked5pider/CoffeeLdr](https://github.com/Cracked5pider/CoffeeLdr)

위의 두 C2 프레임워크가 사용하는 BOF 로더는 모두 TrustedSec 사에서 작성한 [COFFLoader](https://github.com/trustedsec/COFFLoader)를 기반으로 한다. 해당 로더는 DFR을 준수하기 때문에 BOF를 작성할 때도 이를 지키면 Sliver와 Havoc에서 둘 다 실행 가능한 페이로드를 만들 수 있다. 또한 BOF API도 Cobalt Strike와 동일한 시그니처(함수명, 인수의 형식 및 개수)로 작성되어 있기 때문에 API 사용도 문제가 없다. 

![image0](/images/bof_api_not_implemented.png)

다만 일부는 미구현되어 있기 때문에 호출은 가능해도 명령으로써의 기능은 하지 못 할 수 있다.

**둘째, 페이로드의 크기가 실행파일 형식의 페이로드보다 압도적으로 작다.** UAC Bypass를 통해 권한 상승을 하는 명령을 Reflective DLL Loading 형식과 BOF 형식으로 각각 페이로드를 생성하면, 페이로드의 크기가 전자는 100KB 이상, 후자는 3KB 미만이 나온다고 한다. 아마도 이 차이는, BOF에서 호출하는 함수가 대부분 외부 심볼을 참조하므로 코드가 존재하지 않는 것도 있고, 컴파일러가 실행파일에 삽입하는 추가적인 코드 등이 생략됐기 때문일 것으로 보인다. BOF는 페이로드의 크기가 극히 작기 때문에 C2와 비컨 사이에 활성 가능한 통신 채널이 제한적일 때 DNS를 통해 명령을 전달하는 등 차선책을 생각해 볼 수 있다.

# BOF의 약점 

```
Dump of file .\ipconfig.x64.o

File Type: COFF OBJECT

COFF SYMBOL TABLE
...
01B 00000000 UNDEF  notype       External     | __imp_MSVCRT$calloc
01C 00000000 UNDEF  notype       External     | __imp_BeaconOutput
01D 00000000 UNDEF  notype       External     | __imp_MSVCRT$free
01E 00000000 UNDEF  notype       External     | __imp_MSVCRT$vsnprintf
01F 00000000 UNDEF  notype       External     | __imp_KERNEL32$GetProcessHeap
020 00000000 UNDEF  notype       External     | __imp_KERNEL32$HeapAlloc
021 00000000 UNDEF  notype       External     | __imp_KERNEL32$HeapFree
022 00000000 UNDEF  notype       External     | __imp_Kernel32$WideCharToMultiByte
023 00000000 UNDEF  notype       External     | __imp_IPHLPAPI$GetAdaptersInfo
024 00000000 UNDEF  notype       External     | __imp_BeaconPrintf
025 00000000 UNDEF  notype       External     | __imp_IPHLPAPI$GetNetworkParams
...
```
*`__declspec(dllimport)`를 사용하면 심볼 테이블에 x86-32는 접두사로 `__imp__`, x86-64는 접두사로 `__imp_`가 붙는다.*

BOF는 표준 및 외부 라이브러리를 일체 사용할 수 없기 때문에 필연적으로 외부 참조를 남발하는 상황에 놓인다. **문제는 컴파일을 하면 외부 참조는 위와 같이 심볼 테이블에 참조하는 심볼 이름이 남는다는 점이다.** 심지어 링킹을 거치지 않기 때문에 외부 참조 심볼이 해결되지 않고 모두 그대로 테이블에 남은 채로 비컨에게 전달된다. C2가 비컨에게 페이로드를 전달하는 과정과 비컨이 페이로드를 로딩하는 과정 속에서 심볼 테이블에 있는 BOF API 등이 탐지지표로 쓰일 수도 있다. DFR이라는 naming convention과 BOF API의 시그니처 공개로 여러 C2 프레임워크 사이에 호환성이 높아졌는데, 도리어 이게 BOF 페이로드를 특정하는 지표가 된 셈이다.

마지막으로, **결국 오브젝트 파일이 실행되기 위해선 메모리를 할당하고 실행시켜야 한다.** Reflective DLL Loading과 마찬가지로 `PAGE_EXECUTE_READWRITE` 권한이 있는 메모리 영역이 필요하고, 오브젝트 파일은 디스크에서 매핑된 이미지가 아니기 때문에 해당 메모리 영역은 `MEM_IMAGE` 속성이 아닌 `MEM_PRIVATE` 속성을 가진다는 특징이 있다. 

# 결론

C2가 생성하는 페이로드 형태 중 하나인 BOF를 알아봤는데 강점과 약점이 명확했다. 본 글은 Sliver와 Havoc을 오디팅한 내용을 기반으로 작성했기 때문에 BOF의 약점은 어쩌면 상용 C2 프레임워크에선 개선되거나 심지어는 해결됐을 가능성도 있다. 그리고 Sliver와 Havoc도 비컨과 주로 안전한 채널(i.e. TLS)에서 통신하기를 권장하기 때문에 페이로드 전달 과정에서 이를 포착하기란 그리 간단하지는 않다. 

보면서 느꼈던 건 Reflective DLL Loading의 PE 로더에서 한 단계 더 나아가 BOF API를 제공하는 인터프리터?의 개념이 나름 신선했다. 아직은 국내에 BOF의 개념이 그리 알려지지 않은 것 같은데, 페이로드를 BOF로 전달하는 방식이 국내 침해사고에서도 발견되면 국내 AV/EDR 벤더사들은 어떤 방식으로 대응할지 기대된다.

[^1]: Command and Control의 약자. 공격자가 장악한 서버를 말하며 주로 원격에서 명령을 내린다.
[^2]: 일정 시간마다 C2에서 명령을 가져와 수행하는 클라이언트.