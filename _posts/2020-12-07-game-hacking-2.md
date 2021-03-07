---
layout: post
title: "게임핵의 원리에 대해 알아보자 (2) - ESP 편"
author: theori
description:
categories: [ research, korean ]
tags: [ game, cheats, hacks, esp, bone, d3d ]
comments: true
image: assets/images/2020-12-07/image_00.png
---

**ESP**라고 불리는 이 핵은 적의 위치가 화면에 나타난다. ESP는 **ExtraSensory
Perception**, **초감각적 지각**이라는 뜻이다. 월핵과 유사해 보이지만 캐릭터
말고도 물체의 상태나 정보를 화면에 표시해주는 점에서 차이를 보인다.

<div class="row justify-content-center">
<div class="col-12">
<img src="/assets/images/2020-12-07/image_00.png">
</div>
<figcaption><a href="https://www.mpgh.net/forum/showthread.php?t=540031&page=6" target="_blank">https://www.mpgh.net/forum/showthread.php?t=540031&page=6</a></figcaption>
</div>

이 글에서는 ESP 핵의 기본 작동 원리를 알아보고 어떠한 방식으로 구현되는지
살펴본다.


## 기본 원리

ESP의 원리는 메모리를 읽어 필요한 정보를 얻은 후 화면에 표시하는 것이다. 여기서 필수적인 조건은 대상 프로세스의 메모리를 읽을 수 있어야 한다는 것이다. 고려해야 할 요소는 다음과 같다.

- 어떤 데이터를 얻어야 하는가?
- 메모리를 어떻게 읽는가?
- 어떻게 물체의 화면 좌표를 구해야 하는가?
- 어떻게 물체를 사용자에게 보여주는가?

## 어떤 데이터를 얻어야 하는가?

우리가 얻어야 하는 데이터는 게임 플레이를 용이하게 할 수 있는 정보들이다. 예를 들면 물체의 좌표(위치), 회전 방향(바라보는 방향), 닉네임, 생사 상태, 팀, 체력 등이 해당한다. 더 나아가 게임 특성에 따라 투명 상태인지 변신 상태인지 무적 상태인지 특정 스킬(궁, 필살기 등)을 사용 중인지도 알아야 한다. 이러한 데이터들은 메모리 내부에 구조체 혹은 클래스의 형태로 저장되어 있기 때문에 메모리 내에서 탐색을 통해 포인터를 따라가며 찾아내야 한다.

아래는 게임 데이터의 구조체 예시와 도식도이다. 실제 게임 해킹에서 이러한 데이터는 게임 바이너리 리버싱을 통해 알아내야 한다.

```cpp
struct Game
{                     // offset
	Player *players[4]; // 0x0
};

struct Player
{                         // offset
	char nickname[32];      // 0x0 , 닉네임
	float x,y,z;            // 0x20, 현재 좌표
	char pad1[4];           // 0x2C
	float sin_yaw, cos_yaw; // 0x30, 현재 바라보는 각도
	char pad2[4];           // 0x38
	Status *status;         // 0x3C
};

struct Status
{                   // offset
	float hp;         // 0x0, 현재 생명력
	char pad1[4];     // 0x4
	int is_god;       // 0x8, 1이면 무적 상태
	int is_invisible; // 0xC, 1이면 투명 상태
};
```

<div class="row justify-content-center" style="padding-bottom: 10px">
<div class="col-lg-6 col-md-8">
<img src="/assets/images/2020-12-07/image_01.png">
</div>
</div>

분석을 통해 어떠한 데이터를 읽을지 결정했다면 이제 해당 데이터를 실제로 읽어와야
한다.

&nbsp;

## 메모리를 어떻게 읽는가?

이 과정은 이 글의 핵심이 아니기 때문에 간략히 설명하도록 한다. 메모리를 읽는 방법은 크게 대상 프로세스 외부에서 접근하는 방식(이하 External)과 대상 프로세스 내부에서 접근하는 방식(이하 Internal) 2가지 방법이 존재한다. 각 방법은 메모리를 읽는 주체가 다르고, 각자 장단점을 갖고 있다.

### External

