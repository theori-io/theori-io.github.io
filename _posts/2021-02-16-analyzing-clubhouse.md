---
layout: post
title: "Analyzing Clubhouse for fun and profit"
author: theori
description:
categories: [ research, korean ]
tags: [ clubhouse, iOS, app, RTC ]
comments: true
featured: true
image: assets/images/2021-02-16/image_0.png
---

Clubhouse (이하 "클럽하우스")는 2020년 Alpha Exploration Co 사에서 개발한 오디오 채팅 형태의 소셜 네트워킹 앱입니다. 최근 한국에서도 인기를 더하며 초대장이 당근마켓에서 판매되거나 클럽하우스를 사용해보기 위해 중고 아이폰을 구매하는 사람들도 생길 정도였는데요. Elon Musk나 노홍철님과 같은 유명인사들도 가입해서 적극적으로 활동하면서 더 조명을 받은 것 같습니다. 이런 엄청난 인기 덕분에 얼마전 1조원 이상의 기업가치를 인정 받기도 했다는데, 부럽네요..

<div class="row justify-content-center">
<div class="col col-lg-3">
<img src="/assets/images/2021-02-16/image_0.png">
</div>
</div>

&nbsp;

어떻게 보면 그다지 새로울 것 없는 오디오 채팅 앱이지만, 직관적인 UI/UX 뿐만 아니라 다양한 배경을 가진 수 많은 사람들과 함께 현생에서 벗어나 허물없이 대화하고 토론할 수 있다는 점에서 특별하게 다가오는 것 같습니다. 하지만, 화려함 뒤에는 몇 가지 아쉬움도 있는데요. 우선 지금 당장은 iOS/iPad 앱만 존재하기 때문에 Android 유저들은 사용해볼 수 없다는 단점이 있고, 음성 데이터가 중국 서버를 통해 간다는 이야기도 떠돌아서 보안적 이슈에 대한 토론도 진행되었습니다.

클럽하우스의 Android 또는 웹/PC 버전을 만드는 것이 기술적으로 어려워서인지, 정말로 내 대화 내용이 중국 서버에 전송이 되는 것인지 궁금하지 않을 수 없습니다. 그렇다면, 두 눈으로 직접 확인해보면 되겠죠! 아래는 이번 설 명절에 코로나 때문에 어짜피 어디 가지도 못해서 집에서 하루 정도 살펴본 내용을 기반으로 작성한 분석 내용입니다. 시간을 많이 들이지 못해 깊게 본 것은 아니므로 참고용으로만 봐주세요 :)

&nbsp;

## Structure / Flow

클럽하우스는 크게 네 개의 컴포넌트로 이루어져 있습니다 (v0.1.27 기준).

1. Clubhouse App
2. Clubhouse API Server
3. Agora RTC Server
4. PubNub Server

<div class="row justify-content-center">
<div class="col col-lg-8">
<img src="/assets/images/2021-02-16/image_1.png">
</div>
</div>
<figcaption>Figure 1. Clubhouse App structure and flow diagram</figcaption>

클럽하우스는 아마도 단기간에 다자간 실시간 음성대화라는 복잡해질 수 있는 기능들을 자체적으로 모두 구현하는 방향보다는 현존하는 플랫폼들을 효과적으로 활용하는 전략을 세운것으로 보입니다. 그래서, 사용자 인증이나 클럽 관리 등을 제외하고 음성대화 기능과 실시간 상태변화 처리와 같은 코어 기능들은 3rd-party PaaS를 적극 활용하고 있습니다. 조금 더 구체적으로 살펴보면, 실시간 음성대화 기술인 RTC는 Agora.io 서비스를 활용하고 비동기 메세징 기술인 Pub/Sub은 PubNub 서비스를 활용합니다.

클럽하우스 iOS 앱은 클라이언트로써 동작하며 사용자에게 소셜네트워크 기능의 UI/UX를 제공합니다. 기본적으로 회원가입 및 로그인, 사용자/클럽 검색, 사용자 초대, 채널 (방) 목록 표시, 일정 관리, 팔로워/팔로윙 관리, 방 생성, 알람 등의 기능을 제공하고, 당연히 특정 방에 참여하였을 때 참여자 목록과 손들기와 같은 기능을 제공합니다. 앱은 HTTP 리퀘스트를 사용하여 클럽하우스 API 서버와 통신하는데, 조금 전에 예제로 든 기능들을 실제로 서버 사이드와 통신하여 state를 업데이트하는 역할을 합니다.

