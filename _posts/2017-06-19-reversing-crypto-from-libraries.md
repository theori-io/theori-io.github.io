---
layout: post
title: "Reverse Engineering Crypto Algorithms From Libraries"
description: 바이너리 코드로 컴파일 된 암호화 라이브러리에서 암호 알고리즘 추출하고 구현을 해보았다. 특히, 국내 표준 암호 알고리즘이지만 외부에 알려지지 않은 NEAT과 NES 알고리즘에 대해서 알아본다.
modified: 2017-06-19
category:
  - Research
  - Korean
featured: true
mathjax: true
---

한국에서 인터넷 뱅킹이나 홈택스 같은 업무를 보려면 ActiveX 컴포넌트부터 시작해서 매우 다양한 종류의 "보안" 솔루션들을 설치해야만 한다. 하지만, 이러한 보안 솔루션들이 정말로 유저들의 보안을 잘 지켜주도록 설계되고 구현되어 있을까? 과거에도 이미 몇 번 이러한 모듈들이 오히려 공격 표면이 되어 많은 사용자들을 위험에 빠뜨린 사건이 있었지만, 그 구조나 모듈 등이 그 이후로 크게 바뀌지 않았다는 생각이 든다.

공인인증서 관리 프로그램, 키보드 보안 프로그램, 개인 방화벽 프로그램 등 난무하는 이 보안 솔루션들은 얼마나 공격으로부터 유저를 보호할 수 있는지, 그리고 어떻게 해당 솔루션들을 우회하여 "허용되지 않은 플랫폼"에서도 웹서비스를 사용할 수 있는지에 대해서 의문을 가졌고 그에 대한 해답을 찾기 위해 해당 솔루션들을 살펴보기 시작했다.



### 보안 솔루션

거의 모든 곳에서 사용되고 있다고 해도 과언이 아닌 VeraPort 플러그인 매니져를 통해서 각 사이트나 부서별로 요구되는 보안 모듈들을 다운받아 설치하고 관리한다.

<img src="{{ site.url }}/images/2017-06-19/veraport.png">

Theori 한국지사의 주거래 은행인 KEB 하나은행에서 사용되는 보안체계가 얼마나 안전한지 궁금증이 들었기에, 제일 먼저 살펴본 프로그램은 한컴시큐어에서 만든 XecureWeb 컴포넌트였다. 이제 KEB 하나은행을 포함한 대부분의 국내 은행 사이트들은 개인 뱅킹의 통신구간 암호화를 위해 TLS 사용을 하는데, KEB 하나은행의 기업용 사이트는 여전히 HTTP로 연결되고 암호화나 복호화가 필요한 부분에서는 XecureWeb 모듈을 이용하는것을 발견했다.



#### XecureWeb

간단히 살펴본 바로는 XecureWeb은 다음과 같은 구조로 되어있다. 우선 사용자가 Internet Explorer를 통해 은행 사이트를 접속하게 되면 XecureWeb의 Active X 플러그인이 로드되고 해당 플러그인은 사용자의 컴퓨터에서 돌아가는 ClientSM.exe 이라는 클라이언트 세션 매니져 프로그램과 TCP를 이용해 통신한다. 이때, 세션 ID와 Path를 전송한다. 세션 ID는 기존에 핸드쉐이크를 진행하여 세션이 형성된 경우에 존재하지만, 첫 연결일 경우에는 아래 그림에 설명된 핸드쉐이크를 통하여 세션 ID 및 연동되는 SSL 파라미터 등을 생성한다. Path는 현재 접속하려는 URI다.

XecureWeb 클라이언트는 Xgate 서버와 SSL 통신을 하는데, 기존 SSL 프로토콜에서 조금 변형 된 형식을 사용한다. 정확히 말하면 클라이언트 HELO 메세지 패킷에 Path 문자열을 포함하는데 Path의 길이를 나타내는 2바이트 데이터와 Path 값이 추가된다. Xgate 서버는 보통 SSL 서버와 다르지 않은데, SSL 핸드쉐이크 과정에서 생성된 세션 ID에 해당하는 MasterKey를 생성한다. 다만 클라이언트가 보낸 Path와 세션 ID를 tuple로 특정 SSL 파라미터를 맵핑한다. SSL 파라미터는 서버와 클라이언트 각각에서 쓰일 수 있는 MAC Key, Encryption Key, 그리고 IV를 포함한다.