외부 프로세스 또는 커널에서 게임 프로세스의 메모리에 접근하는 방법이다. Windows에서 제공하는 API를 이용하여 구현할 수 있다. 구현 방법에는 크게 유저 모드 API를 사용하는 방법과 커널 모드 API를 사용하는 방법으로 나눠진다. 

그중 유저 모드 API를 사용한 구현 과정을 간략히 소개한다. 게임 프로세스의 Process ID를 구하고 `OpenProcess()`로 프로세스를 열어 Process Handle을 얻는다. 얻은 Process Handle을 이용하여 `ReadProcessMemory()`로 게임 프로세스의 메모리를 읽는다.

External 방법은 다음과 같은 특징을 갖는다.

- 메모리를 읽을 때마다 API를 호출해야 하므로 메모리 접근 속도가 비교적 느리다.
- 게임 프로세스 외부에서 실행되기 때문에 탐지될 요소가 많다.

External 방식을 통해 Game 구조체의 주소로부터 각 플레이어의 위치를 구하는 코드는 다음과 같이 작성할 수 있다.

```cpp
struct Game game;
uint64_t game_addr = <game 구조체 주소>;

// Mem.Read(uint64_t address, size_t length, void* output);
Mem.Read(game_addr, sizeof(struct Game), &game);

for(uint64_t i; i < 4; i++){
	struct Player player;
  uint64_t player_addr = (uint64_t)game.players[i];
	Mem.Read(player_addr, sizeof(struct Player), &player);
	printf(
		"player %d: pos(%f, %f, %f)\n",
		i, player.x, player.y, player.z
	);
}
```

&nbsp;

### Internal

일반적으로 DLL Injection 혹은 Code Injection을 통해 게임 프로세스 내부에서 임의의 코드를 실행 시켜 메모리에 접근하는 방법을 사용한다.

Internal 방법은 다음과 같은 특징을 갖는다.

- 핵 코드가 게임 프로세스 내부에서 실행되면서 게임과 같은 프로세스로 취급되기 때문에 게임의 메모리를 접근하기 쉽다.
- 메모리를 읽을 때 포인터(`*`)로 접근할 수 있어 개발하기 편하다.
- 메모리 접근 속도가 비교적 빠르다.

Internal 방식을 통해 Game 구조체의 주소로부터 각 플레이어의 위치를 구하는 코드는 다음처럼 작성할 수 있다.

```cpp
uint64_t game_addr = <game 구조체 주소>;
struct Game *game = (Game*)game_addr;

for(uint64_t i; i < 4; i++){
	printf(
		"player %d: pos(%f, %f, %f)\n",
		i,
		game->player[i]->x, game->player[i]->y, game->player[i]->z
	);
}
```

&nbsp;

## 어떻게 물체의 화면 좌표를 구해야 하는가?

가장 중요하고 복잡한 과정이다. 오브젝트의 좌표를 얻었지만, 이 좌표는 월드 좌표(게임상의 3D 좌표)이기 때문에 화면에 그릴 때 사용하는 화면 좌표(2D 상의 좌표)와 다르다. 그래서 이를 변환하는 과정이 필요하다.

이 과정이 World to Screen, 월드 좌표에서 화면 좌표로, 즉 3D에서 2D로의 변환이다.
좌표 변환이라는 것이 무엇인지 아래 그림을 통해 나타내 보았다.

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="/assets/images/2020-12-07/image_02.png">
<figcaption><a href="https://www.oreilly.com/library/view/webgl-up-and/9781449326487/ch01.html" target="_blank">https://www.oreilly.com/library/view/webgl-up-and/9781449326487/ch01.html</a></figcaption>
</div>
</div>

이 그림에 대해 이해하려면 먼저 카메라에 대한 개념이 필요하다. 우리가 현실에서 눈을 통해 물체들을 보는 것처럼, 게임 내에서도 물체를 보기 위한 '무언가' 가 필요하다. 바로 이 '무언가'를 카메라라고 한다. 게임 내 물체들은 이 카메라를 통해 화면에 보이게 된다. 위 그림에서 카메라는 눈 표시가 있는 지점에 존재한다.