Agora 서버를 통해 RTC 세션을 맺기 위한 정보 교류가 끝나면, 음성 대화에 관련한 모든 부분은 Agora RTC SDK에서 관장하게 되고 대화 참여자에 대한 실시간 상태 변화는 PubNub SDK를 통해 다른 클라이언트들과 정보를 주고 받으며 업데이트 합니다.

##### Registration / Login Flow

- 전화번호 입력
- 문자로 전송 받은 인증코드 입력
- Wait list에 있는 상태인지 확인
  - 맞다면, 아이디 생성 및 대기 안내
  - 아니라면, 채널 목록 불러오기 및 표시
- 특이사항: Wait list 상태와 별개로 사용자 Token은 서버로부터 전달 받으며, Wait list에 속한 계정이라고 해도 일부 API는 사용 가능

##### Channel Join / Leave Flow

- 참여하고자 하는 방 선택
  - 방 세부 정보 및 참여자 목록 전달
  - RTC Token + PubSub Token 전달
- 실시간 대화 청취
  - RTC 채널 조인 및 채널 정보 Subscribe
    - 오디오 데이터 수신 + Playback
    - 참여자 정보 (join/leave 등) 교류
- 방 나가기
  - RTC 채널 destruction 및 PubSub 해제

##### Speaker / Moderator Flow

- 모더레이터가 참여자 지목하여 모더레이터 또는 스피커로 승격
  - 스피커 Role을 가진 RTC Token 전달 + 관련된 PubSub 채널 Subscribe
- 스피커 권한을 갖게 되어 오디오 데이터 송출 가능
- 모더레이터 권한으로 강퇴, 청취자로 변경 등 가능



이제 기본적인 구조를 이해했으니 각각의 컴포넌트를 조금 더 깊게 살펴보도록 하겠습니다.

&nbsp;

## REST API

API 서버를 포함한 클럽하우스의 거의 모든 엔드포인트는 Cloudflare 인프라 뒤에 위치하고 있습니다. 그래서 클럽하우스 API 서버 주소인 www.clubhouseapi.com을 향하는 대부분의 리퀘스트에는 `__cfduid` 쿠키가 붙어 있습니다. 또한, API 서버에 전송되는 리퀘스트에는 언어, 사용자 고유 ID, 앱 버전이나 기기 고유 값과 같은 클럽하우스만의 HTTP 헤더가 존재합니다. User-Agent도 마찬가지로 앱 빌드 번호를 포함한 커스텀 UA로 셋팅되어 보내집니다. 마지막으로, 모든 리퀘스트에는 가입/로그인 시 서버로부터 부여 받은 Token이 `Authorization` 헤더에 포함됩니다.

```
CH-Languages: en-US
CH-UserID: 1234567890
CH-Locale: en_US
CH-AppBuild: 297
CH-AppVersion: 0.1.27
CH-DeviceId: 7CAF8200-EC2B-4392-A62B-62D41AFB7648
User-Agent: clubhouse/297 (iPhone; iOS 14.4; Scale/2.00)
Authorization: Token ef1f1be31620226ea1dee33edfc6e3feecc5036f
```

각 API 엔드포인트는 https://www.clubhouseapi.com/api/를 base 주소로 하며, 엔드포인트 목록은 아래와 같습니다. 대부분의 엔드포인트의 역할은 이름으로부터 자명합니다.

##### API List (Total: 107, v0.1.27)

```
record_action_trails
start_phone_number_auth
call_phone_number_auth
resend_phone_number_auth
complete_phone_number_auth
check_waitlist_status
get_release_notes
get_all_topics
get_topic
get_clubs_for_topic
get_users_for_topic
update_name
update_displayname
update_bio
update_username
update_twitter_username
update_skintone
add_user_topic
remove_user_topic
update_notifications
add_email
get_settings
update_instagram_username
report_incident
get_followers
get_following
get_mutual_follows
get_suggested_follows_friends_only
get_suggested_follows_all
get_suggested_follows_similar
ignore_suggested_follow
follow
follow_multiple
unfollow
update_follow_notifications
block
unblock
get_profile
get_channel
get_channels
get_suggested_speakers
create_channel
join_channel
leave_channel
active_ping
end_channel
invite_speaker
uninvite_speaker
mute_speaker
make_moderator
accept_speaker_invite
reject_speaker_invite
invite_to_existing_channel
audience_reply
make_channel_public
make_channel_social
block_from_channel
get_welcome_channel
reject_welcome_channel
change_handraise_settings
get_create_channel_targets
update_channel_flags
hide_channel
get_notifications
get_actionable_notifications
ignore_actionable_notification
me
get_online_friends
search_users
search_clubs
check_for_update
get_suggested_invites
invite_to_app
invite_from_waitlist
invite_to_new_channel
accept_new_channel_invite
reject_new_channel_invite
cancel_new_channel_invite
add_club_admin
add_club_member
get_club
get_club_members
get_suggested_club_invites
remove_club_admin
remove_club_member
accept_club_member_invite
follow_club
unfollow_club
get_club_nominations
approve_club_nomination
reject_club_nomination
get_clubs
update_is_follow_allowed
update_is_membership_private
update_is_community
update_club_description
update_club_rules
update_club_topics
add_club_topic
remove_club_topic
get_events
get_events_for_user
get_events_to_start
delete_event
create_event
edit_event
get_event
```

