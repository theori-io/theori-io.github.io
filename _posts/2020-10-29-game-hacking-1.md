---
layout: post
title: "게임핵의 원리에 대해 알아보자 (1) - Wall Hack 편"
author: theori
description:
categories: [ research, korean ]
tags: [ game, cheats, hacks, wall hack, d3d ]
comments: true
featured: true
image: assets/images/2020-10-29/image_10.png
---

FPS 게임에서 자주 발견되는 "Wall Hack" (월핵)은 벽 너머의 적을 보여주어 위치를 알 수 있도록 한다. 월핵을 구현하는 방법은 그래픽 렌더링 라이브러리마다 조금씩 다르다.

![FPS Wall Hack](/assets/images/2020-10-29/image_00.png)

이 글에서는 여러 그래픽 렌더링 라이브러리 중 Microsoft에서 개발한 Direct X(Direct3D, D3D)로 개발된 게임의 월핵을 구현하는 방법을 소개하도록 한다. 먼저 D3D9 버전에서의 예제를 들겠다.

## Z Buffer (Depth Buffer)

D3D9에서 월핵을 구현하기 위해선 렌더링에서 사용되는 `Z Buffer`의 개념을 이해해야 한다. Z Buffer란 렌더링할 때 어떤 물체가 보여야 할지에 대한 여부를 판별하기 위해 사용되는 방법 중 하나다. 위키피디아에 따르면 Z Buffer에 대해 아래와 같이 설명하고 있다.