<img src="{{ site.url }}/images/2017-06-19/xecureweb_handshake.png">

아래에서 더 다루겠지만, 클라이언트는 지원하는 다양한 cipher suite 리스트를 보내지만 서버는 0x103 값에 해당하는 SEED-CBC/HAS-160 cipher suite만을 지원한다.

특정 Session ID와 Path에 대한 SSL Params가 정해지고 나면 Xgate 서버와의 통신은 끊는다. 여기서부터 데이터의 암호화 및 복호화는 XecureWeb 클라이언트가 담당을 한다. ActiveX 플러그인이 브라우저에서의 어떤 데이터를 암호화하고 싶다면 XecureWeb 클라이언트에 보내고, 클라이언트는 가지고 있는 key로 데이터를 암호화하여 반환한다. 이때 암호화는 SEED-CBC 알고리즘을 사용한다.

이 암호화 된 데이터를 HTTP로 전송하게 되는데, GET은 query string을 통해 전달되고 POST는 query string에는 Session ID 값만 들어가고 데이터는 POST 데이터에 들어가게 된다.

<pre>
GET: ?q=&lt;SessionId&gt;;&lt;MAC&gt;;&lt;EncryptedBlob&gt;
POST: ?q=&lt;SessionId&gt;
</pre>

얼핏 보기에도 여기저기 취약할것 같은 구조에 실제로 공격 가능한 시나리오도 금방 머릿속에 들어오지만 더 자세한 분석과 취약점 판단은 다음 기회에 하도록 하겠다.



#### KB & 신한

궁금증이 생겨 거래 은행사는 아니지만 국내에서 많이 사용되고 있는 KB와 신한은행 사이트도 잠깐 살펴봤다.

<img src="{{ site.url }}/images/2017-06-19/kbstar.png">

<img src="{{ site.url }}/images/2017-06-19/shinhan.png">

Chrome 브라우저로 접속하고 로그인 시도를 하려고 보니 콘솔에 에러가 잔뜩 생겼다. 보다시피 KB에서는 로컬호스트의 포트 16107에 HTTP POST 요청을 보내려다가 실패했고, 신한에서는 로컬호스트의 포트 30419에 웹소켓 접속을 하려다가 실패했다. (중간 중간 보이는 astxsvc 서버에 대한 접속 실패는 안랩의 보안 솔루션 엔드포인트 서버에 접속하려다 실패한것으로 보인다)

즉, 두 은행 사이트 모두 로컬호스트에서 어떤 에이전트가 특정 포트를 열어두고 돌고 있다는 것을 가정했다는 뜻이다. KEB와 마찬가지로 추가적인  취약점 분석이나 보안 프로그램 분석 등은 다음 시리즈로 미루겠지만, 이런것들을 우회하려면 javascript (익스텐션) 단에서 해당 기능들을 에뮬레이션 해줘야 한다는 것 정도는 짐작할 수 있었다. 다른 은행들도 지원하는지는 확인해보지 않았지만 KB 같은 경우에는 HTML5 표준을 사용한 브라우저를 통한 인증서 로그인 등을 구현해놓았는데, 올바른 방향으로 조금씩 가고 있다는 것을 느꼈다.

이쯤되면 제목과 본문의 괴리감을 느꼈을 수 있다 (라이브러리에서 암호 알고리즘 추출하는거라며?!). 원래 이 사이드 프로젝트의 목적은 위에 언급한대로 macOS에서 Internet Explorer가 아닌 Chrome 같은 브라우저를 사용하여 인터넷 뱅킹 사이트를 사용할 수 있도록 해주는 브라우저 익스텐션을 만드는것이었다. 하지만, 데이터 보호에 사용되는 암호화/복호화 알고리즘들을 보다보니 잠시 다른 길로 빠져버렸다.. 원래 목적은 다음(?)에 이루도록 하고, 해당 라이브러리가 지원하는 암호 알고리즘을 살펴보고 제목에 충실해보자.



### 암호 알고리즘

위에 간단히 설명한대로, 기본적으로 XecureWeb 클라이언트에서는 Xgate 서버와 통신할 때 변형된 SSL 프로토콜을 사용한다. 이와 관련해서는 이후에 따로 다룰 계획이지만, 사용된 cipher suite은 SEED + HAS-160 였다. 클라이언트에서 지원하는 다른 cipher suite을 나열해보니 (서버는 SEED + HAS-160 조합만 지원하였다), 이미 익숙한 AES, ARIA 등 사이로 다소 생소한 NEAT과 NES라는 이름이 보였다.