&nbsp;

## Agora

Agora.io는 실시간 영상 및 음성 대화 플랫폼을 서비스로 제공하는 곳입니다. 중국과 캘리포니아에 기반을 둔 이 회사는 지난 6월에 나스닥 상장을 하였고, 클럽하우스의 배경 기술을 가진 회사로 주목 받았습니다. 특히, SD-RTN™ (Software Defined Real-time Network) 이라고 하는 자체적인 네트워크 인프라스트럭쳐를 통해 전세계적으로 초저지연 실시간 영상/음성 연결을 해주는 기술을 서비스 형태로 제공하고 있습니다. 또한, 속도 최적화를 위해 UDP를 사용합니다.

> All audio and video services provided by the Agora SDK are deployed and transmitted through the Agora SD-RTN™. Agora deploys over 250 data centers worldwide that use intelligent dynamic routing algorithms to achieve millisecond latency and ensure high availability of Agora's service.

플랫폼 지원도 매우 폭 넓은데, Android, iOS, macOS, Web, Windows 등 사실상 거의 대부분의 use case를 커버하는 SDK를 제공할 뿐만 아니라 Electron, Unity, React Native 등과 같은 프레임워크를 지원함으로써 개발자가 쉽고 빠르게 실시간 영상/음성 서비스를 제공할 수 있도록 합니다. 현재 웹 SDK는 4.x 버전이 개발되고 있고, 그 이외 플랫폼에서는 최신 버전이 3.3.0입니다. 클럽하우스의 경우에는 3.0.1.1 버전의 iOS용 SDK를 사용하고 있습니다.

최근 개발자 문서나 레퍼런스는 대부분 영어와 중국어로 작성되고 있으나, 이전 버전에는 중국어로만 된 문서와 코드 주석 등이 존재하는 것도 확인되었습니다. Agora의 서버코드는 오픈소스가 아니기 때문에 검토할 수 없었으나, 개발자 문서를 살펴보며 전체적인 구성을 이해하고 보안적으로 중요한 부분들을 최대한 확인해보았습니다.

##### App ID, App Certificate

Agora에서 어떤 앱에 대한 서비스를 처리해야하는지 알기 위해서 고유한 ID를 부여하는데, 이 App ID는 클럽하우스 앱 내에 내장된 `Info.plist` 에서 `AGORA_KEY` 라는 이름으로 저장되어 있습니다. App Certificate은 인증 토큰을 발급하기 위해 랜덤하게 생성된 문자열이며 앱 개발자가 개발자 센터에서 생성할 수 있습니다.

##### RTC Token Generation

Dynamic key라고도 불리는 Token은 사용자가 채널에 조인할 때 인증하는 용도로 사용하게 됩니다. Agora에서 dynamic key는 RTM (Real-time Messaging) 토큰과 RTC (Real-time Communication) 토큰, 크게 두 가지로 구성되지만 클럽하우스에서는 RTC 부분만 활용하게 됩니다. 이 토큰을 생성하는 것은 서비스 운영자 (즉, 클럽하우스) 서버에서 구현하게 되어 있는데, 각종 언어로 된 샘플 코드가 제공됩니다.

<div class="row justify-content-center">
<div class="col">
<img src="/assets/images/2021-02-16/image_2.png">
</div>
</div>
<figcaption>Figure 2. Token Generation (from Agora.io)</figcaption>