따라서 물체들은 실제 위치가 아닌 카메라 기준으로 보이게 된다. 그래서 실제 위치만으로는 화면에 어디에 표시되어야 하는지 알 수 없다. 그렇기에 이런 실제 위치(월드 좌표)를 카메라 기준으로 어디에 보이는지(화면 좌표)로 변환하는 과정이 필요하다. 이를 변환하는 방법은 다양한데 이 글에서는 2가지를 다룰 것이다.

1. Direct 3D 함수를 이용한 좌표 변환
    - D3D 함수를 호출해 필요한 값을 얻을 수 있어 개발이 편리하다.
    - D3D 함수를 사용하기 위해 DLL 인젝션이 필요하다. 이에 따라 보통은 Internal로 구현된다.
2. 행렬을 이용한 좌표 변환
    - 좌표를 수동으로 직접 계산해야 하므로 좌표 변환에 필요한 값을 직접 찾아야 한다.
    - 좌표 변환에 필요한 값을 얻을 수만 있다면 External로 구현할 수 있다.

### **Direct 3D 함수를 이용한 좌표 변환**

월드 좌표를 화면 좌표로 변환하기 위해 사용하는 D3D 함수는 `D3DXVec3Project()`이다. 

```c
D3DXVECTOR3* D3DXVec3Project(
  _Inout_       D3DXVECTOR3  *pOut,
  _In_    const D3DXVECTOR3  *pV,
  _In_    const D3DVIEWPORT9 *pViewport,
  _In_    const D3DXMATRIX   *pProjection,
  _In_    const D3DXMATRIX   *pView,
  _In_    const D3DXMATRIX   *pWorld
);
```

<figcaption><a href="https://docs.microsoft.com/en-us/windows/win32/direct3d9/d3dxvec3project" target="_blank">https://docs.microsoft.com/en-us/windows/win32/direct3d9/d3dxvec3project</a></figcaption>

이는 게임상의 3D 좌표를 화면상의 2D 좌표로 변환해주는 핵심 함수이다. `pV` 의 월드 좌표가 변환되어 `pOut` 에 화면 좌표가 저장된다. 