검색을 해보아도 SEED나 ARIA와 달리 많은 자료가 나오지 않았고, 그나마 구글에 돌아다니던 [발표자료](http://www.mathnet.or.kr/real/2006/12/KimByeonsu.ppt){:target="_blank"}에서 그 기원을 찾을 수 있었다. 1998년에 국내 표준으로 등록된 HAS-160이나 현재 국내 민간용으로 제일 많이 사용되고 있는 표준암호인 SEED (1999), 그리고 2004년에 한국 국가 표준 암호 ARIA가 제정되기 이전인 1997년에 NEAT은 국가기관용 표준암호로 제정되었다. 그리고 6여년 뒤인 2003년에 NES로 대체되었다.

<img src="{{ site.url }}/images/2017-06-19/block_cipher_history.png">

하지만, 보통의 암호 알고리즘들과는 다르게 구현이라던가 알고리즘 수식 등을 전혀 찾아볼 수 없었고 이 점이 매우 흥미로웠는데, 그리 오래 가지 않아 어떤 한 [블로그](http://m.blog.naver.com/kimsumin75/20052757958){:target="_blank"}에서 그 이유를 발견하게 되었다.

<img src="{{ site.url }}/images/2017-06-19/blog.png">

글쓴이에 따르면 국가기관용 표준암호인 NEAT는 비공개 알고리즘으로, 구조에 대한 스펙이나 코드가 공개되어 있지 않다. 바로 보안 필드에서 흔히 볼 수 있는 "Security through Obscurity"의 한 장면을 목격하고 있는 것이다. 이렇게 숨겨져 있으니 오히려 해커 본능(?)을 자극해서 어떤 암호 알고리즘이기에 공개조차 안했던 것인지 궁금해졌다. 누군가는 해당 암호 체계를 구현해서 사용했어야할 것이기 때문에, 각 알고리즘의 구현체들을 찾아나섰다.



### National Encryption AlgoriThm (NEAT)

1997년에 제정된 이 표준암호는 이후에 언급할 NES보다 구현체 찾기가 수월했다. 우리가 앞서 살펴보았던 XecureWeb에 포함된 암호 모듈에 구현되어있기 때문이었는데, SEED나 ARIA와 달리 소스코드가 존재하는게 아니기 때문에 바이너리 코드를 대상으로 리버스 엔지니어링을 해서 복구했다. NEAT는 1991년에 소개된 IDEA cipher와 구조적인 부분을 많이 공유하는데, 1994년에 만들어진 RC5에서 사용된 데이터 의존 로테이션 (data-dependent rotation)을 차용하기도 했다.

##### Operation

NEAT는 128-bit 블록 사이즈와 128-bit 키 사이즈를 가진다. 데이터를 변형하는 full-round 12번과 마지막으로 결과를 도출하는 half-round 한 번을 실행하는 총 12.5 라운드를 거쳐 암호화와 복호화를 진행한다. Fiestel 네트워크 기반 알고리즘으로 암호화와 복호화 루틴이 거의 동일하다 — 복호화 시에는 암호화 때와는 다른 키 스케쥴링을 먼저 진행해야한다.

IDEA와 마찬가지로 NEAT는 각기 다른 수학 [군](https://ko.wikipedia.org/wiki/%EA%B5%B0_(%EC%88%98%ED%95%99)){:target="_blank"}을 넘나드는 계산을 통해서 보안성을 높인다. 정확히 말하면 다음 연산들을 16-bit 단위로 처리한다:

* Bitwise XOR (다이어그램에서 ⊕로 표시)
* Addition modulo 2<sup>16</sup> (다이어그램에서 ⊞로 표시)
* Multiplication modulo 2<sup>16</sup>+1 (다이어그램에서 ⊙로 표시)
  * 입력 값에서의 0x0000은 0x10000으로 계산되고, 결과 값에서의 0x10000은 0x0000으로 계산된다.
* Rotation (다이어그램에서 &#8920;로 표시)

각 라운드에 사용되는 F와 그 역함수는 다음 구조를 가진다. 각 함수는 입력으로 들어오는 64-bit 데이터를 4개의 16-bit 데이터로 나눈 뒤, 위에서 설명한 연산들을 이용하여 계산한다. 여기서 사용되는 라운드 키 (K<sub>1</sub>, K<sub>2</sub>, K<sub>3</sub>, K<sub>4</sub>)는 키 스케쥴링을 통해 생성한다.

<img src="{{ site.url }}/images/2017-06-19/neat_f.png" style="width: 49%">
<img src="{{ site.url }}/images/2017-06-19/neat_fi.png" style="width: 49%">
<figcaption>F 함수와 F <sup>-1</sup> 함수</figcaption>

NEAT의 라운드는 다음과 같은데, 한 블록인 128-bit를 64-bit 두 개로 나눈 뒤 각각을 F와 F의 역함수인 F<sup>-1</sup>에 넣은 뒤 그 결과 값들을 MIX 함수를 통해 섞음으로서 복잡도를 높인다. 앞서 설명한대로 IDEA와 비슷한 구조를 가지지만, 입력 데이터에 의존하는 rotation 연산이 들어있다.

<figure>
<img src="{{ site.url }}/images/2017-06-19/neat_round.png">
<figcaption>NEAT Round</figcaption>
</figure>

MIX 함수는 다른 암호 알고리즘에서 쉽게 찾아볼 수 없는 구조였는데, 라운드 믹스 값에 따라 상수 테이블에서 값을 읽어들인 후 인덱스로 사용하여 입력값들을 서로 XOR 한다. (실제 구현은 코드 참고)

그리고 half-round인 마지막 라운드에서는 MIX 함수를 사용하지 않고 F와 F<sup>-1</sup>의 결과값을 스왑한다.

<figure>
<img src="{{ site.url }}/images/2017-06-19/neat_half_round.png">
<figcaption>NEAT Half-Round</figcaption>
</figure>

이로써 모든 작업이 끝나고, 마지막으로 16-bit로 나뉘어있는 데이터들을 바이트 배열로 전환한다.

##### Key Schedule

NEAT의 키 스케쥴링은 암호화 과정과 흡사하다. 암호 키를 데이터 블록으로 사용하여 7라운드를 진행하여 총 13개의 64-bit 라운드 키를 생성한다. 하지만, 보통 라운드에 사용되는 라운드키를 생성하는 과정이므로 라운드 키를 사용하는 곳에 아래의 상수 라운드 키를 사용한다.

{% highlight c %}
const uint16_t Ks[] = { 0xFD56, 0xC7D1, 0xE36C, 0xA2DC };
{% endhighlight %}

마지막으로 각 라운드의 MIX 작업에 사용되는 Round Mix 값은 생성된 라운드 키들을 XOR 연산하여 만든다.

##### Decryption

NEAT의 복호화 과정은 암호화 과정과 거의 동일하다. 물론, 사용되는 라운드 키의 순서와 값들을 뒤집어 (invert) 주어야 하고 Round Mix 값들 역시 거꾸로 뒤집어 준다. 이 상태로 기존 루틴을 실행해주면 된다. (MIX 함수의 특성상 복호화 때 스왑하는 값들도 순서를 바꾸어 주어야 한다 — 코드 참고)



### National Encryption Standard (NES)

NEAT가 표준으로 제정된지 6여년 뒤인 2003년에 새로운 국가기관용 블록 암호인 NES가 표준으로 제정된다. 이름에서도 살짝 엿볼 수 있듯이 1998년에 만들어진 AES를 많이 닮아있다. 그 한 예로, NES는 Feistel 구조 암호 알고리즘이었던 NEAT과는 다르게 Substitution-permutation Network (SPN — 대체-치환 네트워크) 구조로 되어있다는 점이 특징이다. 하지만, 2004년에 국가 표준 암호로 제정된 ARIA와 마찬가지로 2000년도에 AES의 후속작(?)으로 소개되었던 Anubis나 KHAZAD에서 사용된 Involutional SPN 구조로 설계됨으로서 SPN 구조에도 불구하고 복호화와 암호화 알고리즘에 거의 차이가 없다.

NES의 구현체를 찾는 일은 NEAT의 그것을 찾는것보다 조금 더 어려웠는데, 아무래도 더 최신 알고리즘이기도 하고 관리도 더 철저히 된 듯 하다. 웬만한 국내 암호 모듈들도 NES 알고리즘을 지원하는 버전은 따로 빌드하여 관리하는듯 싶다. 구현체를 찾아 헤매이던 도중 국내의 한 VPN 클라이언트의 암호 모듈이 NES를 지원하는것을 발견했다.

##### Operation

NES는 256-bit 블록 사이즈와 256-bit 키 사이즈를 가진다. NES의 연산들은 8x4 또는 4x8 column-major order 행렬 (matrix)로 나타낸 state를 상대로 이루어지며, 모든 계산은 [Rijndael's finite field](https://en.wikipedia.org/wiki/Finite_field_arithmetic#Rijndael.27s_finite_field){:target="_blank"}에서 이루어진다. 그래서 덧셈은 단순히 XOR이 되고, 곱셈은 irreducible polynomial인 $$ x^8+x^4+x^3+x+1 $$ modulo를 사용한다. 예를들어, 32 바이트 (256-bit) 블록을 8x4 행렬로 나타내면 다음과 같다. 또한, Substitution과 Permutation은 각각 S-Box와 P-Box를 이용해서 계산한다.

$$
\begin{bmatrix}
  a_{0} &  a_{8} & a_{16} & a_{24} \\
  a_{1} &  a_{9} & a_{17} & a_{25} \\
  a_{2} &  a_{10} & a_{18} & a_{26} \\
  a_{3} &  a_{11} & a_{19} & a_{27} \\
  a_{4} &  a_{12} & a_{20} & a_{28} \\
  a_{5} &  a_{13} & a_{21} & a_{29} \\
  a_{6} &  a_{14} & a_{22} & a_{30} \\
  a_{7} &  a_{15} & a_{23} & a_{31}
\end{bmatrix}
$$

NES 알고리즘은 크게 네 가지 스텝을 거치며, 첫번째와 마지막을 제외한 보통 라운드 (Round 1~11) 또한 네 가지 연산을 통해 데이터를 변형한다. 또한, NES는 두 개의 P-Box를 가지는데, 라운드가 짝수냐 홀수냐에 따라 각기 다른 P-Box를 사용한다.

<img src="{{ site.url }}/images/2017-06-19/nes_full.png" style="width: 32%">
<img src="{{ site.url }}/images/2017-06-19/nes_round.png" style="width: 32%">
<img src="{{ site.url }}/images/2017-06-19/nes_final_round.png" style="width: 32%">

<figcaption>NES 전체과정  |  NES Round  |  NES Final Round</figcaption>

1. Key Schedule (Key Expansions) — 암호키를 이용하여 13개의 32-byte 라운드 키를 생성해낸다 (총 416 바이트).
2. Initial Round (Round 0)
   1. AddRoundKey — state의 각 바이트가 해당 라운드 키의 각 바이트와 bitwise XOR 되어 저장된다.
3. Rounds (Round 1~11)
   1. Substitution — S-Box에 따라 state을 변형시킨다.
   2. Transposition — state를 4x8 행렬로 취급하고 행렬대치 (transpose) 시켜서 8x4 행렬로 변환한다.
   3. Permutation — P-Box의 값을 이용하여 행렬 곱셈을 한다.
   4. AddRoundKey
4. Final Round (Round 12 — Permutation 단계가 없음)
   1. Substitution
   2. Transposition
   3. AddRoundKey

다이어그램에서는 AES의 연산과 비슷하게 표현하기 위해 SubBytes와 MixColumns라고 표기했지만, Substition과 Permutation이라고 생각하면 된다.

Substitution과 Transposition 스텝을 수식으로 살펴보면 다음과 같다.
$$
\begin{bmatrix}
  S(a_{0}) &  S(a_{8}) & S(a_{16}) & S(a_{24}) & S(a_{4}) & S(a_{12}) & S(a_{20}) & S(a_{28}) \\
  S(a_{1}) &  S(a_{9}) & S(a_{17}) & S(a_{25}) & S(a_{5}) & S(a_{13}) & S(a_{21}) & S(a_{29}) \\
  S(a_{2}) &  S(a_{10}) & S(a_{18}) & S(a_{26}) & S(a_{6}) & S(a_{14}) & S(a_{22}) & S(a_{30}) \\
  S(a_{3}) &  S(a_{11}) & S(a_{19}) & S(a_{27}) & S(a_{7}) & S(a_{15}) & S(a_{23}) & S(a_{31})
\end{bmatrix}
^{\LARGE{T}}
=
\begin{bmatrix}
  b_{0} &  b_{8} & b_{16} & b_{24} \\
  b_{1} &  b_{9} & b_{17} & b_{25} \\
  b_{2} &  b_{10} & b_{18} & b_{26} \\
  b_{3} &  b_{11} & b_{19} & b_{27} \\
  b_{4} &  b_{12} & b_{20} & b_{28} \\
  b_{5} &  b_{13} & b_{21} & b_{29} \\
  b_{6} &  b_{14} & b_{22} & b_{30} \\
  b_{7} &  b_{15} & b_{23} & b_{31}
\end{bmatrix}
$$
예리한 독자는 발견했겠지만, Substitution된 결과 행렬을 그대로 Transposition을 적용하는것이 아니고 마치 8x4 행렬을 두 개의 4x4 행렬로 자른 뒤 이어붙여 4x8 행렬로 만든 듯한 행렬에 대해서 Transpose를 한다. 즉, 결과적으로 $$ b_{1} = S(a_{8}), b_{4} = S(a_{4}) $$ 가 된다.

Permutation 스텝은 P-Box를 행렬 곱셈하는데, 다음은 짝수 라운드에서 해당 과정을 진행하는 것을 수식으로 나타낸 것이다.

$$
\begin{bmatrix}
    3 & 7 & 5 & 2 & 5 & 4 & 2 & 3 \\
    3 & 3 & 7 & 5 & 2 & 5 & 4 & 2 \\
    2 & 3 & 3 & 7 & 5 & 2 & 5 & 4 \\
    4 & 2 & 3 & 3 & 7 & 5 & 2 & 5 \\
    5 & 4 & 2 & 3 & 3 & 7 & 5 & 2 \\
    2 & 5 & 4 & 2 & 3 & 3 & 7 & 5 \\
    5 & 2 & 5 & 4 & 2 & 3 & 3 & 7 \\
    7 & 5 & 2 & 5 & 4 & 2 & 3 & 3 \\
\end{bmatrix}
\begin{bmatrix}
  b_{0} &  b_{8} & b_{16} & b_{24} \\
  b_{1} &  b_{9} & b_{17} & b_{25} \\
  b_{2} &  b_{10} & b_{18} & b_{26} \\
  b_{3} &  b_{11} & b_{19} & b_{27} \\
  b_{4} &  b_{12} & b_{20} & b_{28} \\
  b_{5} &  b_{13} & b_{21} & b_{29} \\
  b_{6} &  b_{14} & b_{22} & b_{30} \\
  b_{7} &  b_{15} & b_{23} & b_{31}
\end{bmatrix}
=
\begin{bmatrix}
  c_{0} &  c_{8} & c_{16} & c_{24} \\
  c_{1} &  c_{9} & c_{17} & c_{25} \\
  c_{2} &  c_{10} & c_{18} & c_{26} \\
  c_{3} &  c_{11} & c_{19} & c_{27} \\
  c_{4} &  c_{12} & c_{20} & c_{28} \\
  c_{5} &  c_{13} & c_{21} & c_{29} \\
  c_{6} &  c_{14} & c_{22} & c_{30} \\
  c_{7} &  c_{15} & c_{23} & c_{31}
\end{bmatrix}
$$

행렬 계산으로 state을 변경한 후 라운드 키를 섞기 위해서 [c<sub>0</sub>, c<sub>1</sub>, …, c<sub>31</sub>]의 1차원 배열로 취급한다.

##### Key Schedule

NES의 키 스케쥴링은 암호키를 기반하여 위에서 설명한 라운드 연산 (Substitution, Permutation, AddRoundKey)을 이용하여 키 확장을 하여 라운드 키를 생성하고, 여기서는 홀수나 짝수의 라운드와 상관 없이 홀수용 P-Box의 inverse를 사용하여 Permutation을 진행한다. 또한, AES와는 다르게 상수 테이블을 사용하지 않고, 암호키와 inverse S-/P-box만 사용하므로 암호키 값의 영향이 크다.

##### Decryption

NES의 복호화는 앞서 설명한대로 Involutional SPN 구조로 설계되었기 때문에 암호화와 같은 프로세스 루틴을 사용한다. 다만, 복호화를 위한 키 스케쥴링을 해주어야 한다. 구조상 라운드 키들만 변형해주면 되는데, 첫번째와 마지막 라운드 키는 그대로 사용이 가능하다. 복호화 시에는 마찬가지로 S-Box의 inverse를 사용하며, 홀수 라운드키인지 짝수 라운드키인지에 따라서 각각 알맞은 P-Box의 inverse를 사용해야 한다.



### 마치며

##### 결론과 앞으로의 방향

우리는 국내 보안 솔루션에서 지원하는 암호 알고리즘들 중 공개되지 않은 국가 기관용 표준인 NEAT과 NES 블록 암호 알고리즘에 대해서 알아봤다. 비공개 알고리즘인 만큼 소스코드나 스펙을 찾을 수 없었기에 보안 솔루션에 탑재된 모듈들을 상대로 리버스 엔지니어링을 통해 분석했다. 바이너리 코드로 컴파일이 되는 과정에서 최적화 등 때문에 쉽게 이해할 수 없는 코드가 되어서 조금 고생하긴 했지만, 현존하는 다른 블록 암호 알고리즘과 많은 부분을 공유하고 있어서 알고리즘 재구현에 큰 어려움은 없었다.

수 많은 암호 알고리즘이 설계되고 보안성을 점검 받는다. 때에 따라서 예기치 못한 취약점이 발견되고 더이상 안전한 암호 시스템으로 사용되지 못하는 경우도 생긴다. 하지만, 그러한 검증을 위해서는 우선 알고리즘과 구조에 대해서 아는 것이 중요하다. 우리는 암호학자가 아니므로 해당 알고리즘들에 대한 cryptanalysis는 그 분야의 전문가들에게 맡기도록 한다 :)

이미 몇 번 언급한대로 최초 목표였던 브라우저 익스텐션을 통한 보안 솔루션 emulation은 달성하지 못했지만, 알려지지 않은 암호 스펙과 알고리즘을 알아내어 이해하고 구현까지 했다는데에 의의를 두도록 한다. 다음 번엔 다시 원래 목표로 돌아가서 국내 금융 기관에서 사용되는 보안 솔루션들의 구조와 취약점에 대해서 더 알아보도록 하겠다.

##### 테스팅 및 검증

비공개 알고리즘이기 때문에 우리의 분석이나 구현이 정확히 스펙과 맞아 떨어지는지 확인할 방법이 없다. 그렇기 때문에, 우리의 구현체가 적어도 보안 모듈들에 있는 코드와 같은 결과를 내는지에 대한 테스팅과 검증을 진행했다. 두 알고리즘 모두 Linux에서 테스트 했다. NEAT 같은 경우에는 XecureWeb의 리눅스 배포 버전에 포함된 모듈(.so)을 사용했고, NES는 SecureWorks TRUiN 패키지에 포함된 윈도우 용 DLL을 @taviso의 [loadlibrary](https://github.com/taviso/loadlibrary){:target="_blank"}를 이용하여 해당 함수들을 후킹해서 결과를 비교했다.

##### 오픈소스

NEAT과 NES를 구현한 C와 Python 코드를 [Theori GitHub에 오픈소스](https://github.com/theori-io/kr-crypto){:target="_blank"} 한다. 실제 구현 내용과 알고리즘에 대한 이해를 위해 (약간의) 주석을 보고 싶다면 C 코드를 참고하는 것을 추천한다. Python 구현체는 NEAT과 NES를 블록 암호 바탁으로 하는 간단한 CBC 모드를 구현해서 임의 길이의 메세지를 암호화하고 복호화 할 수 있다.

빌드 및 사용 방법은 같이 포함된 README 내용을 참고하기 바란다. 윈도우 DLL 테스팅에 쓰인 코드도 포함한다.

분석과 테스팅에 사용한 npXecureMacuxNPPlugin.so와 gpkiapi_wc.dll은 저작권의 문제로 직접 배포할 수 없으나, 해당사들이 배포하는 다음 패키지들을 통하여 추출할 수 있다.

* XecureWeb 리눅스 (Ubuntu) 64bit 버전: [https://spot.wooribank.com/pot/Dream?withyou=CQSCT0067](https://spot.wooribank.com/pot/Dream?withyou=CQSCT0067){:target="_blank"}
* TRUiN VPN 클라이언트: [https://vpn.ccn.go.kr/web/updatefiles/kr/truin-x64.exe](https://vpn.ccn.go.kr/web/updatefiles/kr/truin-x64.exe){:target="_blank"}