위와 같은 형태로 생성되고 사용되는데, 즉 클럽하우스 서버에서 생성되고 발급된 토큰을 클라이언트에서 받아서 Agora SDK를 사용해 직접 Agora 서버에 접속하는 방식입니다. 그렇다면, 이 토큰은 어떻게 생성될까요? Python3로 된 예제 코드는 [여기](https://github.com/AgoraIO/Tools/blob/master/DynamicKey/AgoraDynamicKey/python3/src/AccessToken.py)서 확인할 수 있습니다.

간단하게 설명하자면, 앱 ID, 채널 이름, 유저 ID를 비롯한 값들과 유저가 해당 채널에서 어떤 권한을 가지는지 명시하는 role (Publisher 혹은 Subscriber)을 유효기간과 함께 담아서 RTC 토큰을 생성하게 됩니다. 여기서 주목해야할 부분은 사용자가 임의로 role을 변경하지 못하도록 개발자에게만 알려진 App Certificate을 이용하여 HMAC을 계산한다는 점입니다.

```python
val = self.appID.encode('utf-8') + self.channelName.encode('utf-8') + self.uidStr.encode('utf-8') + m
signature = hmac.new(self.appCertificate.encode('utf-8'), val, sha256).digest()
```

위 `signature` 값이 토큰에 포함되며, 토큰을 클라이언트가 앱 서버로부터 받아서 Agora 서버에 보내는 사이 메세지가 변조되지 않았는지 여부를 확인하는데 사용됩니다. Agora 서버 코드는 공개되지 않았으므로 실제로 검증을 하는지 확인하기 어려우나, role에 해당하는 페이로드 부분만 변경하고 보내면 서버에서 실패 코드가 반환되는 것을 확인하였습니다.

##### Role-based Privilege Separation

모든 참여자가 같은 권한을 갖는 것이 아니라 오디오 송출 권한을 가진 Publisher (클럽하우스에서 Speaker에 해당)와 듣기만 가능한 Subscriber (클럽하우스에서 Audience에 해당)로 권한 분리가 되어있고 위에 언급한대로 이 권한은 발급 받은 토큰에 포함되어 있습니다. 클럽하우스 API를 통해서 Speaker 역할을 받을 때 해당 권한이 포함된 토큰을 받게되어 접속하게 됩니다. 클럽하우스에서 청중 목록에 있다가 스피커로 올라가게 되면 잠깐 동안 오디오가 끊겼다가 다시 접속되는 것 같은 현상을 경험하게 되는데, 이는 기존 세션은 종료하고 새 권한이 추가된 토큰으로 재접속하기 때문이라고 판단됩니다.

즉, Publisher role이 있어야만 오디오 데이터를 송출할 수 있기 때문에 단순 청중 참여자로 접속해서 임의로 오디오를 송출하는 트롤링은 하지 못하도록 되어 있습니다.

##### Separate Audio Stream

오디오 미디어 스트림은 채널에 접속해서 송출하는 사용자 별로 따로 전송받을 수 있어서, 각자 트랙을 녹음한 후 후처리로 mixing이 가능합니다. 오디오에 대한 codec은 모드에 따라 ILBC, SILK, NOVA, HVXC, AAC 등을 지원합니다.

##### Encryption

Agora.io에서는 데이터에 대한 암호화를 크게 두 가지 방법으로 지원합니다.

<div class="row justify-content-center">
<div class="col">
<img src="/assets/images/2021-02-16/image_3.png">
</div>
</div>
<figcaption>Figure 3. Agora Data Encryption</figcaption>

*Built-in 암호화*

- 단말에서 데이터를 암호화해서 보내고 다른 단말에서 데이터를 복호화하는 방식의 End-to-end (E2E; 단말간) 암호화를 지원합니다. 지원하는 암호 모드는 AES-128-XTS, AES-128-ECB, AES-256-XTS, SM4-128-ECB 이고, 암호화 키의 생성과 저장, 전달, 그리고 검증은 SDK 사용자 (즉, 클럽하우스)가 직접적으로 컨트롤 할 수 있습니다. 예를 들어, 클럽하우스 API 서버가 클라이언트에게 RTC 토큰을 생성하여 전달하듯 각 채널에 대한 암호화 키를 생성하여 전달하는 방식을 통해 Agora 서버에서는 데이터의 내용을 볼 수 없도록 할 수 있습니다.

*Custom 암호화*

- 만약 SDK 사용자가 원한다면 임의의 암호화 함수를 구현하여 사용할 수도 있습니다. 해당 함수는 오디오 데이터가 인코딩 되기 전에 적용되고, 받는 쪽에서는 디코딩 이후에 적용되는 방식으로 Agora 쪽에서는 (클라이언트를 리버스 엔지니어링 하기 전까지는) 암호화 방식 자체도 알 수 없도록 구현할 수 있습니다.

&nbsp;

## PubNub

PubNub은 실시간 커뮤니케이션 레이어를 제공하는 서비스입니다. 사실, 이 부분은 자세히 분석하지 않았으므로 많이 이야기 할 것은 없지만 RTC 토큰과 마찬가지로 클럽하우스 API 서버에서 접속하는 방에 대한 PubNub 토큰을 전달 받아서 Publish 및 Subscribe를 하고 데이터를 교환하게 됩니다. 특정 방에 참여하거나 누군가를 스피커 또는 모더레이터로 승격시키거나, 손을 들거나 하는 행위를 하는 것에 대해서 실시간으로 업데이트 하기 위해 사용되는 것 같습니다.

다음은 현재 PubNub 채널을 통해 오고가는 메세지의 action 목록입니다.

##### Action List (Total: 22, v0.1.27)

```
join_channel
leave_channel
add_speaker
remove_speaker
end_channel
make_channel_public
make_channel_social
reject_welcome_channel
make_moderator
change_handraise_settings
raise_hands
unraise_hands
invite_to_new_channel
accept_new_channel_invite
reject_new_channel_invite
cancel_new_channel_invite
invite_speaker
uninvite_speaker
reject_speaker_invite
accept_speaker_invite
remove_from_channel
mute_speaker
```

&nbsp;

## Security

클럽하우스 iOS 앱은 기본적으로 모든 HTTP 통신을 TLS 위에서 하기 때문에 API 서버와의 통신은 안전하게 암호화 되어 보호됩니다. 또한, Certificate Pinning을 적용하여 임의의 root certificate을 등록하는 단순한 Man-in-the-Middle (MITM) 방식을 통한 패킷 열람을 방지하였습니다. 하지만, Frida에 기반한 objection 툴을 활용하면 손쉽게 Certificate Pinning을 무력화하여 통신 내용을 확인할 수 있습니다.

현재 클럽하우스는 어떤 보안 공격에 노출되어 있을까요? 분석 자체를 워낙 급하게 진행했기 때문에 전수조사는 하지 못했지만, 위 내용을 이해하는 과정을 통해 클럽하우스에서 발생할 수 있는 보안 취약점들에 대해서 간략하게 정리해보았습니다.

##### 1. Potential Account Takeover

클럽하우스를 가입할 때 필요한 정보는 문자 메세지를 수신할 수 있는 유효한 전화번호 하나 입니다. 전화번호를 입력하고 진행하면 해당 번호로 4자리의 코드가 전송되게 되고 해당 번호를 입력하면 전화번호 인증이 완료됩니다. 위에서 설명한대로 가입 뿐만 아니라 로그인을 할 때도 사실상 같은 흐름을 가지게 되는데, 즉 공격자가 공격 대상의 전화번호와 로그인 과정에서 생성된 4자리 코드를 안다면 추가적인 검증 없이 바로 해당 계정을 획득할 수 있습니다. 4자리 코드는 10,000 가지의 경우의 수 밖에 없기 때문에 상황에 따라 보안적으로 매우 위험할 수 있는데요.

두 가지 공격 시나리오를 생각해볼 수 있습니다.

1. 공격 대상을 특정할 경우
   - 공격 대상의 전화번호를 고정하고 계속해서 4자리 코드를 brute-force 방식으로 시도하는 방법으로 인증에 성공할 때 까지 인증 시도를 합니다.
2. 공격 대상을 특정하지 않는 경우
   - (클럽하우스 계정과 연동 된) 다량의 무작위 전화번호에 대한 인증 시도를 합니다. 단, 각 번호에 대해서는 인증을 매우 적게 시도합니다.

각 시도는 일만 분의 일, 즉 0.01% 확률로 성공할 수 있으므로 약 7,000번 (또는 7,000개의 전화번호) 정도 시도하면 성공할 확률이 약 50%이며 이는 매우 현실적인 공격 가능성을 뜻합니다. 하지만, 위 시나리오는 클럽하우스에서 위와 같은 시도를 전혀 제한하지 않을 경우의 "이상적인" 상황에 속합니다. 모든 가능성을 테스트 해본 것은 아니기에 완벽하지 않을 수 있지만, 간단한 테스트를 통해 4자리 코드를 인증하는 부분에서는 다음과 같은 rate limit이 적용되어있는 것을 확인했습니다.

- 문자 한 번 당 시도할 수 있는 횟수는 **총 3회** 입니다. 3회 이상 잘못된 코드를 넣을 시, 기존 4자리 코드는 무효화 되며 문자 인증 과정을 다시 처음부터 진행해야 합니다.
- 같은 전화번호로 계속해서 인증에 실패할 시, 해당 번호에 대해 **30분 가량 문자 인증 기능이 정지** 됩니다.

그렇다면, 이런 제약사항을 고려했을 때 공격의 위험도는 얼마나 될까요? 매번 새로운 코드를 맞춰야 하는 것이 아닌 하나의 코드 당 3번의 시도를 할 수 있으므로 공격 성공 확률은 조금 높아지게 되지만, 연속해서 실패할 경우 해당 번호에 대한 문자 인증 기능이 일시적으로 정지되기 때문에 현실적인 위험도는 많이 낮아지게 됩니다. 하지만, 공격자가 인내심을 가지고 며칠에 걸쳐서 시도한다면 약 3주 정도만에 원하는 계정을 탈취할 수 있게 됩니다. 물론 피해자의 핸드폰 번호로 많은 인증코드 문자가 발송되어 꽤 눈에 띄긴 하겠네요.

공격자가 운이 정말 좋아서 사회적으로 영향력이 있는 정치인이나 기업대표, 혹은 연예인들의 계정을 그들이 자는 동안 탈취할 수 있다면 얼굴이 보이지 않는 소셜 네트워크 특성 상 큰 혼란을 야기할 가능성이 있겠군요. 목소리는 많이 발달한 머신러닝의 힘을 빌리는 방법도 있을 것 같고, 요즘 클럽하우스에서 매우 인기가 많은 성대모사 방의 장인들을 생각해보면 아찔하기도 합니다.

이 공격 가능성이 큰 문제가 되는 또 한가지 이유는 아래 3번 항목에서 이야기 할 "다중 로그인" 때문이기도 합니다. 공격자가 성공적으로 대상 계정을 인증하게 되면 피해자 입장에서는 다른 사람이 자신 계정에 성공적으로 로그인 해 있는지 알 수 있는 방법도 없고, 다른 기기에 로그인 된 세션들을 로그아웃 시키는 기능도 없기 때문입니다. 보다 안전한 계정 환경을 위해 클럽하우스에서는 다음과 같은 보완을 하면 좋을 것 같습니다.

- 인증코드 자리수 6자리 이상으로 증가
- 현재 로그인 된 세션 관리 기능 추가 (로그인 내역 열람 및 세션 강제 로그아웃)



##### 2. Unencrypted Voice/Data Channels

위에서 언급한대로 Agora에서는 데이터에 대한 암호화를 지원합니다. 하지만, 현재 클럽하우스에서 송수신하는 오디오 데이터와 컨트롤 데이터는 암호화 되지 않고 있습니다.

{:.text-center}
![image_4](/assets/images/2021-02-16/image_4.png)

채널 ID, 유저 ID, RTC 토큰, 내부 IP 등이 다수 패킷에 포함되고 RTC 메세지 또한 그대로 UDP 패킷에 노출되는 것을 확인할 수 있습니다.

{:.text-center}
![image_5](/assets/images/2021-02-16/image_5.png)

이러한 설정 때문에 도/감청 및 변조 등 보안적으로 문제가 발생할 수 있는데, 크게 두 가지 시나리오를 생각해볼 수 있습니다.

1. Stealth 도청/감청 + 트롤링
   - 패킷에 포함되는 정보만으로 해당 RTC 채널을 접속하여 음성 데이터를 수집할 수 있고, 스피커 권한까지 있는 토큰이라면 임의의 오디오 데이터를 보내서 트롤할 수 있는 위험성이 있습니다.
   - 특히, 클럽하우스 API를 거치지 않고 직접적으로 RTC 채널에 접속할 경우 클럽하우스 앱 UI에서는 청취자 혹은 스피커로 표시되지 않기 때문에 강제로 내보내는 것이 불가능합니다.
2. 데이터 변조
   - UDP 패킷이 암호화 되어 있지 않기 때문에 중간에서 ARP spoofing 등의 공격을 통해 패킷의 내용을 변조하여 오디오 스트림 데이터를 바꾸어 아예 다른 오디오가 들리도록 할 수도 있습니다.



##### 3. Miscellaneous

*Multiple login sessions*

위에서 이미 언급했지만, 현재 클럽하우스에서는 공격자가 성공적으로 대상 계정에 로그인 해도 피해자 입장에서는 누군가가 자신 계정에 성공적으로 로그인 해 있는지 알 수 있는 방법이 없고, 다른 기기에 로그인 된 세션들의 목록을 보거나 로그아웃 시키는 기능도 없습니다. 현재 로그인 된 세션 관리 기능 (로그인 내역 열람 및 세션 강제 로그아웃)이 추가되면 좋겠습니다. 본인 기기가 아닌 곳에서 로그인을 하셨다면, 꼭 로그아웃 하시기 바랍니다.



*Chinese Servers*

Cloudflare 프록시를 사용하기 때문에 정확한 IP의 획득이 어렵지만, 클럽하우스의 서버들은 모두 중국 이외의 지역에 있는 AWS 인스턴스에 존재하는 것으로 판단됩니다. 하지만, Agora SDK에서는 네트워크 latency를 가장 줄이는 최적의 경로를 찾기 위해서 다양한 서버들과 데이터를 주고 받는데 이 과정에서 중국에 위치한 서버들과 통신하는 것을 확인할 수 있었습니다.

<div class="row justify-content-center">
<div class="col">
<img src="/assets/images/2021-02-16/image_6.png">
</div>
</div>
<figcaption>Figure 4. Talking with Alibaba Server in Mainland China (112.126.96.46)</figcaption>

이에 대해서 클럽하우스 측은 지난 12일, 다음과 같은 답변을 내놓았습니다.


> For example, for a small percentage of our traffic, network pings containing the user ID are sent to servers around the globe -- which can include servers in China -- to determine the fastest route to the client. Over the next 72 hours, we are rolling out changes to add additional encryption and blocks to prevent Clubhouse clients from ever transmitting pings to Chinese servers. We also plan to engage an external data security firm to review and validate these changes.

요약하자면 트래픽의 일부 중 클라이언트에게 가장 빠른 경로를 찾아주기 위해 중국에 있는 서버를 포함한 세계적으로 포진해 있는 서버들에게 사용자 ID가 전송되었고, 클럽하우스 앱을 72시간 내에 추가적인 암호화를 적용하고 앱이 중국에 위치한 서버에는 아예 아무것도 전송하지 않도록 패치하도록 하겠다고 했습니다. 아마도 위에서 언급한 암호화를 적용하지 않을까 싶습니다.  ~~마지막에는 외부 보안 회사를 고용해서 새로 적용될 패치를 검증하겠다고 하는데, 저희에게 맡겨줬으면 좋았을 걸 그랬네요..허허.~~

하지만, 조금 충격적인 것은 위 답변 내용이 사실이 아닌 것으로 판단된다는 것입니다. 직접 진행한 실험 결과에 따르면 사용자 ID 뿐만 아니라 위에서 보안 이슈로 언급했던 암호화 되지 않은 RTC 토큰 등이 함께 전송되어 언제든지 해당 채널에 접속하여 오디오 스트림을 엿들을 수 있었기 때문입니다.



> These SDKs support network geofencing in the following regions: global (default), North America, Europe, Asia (excluding Mainland China), Japan, India, and Mainland China. Once a customer specifies a region using geofencing, no audio, video, or message can access Agora servers outside that region.

또한, Agora에서는 Geofencing을 지원합니다. 아마 이 기능을 활용해서 중국에 위치한 서버에는 아무것도 보내지 않도록 구현할 계획인것 같은데, SDK가 지원하는 기능만으로 해당 기능을 구현하기는 어려워보입니다. 왜냐하면, Geofencing에 대한 documentation을 잘 읽어보면 "After enabling geofencing, the Agora SDK only connects to Agora servers within the specified region." 구절이 있는데, 즉 Geofencing을 활용해서 특정 지역 (예: 북미, 유럽, 일본)에만 접속하게 할 수는 있지만 특정 지역 (예: 중국)을 제외한 다른 서버들은 허용하는 옵션은 없기 때문입니다. 하지만, 클럽하우스가 아마 최근 Agora 트래픽 지분의 대부분을 차지하고 있을테니 두 회사가 알아서 잘 구현하겠죠 ㅎㅎ

&nbsp;

## Conclusion

이 글을 통해 요즘 핫한 클럽하우스 서비스가 어떤식으로 설계되었는지, 구성은 어떠한지, 그리고 많은 사람들이 궁금해했던 보안 문제는 없는지 등에 대해서 조금이나마 설명이 되었기를 바랍니다. 위에서 다룬 것 이외에도 다양한 기능들이 존재하지만, 설 연휴 중 하루를 할애하여 궁금한 부분을 빠르게 해결하기 위해 선택과 집중을 했음을 양해 부탁드립니다. 또한, 분석한 내용을 기반으로 PC에서 구동 가능한 클라이언트를 작성하긴 했지만 법적 문제와 사용 시 밴 위험성은 항상 존재하기 때문에 공개하지 않습니다.

혹시라도 여유가 생겨서 더 살펴볼 기회가 있다면, 추후에 관련 내용은 업데이트 하도록 하겠습니다. 아마 암호화 로직이 들어간 다음이 되지 않을까 싶네요 :)