> 어떤 물체가 그려질 때 만들어진 픽셀의 깊이 정보(z 좌표)는 버퍼(Z 버퍼 혹은 깊이 버퍼)에 저장된다. 이 버퍼는 (x-y)의 2차원 좌표를 기준으로 해당하는 각각의 스크린 픽셀 요소들로 정렬되어 있다. 만약 다른 물체가 같은 픽셀에 그려져야 할 때, Z 버퍼링은 현재 픽셀과 새로 그려질 픽셀 중 어떤 것이 관찰자에게 더 가까운지 깊이를 비교한다. Z 버퍼에 기록되도록 새로 선택된 깊이는 이전의 깊이를 덮어쓴다. 즉, Z 버퍼는 '더 가까운 물체가 더 먼 물체를 가린다' 라는 직관적 깊이 관념을 정확하게 따를 수 있게 돕는다. ([https://ko.wikipedia.org/wiki/Z_버퍼링](https://ko.wikipedia.org/wiki/Z_%EB%B2%84%ED%8D%BC%EB%A7%81))

아래 그림처럼 3개의 도형이 있는 상황을 가정하여 Z Buffer가 어떤 식으로 동작하는지 알아보자.

<div class="row justify-content-center">
<div class="col-8">
<img src="/assets/images/2020-10-29/image_01.png">
</div>
</div>

<figcaption><a href="http://www.racketboy.com/retro/about-video-games-rasterization-and-z-buffer" target="_blank">http://www.racketboy.com/retro/about-video-games-rasterization-and-z-buffer</a></figcaption>

이 그림에서는 도형을 S2, S1, S3의 순으로 그리고 있다.

오른쪽에 있는 2차원 배열은 초기화된 직후의 Z Buffer 상태부터 S2, S1, S3 도형을 순서대로 그린 후의 Z Buffer 상태이다. 각각의 상태에 대해 해석해보자.

1. 아무것도 그리지 않은 초기 Z Buffer 상태이다.
2. S2 도형을 그리고 난 후 Z Buffer 상태이다.
3. S1 도형을 그리고 난 후 Z Buffer 상태이다. S2 도형과 겹치는 부분은 깊이 값을
   비교했을 때 Z Buffer에 기록되어 있는 값이 더 크기 때문에 해당 영역은 업데이트
   되지 않았다.
4. S3 도형을 그리고 난 후 Z Buffer 상태이다. S1, S2 도형과 겹치는 부분은 깊이
   값을 비교했을 때 Z Buffer에 기록되어 있는 값이 더 작기 때문에 해당 영역은 S3
   도형의 깊이 값으로 업데이트 되었다.

만약 특정 물체(예: 적 플레이어)를 렌더링할 때 Z Buffer 기능을 비활성화하면 렌더링 엔진은 해당 물체가 보여야 할 지의 여부를 구별할 수 없어 물체를 항상 화면에 보여줄 것이다. 이것이 Z Buffer를 사용한 월핵의 기본 원리이다. 다만 Z Buffer 기능을 비활성화하기 위해 렌더링의 흐름을 제어해야 하는데 이를 위해서는 Direct3D의 라이브러리 함수를 후킹해야 한다. 아래에서는 어떤 함수를 후킹해야 하는지와 후킹하는 방법에 대해 이야기한다.

## D3D9 Hook

D3D9의 함수를 후킹하기 위해서는 후킹 할 함수의 주소를 알아내는 것이 선행되어야 한다. 먼저 Direct3D가 동작하는 기본 원리를 알아보자.

아래는 Direct3D 인터페이스를 생성하는 코드이다.

```cpp
// this function initializes and prepares Direct3D for use
void initD3D(HWND hWnd){
    d3d = Direct3DCreate9(D3D_SDK_VERSION);    // create the Direct3D interface

    D3DPRESENT_PARAMETERS d3dpp;    // create a struct to hold various device information

    ZeroMemory(&d3dpp, sizeof(d3dpp));    // clear out the struct for use
    d3dpp.Windowed = TRUE;    // program windowed, not fullscreen
    d3dpp.SwapEffect = D3DSWAPEFFECT_DISCARD;    // discard old frames
    d3dpp.hDeviceWindow = hWnd;    // set the window to be used by Direct3D

    // create a device class using this information and information from the d3dpp stuct
    d3d->CreateDevice(D3DADAPTER_DEFAULT,
                      D3DDEVTYPE_HAL,
                      hWnd,
                      D3DCREATE_SOFTWARE_VERTEXPROCESSING,
                      &d3dpp,
                      &d3ddev);
}
```

<figcaption><a href="http://www.directxtutorial.com/Lesson.aspx?lessonid=9-4-1" target="_blank">http://www.directxtutorial.com/Lesson.aspx?lessonid=9-4-1</a></figcaption>

Direct3D 라이브러리는 대부분의 export 되어 있는 형태가 아니기 때문에
[CreateDevice](https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3d9-createdevice){:target="_blank"}
함수를 통해 생성된 [IDirect3DDevice9](https://docs.microsoft.com/en-us/windows/win32/api/d3d9helper/nn-d3d9helper-idirect3ddevice9){:target="_blank"}
인터페이스를 사용하여 렌더링 함수들을 호출해야 한다. 

이 정보들을 바탕으로 D3D9의 함수를 후킹하는 과정을 요약하면 다음과 같다. 

1. 게임 프로세스의 메모리에 접근하기 위해 DLL을 인젝션한다. 
2. vtable의 주소를 찾는다. (Pattern Matching/Scanning)
3. `DrawIndexedPrimitive` 함수의 주소를 구한다. (해당 함수 용도는 추후 설명)
4. 함수에 Inline Hook을 설치한다.
5. 숨겨진 물체를 보이게 한다.

여기서 vtable이란 C++ 클래스의 멤버 함수 중 가상 함수가 존재하는 경우 생성되는 함수 포인터 배열이다. 이 글에서는 `IDirect3DDevice9` 인터페이스의 vtable에 렌더링 함수들이 **정해진 순서대로** 존재한다는 것만 알면 된다. 따라서 vtable 주소만 구하면 모든 가상 함수의 주소를 알 수 있게 된다.

#### 1. 게임 프로세스의 메모리에 접근하기 위해 DLL을 인젝션한다

DLL Injection 없이 외부 프로세스에서 메모리 관련 WIN API를 사용하는 방식(External Hook이라고도 함)으로도 월핵 구현이 가능하지만 보통 개발이 편한 DLL Injection을 사용한다. 다만, 해당 내용에 대해 이번 글에서는 그림으로 간략하게 표현하고 자세히 다루지 않는다.

![DLL injection](/assets/images/2020-10-29/image_02.png)

&nbsp;

#### 2. vtable의 주소를 찾는다 (Pattern Matching/Scanning)

이제 `IDirect3DDevice9` 인터페이스의 vtable을 찾아야 한다. 이 인터페이스는 [CreateDevice](https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3d9-createdevice){:target="_blank"} 함수를 통해 생성되기 때문에 이 함수를 기점으로 분석해야 한다.

`CreateDevice` 함수는 7번째 인자인 `ppReturnedDeviceInterface`로
`IDirect3DDevice9` 인터페이스를 반환한다. `ppReturnedDeviceInterface` 변수는
`CreateDevice` 함수 시작 시 0으로 초기화된 후 다음 부분에서 설정된다.

```cpp
LONG __stdcall CEnum::CreateDevice(
	CEnum *this, // this 포인터기 때문에 msdn에는 이 부분이 없음
	unsigned int a2,
	enum _D3DDEVTYPE a3,
	HWND a4,
	unsigned int a5,
	struct _D3DPRESENT_PARAMETERS_ *a6,
	struct IDirect3DDevice9 **ppReturnedDeviceInterface)
{
/*  
HRESULT CreateDevice(
  UINT                  Adapter,
  D3DDEVTYPE            DeviceType,
  HWND                  hFocusWindow,
  DWORD                 BehaviorFlags,
  D3DPRESENT_PARAMETERS *pPresentationParameters,
  IDirect3DDevice9      **ppReturnedDeviceInterface
);
*/
...
v12 = CEnum::CreateDeviceImpl(
            (CEnum *)v7,
            v8,
            a3,
            (HWND)*(&var_164 + 1),
            a5,
            v23,
            v22,
            (struct IDirect3DDevice9Ex **)&var_164,
            v11);
    v13 = var_164;
    *(&var_164 + 1) = v12;
    *ppReturnedDeviceInterface = (struct IDirect3DDevice9 *)var_164; // 이 곳에서 인터페이스 값이 설정됨
...
```

`var_164` 값으로 `ppReturnedDeviceInterface` 값이 설정되는데 이 값은 `CEnum::CreateDeviceImpl` 함수의 8번째 인자에서 설정된다. `CreateDeviceImpl` 함수를 따라가 보면 다음과 같다.

```cpp
int __thiscall CEnum::CreateDeviceImpl(
	CEnum *this,
	unsigned int a2,
	enum _D3DDEVTYPE a3,
	HWND a4,
	unsigned int a5,
	struct _D3DPRESENT_PARAMETERS_ *arg_10,
	const struct D3DDISPLAYMODEEX *a7,
	struct IDirect3DDevice9Ex **ppReturnedDeviceInterface,
	struct _D3D9ON12_ARGS *a9
){
    if ( hLibModule )
    {
      v26 = CD3DHal::CD3DHal((CD3DHal *)hLibModule); // v26은 이 함수의 반환값으로 설정됨
      goto LABEL_36;
    }
...
    if ( !v27 )
    {
      *ppReturnedDeviceInterface = (struct IDirect3DDevice9Ex *)v26; // ppReturnedDeviceInterface이 v26으로 설정됨
      return 0;
    }
    (*(void (__thiscall **)(CD3DHal *, int))(*(_DWORD *)v26 + 696))(v26, 1);
    D3DRecordHRESULT(
      (size_t)"Failed to initialize D3DDevice. CreateDeviceEx Failed.",
      (struct _hrCapture *)0xDEADBEEF,
      "windows\\directx\\dxg\\inactive\\d3d9\\d3d\\fe\\d3ddev.cpp",
      1068);
```

`ppReturnedDeviceInterface` 에 들어가는 값은 `v26`에서 왔고, `v26`은 `CD3DHal::CD3DHal` 함수에서 왔다. `CD3DHal::CD3DHal` 함수는 아래와 같다.

```cpp
CD3DHal *__thiscall CD3DHal::CD3DHal(CD3DHal *this){
  CD3DHal *v1; // esi

  v1 = this;
  CD3DBase::CD3DBase(this);
  *(_DWORD *)v1 = &CD3DHal::`vftable'; // vtable을 초기화 하는 부분
  *((_DWORD *)v1 + 3220) = 0;
  *((_DWORD *)v1 + 3218) = 0;
  *((_DWORD *)v1 + 3219) = 0;
  *((_DWORD *)v1 + 3269) = 0;
  *((_DWORD *)v1 + 3272) = 0;
  *((_DWORD *)v1 + 3278) = 0;
  *((_DWORD *)v1 + 3279) = 0;
  *((_DWORD *)v1 + 3280) = 0;
  *((_DWORD *)v1 + 3281) = 0;
  *((_DWORD *)v1 + 4088) = 0;
  *((_DWORD *)v1 + 4091) = 0;
  *((_DWORD *)v1 + 4094) = 0;
  *((_DWORD *)v1 + 4097) = 0;
  *((_DWORD *)v1 + 4100) = 0;
  *((_BYTE *)v1 + 16404) = 0;
  *((_DWORD *)v1 + 3275) = 0;
  return v1;
}
```

7번째 라인에 보이는 ``&CD3DHal::`vftable'``이 `IDirect3DDevice9` 인터페이스의 vtable이다.

이제 `CD3DHal::CD3DHal` 함수의 코드 부분을 패턴으로써 사용하여 d3d9.dll 상의 메모리를 스캔해 이 함수를 찾고 vtable의 주소를 알아낼 수 있다.

패턴을 찾기 위해 어셈블리어로 확인해보면 다음과 같다.

```cpp
public: __thiscall CD3DHal::CD3DHal(void) proc near
8B FF             mov     edi, edi
56                push    esi
8B F1             mov     esi, ecx
E8 05 73 00 00    call    CD3DBase::CD3DBase(void)
33 C0             xor     eax, eax
C7 06 24 1D 00 10 mov     dword ptr [esi], offset const CD3DHal::`vftable' // 이 부분부터 패턴 시작
89 86 50 32 00 00 mov     [esi+3250h], eax
89 86 48 32 00 00 mov     [esi+3248h], eax
89 86 4C 32 00 00 mov     [esi+324Ch], eax
89 86 14 33 00 00 mov     [esi+3314h], eax
89 86 20 33 00 00 mov     [esi+3320h], eax
89 86 38 33 00 00 mov     [esi+3338h], eax
89 86 3C 33 00 00 mov     [esi+333Ch], eax
89 86 40 33 00 00 mov     [esi+3340h], eax
89 86 44 33 00 00 mov     [esi+3344h], eax
89 86 E0 3F 00 00 mov     [esi+3FE0h], eax
89 86 EC 3F 00 00 mov     [esi+3FECh], eax
89 86 F8 3F 00 00 mov     [esi+3FF8h], eax
89 86 04 40 00 00 mov     [esi+4004h], eax
89 86 10 40 00 00 mov     [esi+4010h], eax
88 86 14 40 00 00 mov     [esi+4014h], al
89 86 2C 33 00 00 mov     [esi+332Ch], eax
8B C6             mov     eax, esi
5E                pop     esi
C3                retn
                  public: __thiscall CD3DHal::CD3DHal(void) endp
```

vtable을 설정하는 부분부터 추출한 옵코드는 `C7 06 24 1D 00 10 89 86 50 32 00 00 89 86`이다. 하지만 환경에 따라 vtable의 오프셋과 뒤에 따라오는 mov 어셈블리의 오프셋은 달라질 수 있다. 따라서 가변적인 부분을 `??`로 치환하면 `C7 06 ?? ?? ?? ?? 89 86 ?? ?? ?? ?? 89 86` 가 되고, 이것이 최종적으로 vtable을 찾을 때 사용하게 될 패턴이다.

패턴을 찾을 때는 아래 형태의 함수를 많이 사용한다.

```cpp
bool bCompare(const BYTE* pData, const BYTE* bMask, const char* szMask){
    for(;*szMask;++szMask,++pData,++bMask)
        if(*szMask=='x' && *pData!=*bMask ) 
            return false;
 
    return (*szMask) == NULL;
}
 
DWORD FindPattern(DWORD dwAddress,DWORD dwLen,BYTE *bMask,char * szMask){
    for(DWORD i=0; i < dwLen; i++)
        if( bCompare( (BYTE*)( dwAddress+i ),bMask,szMask) )
            return (DWORD)(dwAddress+i);
 
    return 0;
}
```

구한 패턴과 `FindPattern` 함수를 사용하여 다음과 같이 vtable의 주소를 구할 수 있다.

```c
DWORD table = FindPattern(
    (DWORD)hModule,
    0x128000,
    (PBYTE)"\xC7\x06\x00\x00\x00\x00\x89\x86\x00\x00\x00\x00\x89\x86",
    "xx????xx????xx" // 가변적인 주소 부분은 ?로 마스킹
);
memcpy(&vTable, (void*)(table+2), 4);	// vtable 주소 값을 복사
```

&nbsp;

#### 3. DrawIndexedPrimitive 함수 주소를 구한다

D3D9에서 물체 혹은 도형을 그릴 때 사용하는 함수로는 `DrawPrimitive`, `DrawPrimitiveUp`, `DrawIndexedPrimitiveUp`, `DrawIndexedPrimitive` 등이 있다. 이 중 [DrawIndexedPrimitive](https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-drawindexedprimitive){:target="_blank"} 함수를 사용한 이유는 Direct3D 개발 과정에서 성능상의 문제로 이 함수를 주로 사용하기 때문이다. 

`DrawIndexedPrimitive` (이하 'DIP') 함수 주소를 찾을 때는 vtable index를 이용한다.
컴파일된 d3d9.dll 바이너리는 vtable index가 고정되어 있기 때문에 미리 구한 (혹은 공개된) vtable index를 이용할 수 있다.

```cpp
#define QUERY_INTERFACE             0
#define ADDREF                      1
#define RELEASE                     2
#define TESTCOOPERATIVELEVEL        3
#define GETAVAILABLETEXTUREMEM      4
...
#define ENDSCENE                    42
...
#define DRAWINDEXEDPRIMITIVE    82
#define DRAWPRIMITIVEUP         83
#define DRAWINDEXEDPRIMITIVEUP  84
```

IDA를 통해 `DRAWINDEXEDPRIMITIVE`의 vtable index가 맞는지 확인해보자.

```cpp
.text:10001D24 const CD3DHal::`vftable' dd offset CBaseDevice::QueryInterface(_GUID const &,void * *)
...
.text:10001E6C                          dd offset CD3DBase::DrawIndexedPrimitive(_D3DPRIMITIVETYPE,int,uint,uint,uint,uint)
```

vtable의 주소는 0x10001D24 이고 `DRAWINDEXEDPRIMITIVE`값은 82 이므로

```cpp
0x10001D24 + 82 * 4(32-bit 포인터 크기) == 0x10001E6C 
```

`CD3DBase::DrawIndexedPrimitive` 함수 포인터의 주소 값과 동일한 걸 확인할 수 있다.

&nbsp;

#### 4. 함수에 Inline Hook을 설치한다

DIP 함수의 주소를 구했으니 이제 함수를 후킹해 Z Buffer 기능을 비활성화해야 한다. 함수를 후킹하는 방법은 다양하나 이 글에서는 `Inline Hook` 기법에 대해 설명한다.

먼저 Inline Hook에 대해 잘 모르는 독자를 위해 원리만 간단히 짚고 넘어가도록 한다. 

<div class="row justify-content-center">
<div class="col col-md-6">
<img src="/assets/images/2020-10-29/image_03.png">
<figcaption><a href="https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/huntusenixnt99.pdf" target="_blank">Detours: Binary Interception of Win32 Functions</a></figcaption>
</div>
<div class="col col-md-6">
<img src="/assets/images/2020-10-29/image_04.png">
<figcaption>후킹된 DIP 함수의 흐름</figcaption>
</div>
</div>

좌측 그림을 보면 `TargetFunction` (후킹 대상 함수, 우측 그림의 DIP 함수에 해당)의
프롤로그를 `jmp` 명령어로 패치하여 후킹 함수 (사진에서는 `DetourFunction`)을
실행하도록 한다. Trampoline은 후킹했던 함수의 Original Function을 호출하고 싶을
때 사용한다 (우측 그림의 oDIP 함수에 해당). Trampoline 코드에 후킹으로 인해
유실됐던 프롤로그를 복사한 후 Original Function+5[^1]로 점프하게 해 원본 함수를 사용할 수 있도록 한다.

Inline Hook에 대해 좀 더 자세한 정보는 [Detours: Binary Interception of Win32 Functions 논문](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/huntusenixnt99.pdf){:target="_blank"}
에서 확인할 수 있다. (관련 라이브러리 :
[MinHook](https://github.com/TsudaKageyu/minhook){:target="_blank"},
[Detours](https://github.com/microsoft/Detours){:target="_blank"})

간단하게 "hooked!\n"를 출력하는 함수로 후킹 하면 다음과 같은 형태가 된다.

```cpp
#define DRAWINDEXEDPRIMITIVE    82 // DrawIndexedPrimitive의 vtable index

DWORD FindPattern(DWORD dwAddress, DWORD dwLen, BYTE *bMask, char * szMask){
	// 생략
} // 2. 에서 언급한 FindPattern 함수

typedef HRESULT(WINAPI* tDrawIndexedPrimitive)(
	LPDIRECT3DDEVICE9 pDevice,
	D3DPRIMITIVETYPE pType,
	INT BaseVertexIndex,
	UINT MinVertexIndex,
	UINT NumVertices,
	UINT startIndex,
	UINT primCount
); // DrawIndexedPrimitive 함수 원형 선언
tDrawIndexedPrimitive oDrawIndexedPrimitive = NULL;

void *DetourFunction(BYTE *src, const BYTE *dst, const int len){ // inline hook을 설치하는 함수
	BYTE *jmp = (BYTE*)malloc(len + 5);
	DWORD dwBack;

	VirtualProtect(src, len, PAGE_EXECUTE_READWRITE, &dwBack);
	memcpy(jmp, src, len);
	jmp += len;
	jmp[0] = 0xE9;
	*(DWORD*)(jmp + 1) = (DWORD)(src + len - jmp) - 5;
	src[0] = 0xE9;
	*(DWORD*)(src + 1) = (DWORD)(dst - src) - 5;
	for (int i = 5; i < len; i++)
		src[i] = 0x90;
	VirtualProtect(src, len, dwBack, &dwBack);
	VirtualProtect(jmp, len, PAGE_EXECUTE_READWRITE, &dwBack);
	return (jmp - len);
}

HRESULT __stdcall hkDrawIndexedPrimitive(
	LPDIRECT3DDEVICE9 pDevice,
	D3DPRIMITIVETYPE pType,
	INT BaseVertexIndex,
	UINT MinVertexIndex,
	UINT NumVertices,
	UINT startIndex,
	UINT primCount
){
	// "hooked!\n"를 출력하는 함수
	printf("hooked!\n");
	return oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount);
}

void initHook(void) {
	DWORD hD3D = (DWORD)GetModuleHandle(L"d3d9.dll");
	DWORD addr = FindPattern(
		hD3D,
		0x128000, // d3d9.dll 모듈 크기, 버전별로 다를 수 있음
		(PBYTE)"\xC7\x06\x00\x00\x00\x00\x89\x86\x00\x00\x00\x00\x89\x86",
		"xx????xx????xx"
	);
	if (addr) {
		DWORD vtableAddress;
		memcpy(&vtableAddress, (void *)(addr + 2), 4);
		oDrawIndexedPrimitive = (tDrawIndexedPrimitive)DetourFunction(
			(PBYTE)vtableAddress[DRAWINDEXEDPRIMITIVE],
			(PBYTE)hkDrawIndexedPrimitive,
			5
		);
	}
}
```

주의해야 할 사항은 x86에서 `DrawIndexedPrimitive` 함수는 호출 규약이 호출된 함수가 스택을 정리하는 stdcall 이라는 점이다. 후킹 시 호출규약이 다르면 스택 오프셋이 달라지기 때문에 후킹 함수에 꼭 `__stdcall` 이나 `WINAPI` 를 선언하여 호출 규약을 지정해야 한다.

이제 위 코드의 `hkDrawIndexedPrimitive` 함수에 코드를 작성해 사물이 렌더링되는 시점에 원하는 코드를 실행할 수 있다.

<div class="row justify-content-center">
<div class="col-6">
<img src="/assets/images/2020-10-29/image_05.png">
</div>
</div>
<figcaption>후킹 전</figcaption>

<div class="row justify-content-center">
<div class="col-6">
<img src="/assets/images/2020-10-29/image_06.png">
</div>
</div>
<figcaption>후킹 후</figcaption>

&nbsp;

#### 5. 숨겨진 물체를 보이게 한다.

렌더링할 때 숨겨진 물체가 보이게 하는 순서는 다음과 같다.

1. Z Buffer 비활성화
    - DIP 함수를 통해 물체를 그리기 전에 Z Buffer를 비활성화해 물체가 벽 너머에서도 보일 수 있게 만든다.
2. `oDrawIndexedPrimitive` 호출
    - Z Buffer가 비활성화된 상태에서 물체를 그리기 위해 원본 `DrawIndexedPrimitive` 함수를 호출한다.
3. Z Buffer 활성화
    - Z Buffer를 다시 활성화해서 다른 물체가 정상적으로 그려질 수 있도록 한다.

Z Buffer를 활성화/비활성화 시 [SetRenderState](https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-setrenderstate){:target="_blank"} 함수를 사용한다. 함수의 원형은 다음과 같다.

```cpp
HRESULT SetRenderState(
	D3DRENDERSTATETYPE State,
	DWORD              Value
);
```

이 함수의 첫 번째 인자에 `D3DRS_ZENABLE`를 주고 두 번째 인자에 `D3DZB_TRUE` 또는 `D3DZB_FALSE` 주는 것으로 Z Buffer를 활성화/비활성화 할 수 있다. 

D3D9 기본 샘플에 적용해 보았다. 샘플은 Direct X SDK에 기본으로 설치되어 있는 `DirectX Sample Browser`나  `$DIRECT_SDK\Samples\C++\Direct3D\Bin\x86` 에서 찾을 수 있다.

```cpp
void __stdcall hkDrawIndexedPrimitive(LPDIRECT3DDEVICE9 pDevice, D3DPRIMITIVETYPE pType, INT BaseVertexIndex, UINT MinVertexIndex, UINT NumVertices, UINT startIndex, UINT primCount)
{
	pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_FALSE); // Z Buffer 비활성화
	// Drawing 전
	oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount);
	// Drawing 후
	pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_TRUE); //Z Buffer 활성화
}
```

<div class="row justify-content-center">
<div class="col col-md-6">
<img src="/assets/images/2020-10-29/image_08.png">
<figcaption>적용 전</figcaption>
</div>
<div class="col col-md-6">
<img src="/assets/images/2020-10-29/image_09.png">
<figcaption>적용 후</figcaption>
</div>
</div>

가려진 기둥과 바닥 속 물체가 보이는 걸 확인할 수 있다.

&nbsp;

#### Advanced

위 구현에는 어떤 물체를 보이게 할지에 대한 조건이 없다. 즉, 이 코드는 조건 없이 Z Buffer를 비활성화하므로 DIP 함수를 사용하는 모든 물체가 보이게 된다. 이렇게 된다면 우리가 원하는 대상뿐만 아니라 맵의 모든 오브젝트가 보이게 된다. 이 핵을 GlassWall 이라고 부르기도 한다.

그렇다면 원하는 물체만 보이게 할 수는 없을까? 우리는 이미 렌더링에 관한 모든 제어를 할 수 있다. 그러므로 렌더링하는 물체가 우리가 원하는 물체인지에 대한 조건만 추가한다면 원하는 물체만 보이게 구현할 수 있다.

물체를 구별하는 방법은 여러 가지가 있지만 간단하고 흔히 사용되는 방법은 Stride 값을 이용하는 것이다. 여기서 Stride 값이란 정점 버퍼 구조체의 크기를 가지고 있는 값이다. 따라서 현재 렌더링하고 있는 물체의 Stride 값을 구해 우리가 원하는 물체의 Stride 값과 같다면 Z Buffer를 비활성화하는 방식으로 구현할 수 있다.

물체의 Stride 값을 구하는 가장 쉬운 방법은 비교하는 Stride 값을 조금씩 증가시키면서 어떤 값일 때 우리가 원하는 물체가 표현되는지 직접 확인하는 방법이다. 정점 버퍼 구조체의 크기가 크지 않기 때문에 해당 방법을 이용하면 손쉽게 우리가 원하는 물체의 Stride 값을 구할 수 있다.

```c
HRESULT __stdcall hkDrawIndexedPrimitive(
	LPDIRECT3DDEVICE9 pDevice,
	D3DPRIMITIVETYPE pType,
	INT BaseVertexIndex,
	UINT MinVertexIndex,
	UINT NumVertices,
	UINT startIndex,
	UINT primCount
){
	IDirect3DVertexBuffer9* pStreamData;
	UINT iOffsetInBytes, iStride;
	pDevice->GetStreamSource(0, &pStreamData, &iOffsetInBytes, &iStride);
	if (iStride == 32) // Stride 값 비교
	{
		pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_FALSE); // Z Buffer 비활성화
		// Drawing 전
		oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount);
		// Drawing 후
		pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_TRUE); // Z Buffer 활성화
	}
	return oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount); // Stride 값이 우리가 원하는 값이 아니더라도 호출되어야 함
}
```

정점 버퍼 구조체와 Stride 값은 [GetStreamSource](https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-getstreamsource){:target="_blank"} 함수로 구할 수 있다.
그리고 Stride 값이 일치하지 않아도 물체는 그려져야 하므로 `oDIP` 함수는 항상 호출되어야 한다.

더 세부적인 조건으로 물체를 필터링하고 싶다면 DIP 함수의 `NumVertices`인자를 이용해 물체의 정점 개수를 확인하는 방법도 있다.

그런데 글을 읽으면서 '내가 본 월핵은 이게 아닌데? 특별한 색이 칠해져 있었는데?' 라는 생각이 들 수도 있다. 그리고 벽 뒤에 캐릭터가 보여도 이 캐릭터가 벽 앞에 있는 건지 벽 뒤에 있는 건지 구분하기가 어렵다. 그래서 핵 개발자들은 색이 있는 월핵을 개발하였다. 이를 해외포럼에서는 Chams라고 부르지만 국내에서는 '형광월핵'으로도 부르기도 한다.

<div class="row justify-content-center">
<div class="col-10">
<img src="/assets/images/2020-10-29/image_10.png">
</div>
</div>

<figcaption>형광월핵 (출처: <a href="https://www.unknowncheats.me/forum/cs-go-releases/121199-csgo-simple-chams.html" target="_blank">unknowncheats</a>)</figcaption>

Chams는 색을 입히기 위해 물체에 우리가 생성한 텍스쳐를 설정한다. 아래는 새로운 텍스쳐를 생성해 색을 입히는 코드이다.

```cpp
HRESULT GenerateTexture(IDirect3DDevice9* pD3Ddev, IDirect3DTexture9 **ppD3Dtex, DWORD colour32){
	if (FAILED(pD3Ddev->CreateTexture(8, 8, 1, 0, D3DFMT_A4R4G4B4, D3DPOOL_MANAGED, ppD3Dtex, NULL))){
		return E_FAIL;
	}
	WORD colour16 = 
		(WORD)(((colour32 >> 28) & 0xF) << 12)|
		(WORD)(((colour32 >> 20) & 0xF) << 8) |
		(WORD)(((colour32 >> 12) & 0xF) << 4) |
		(WORD)(((colour32 >> 4)  & 0xF) << 0;

	D3DLOCKED_RECT d3dlr;
	(*ppD3Dtex)->LockRect(0, &d3dlr, 0, 0);
	WORD *pDst16 = (WORD*)d3dlr.pBits;
	for (int xy = 0; xy < 8 * 8; xy++)
	{
		*pDst16++ = colour16;
	}
	(*ppD3Dtex)->UnlockRect(0);
	return S_OK;
}

bool generated = false;
LPDIRECT3DTEXTURE9 red, green;
HRESULT __stdcall hkDrawIndexedPrimitive(LPDIRECT3DDEVICE9 pDevice, D3DPRIMITIVETYPE pType, INT BaseVertexIndex, UINT MinVertexIndex, UINT NumVertices, UINT startIndex, UINT primCount)
{
	IDirect3DVertexBuffer9* pStreamData;
	UINT iOffsetInBytes, iStride;
	if (!generated) // 이미 빨간색, 초록색 텍스쳐가 생성되어 있으면 다시 생성할 필요가 없음
	{
		GenerateTexture(pDevice, &red, D3DCOLOR_ARGB(255, 255, 0, 0)); // 빨간색 텍스쳐
		GenerateTexture(pDevice, &green, D3DCOLOR_ARGB(255, 0, 255, 0)); // 초록색 텍스쳐
		generated = true;
	}
	pDevice->GetStreamSource(0, &pStreamData, &iOffsetInBytes, &iStride); // 물체의 stride값을 가져옴
	if (iStride == 32)
	{
		pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_FALSE); // Z Buffer 비활성화
		pDevice->SetTexture(NULL, red); // 물체를 빨간색 텍스쳐로 설정
		// Drawing 전
		oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount);
		// Drawing 후
		pDevice->SetRenderState(D3DRS_ZENABLE, D3DZB_TRUE); // Z Buffer 활성화
		pDevice->SetTexture(NULL, green); // 물체를 초록색 텍스쳐로 설정
	}
	return oDrawIndexedPrimitive(pDevice, pType, BaseVertexIndex, MinVertexIndex, NumVertices, startIndex, primCount);
}
```

`GenerateTexture` 함수는 D3D함수인 [CreateTexture](https://docs.microsoft.com/en-us/windows/
win32/api/d3d9/nf-d3d9-idirect3ddevice9-createtexture){:target="_blank"} 함수로 텍스쳐를 생성하는 함수이다. 후킹 과정은 다음과 같다.

먼저 `GenerateTexture` 함수로 빨간색, 초록색의 텍스처를 하나씩 생성한다. 현재 물체가 우리가 표현하고 싶은 물체라면 Z Buffer를 비활성화하고 oDIP 함수를 호출하기 전에 [SetTexture](https://docs.microsoft.com/en-us/windows/win32/api/d3d9helper/nf-d3d9helper-idirect3ddevice9-settexture){:target="_blank"} 함수로 물체에 빨간색 텍스처를 설정한다. 따라서 해당 물체는 벽 뒤에 가려진 부분을 포함한 모든 부분이 빨간색으로 나타난다. 다음으로 Z Buffer를 활성화하고 초록색 텍스처를 설정한 후 다시 oDIP 함수를 호출하면 시야에 보이는 물체의 부분만 초록색으로 그려지게 된다.

원뿔형 기둥의 stride 값과 정점의 개수를 구하여 다음과 같이 조건을 적용하였다.

```cpp
if (iStride == 32 && (NumVertices == 58 || NumVertices == 98))
```

다음 사진은 Chams를 적용한 모습이다.

![Chams](/assets/images/2020-10-29/image_11.png)

기둥의 가려진 부분은 빨간색으로, 보이는 부분은 초록색으로 나타난다.


## DirectX 11

DirectX 11(이하 D3D11)버전은 D3D9와 구조가 다르기 때문에 후킹하는 함수와 사용하는 원리가 조금 다르다. D3D9에서는 `DrawIndexedPrimitive` 함수를 후킹하였지만 D3D11에서는 `DrawIndexed` 함수를 후킹해야 한다. 또한 D3D11에는 `SetRenderState` 함수를 사용하여 Z Buffer를 비활성화하는 것이 불가능하므로 `D3D11_DEPTH_STENCIL_DESC` 구조체를 통해 Z Buffer를 비활성화 해야 한다. `D3D11_DEPTH_STENCIL_DESC`구조체의 `DepthEnable`필드를 False로 설정하면 깊이 테스트가 비활성화되어 엔진이 물체의 깊이를 구별할 수 없게 된다.

다음과 같은 과정을 거쳐 D3D11에서 월핵을 구현할 수 있다.

1. 기존의 Depth Stencil State를 가져온다.
    - [OMGetDepthStencilState](https://docs.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-omgetdepthstencilstate){:target="_blank"} 함수를 통해 가져올 수 있다.
2. 가져온 Depth Stencil State의 `DepthEnable`필드를 False로 설정한다.
    - `GetDesc` 함수를 통해 [D3D11_DEPTH_STENCIL_DESC](https://docs.microsoft.com/en-us/windows/win32/api/d3d11/ns-d3d11-d3d11_depth_stencil_desc){:target="_blank"} 디스크립터를 가져와 수정할 수 있다.
    - 수정한 디스크립터는 [CreateDepthStencilState](https://docs.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11device-createdepthstencilstate){:target="_blank"} 함수를 호출하여 새로운 Depth Stencil State로 생성할 수 있다.
3. Depth Stencil State를 교체한다.
    - [OMSetDepthStencilState](https://docs.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-omsetdepthstencilstate){:target="_blank"} 함수를 사용하여 새로운 Depth Stencil State를 현재 컨텍스트에 적용한다.

코드로 표현하면 다음과 같다.

```cpp
UINT ref;
ID3D11DepthStencilState *state;
D3D11_DEPTH_STENCIL_DESC desc;

pContext->OMGetDepthStencilState(&state, &ref);
state->GetDesc(&desc);
desc.DepthEnable = false;
pDevice->CreateDepthStencilState(&desc, &state);
pContext->OMSetDepthStencilState(state, ref);
```

D3D9와 마찬가지로 `DepthEnable` 필드를 활성화/비활성화하면서 원하는 물체만 표현될 수 있도록 해야 한다. 이 부분의 구현은 독자에게 숙제로 남겨두도록 한다 😎

## 마치며

지금까지 월핵에 사용되는 Direct3D 후킹의 원리를 분석하고 구현해 보았다. 게임핵이라 하면 보통 수치나 게임의 코드 등 메모리를 단순하게 변조하는 것이 연상되는데 월핵은 이와 달리 그래픽과 관련된 코드를 후킹하는 것으로 구현되는 것을 알 수 있었다.

다음 글에서는 최근 월핵보다 훨씬 더 많이 사용되고 있는 ESP 핵과 많은 FPS 온라인 게임들을 괴롭히는 Aimbot에 대해 다룰 예정이다.

[^1]: jmp로 패치한 후킹 코드의 크기가 5바이트이기 때문