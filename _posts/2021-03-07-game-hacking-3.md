---
layout: post
title: "게임핵의 원리에 대해 알아보자 (3) - 에임봇 편"
author: theori
description:
categories: [ research, korean ]
tags: [ game, cheats, hacks, aimbot, fps ]
comments: true
featured: true
image: assets/images/2021-03-07/image_00.png
---

에임봇(Aimbot)은 플레이어가 조준점을 움직이지 않아도 자동으로 적을 조준시키는 핵이다. 이 핵은 국내에서 '에임핵', '오토에임'으로 알려졌지만, 해외에서는 오래전부터 에임봇이라는 이름으로 사용되어 왔다. 그럼, 이 에임봇의 원리에 대해 알아보자.

에임봇은 구현 방식이 다양한데 게임 내 메모리를 읽어서 필요한 정보를 계산해서 조준하는 방식과 게임 화면의 이미지를 활용하여 타겟을 찾는 이미지 서치 방식 등이 존재한다. 다만, 이번 글에서는 이미지 서치 방식은 제외한다.

적 위치를 계산하는 방법에 따라 조준 방식을 크게 2가지로 나눌 수 있다.

- **각도를 계산하여 화면을 회전하는 방식**
- **화면 좌표를 계산하여 마우스를 이동하는 방식**

각각의 방식에 대해 자세히 들여다보자.

&nbsp;

## 각도를 계산하여 화면을 회전하는 방식

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="/assets/images/2021-03-07/image_00.png" width="100%">
<figcaption>Overwatch 게임 화면</figcaption>
</div>
</div>

위 그림에서 흰색 점이 조준점이고 초록 별이 조준할 대상이라고 하자. 각도를 계산하여 회전하는 방식은 화면상에서 조준점을 대상에 직선 이동하는 것이 아니라, 파란 화살표처럼 게임 내 화면(카메라)을 회전하는 방식이다. 이때 회전할 각도를 계산하는 것이 이 조준 방식의 핵심이다. 게임에는 화면 회전을 위해 카메라의 방향을 나타내는 값이 존재한다. 게임 메모리 내 화면 방향을 제어하는 값을 변조하면 화면을 회전시킬 수 있다. 임의의 각도로 이 메모리를 변조하면 화면이 회전되고 조준점이 이동된다. 이것이 각도로 화면을 회전하여 조준하는 과정이다.

먼저, FPS 게임이 화면을 어떻게 움직이는지 알아야 한다. 이를 위해 게임이 물체의 방향을 표현하는 방법을 알아보자.

&nbsp;

### 게임 내 물체의 방향 표현