긴 글 읽어주셔서 감사합니다. 클럽하우스에서 보아요~👋 (@brianairb 팔로우 부탁드립니다!)

&nbsp;


---

&nbsp;

## Update #1: v0.1.28

블로그 글을 공개하고 보니 v0.1.28 버전으로 업데이트가 있어서, 빠르게 살펴봤습니다. 사실상 v0.1.27과 기능면에서 크게 달라진 것은 없는데, 위에서 언급한대로 Geofencing과 Encryption 관련 부분이 변경되었습니다.

##### Geofencing

클럽하우스 앱에서는 이제 `AgoraRtcEngineConfig` 클래스에 있는 `areaCode` 값을 0xFFFFFFFE (즉, ~0x1)으로 셋팅함으로써 연결 허용되는 지역에서 중국을 제외하게 됩니다. 사실 위에서는 한 지역만 제외하는 것은 어려울 것이라고 했는데, 이는 docs 문서에 있었던 문구만 보고 그렇게 생각을 했었습니다. 하지만, 실제로 API 함수에 대한 docs를 살펴보니 "You can use the bitwise OR operator (\|) to specify multiple areas." 라고 추가적으로 설명되어 있는 것을 발견하였습니다. 동시에 여러 지역을 설정할 수 있는 것이죠.

