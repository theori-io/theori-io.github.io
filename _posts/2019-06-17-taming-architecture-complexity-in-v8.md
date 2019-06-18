---
layout: post
title: "Taming architecture complexity in V8 (Translation)"
description: CodeStubAssembler (CSA)는 V8의 중요한 구성 요소 중 하나로 최근 자주 사용되고 있는 언어인 Torque와도 관련이 깊습니다. V8 블로그의 공식 문서를 보면서 CSA에 대해 알아봅시다.
modified: 2019-06-17
category:
  - Research
  - Korean
featured: true
imagefeature: false
---

CSA는 크롬 내부의 자바스크립트 엔진인 V8에서 사용되는 중요한 구성 요소입니다. 
V8의 속도 향상에 크게 기여하고 있으며, 최근 V8에서 내부적으로 사용되는 언어인 Torque와도 관련이 깊습니다.
그뿐 아니라 관련 코드에서 여러 버그 수정이 이뤄지고 있기도 합니다.
이 글은 V8 블로그의 글을 번역하고 주석을 달아 CSA에 대한 내용을 이해하기 쉽도록 국문으로 작성한 글입니다.
원문은 [이 링크](https://v8.dev/blog/csa)를 타고 들어가시면 볼 수 있습니다.

## Introduction

이 글에서는 V8의 구성 요소 중 지난 몇 번의 릴리즈에서 [큰](https://v8.dev/blog/optimizing-proxies) [성능](https://twitter.com/v8js/status/918119002437750784) [향상](https://twitter.com/_gsathya/status/900188695721984000)을 달성하는 데 유용한 도구였던 CodeStubAssembler (CSA)를 소개하고자 합니다. CSA는 V8 팀이 높은 안정성으로 저수준 Javascript 기능을 빠르게 최적화할 수 있게 해 팀의 개발 능력을 비약적으로 향상시켰습니다.

## A brief history of builtins and hand-written assembly in V8

V8에서 CSA의 역할을 이해하기 위해서는 CSA 개발에 이르기까지의 문맥과 역사를 이해하는 게 중요합니다. 

V8은 여러 기술들을 사용해 자바스크립트의 성능을 이끌어냅니다. 오래 실행되는 자바스크립트 코드에서는 V8의 [TurboFan](https://v8.dev/docs/turbofan) 최적화 컴파일러가 ES2015+ 기능의 속도를 높입니다. 그러나 V8의 기본 성능을 높이기 위해서는 짧은 자바스크립트 코드도 효율적으로 실행해야 합니다. 특히 [ECMAScript 사양](https://tc39.es/ecma262/)에 의해 정의된 모든 자바스크립트 프로그램에서 유효한 사전 정의된 객체에서의 builtin 함수에선 더 그렇습니다. `[1]`

역사적으로, 많은 builtin 함수들은 [self-hosted](https://en.wikipedia.org/wiki/Self-hosting_(compilers))되었습니다. 즉, V8의 개발자가 자바스크립트로 builtin 함수의 코드를 작성했습니다. Self-hosted builtin들은 성능 향상을 위해 V8이 사용자 제공 자바스크립트 코드를 최적화할 때와 동일한 메커니즘을 사용합니다. 사용자 제공 코드와 마찬가지로, self-hosted builtin들은 최적화 컴파일러에 의해 컴파일되기 때문에 타입 피드백을 수집하는 워밍업 페이즈를 필요로 합니다. `[2]`

이 기술은 특정 상황에서 꽤 좋은 성능을 끌어내지만, 이를 훨씬 더 발전시킬 수 있습니다. 가령, `Array.prototype`에 미리 정의된 함수들에 대한 정확한 semantic은 [표준 스펙의 자세한 부분](https://tc39.es/ecma262/#sec-properties-of-the-array-prototype-object)에 명시되어 있습니다. V8의 개발자들은 자주 발생하고 중요한 몇몇 경우들에 대한 동작과 사양을 정확히 알고 있으며, 이를 활용해 커스텀 함수를 만듭니다. 이렇게 만들어진 `최적화된 builtin`들은 첫 호출부터 최적화되어 있어 워밍업 페이즈를 거치거나 최적화 컴파일러를 호출할 필요 없이 대부분의 경우를 핸들링할 수 있습니다.

이러한 hand-written built-in 함수들의 성능을 최대한 끌어내기 위해 전통적으로 V8 개발자들은 최적화된 built-in 함수들을 어셈블리로 작성해 왔습니다. 어셈블리를 사용함으로써 V8 C++ 코드에서 비용이 많이 드는 호출을 피하고 자바스크립트 함수들을 호출할 때 내부적으로 V8의 레지스터 기반 [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)를 사용해 매우 빠릅니다.

이와 같은 이점으로 인해 V8은 여러 해 동안 `각 플랫폼마다` 수만 줄의 어셈블리 코드를 포함해 왔습니다. 이런 builtin들은 성능 향상에는 훌륭하지만 언어의 새로운 표준이 등장하게 되면 수정해야 하고, 이들의 유지보수와 확장 작업은 굉장히 힘들며 잦은 오류가 발생합니다.

## Enter the CodeStubAssembler

V8 개발자들은 수년 동안 hand-written 어셈블리 코드의 장점을 가지면서 유지보수하기 쉬운 builtin을 만드는 딜레마에 시달렸습니다.

TurboFan의 출현으로 인해 이는 가능하게 되었습니다. TurboFan의 백엔드는 저수준 기계 코드를 위해 크로스플랫폼 [IR](https://en.wikipedia.org/wiki/Intermediate_representation)을 사용합니다. 이 저수준 IR은 모든 플랫폼에서 좋은 코드를 생산하기 위해 명령어 선택기(instruction selector), 레지스터 할당기(register allocator), 명령어 스케쥴러(instruction scheduler)에 입력됩니다. 또한 백엔드는 어떻게 커스텀 레지스터 기반 ABI를 사용하고 호출하는지, 어떻게 기계수준 tail call을 지원하는지, 어떻게 leaf function에서 스택 프레임의 생성을 제거하는지와 같은 hand-written assembly builtin에서 사용되는 여러 기술들을 알고 있습니다. 이런 정보는 TurboFan의 백엔드가 V8의 다른 부분과 잘 결합되는 빠른 코드를 생성할 수 있도록 합니다.

이런 기능들의 결합으로 처음으로 hand-written 어셈블리 코드의 유지 보수 가능한 대안이 실현되었습니다. V8 팀은 TurboFan의 백엔드 위에 구축된 portable assembly를 정의하는 새로운 구성 요소를 만들었습니다. 이를 CodeStubAssembler (CSA) 라고 합니다. CSA는 자바스크립트를 작성/파싱하거나 TurboFan의 JS-specific한 최적화를 적용하지 않고도 기계수준 IR을 바로 생성할 수 있는 API를 제공합니다. 이 코드 생성을 위한 fast-path는 V8 개발자만이 내부적으로 엔진의 속도를 빠르게 하기 위해 사용할 수 있는 것이지만, 크로스 플랫폼에서 최적화된 어셈블리 코드를 생성하기 위한 이 효율적인 방법은 CSA로 만들어진 builtin을 사용한다면 성능에 민감한 V8 인터프리터의 바이트코드 핸들러인 [Ignition](https://v8.dev/docs/ignition)을 포함해 모든 개발자의 자바스크립트 코드를 빠르게 합니다. 아래는 CSA와 자바스크립트의 컴파일 파이프라인입니다.

<img src="{{ site.url }}/images/2019-06-17/pipeline.png">

CSA 인터페이스는 굉장히 낮은 수준의 연산을 포함하고 어셈블리 코드를 작성해본 사람이라면 누구나 익숙해질 수 있습니다. 예를 들어, "주어진 주소로부터 객체 포인터를 로드한다"나 "두 32비트 숫자를 곱한다"와 같은 연산을 포함하고 있습니다. CSA는 런타임이 아닌 컴파일 중에 여러 정확성 버그들을 잡아내기 위해 IR 수준에서 타입 검사를 진행합니다. 예를 들어, V8 개발자는 실수로 인해 메모리에서 로드한 객체 포인터가 32비트 숫자를 곱하는 명령의 입력으로 쓰이지 않는다고 확신할 수 있습니다. 이런 종류의 검증은 직접 작성한 어셈블리 코드에서는 불가능합니다.

## A CSA test-drive

CSA가 무엇을 제공하는 지 더 정확히 이해하기 위해 간단한 예제를 하나 들어보겠습니다. 우리는 객체가 문자열이라면 그 길이를 반환하는 새로운 builtin을 추가할 것입니다. 만약 입력으로 들어온 객체가 문자열이 아니라면 builtin은 undefined를 리턴합니다.

먼저, V8의 [builtin-definitions.h](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-definitions.h)의 `BUILTIN_LIST_BASE` 매크로에 새로운 `GetStringLength` builtin을 추가하고 하나의 인자를 가진다는 걸 상수 `kInputObject`를 통해 명시합니다.
```cpp
TFS(GetStringLength, kInputObject)
```
`TFS` 매크로는 `TurboFan builtin using standard CodeStub linkage`의 약자로, 코드를 생성하기 위해 CSA를 사용하고 인자는 레지스터를 통해 전달된다는 걸 의미합니다.

그리고 builtin의 내용은 [builtins-string-gen.cc](https://cs.chromium.org/chromium/src/v8/src/builtins/builtins-string-gen.cc)에 정의할 수 있습니다.

```cpp
TF_BUILTIN(GetStringLength, CodeStubAssembler) {
  Label not_string(this);

  // 첫 번째 인자로 정의한 object를 상수로 fetch
  Node* const maybe_string = Parameter(Descriptor::kInputObject);

  // 아래의 IsString 체크는 인자가 smi가 아니라 object pointer라고 가정하기 때문에
  // 먼저 입력이 Smi인지 검사하는 과정이 필요하다.
  // 만약 인자가 Smi라면, |not_string| label로 이동한다.
  GotoIf(TaggedIsSmi(maybe_string), &not_string);

  // Object가 String인지 확인하고, String이 아니라면 |not_string| label로 이동한다.
  GotoIfNot(IsString(maybe_string), &not_string);

  // CSA "매크로" LoadStringLength를 사용해 string의 length를 load하고 리턴한다.
  Return(LoadStringLength(maybe_string));

  // 위의 IsString 검사가 실패했을 때 이동할 label의 location을 정의한다.
  BIND(&not_string);

  // 입력으로 전달된 Object가 String이 아니다. Javascript Undefined 상수를 리턴한다.
  Return(UndefinedConstant());
}
```

위의 예제에서, 두 가지 종류의 명령이 사용되었습니다. 
먼저, `GotoIf`나 `Return`과 같이 직접적으로 하나 혹은 두 개의 어셈블리어로 바뀌는 `primitive` CSA 명령이 있습니다.
또, V8이 지원하는 칩 아키텍처 중 하나에서 찾을 수 있는 가장 자주 사용되는 어셈블리 명령에 대략적으로 대응되는 사전 정의된 CSA `primitive` 명령들의 고정 집합이 있습니다. 
다른 종류는  `LoadStringLength`, `TaggedIsSmi`, `IsString`과 같은 `macro` 명령들입니다. 
이들은 하나 혹은 그 이상의 `primitive`/`macro` 명령들을 인라인화하는 함수들입니다. 
매크로 명령은 자주 사용되는 V8 구현의 관용구들을 캡슐화해 쉽게 재사용할 수 있게 합니다. 
길이에 제한은 없으며 V8 개발자는 필요할 때마다 새로운 매크로 명령을 쉽게 정의할 수 있습니다.

위 코드를 추가하고 V8을 컴파일한 후, V8의 스냅샷을 준비하기 위해 builtin 함수를 컴파일하는 툴인 `mksnapshot`을 실행할 수 있습니다. `--print-code` 옵션을 전달하면 각 builtin에서 생성된 어셈블리 코드를 출력할 수 있습니다. 출력된 결과에 `GetStringLength`를 `grep`하면, x64에서 다음과 같은 결과를 얻을 수 있습니다. (가독성을 좋게 하기 위해 코드 출력을 정리했습니다.)
```asm
	test al, 0x1
	jz not_string
	movq rbx, [rax-0x1]
	cmpb [rbx+0xb], 0x80
	jnc not_string
not_string:
	movq rax, [rax+0xf]
	retl
```
32비트 ARM에선 `mksnapshot`을 통해 다음 코드가 생성됩니다.
```asm
	tst r0, #1
	beq +28 -> not_string
	ldr r1, [r0, #-1]
	ldrb r1, [r1, #+7]
	cmp r1, #128
	bge +12 -> not_string
	ldr r0, [r0, #+7]
	bx lr
not_string:
	ldr r0, [r10, #+16]
	bx lr
```
우리 builtin이 표준이 아닌 (적어도 C++에서는) 호출 규약을 가지고 있다 하더라도 테스트 케이스를 작성할 수 있습니다. 모든 플랫폼에서 builtin을 테스트하기 위해 아래 코드를 [test-run-stubs.cc](https://cs.chromium.org/chromium/src/v8/test/cctest/compiler/test-run-stubs.cc)에 추가할 수 있습니다.  

```c
TEST(GetStringLength) {
	HandleAndZoneScope scope;
	Isolate* isolate = scope.main_isolate();
	Heap* heap = isolate->heap();
	Zone* zone = scope.main_zone();

	// 입력이 string일 때의 case를 테스트
	SubTester tester(isolate, zone, Builtins::kGetStringLength);
	Handle<String> input_string(
		isolate->factory()->
			NewStringFromAsciiChecked("Oktoberfest"));
    Handle<Object> result1 = tester.Call(input_string);
    CHECK_EQ(11, Handle<Smi>::cast(result1)->value());

	// 입력이 string이 아닐 때의 case를 테스트 (e.g. undefined)
	Handle<Object> result2 = 
		tester.Call(factory->undefined_value());
	CHECK(result2->IsUndefined(isolate));
}
```

다양한 종류의 builtin을 위해 CSA를 사용하는 방법과 추가적인 예제들은 [이 위키 페이지](https://v8.dev/docs/csa-builtins)에서 확인할 수 있습니다. 

## A V8 developer velocity multiplier

CSA는 여러 플랫폼을 대상으로 하는 범용 어셈블리 언어 그 이상입니다. CSA는 그동안 해왔던 것처럼 각 아키텍쳐마다 손으로 코드를 작성하는 데 비해 훨씬 빠릅니다. 또한 이는 손으로 만든 어셈블리의 모든 장점을 챙기면서 개발자를 위험한 함정으로부터 보호합니다.

- CSA를 사용하면 개발자는 어셈블리 명령어로 직접 변환되는 저수준 primtive들을 이용해 builtin 코드를 작성할 수 있습니다. CSA의 명령어 선택기가 V8이 타겟팅하는 모든 플랫폼에 대한 코드의 최적화를 보장하기 때문에 개발자가 각 플랫폼의 어셈블리 언어를 자세히 알 필요가 없습니다.
- CSA 인터페이스는 저수준 어셈블리에 의해 조작된 값들이 코드의 작성자가 기대한 타입인지 보장하기 위해 여러 선택 가능한 type을 가지고 있습니다.
- 어셈블리 명령어의 레지스터 할당은 외부적으로 직접 하는 게 아닌 CSA가 자동화해서 처리합니다. 이는 스택 프레임 할당과 builtin이 사용 가능한 양보다 더 많은 레지스터를 사용하거나 call을 할 때 스택에 값을 저장하는 걸 포함합니다. 이렇게 하면 직접 작성된 어셈블리 builtin을 괴롭히는 미묘하고 찾기 어려운 버그들이 전부 제거됩니다. CSA는 생성된 코드를 덜 취약하게 만들면서 정확한 저수준 builtin을 만드는 데 걸리는 시간을 대폭 감소시킵니다.
- CSA는 ABI 효출 규약 (표준 C++ 및 V8 내부적 레지스터 기반 규약)을 이해하므로 CSA에서 생성한 코드와 V8의 다른 부분을 쉽게 상호 운용할 수 있습니다.
- CSA 코드는 C++이기 때문에 여러 builtin에서 재사용할 수 있는 `macro`의 일반적인 코드 생성 패턴을 캡슐화하기 쉽습니다.
- V8은 Ignition의 바이트코드 핸들러를 생성하기 위해 CSA를 사용하므로, 인터프리터의 성능 향상을 위해 CSA-기반 builtin들의 기능을 직접 바이트코드 핸들러에 인라인하기 쉽습니다.
- V8의 테스팅 프레임워크는 별도의 어셈블리 어댑터를 작성할 필요 없이 CSA의 기능과 CSA로 생성된 builtin들의 테스트를 지원합니다.

결국, CSA는 V8 개발의 흐름을 바꿨습니다. 이는 V8을 최적화하는 V8 팀의 능력을 크게 향상시켰습니다. 이는 V8 embedder을 위해 더 많은 Javascript 언어를 더 빨리 최적화 할 수 있게 되었습니다.

## 각주

`[1]` 이는 ECMAScript 표준에서 제공하는 여러 내장 함수들로, `Array.prototype.reverse`와 같은 함수들을 포함합니다.

`[2]` V8의 TurboFan 컴파일러는 최적화된 코드를 생성하기 위해 타입 정보를 수집합니다. 가령, 다음과 같은 코드를 최적화한다고 합시다.
```js
function add(a, b) {
	return a + b;
}
```
자바스크립트는 동적 타입 언어기 때문에 a와 b에 어떤 값이 올지 아무도 모릅니다. 그러나, `add` 함수가 두 개의 `smi`(V8에서 내부적으로 사용하는 정수 타입) 타입을 계속 입력으로 받아 호출된다면, TurboFan은 `add` 함수의 인자로 `smi`가 들어올 것을 가정하고 코드를 생성하게 됩니다. 이렇게 생성된 코드는 a와 b에 올 수 있는 모든 타입을 고려할 필요가 없어 매우 빨라지며, 인자가 `smi` 타입이 아닐 경우의 예외 처리 코드를 추가하는 것으로 안전을 보장합니다.