게임 맵 상에서 물체가 바라보는 방향은 **오일러 앵글(Euler Angle)**로 표현할 수 있다. [사원수](https://ko.wikipedia.org/wiki/사원수){:target="_blank"}(Quaternion, 쿼터니안)라는 개념을 사용하기도 하지만 이 글에서는 오일러 앵글을 기준으로 다룰 것이다.

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="/assets/images/2021-03-07/image_01.png">
<figcaption><a href="http://www.chrobotics.com/library/understanding-euler-angles" target="_blank">http://www.chrobotics.com/library/understanding-euler-angles</a></figcaption>
</div>
</div>

**오일러 앵글**은 위 그림처럼 물체가 놓인 방향을 3차원 공간에서 표현할 수 있는 각도로서 yaw, pitch, roll이라는 축을 사용한다. 위 그림으로 예를 들어 어떤 게임이 오일러 앵글로 비행기의 방향을 표현한다고 하자. 비행기의 기체 앞쪽을 들면 pitch 축 값이 변하고, 자동차처럼 좌우로 회전시키면 yaw 축 값이 변하며, 날개를 회전시키면 roll 축 값이 변한다. 오일러 앵글은 이러한 축으로 물체의 방향을 표현한다.

게임은 카메라를 통해 물체를 화면에 표시한다. 카메라에 대한 설명은 [[게임핵의 원리에 대해 알아보자 (2) - ESP의 원리]](https://theori.io/research/korean/game-hacking-2/){:target="_blank"} 글에서 볼 수 있다. FPS 게임에서 카메라는 플레이어 위치에 존재한다.  마우스로 카메라를 회전시키면 플레이어가 바라보는 방향에 따라 게임 화면이 움직이게 된다. 위 그림의 비행기처럼 카메라도 방향을 표현할 수 있다. 사람에 비유하면 머리를 위아래로 움직이는 것은 pitch 축, 좌우로 돌리는 것은 yaw 축, 옆으로 기울이는 것은 roll 축에 해당한다. FPS 게임은 이렇게 카메라를 회전하여 화면 방향을 바꾼다.

게임상의 화면을 회전하기 위해서는 이 오일러 앵글로 표현된 화면 방향 값을 메모리 내에서 찾아 변조해야 한다. 이 값을 구하는 방법은 게임에 따라 다르므로 이 글에선 다루지 않을 것이다.

&nbsp;

### 회전할 각도 계산

적을 향해 회전할 각도는 두 벡터(플레이어와 적 좌표) 사이의 각도로서 삼각함수를 이용하여 구할 수 있다. 삼각함수는 기하학과 밀접한 관련이 있는 게임 프로그래밍에서 매우 흔히 사용되는 함수이다. 두 벡터가 이루는 각도를 계산하는 방법은 내적 등 여러 방법이 존재하지만, 이 글에서는 삼각함수를 이용하여 계산하는 방법을 다룰 것이다. 예시를 통해 삼각함수를 이용한 각도 계산 방법을 알아보자.

<div class="row justify-content-center mb-4">
<div class="col col-md-auto">
<img src="/assets/images/2021-03-07/image_02.png">
</div>
</div>

위 그림은 플레이어(파란 점)와 적(빨간 점) 플레이어를 게임 맵 위에서 바라본 좌표 평면 예시이다. 이 좌표계는 월드(맵) 좌표계로 플레이어는 맵의 (0, 0)에 위치해있다. 일반적으로는 원점이 아닌 사분면에 있지만, 핵심 원리 설명의 편의를 위해 원점으로 위치를 설정했다. 위 그림처럼 자신이 z 축 방향으로 바라보고 있다고 할 때 yaw 축 각도 값은 0이라고 하자. yaw=0을 기준으로 두 플레이어 사이가 이루는 각도 yaw를 안다면 회전 시켜 적을 향하게 할 수 있을 것이다. 이 회전시킬 각도 yaw를 구하는 것이 계산의 목표이다.

위 그림처럼 두 점의 위치 관계를 삼각형으로 표현하면 x, z 값은 각각 두 점 사이의 x, z 축 거리이다. 먼저 이 삼각형에서 삼각비 tan를 구해본다. tan는 삼각형의 높이와 밑변의 비율(높이/밑변)이다. 예제 삼각형에서 각도 yaw를 기준으로 tan 값을 구해보면 높이는 x, 밑변은 z이므로 x/z이 된다. 삼각함수를 사용하면 입력한 각도에 대한 삼각비 값을 알 수 있다. 삼각함수를 이용하여 각도 yaw의 tan를 구하면 `tan(yaw)=x/z`이 된다.

하지만 우리는 삼각비 값(x/z)이 아니라 회전할 각도 yaw를 구하는 것이 목표이다. 이 각도는 삼각함수 **tan의 역함수(역삼각함수)**인 **arctan**를 사용하여 구할 수 있다. 해당 역함수가 이 조준 방식에서 가장 중요한 함수이다. arctan 함수를 사용하면 삼각형의 변의 길이인 x, z 값만으로 화면을 회전할 각도(yaw)를 계산할 수 있다.

&nbsp;

### 역삼각함수를 이용한 각도 계산

tan 역함수를 이용하여 삼각형의 x, z 값으로 각도 yaw와 pitch를 구해보자. 화면을 회전하기 위해서는 이 두 축의 각도를 모두 구해야 한다. 설명에 사용할 용어는 다음과 같다.

- degree : 도, 일상에서 사용하는 각도 단위
- radian : 라디안, 수학에서 사용하는 각 크기를 나타내는 국제 표준 단위
- rad2deg : 라디안을 도(degrees)로 변환하는 함수

&nbsp;

### yaw 각도 계산

tan 역함수는 C 런타임 라이브러리에서 제공하는 [atan2()](https://docs.microsoft.com/en-us/cpp/c-runtime-library/reference/atan-atanf-atanl-atan2-atan2f-atan2l){:target="_blank"}로 사용할 수 있고 원형은 아래와 같다.

```c
double atan2(double y, double x);
```

y는 높이, x는 밑변으로 인자를 받는다. 예제의 삼각형에 `atan2()`를 적용해보자.

```js
atan2(1, sqrt(3)) => rad2deg(0.524 radians) => 30.0 deg
```

`atan2()` 함수에 높이 1과 밑변의 길이 sqrt(3)을 넣어 0.524 라디안이라는 결과를 얻었다.
일반적으로 수학에서 각도는 라디안이라는 국제 표준 단위를 사용하기 때문에 결과를 쉽게 확인하기 위해 rad2deg 함수를 통해 각도로 변환했다. 
0.524 라디안을 각도로 바꾸면 30도가 된다. 이 각도가 올바르게 구한 값인지 다시 tan 값을 구해 확인해 본다. tan(30도)는 1/√3이므로 높이 1, 밑변 √3인 예제 삼각형과 일치하는 것을 보아 올바르게 구한 것을 확인할 수 있다.

이를 통해  `atan2()` 함수를 사용하여 높이 x, 밑변 z만으로 각도 yaw를 구했다. 이것이 우리가 회전시킬 각도를 구하는 기본 원리이다.
z축을 기준으로 원점과 빨간 점이 이루는 각을 구했으므로 아래 그림처럼 yaw 축을 구한 각도 값으로 변경하면 적을 향해 조준할 수 있다.

&nbsp;

### pitch 각도 계산

yaw 값을 구했으므로 이제 pitch 축에서 회전할 각도를 계산해보자. yaw를 구했을 때와 같은 예제를 사용할 것이다.

<div class="row justify-content-center mb-4">
<div class="col col-md-auto">
<img src="/assets/images/2021-03-07/image_02.png">
</div>
</div>

yaw 각도 계산에서 그림을 다시 보면 xz 평면 상에서 적 플레이어는 위와 같이 있다.

<div class="row justify-content-center mb-4">
<div class="col col-md-auto">
<img src="/assets/images/2021-03-07/image_03.png">
</div>
</div>

이번엔 위 그림처럼 3차원상에서 적(빨간 점)이 xz 평면 위에 y만큼 떠 있고 플레이어는 x축 방향을 바라보고 있다고 가정한다. 이때 플레이어의 pitch 축 각도 값은 0이라고 하자. 플레이어는 적을 향하기 위해 그림의 각도 pitch만큼 회전해야 한다.

xy 평면의 삼각형을 보면 밑변의 길이는 xz 평면의 삼각형의 x일 것 같지만 실제로는 xz 평면 삼각형의 빗변인 d가 xy 평면 삼각형 밑변의 길이이다. 위 그림은 3차원상에서 2차원으로 투영시킨 모습이기 때문에 x로 보이는 것이다. 3차원 공간에서 보게 되면 아래 그림과 같게 된다.

<div class="row justify-content-center mb-4">
<div class="col col-md-auto">
<img src="/assets/images/2021-03-07/image_04.png">
</div>
</div>

d가 x축으로 투영됐기 때문에 xy 평면상에서 밑변의 길이가 x인 것처럼 보인 것이다. 3차원으로 보면 원래 pitch의 밑변은 xz 평면 삼각형의 빗변인 d이다. 여기서 d는 삼각형의 빗변 길이이기 때문에 피타고라스의 정리로 쉽게 구할 수 있다. 밑변 d의 길이는 `sqrt(x * x + z * z)`이므로 `sqrt(1 * 1 + sqrt(3) * sqrt(3))`인 2가 된다.

xy 평면 삼각형을 d로 다시 나타내면 아래 그림과 같게 된다.

<div class="row justify-content-center mb-4">
<div class="col col-md-auto">
<img src="/assets/images/2021-03-07/image_05.png">
</div>
</div>

우리가 구해야 할 각도 pitch도 tan의 역함수인 `atan2()`로 계산할 수 있다. 위 그림에서 높이는 y,  밑변은 d이므로 `atan2(y, d)`로 각도 pitch를 구할 수 있다. 

```c
d = sqrt(x*x + z*z) // 피타고라스 정리
atan2(y, d) => pitch 
atan2(2 * sqrt(3), sqrt(1 * 1 + sqrt(3) * sqrt(3))) => rad2deg(1.048 radians) => 60.0 degrees
```

위 코드로 계산해보면 예제 삼각형의 각도 pitch는 60도인 것을 알 수 있다. 특수각 tan(60)은 √3(2√3/2)이므로 각도를 올바르게 구한 것을 확인할 수 있다. 

이렇게 위치 좌표만으로 삼각함수의 역함수를 사용하여 플레이어와 적이 이루는 각도를 계산할 수 있다. 이제 메모리에서 화면의 방향 값을 계산한 각도로 변경하면 화면이 회전되고 적을 향해 조준할 수 있을 것이다. 

&nbsp;

### 구현 코드

다음은 계산한 각도로 화면의 방향 값을 변경하여 적을 조준하는 코드이다.

```cpp
// 오일러 앵글 구조체
struct Angle
{
    float yaw, pitch;
};

// 적과 자신이 이루는 각도를 계산하는 함수
void CalcAngle(D3DXVECTOR3 *enemy, Angle *angle) 
{
    GetMyPos(&my); // 자신의 좌표를 구하는 의사 함수
    D3DXVECTOR3 delta = *enemy - my; 
    float x, y, z, d;
    x = delta.x;
    y = delta.y;
    z = delta.z;
    d = sqrt(x*x + z*z); // 삼각형의 빗변 길이 (피타고라스의 정리)
    angle->yaw = atan2(x, z); 
    angle->pitch = atan2(y, d);
}

// 적 좌표를 받아 적을 향해 조준점을 변경하는 함수
void SetAim(D3DXVECTOR3 *enemy)
{
    Angle angle;
    CalcAngle(enemy, &angle);
    SetYaw(angle.yaw); // 자신의 yaw 값을 변경하는 의사 함수
    SetPitch(angle.pitch);  // 자신의 pitch 값을 변경하는 의사 함수
}
```

`SetAim()` 함수는 적 좌표를 받아 회전할 각도를 계산하고 화면의 방향 값을 변경하여 적을 조준한다.

내부에서 사용되는 `CalcAngle()` 함수는 `enemy`라는 적 좌표 벡터를 인자로 받고 계산한 각도를 `angle`이라는 구조체에 저장한다. 

`CalcAngle()` 함수를 보면 `enemy` 벡터와 `my` 벡터를 빼서 `delta` 벡터를 만든다. 여기서 적과 자신의 좌표를 빼는 이유는 플레이어와 적이 이루는 삼각형을 벡터로 표현하기 위해서이다. 이 delta 벡터는 점과 점 사이의 차이를 나타내므로 예제 삼각형의 x, y, z에 해당한다. 예제에서는 설명의 편의를 위해 자신을 원점으로 계산해서 좌표를 빼지 않았다. 하지만 실제로 회전시킬 각도는 적과 자신의 위치에 따라 상대적으로 달라진다. 이러한 이유로 좌표의 차이값을 표현한 벡터로 각도를 계산해야 한다.

그다음 `d = sqrt(x*x + z*z)` 로 pitch 값 계산에 쓸 삼각형의 빗변을 구한다. `atan2(x, z)`로 yaw값을  구하고 `atan2(y, d)` 로 pitch 값을 구할 수 있다. 

이제 `SetYaw()`와 `SetPitch()` 함수로 게임내 자신의 yaw, pitch 메모리를 변조한다. 계산한 yaw, pitch 값으로 변조하면 화면이 회전되고 조준점이 적으로 향하게 된다.

각도를 계산하여 화면을 회전하는 에임봇은 삼각함수로 화면을 회전할 각도를 계산한 후, 메모리를 변조하여 화면을 회전하는 방법으로 구현할 수 있었다. 이 방식은 적과 자신의 좌표만으로 조준할 각도를 계산할 수 있어 쉽게 얻을 수 있는 데이터로 계산할 수 있다는 장점을 갖고 있다. 각도를 계산해야 해서 어렵게 느껴질 수 있지만, 기하학 기초 지식으로 계산할 수 있고 계산에 필요한 데이터가 적어 구현 난도가 비교적 낮다. 

이제 다른 조준 방식을 알아보자.

&nbsp;

## 화면 좌표를 계산하여 마우스를 이동하는 방식

<div class="row justify-content-center">
<div class="col col-md-8">
<img src="/assets/images/2021-03-07/image_06.png" width="100%">
<figcaption>Overwatch 게임 화면</figcaption>
</div>
</div>

이 방식은 마우스를 화살표처럼 대상에 직선 이동하여 조준하는 방식이다.
적 좌표를 화면 좌표로 변환한 후 변환한 화면 좌표로 마우스를 이동하여 조준시킬 수 있다.
사전에 ESP를 구현했다면 이 과정을 간단히 구현할 수 있다. ESP의 화면 좌표 변환 과정이 사용되기 때문에 좌표 변환 구현이 선행되어야 한다. 좌표 변환에 대한 설명은 [[게임핵의 원리에 대해 알아보자 (2) - ESP의 원리]](https://theori.io/research/korean/game-hacking-2/){:target="_blank"} 글에서 볼 수 있다.

좌표 변환을 통해 적 좌표를 화면 좌표로 변환했다면 변환한 화면 좌표에 마우스를 이동해야 한다. 윈도우에서 입력 제어 API로 제공하는 `SendInpt()`를 사용한다.

```cpp
UINT SendInput(
    UINT    cInputs,
    LPINPUT pInputs,
    int     cbSize
);
```

<figcaption><a href="https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput" target="_blank">https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendinput</a></figcaption>

`SendInput()` API는 입력을 보낼 개수, 입력 정보가 포함된 버퍼, 버퍼의 개수를 인자로 받는다.

아래는 입력받은 화면 좌표로 마우스를 이동시키는 코드이다.

```cpp
void MoveCursor(int x, int y)
{
    x -= window.width / 2; // x -= 게임 화면 너비 / 2
    y -= window.height / 2; // y -= 게임 화면 높이 / 2
    INPUT input = { 0 };
    input.type = INPUT_MOUSE;
    input.mi.dwFlags = MOUSEEVENTF_MOVE; // 마우스를 상대적으로 이동
    input.mi.dx = x; // 이동할 x 좌표 지정
    input.mi.dy = y; // 이동할 y 좌표 지정
    SendInput(1, &input, sizeof(input)); // 현재 위치에서 x, y 좌표 만큼 마우스 커서를 이동
}

void MoveAim(D3DXVECTOR3 *enemy_world)
{
    D3DXVECTOR3 enemy_screen;
    WorldToScreen(enemy_world, &enemy_screen); // 적 월드 좌표를 화면 좌표로 변환
    MoveCursor(enemy_screen.x, enemy_screen.y); // 적 화면 좌표로 마우스 커서를 이동
}
```

`MoveAim()` 함수는 적 월드(맵) 좌표를 받아 화면 좌표로 변환하고, 변환한 화면 좌표로 마우스 커서를 이동 시켜 조준한다. 월드 좌표를 화면 좌표로 변환하는 과정이 필요하기 때문에 사전에 구현한 `WorldToScreen()`라는 함수를 사용하여 좌표를 변환한다. 변환한 화면 좌표는 `MoveCursor()` 함수에 전달된다.

`MoveCursor()` 함수는 입력받은 화면상의 적 좌표로 마우스를 이동시키는 함수이다. FPS 게임은 일반적으로 마우스 좌표가 게임 화면의 중앙에 고정되어 있기 때문에 중앙을 기준으로 마우스를 상대적으로 이동시켜야 한다. `window.width`, `window.height`는 사전에 구한 게임 화면의 너비, 높이 값이다. 인자 x, y에 화면 크기의 절반(`window.width/2`, `window.height/2`)을 빼서 화면 중앙(조준점)으로부터의 상대 좌표를 구한다. 마우스를 상대적으로 이동시킬 수 있는 `MOUSEEVENTF_MOVE` 플래그와 함께 `SendInput()` API로 마우스를 이동시키면 적이 위치한 화면 좌표에 조준점이 이동되어 적을 조준하게 된다.

이렇게 적 월드 좌표를 화면 좌표로 변환하여 마우스를 이동한 후 조준하는 에임봇을 구현할 수 있다.

이 방식은 마우스 제어 API를 사용하기 때문에 메모리를 수정하지 않아도 된다는 장점이 있다. 하지만 적의 월드 좌표를 화면 좌표로 변환하는 과정이 비교적 복잡하고 이 과정에 필요한 데이터가 많이 요구된다.

<hr/>

지금까지 알아본 두 조준 방식을 요약해 보면 아래와 같다.

**각도를 계산하여 회전하는 방식**

- 화면을 회전하여 조준한다.
- 삼각함수를 사용하여 회전할 각도를 계산한다.
- 메모리 변조가 필요하다.
- 적과 자신의 위치만으로 계산이 가능하다. → 계산이 쉽다.

**화면 좌표를 계산하여 이동하는 방식**

- 마우스를 이동 시켜 조준한다.
- 적 좌표를 화면 좌표로 변환해야 한다.
- 메모리 변조가 필요하지 않다. (마우스 제어 API 사용)
- 적, 자신의 위치, 그리고 화면 좌표 변환에 사용되는 데이터가 추가로 필요하다. → 계산이 복잡하다.

&nbsp;

## Advanced - Visible Check

지금까지 구현한 에임봇에 문제점이 존재한다. [[게임핵의 원리에 대해 알아보자 (1) - Wall Hack 편]](https://theori.io/research/korean/game-hacking-1/){:target="_blank"} 글에서 기본 월핵의 문제점과 유사하게 벽 뒤 플레이어를 구분하지 못한다는 것이다.

그래서 벽 뒤에 숨어 있는 적도 불필요하게 조준해서 벽을 가리키게 되는 현상이 나타난다. 적이 보이지 않는 벽을 강제로 조준하게 되면 오히려 에임봇을 사용하는 것이 더 불편할 것이다. 이런 문제를 개선하기 위해 적이 물체에 가려져 있지 않은지 조건을 추가해야 한다. 

플레이어가 물체에 가려져 있지 않은 상태, 즉 적을 볼 수 있는 상태를 visible이라 하고 이를 검사하는 과정을 visible check라고 부른다. 아래는 적의 visible 상태를 검사하여 에임봇을 작동시키는 예시 코드이다.

```cpp
if (CheckVisible(&enemy)) // enemy를 볼 수 있는지 확인
    SetAim(&enemy); // 조준점을 enemy에게 이동
```

`CheckVisible()` 함수는 visible 상태를 반환한다고 할 때 적을 명중시킬 수 있을 때만 에임봇으로 조준시킬 수 있다.

이 검사를 통해 에임봇을 화면에 보이는 적만 조준할 수 있도록 개선할 수 있다. 벽을 구분하는 방법은 게임 엔진에 따라 다양하지만, 이 글에선 간단하게 2가지 방법을 다루기로 한다.

&nbsp;

### 메모리 내 상태 값으로 검사

게임이 사용한 게임엔진과 구현 방법에 따라 적 플레이어의 상태를 저장하는 구조체에 적을 볼 수 있는지에 대한 여부가 저장되는 경우가 있다. 이 경우 간단하게 해당 데이터를 읽어 상대방을 볼 수 있는지 확인할 수 있다. 

하지만 이러한 데이터가 구조체에 없거나 구하기 힘들다면 다음 방법을 사용해야 한다.

&nbsp;

### 게임엔진의 함수로 검사

게임 엔진 내부에서 사용하는 충돌 검사 함수를 사용하여 visible 상태를 검사하는 방식이다. 내 위치에서 적이 보인다는 의미는 적과 플레이어 사이에 충돌이 가능하다는 것이기 때문에 충돌 검사 함수를 사용하여 visible 상태를 검사할 수 있다. 

예를 들면 아래 그림처럼 충돌이 발생하면 1, 충돌이 발생하지 않으면 0을 리턴하는 충돌 검사 함수가 있다.

<div class="row justify-content-center mb-4">
<div class="col col-md-6">
<img src="/assets/images/2021-03-07/image_07.png">
</div>
</div>

우리는 충돌 결과가 1이면 적이 보이는 것이고 0이면 보이지 않는 것으로 visible을 판단할 수 있게 된다. 

```cpp
bool CheckVisible(Player *enemy) 
{
 // 적 좌표와 자신의 좌표를 충돌 검사 함수에 넣어 충돌 결과 반환
 return CheckCollidable(&enemy->pos, &my_pos);  
}
```

이 방법은 위 코드처럼 게임 내 충돌 검사 함수( `CheckCollidable()`)를 직접 호출해서 사용한다. 충돌 검사 함수를 직접 호출해야 하기 때문에 게임 엔진의 함수 위치를 메모리에서 찾아야 사용할 수 있다. 

게임 엔진에 따라 충돌 검사 함수가 다르기 때문에 게임 엔진의 구조를 알아야 찾을 수 있다. 이러한 이유로 메모리 값으로 비교하는 검사에 비해 구현하기 어렵지만, 일반적으로 게임 엔진에는 충돌 검사 함수가 존재하기 때문에 항상 사용할 수 있는 방법이다.

&nbsp;

## Advanced - Magic Bullet

에임봇보다 발전된 이 핵은 발사되는 총알의 위치와 방향을 조작하여 명중시키는 핵이다. 멀리 떨어져 적이 보이지 않아도 죽일 수도 있다. 총알을 발사하거나 탄도를 처리하는 함수를 후킹하여 구현된다. 예를 들어, 총알을 발사할 때 출발 지점과 목적 지점을 함수에 전달하는 게임 엔진이 있다. 이때 이 함수를 후킹하여 출발 지점을 적의 머리 위로, 목적 지점을 적의 위치로 변조하게 된다면 어떻게 될까? 

<div class="row justify-content-center mb-4">
<div class="col col-md-6">
<img src="/assets/images/2021-03-07/image_08.png">
</div>
</div>

위 그림처럼 조준점이 적을 향하지 않아도 총을 발사할 때마다 탄환이 적의 머리 위에서 아래로 발사될 것이다. 아래는 이 과정에 대한 의사 코드이다.

```cpp
Fire(Vector3 start, Vector3 end); // Hook to hkFire();

hkFire(Vector3 start, Vector3 end)
{
    start = enemy[i]->pos + Vector3(0, 50, 0); // 적의 위
    end = enemy[i]->pos; // 적 위치
    origFire(start, end);	// 원래 Fire 함수
}
```

총알을 발사하거나 탄도를 처리하는 함수 또한 게임에 따라 다르다. 총알을 발사할 때 충돌 체크 함수를 이용하여 적과 총구가 충돌이 가능한지 여부로 발사를 처리하는 경우도 있다.

&nbsp;

## 마치며

지금까지 에임봇의 원리를 알아보았다. 에임봇 구현은 기초적인 기하학 개념이나 이전 글의 좌표 변환 과정이 필요했다. 또한 조준점만 이동시키는 것이 끝이 아니라 벽을 구분해야 한다는 문제점이 존재했다. 이 문제점은 게임 엔진의 충돌 검사 함수를 사용해서 해결할 수 있었고 이 함수를 찾기 위해 게임 엔진 구조를 파악해야 한다는 것을 알 수 있었다.

&nbsp;