```objc
typedef NS_ENUM(NSUInteger, AgoraIpAreaCode ) {
   AgoraIpAreaCode_CN = ( 1 < < 0 ),
   AgoraIpAreaCode_NA = ( 1 < < 1 ),
   AgoraIpAreaCode_EUR = ( 1 < < 2 ),
   AgoraIpAreaCode_AS = ( 1 < < 3 ),
   AgoraIpAreaCode_GLOBAL = ( 0 xFFFFFFFF ),
};
```

위 정의를 보면 알 수 있듯 CN (중국) AreaCode는 0x1의 값으로 표현됩니다.



##### Encryption

아직 실제로 적용은 안된 것 같습니다만, 서버사이드에서 원하면 언제든지 활성화 할 수 있도록 지원하는 코드가 추가 되었습니다. API 서버에서 Encryption key를 전달받았을 때만 아래 소개할 코드 형태로 암호화를 활성화 하게 됩니다. 본문에서 설명한 내용 중 Built-in 암호화 방식을 사용하는데, 기존에 클럽하우스 API를 통해 RTC Token를 전달되는 것과 마찬가지로 Encryption key가 전달되고 해당 key를 사용해서 Agora RTC 채널 세션을 만들게 됩니다. 사용되는 Encryption Mode는 `AES256XTS`, 즉 AES-256-XTS 입니다.

```swift
let config = AgoraEncryptionConfig()
config.encryptionKey = encryptionKey
config.encryptionMode = .AES256XTS
agoraKit.enableEncryption(true, encryptionConfig: config)
```

즉, 해당 기능이 활성화되어 사용되면 암호화 키가 API 서버에서 직접 클라이언트로 제공되므로 Agora 서버에서는 암호화 된 데이터를 열람할 수 없게 됩니다. 마찬가지로, UDP 패킷으로 전송되는 데이터가 암호화 되고 기능이 활성화 된 후 직접 검증이 필요하긴 하겠지만 documentation 대로 SDK가 정말로 Agora 서버에게 절대 이 Encryption key를 보내지 않는다면 위에서 언급한 Public Wi-Fi 등의 시나리오에서도 도/감청 및 변조가 불가능해지게 됩니다.

&nbsp;

빠른 대처를 한 클럽하우스에게 박수를 쳐줘야할지, 이미 이런 기능들을 설계하고 구현해둔 Agora를 칭찬해야할지 모르겠네요. 물론 처음부터 이런 우려사항이 없도록 구현됐다면 가장 좋았겠지만, 둘 다 칭찬합니다!

&nbsp;