인자로 들어갈 Viewport와 3개의 Matrix (`pProjection`, `pView`, `pWorld`)는 `GetViewport()`와 `GetTransform()` 함수를 이용해 얻을 수 있다. `GetViewport()`와 `GetTransform()` 함수는 디바이스의 메서드이기 때문에 디바이스 인스턴스(`pDevice`)가 필요하다. 따라서 `EndScene()`이나 DIP Hook 등의 방법으로 게임이 사용하는 `pDevice`를 얻어야 두 함수를 호출할 수 있다. 이에 관한 내용은 [[게임핵의 원리에 대해 알아보자 (1) - Wall Hack 편]](https://theori.io/research/korean/game-hacking-1/){:target="_blank"} 에서 알 수 있다.
`D3DXVec3Project()`는 단순 계산 함수이므로 디바이스 인스턴스가 필요하지 않다.

```c
typedef struct D3DVIEWPORT9 {
  DWORD X;
  DWORD Y;
  DWORD Width;
  DWORD Height;
  float MinZ;
  float MaxZ;
} D3DVIEWPORT9, *LPD3DVIEWPORT9;

HRESULT GetViewport(
  D3DVIEWPORT9 *pViewport
);
```

<figcaption style="padding-bottom:0"><a href="https://docs.microsoft.com/en-us/windows/win32/direct3d9/d3dviewport9" target="_blank">https://docs.microsoft.com/en-us/windows/win32/direct3d9/d3dviewport9</a></figcaption>

<figcaption><a href="https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-getviewport" target="_blank">https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-getviewport</a></figcaption>


viewport는 게임 화면의 크기, 위치 등의 값을 가진다. 화면 크기는 좌표 변환 시 실제 화면에 물체를 나타내는 투영과정에서 물체의 크기를 조절하는 데에 사용된다.  `GetViewport()` 함수는 이 viewport 값을 구해준다. 

```c
HRESULT GetTransform(
  D3DTRANSFORMSTATETYPE State,
  D3DMATRIX             *pMatrix
);
```

<figcaption><a href="https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-gettransform" target="_blank">https://docs.microsoft.com/en-us/windows/win32/api/d3d9/nf-d3d9-idirect3ddevice9-gettransform</a></figcaption>

이 함수는 변환에 필요한 게임상의 행렬 World Matrix, View Matrix, Projection Matrix를 구해준다. 이 행렬이 어떤 계산에 쓰이는지는 다음 파트에서 설명한다.

인자 `State`는 구할 행렬의 타입을 지정하는 값으로서 `D3DTS_VIEW` , `D3DTS_PROJECTION`, `D3DTS_WORLD` 등의 값을 가질 수 있다. 이 값에 따라 `pMatrix` 에 각각 View Matrix, Projection Matrix, World Matrix가 저장된다.

이렇게 `D3DXVec3Project()` 에 필요한 인자를 전부 구했다면 함수를 호출하여 월드 좌표를 화면 좌표로 변환할 수 있다.

```cpp
void WorldToScreen(
	D3DXVECTOR3 *world_pos, // input
	D3DXVECTOR3 *screen_pos // output
){
	D3DVIEWPORT9 vp;
	D3DXMATRIX proj_matrix, view_matrix, world_matrix;
	pDevice->GetViewport(&vp);
	pDevice->GetTransform(D3DTS_VIEW, &view_matrix);
	pDevice->GetTransform(D3DTS_PROJECTION, &proj_matrix);
	pDevice->GetTransform(D3DTS_WORLD, &world_matrix);
	D3DXVec3Project(screen_pos, world_pos, &vp, 
					&proj_matrix, &view_matrix, &world_matrix);
}
```

&nbsp;

### 행렬의 수학적 계산을 이용한 좌표 변환

이 방법은 행렬 변환(Matrix Transformation) 개념이 사용되고 이를 이용한 좌표 변환에는 2가지가 존재한다.

- View Matrix를 이용한 좌표 변환
    - View Matrix 값을 찾아야 한다.
    - Matrix에 변환에 필요한 값이 미리 계산되어 있으므로 계산 과정이 간단하다.
- 카메라의 방향을 이용한 좌표 변환
    - 카메라의 방향(Angle)과 위치(Location)만으로 계산이 가능하다.
    - 계산 과정이 비교적 복잡하다.

카메라의 방향을 이용한 좌표 변환 방법은 비교적 계산 과정이 복잡하고 컴퓨터 그래픽스 기초 지식이 많이 필요하므로 생략했다.

**View Matrix를 이용한 좌표 변환**

필요한 요소는 물체의 월드 좌표와 View Matrix이다. View Matrix는 변환 행렬로서 월드 좌표를 카메라 좌표계로 변환해준다. 월드 좌표와 View Matrix를 곱하면 카메라 좌표계로 변환할 수 있다. 따라서 View Matrix를 구하면 카메라 위칫값을 따로 구하지 않아도 되기 때문에 편리하게 월드 좌표를 화면 좌표로 변환할 수 있다. View Matrix를 구하는 방법은 D3D의 경우 View Matrix를 계산하는 `D3DXMatrixLookAtLH()` 함수를 후킹해 생성되는 값을 찾거나 메모리를 스캔해 게임 엔진이 저장해둔 View Matrix를 찾아내는 방법이 있다.

View Matrix를 구했다면 구한 행렬과 물체의 월드 좌표를 곱하여 카메라상의 x, y, z 좌푯값을 구할 수 있다. 식으로 나타내면 다음과 같다.

```cpp
x = world.x * view._11 + world.y * view._21 + world.z * view._31 + view._41;
y = world.x * view._12 + world.y * view._22 + world.z * view._32 + view._42;
z = world.x * view._13 + world.y * view._23 + world.z * view._33 + view._43;
w = world.x * view._14 + world.y * view._24 + world.z * view._34 + view._44;
```
이때 w라는 개념이 등장한다. w는 동차 좌표계([Homogeneous coordinates](https://en.wikipedia.org/wiki/Homogeneous_coordinates){:target="blank"})에서 사용하는 개념이다. 동차 좌표계란 좌표의 차원을 1차원 증가 시켜 표현하는 방식으로, 예를 들어 일반적으로 3차원을 x, y, z 좌표로 표현한다면, 동차 좌표계에서는 x, y, z, w와 같이 표현하는 방식이다. 일반적인 좌표계의 변환은 다음과 같이 이뤄진다.

```cpp
x, y, z -> x, y, z, 1 // 일반 좌표계 -> 동차 좌표계
x, y, z, w -> x/w, y/w, z/w // 동차 좌표계 -> 일반 좌표계
```

동차 좌표계를 사용하는 이유는 좌표변환에 있어서 편리함이 많기 때문에 사용하는 것이다. 이에 대한 상세한 내용은 컴퓨터 그래픽스에 더 관련 있는 개념이기 때문에 생략하도록 하겠다.

다시 좌표 변환으로 돌아와서, 이렇게 구한 (x, y, z, w) 좌표는 카메라상의 좌표계이다. 즉 카메라 기준으로 좌표가 얼마나 떨어져 있는지 표현한 값이다. 이를 다시 일반 좌표계 (x, y, z) 형태로 변환하고 물체로부터의 거리를 반영하여 원근 투영(Perspective Projection)을 해야 한다. (아래 그림 중 좌측)

<div class="row justify-content-center">
<div class="col col-lg-8">
<img src="/assets/images/2020-12-07/image_03.png">
<figcaption><a href="https://glumpy.readthedocs.io/en/latest/tutorial/cube-ugly.html" target="_blank">https://glumpy.readthedocs.io/en/latest/tutorial/cube-ugly.html</a></figcaption>
</div>
</div>

원근 투영은 삼각함수를 사용한 계산이기 때문에 계산 자체는 복잡하지 않다. D3D에서도 `D3DXMatrixPerspectiveFovLH()` 함수를 사용해서 원근 투영을 할 수 있다. 다만 이때 FOV(Field Of View, 시야각)와 화면의 가로세로 길이가 필요하다. 화면의 가로세로 길이는 쉽게 구할 수 있지만, FOV의 경우에는 게임을 리버싱하여 어디에 저장하는지 알아내야 한다. 이 과정까지 지나면 화면 좌표를 구할 수 있다.

&nbsp;

## 어떻게 물체를 사용자에게 보여주는가?

마지막으로 화면상으로 변환한 좌표를 적절한 형태(사각형, 뼈대 등)로 사용자에게 보여주어야 한다. 이를 구현하기 위한 방법은 GDI, DirectX, OpenGL 등 여러 가지 방법이 존재한다. 이 글에서는 DirectX를 사용하여 물체를 사용자에게 보여주는 방법을 살펴본다.

### EndScene 함수 후킹

D3D에서 렌더링을 마무리할 때 사용하는 함수인 `EndScene()`을 후킹해 그리는 방법이 있다. `EndScene()` 함수를 후킹하여 그리는 방법의 장점은 D3D 기능을 활용하여 편리하게 물체를 화면에 그릴 수 있다는 점이다. 단점으로는 게임 내 화면에 덮어 그리기 때문에 게임 화면이 게임사에 의해 모니터링된다면 탐지될 수 있다.

### Overlay

Overlay는 새로운 D3D 창을 생성해 게임과 겹쳐 보이도록 하는 기법이다. D3D를 이용해 물체 그리는 것은 동일하나 투명한 창이 생성된다는 점에서 `EndScene()` 함수를 후킹하는 방법과 차이를 보인다.

<div class="row justify-content-center">
<div class="col">
<img src="/assets/images/2020-12-07/image_04.png">
</div>
</div>

<div class="row justify-content-center">
<div class="col col-6 col-md-8">
<img src="/assets/images/2020-12-07/image_05.png">
</div>
</div>

아래는 새로운 창을 생성하는 코드이다.

```c
hWnd = CreateWindowEx(WS_EX_LAYERED | WS_EX_TRANSPARENT, wc.lpszClassName, "", WS_POPUP, rc.left, rc.top, s_width, s_height, NULL, NULL, wc.hInstance, NULL);
SetLayeredWindowAttributes(hWnd, RGB(0, 0, 0), 0, ULW_COLORKEY);
SetLayeredWindowAttributes(hWnd, 0, 255, LWA_ALPHA);
```

창을 생성할 때 `WS_EX_LAYERED` 와 `WS_EX_TRANSPARENT` 속성으로 창을 투명하게 하고 겹칠 수 있도록 한다. 그리고 `ULW_COLORKEY`와`LWA_ALPHA`로 특정한 색의 투명도를 지정할 수 있다. 

```c
d3d = Direct3DCreate9(D3D_SDK_VERSION);
// ...
d3dpp.BackBufferFormat = D3DFMT_A8R8G8B8;  
// ...
d3d->CreateDevice(D3DADAPTER_DEFAULT, D3DDEVTYPE_HAL, hWnd, D3DCREATE_SOFTWARE_VERTEXPROCESSING, &d3dpp, &d3ddev)
```

창을 생성한 이후 D3D를 사용해서 물체를 그리면 된다. 이때 유의해야 할 점은 D3D 디바이스를 생성할 때 투명한 물체를 그릴 수 있도록 `BackBufferFormat` 를 투명도를 지원하는 포맷으로 설정해야만 한다.

이 기법의 장점은 게임 화면과는 별도의 D3D 디바이스로 그려지기 때문에 게임 내 화면 모니터링을 피할 수 있다는 것이다.

## Advanced - Bone ESP

3D 모델은 뼈(Bone)로 구성되어 있고, 이 뼈를 보여주는 형태의 ESP를 Bone ESP라고 한다. 일반적인 ESP는 점이나 박스로만 이루어져 있는데, 이러한 ESP는 복잡한 3D 환경에서의 충돌을 정확히 구현할 수 없다. Bone ESP는 이 한계점을 극복하기 위해 실제 3D 모델의 Bone을 추적하는 것을 목표로 한다.

<div class="row justify-content-center">
<div class="col col-lg-8">
<img src="/assets/images/2020-12-07/image_99.jpg">
</div>
<figcaption><a href="https://www.quaddicted.com/webarchive/www.planetfortress.com/tf2models/tuto/ms3d_sc/tuto_ms3d_sc4.htm" target="_blank">https://www.quaddicted.com/webarchive/www.planetfortress.com/tf2models/tuto/ms3d_sc/tuto_ms3d_sc4.htm</a></figcaption>
</div>

메모리상에서 플레이어 구조체 근처에 뼈의 정보가 함께 저장되어 있을 확률이 높다. 뼈 좌표가 갖는 특징은 메모리 내 실제 좌푯값과 비슷하지만 렌더되는 환경에 따라 값이 상이하다는 점이다. 예를 들어 메모리상에서의 캐릭터의 좌표가 고정되어 있어도 모델상에서의 뼈는 환경, 플레이어의 액션, 모핑 액션에 의해 위치가 자주 변화한다.

뼈의 좌표를 찾았다면 보통은 Bone Index Logger를 사용하여 부위를 식별하는 작업을 거친다.

<div class="row justify-content-center">
<div class="col col-lg-8">
<img src="/assets/images/2020-12-07/image_06.png">
</div>
</div>

게임 내부적으로 뼈를 관리하는 과정에서 고유의 숫자가 붙는데, 이 숫자를 뼈의 위치에 그려주는 것이 Bone Index Logger다. 핵 개발자는 이것을 보고 각 뼈가 어느 부위에 있는지 식별할 수 있다. 이렇게 얻은 정보를 통해 각 뼈를 올바르게 이어주면 아래와 같이 Bone ESP가 완성된다.

<div class="row justify-content-center">
<div class="col col-lg-8">
<img src="/assets/images/2020-12-07/image_07.png">
</div>
<figcaption><a href="https://www.unknowncheats.me/forum/insurgency/126174-insurgency-little-offsets-4.html" target="_blank">https://www.unknowncheats.me/forum/insurgency/126174-insurgency-little-offsets-4.html</a></figcaption>
</div>

&nbsp;

# 마치며

이번 글에서는 ESP 핵이 어떻게 동작하는지 알아보았다. ESP 핵을 구현하기 위해서는 월핵과는 다르게 직접 물체의 월드 좌표를 구하고 화면 좌표로 변환한 후 화면에 그려주는 과정이 필요하다는 것을 알 수 있었다. 

다음 글에서는 많은 FPS 온라인 게임들을 괴롭히는 Aimbot에 대해 다룰 예정이다.
