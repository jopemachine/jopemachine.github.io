---
layout: post
title: Raft Paper (In Search of an Understandable Consensus Algorithm) 번역 및 정리
subtitle: Paper Study
author: jopemachine
tags:
  - Paper Study
  - Raft
  - Distributed System
header-img: img/header-img/network.jpg
header-mask: 0.3
last-update: December 22, 2022
---

# Raft Paper 스터디

해당 포스팅은 [Raft 논문 - In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)을 개인적인 공부 목적으로 번역하고 정리한 것이다.

포스팅 내 본문의 순서는 논문의 순서와 일치하지 않을 수 있다.

<!-- [aioraft-ng](https://github.com/lablup/aioraft-ng)에 Log replication, Membership change 기능을 추가 구현하기 위해 해당 논문을 공부해보게 되었다. -->

계속 공부하면서 업데이트 중...

## Raft란?

[Raft](https://raft.github.io/)는 MongoDB, etcd 등 다양한 분산 시스템에서 사용되는 실용적인 [합의 알고리즘](https://ko.wikipedia.org/wiki/%ED%95%A9%EC%9D%98_(%EC%BB%B4%ED%93%A8%ED%84%B0_%EA%B3%BC%ED%95%99))이다.

기존의 [Paxos](https://ko.wikipedia.org/wiki/팩소스_(컴퓨터_과학)) 같은 합의 알고지즘에 비해 훨씬 이해하거나 구현하기 쉬우면서 (9.1절 참고) 이에 필적할만한 성능을 제공한다. 특히 이해 가능성(Understandability)은 Raft 알고리즘의 가장 중요한 목표이기도 하다 (4절 참고).

Raft는 크게 두 가지 접근을 통해 이해 가능성을 달성한다.

1. **문제를 나눠서 접근하기**: `Leader Election`, `Log Replication`, `Membership changes`.

2. **고려해야 하는 상태의 수를 줄여 상태 공간 단순화 하기**.

## Raft만의 특징

Raft는 다른 분산 시스템 알고리즘들과 비슷하게 작동하지만 아래는 Raft만의 특징들이다.

- `Strong Leader`: Raft는 다른 합의 알고리즘들에 비해 리더에게 더 많은 권한을 부여한다. 예를 들어 로그 엔트리들은 리더에서 다른 팔로워 서버들로 단방향으로 흐르게 된다. 이러한 단방향 흐름은 복제된 로그들을 관리하기 수월하고, Raft 알고리즘을 이해하기 쉽게 만들어준다.

- `Leader Selection`: Raft는 리더를 선출할 때 무작위 타이머를 사용한다. 무작위 타이머는 아주 약간의 구현 추가로 충돌 문제를 단순하고 빠르게 해결해준다.

- `Membership Changes`: Raft의 메커니즘은 클러스터 내 서버들을 변경할 때, 업데이트 기간 동안 서로 다른 설정들이 overlap 되는, `Joint consensus`라는 새로운 접근 방식을 사용한다. 이러한 접근 방식은 클러스터가 운영 중일 동안 설정 업데이트를 가능하도록 만들어준다.

## 합의 알고리즘

- 전형적으로 복제된 상태 머신은 복제된 로그를 사용하여 구현된다. 각각의 로그들은 같은 순서로 같은 명령의 집합을 포함하며, 따라서 각 상태 머신들은 같은 순서의 명령들을 처리하게 된다. 각 상태 머신들은 결정적이기 때문에 같은 상태라면 같은 출력이 보장된다.

- 복제된 로그들의 일관성을 보장하는 것이 합의 알고리즘의 역할이다.

- 합의 모듈은 클라이언트들의 요청을 받아 로그에 추가한다. 특정 서버에 장애가 발생하더라도 같은 요청들이 같은 순서로 복제되도록 보장한다. 결과적으로 서버들은 단일의 신뢰성 높은 상태 머신을 만들게 된다.

![](/img/posts/Paper-Review/2022-12-19-Raft-Review/1.png)

### 합의 알고리즘으로서 만족해야 하는 조건들

합의 알고리즘은 전형적으로 다음의 조건들을 만족해야 한다.

- 모든 [Non-Byzantine 상태](https://ko.wikipedia.org/wiki/%EB%B9%84%EC%9E%94%ED%8B%B0%EC%9B%80_%EC%9E%A5%EC%95%A0_%ED%97%88%EC%9A%A9)에서 안전성을 보장해야 한다. (예를 들어, 네트워크 패킷 딜레이, 패킷 로스, 파티션, 중복, reordering 등...)

- 다수의 서버들이 운용되고 있는 한, fully functional available 해야 하며, 각각의 서버들은 클라이언트와 통신할 수 있어야 한다.

- 로그의 일관성을 보장하기 위해 타이밍에 의존하면 안 된다. 또한, 잘못된 타이밍은 지연을 유발하거나 최악의 경우 가용성 문제를 일으킬 수 있다.

- 일반적으로 소수의 느린 서버들은 시스템의 전반적인 성능에 영향을 끼치지 않는다.

## 서버의 상태

- 특정 시각에 서버들은 `Leader`(리더), `Follower`(팔로워), `Candidate`(후보자) 셋 중 하나의 상태에 속하게 된다.

- 리더는 모든 클라이언트의 요청을 받는다. 만약 팔로워로 요청이 전달되면 리더에게 리다이렉트 된다.

- 팔로워는 그들 스스로는 어떤 요청도 보내지 않으며, 리더와 후보자들로부터의 요청에 단순히 반응하기만 하는 수동적인 상태이다.

- 후보자는 리더 선출에 사용된다.

![](/img/posts/Paper-Review/2022-12-19-Raft-Review/2.png)

### 리더와 팔로워

하나의 리더를 선출함으로써 합의를 구현한다. 리더는 복제 로그들을 관리하기 위한 책임을 맡는다. 클라이언트들에게 로그 엔트리를 전달 받고, 다른 서버들에 요청을 복제하며 상태 머신에 로그 엔트리들을 적용해야 할 때 알려준다.

리더를 갖는 것은 복제 로그들의 관리를 단순화 해 준다. 예를 들어 리더는 다른 팔로워들과의 소통 없이 새로운 로그 엔트리들을 어디에 놓아야 하는지 결정할 수 있다. 또한 데이터는 리버에서 팔로워로 일방향으로 흐르게 된다. 리더에 장애가 생기게 되면 새 리더가 선출된다.

## Term

- Raft는 시간을 `term`이라고 하는 임의의 길이로 쪼갠다. `term`은 연속된 정수들로서 논리적인 시계의 역할을 맡는다. 각각의 서버들에 `current term`을 저장해 사용함으로써 stale 된 리더를 탐지할 수 있다. `current term`은 서버들이 통신할 때 마다 교환되며 만약 한 서버의 `current term`이 다른 서버의 `current term`보다 작다면 해당 `current term`의 값을 더 큰 값으로 업데이트 한다.

- 만약 후보자나 리더가 자신의 `term` 값이 구식임을 확인하면 즉시 팔로워 상태로 돌아간다.

- 만약 서버가 구식 `term` 값을 포함하는 요청을 전달 받으면 해당 요청을 거절한다.

## Raft 구현에 사용되는 RPC

- 기본적인 Raft 알고리즘에서의 서버들의 통신에는 오직 `AppendEntries`(AE), `RequestVote`(RV)라는 두 종류의 RPC만 필요하다. `AppendEntries`는 리더가 로그 엔트리들을 복제하도록 만들고, heartbeat를 위해 필요하다.

- 만약 RPC가 타임 아웃되면 재요청하게 되며, 최적의 성능을 위해 RPC를 병렬로 요청한다.

## Raft 알고리즘 관련 주요 성질들

Raft는 아래와 같은 성질들이 항상 참임을 보장한다.

- `Election Safety`: 주어진 `term`마다 하나의 리더만 선출된다.

- `Leader Append-Only`: 리더는 절대 로그를 덮어쓰거나 삭제하지 않는다. 오직 새로운 엔트리를 추가하기만 한다.

- `Log Matching`: 만약 두 로그가 동일한 인덱스와 `term` 값을 가진 엔트리를 포함한다면, 두 로그의 엔트리들은 주어진 그 엔트리의 인덱스까지 모두 동일하다.

- `Leader Completeness`: 만약 한 로그 엔트리가 어떤 `term` 값에 커밋된다면, 해당 엔트리는 더 높은 `term` 값들을 갖는 리더들의 로그에 존재하게 될 것이다.

- `Safety Machine Property`: 어떤 서버가 로그 엔트리를 주어진 인덱스에 그 서버의 상태 머신에 적용할 경우, 다른 서버들은 같은 인덱스에 다른 로그 엔트리를 적용할 수 없다.

## 리더 선출 (Leader Election)

Raft는 heartbeat이라는 메커니즘을 통해 리더를 선출한다. 서버들은 시작할 때 팔로워 상태로 시작한다. 팔로워 상태의 서버는 리더나 후보자에게 유효한 RPC들을 받을 때 까지 팔로워 상태를 유지한다.

리더는 모든 팔로워들에게 주기적으로 heartbeat 신호를 보내는데, 이것은 빈 로그 엔트리를 가진 `AppendEntries` RPC로 구현된다.

만약 리더가 `Election timeout`이라고 부르는 정해진 기간 동안 RPC를 보내지 않으면, 리더에 장애가 생긴 것으로 간주하고 새로운 리더 선출 프로세스를 시작한다.

선거 시작을 위해 팔로워는 `current term`을 증가시키고 후보자 상태로 전이한다. 그리고 자신에게 투표하고 `RequestVote` RPC를 클러스터 내 서버들에게 병렬로 발행한다.

후보자는 아래 세 가지 일들 중 하나가 일어날 때 까지 후보자 상태를 유지한다.

1. 해당 후보자가 선거에 승리해 리더가 됨.

2. 다른 후보자가 리더로 선출됨.

3. 선거 시간이 끝났지만 어떤 서버도 리더로 선출되지 못함.

후보자는 클러스터 내 같은 `term` 값을 가진 다수의 서버들에게 투표를 받게 되면 리더로 선출된다. 리더로 선출된 후엔 추가적인 선거를 방지하기 위해 다른 모든 서버들에게 heartbeat를 보낸다.

투표 결과를 기다리는 동안 후보자는 본인이 리더라고 주장하는 서버로부터 `AppendEntries` RPC를 받게 될 수 있다. 이 경우 해당 리더의 `term` 값이 구식이라면 해당 RPC는 거절된다. 구식이 아닌 경우 리더가 클러스터 내에 이미 존재하므로 후보자는 다시 팔로워 상태로 돌아간다.

만약 너무 많은 팔로워들이 후보자로 동시에 참가하면, 표가 분산되어 어떤 서버도 리더로 선출되지 못할 수 있다.

이런 일이 발생하면 각 후보자들의 `RequestVote` RPC들은 모두 타임아웃 될 것이고, 그들의 `term` 값을 증가시킨 후 새 선거를 시작하게 된다. 그러나 여기에서 추가적인 조치를 취하지 않는다면 *표 분산(Vote split)*은 영원히 계속될 것이다.

Raft는 `Election timeout` 값을 무작위로 선정함으로써 이런 표 분산 문제를 해결한다.

표 분산 문제를 막기 위해 `Election timeout` 값은 *150ms - 300ms*의 고정된 무작위 값으로 선정된다.

무작위성으로 인해 어떤 하나의 서버가 먼저 투표에 승리할 것이고, 다른 후보자들이 타임 아웃 되기 전에 heartbeat를 보낼 것이다.

## 로그 복제 (Log Replication)

리더가 선출되면 클라이언트 요청들에 대해 서비스 하기 시작한다.

각 클라이언트 요청들은 복제된 상태 머신들에 의해 실행되어야 하는 명령을 포함한다. 리더는 그 명령을 새로운 엔트리로 자신의 로그에 추가한다.

그리고 `AppendEntries` RPC들을 병렬로 실행해 각 서버들에게 해당 엔트리를 복제하도록 한다. 해당 엔트리가 안전하게 복제되고 나면, 리더는 자신의 상태 머신에 해당 엔트리를 적용해 실행 결과를 클라이언트에 돌려주게 된다.

만약 팔로워 서버에 장애가 발생하거나, 패킷 로스가 발생하는 경우 (클라이언트에 응답한 이후에도) 지속적으로 Retry 요청을 보내 모든 팔로워들이 최종적으로 같은 로그 엔트리들을 저장하도록 만든다.

로그들은 아래 그림처럼 구조화 된다.

![](/img/posts/Paper-Review/2022-12-19-Raft-Review/3.png)

### 상태 머신의 작동

각각의 로그 엔트리들은 해당 엔트리를 저장했을 때의 `term` 값과 함께 상태 머신 명령을 저장한다.

로그 엔트리의 `term` 값들은 로그들 간에 비일관성을 탐지하기 위해 사용되며, 각 로그 엔트리들은 또한 해당 로그에서의 위치를 파악하기 위한 정수 인덱스를 갖는다.

리더는 각 로그 엔트리들을 언제 상태 머신들에 적용해도 괜찮은지를 판단한다. 적용해도 괜찮다고 판단한 로그 엔트리들을 커밋 되었다고(Committed) 부른다.

Raft는 커밋된 로그 엔트리들에 안전함, 견고함을 보장하며, 이 엔트리들은 최종적으로 이용 가능한(Available) 상태 머신들에 의해 실행될 것이다.

로그 엔트리는 리더가 해당 엔트리를 대다수의 서버들에 복제했을 때 커밋된다.

이 때 또한 해당 엔트리보다 앞서 있는 다른 로그들 또한 커밋된다. (해당 리더 이전의 리더들이 만든 엔트리들까지 포함하여)

리더는 커밋된 엔트리들 중 가장 높은 엔트리의 인덱스를 추적하며, 해당 인덱스 값을 heartbeat를 포함한 모든 `AppendEntries` RPC에 포함시킨다.

팔로워들은 특정 엔트리가 커밋되었다는 것을 알게 되면 자신의 상태 머신에 해당 엔트리를 로그 순서대로 적용한다.

### Log Matching Property

래프트 로그 메커니즘은 서버들 간에 높은 응집성을 갖도록 설계되었다. 이 점은 단지 시스템의 행동을 단순화하고 예측 가능하게 만들어 줄 뿐 아니라, 안정성을 보장해주는 중요한 요인이다.

Raft는 아래 특징들을 보장함으로써, **Log Matching Property**가 가능하도록 만들어준다.

* 만약 서로 다른 로그의 두 엔트리가 같은 인덱스와 `term` 값을 가진다면, 두 엔트리는 같은 명령어를 저장한다.

* 만약 서로 다른 로그의 두 엔트리가 같은 인덱스와 `term` 값을 가진다면, 해당 엔트리 앞의 모든 엔트리들 또한 동일하다.

첫 번째 성질은 리더가 주어진 로그 인덱스와 `term`에 대해 기껏해야 1개의 엔트리를 만들어내며, 로그 엔트리들은 절대 그들의 위치를 바꾸지 않는다는 점에서 유도할 수 있다.

두 번째 성질은 `AppendEntries` RPC를 수행할 때 간단한 *일관성 검사*를 수행함으로써 만족될 수 있다. `AppendEntries` RPC를 보낼 때 리더는 새 엔트리의 바로 앞 엔트리의 인덱스 및 `term` 값을 포함시킨다. 팔로워가 그들의 로그에서 같은 인덱스와 `term` 값을 갖는 엔트리를 찾지 못하면 해당 `AppendEntries` RPC는 거절된다. 이 일관성 검사가 `Log Matching Property`를 유도한다.

로그의 초기 상태는 `Log Matching Property`를 만족한다. 그리고 위 일관성 검사에 의해 로그가 늘어날 때 마다 `Log Matching Property`가 보존된다. 결과적으로 `AppendEntries`가 성공적으로 리턴될 때 마다 리더는 각 팔로워들의 로그가 자신의 로그와 추가된 새 엔트리 항목까지 동일할 것이라는 것을 알 수 있다.

### 비일관성 문제 해결

장애가 없는 경우 리더의 로그들과 팔로워의 서버들은 일관성을 갖는다. 따라서, `AppendEntries`의 일관성 검사는 절대 실패하지 않을 것이다.

그러나 리더의 장애는 로그를 비일관적으로 만들 수 있다. (구식 리더가 로그 엔트리들을 완전히 복사하지 않았을 수 있다.) 이러한 비일관성은 리더와 팔로워들에게 일련의 장애를 유발할 수 있다.

리더에 있는 엔트리들의 일부가 팔로워에 존재하지 않거나 리더에 없는 추가적인 엔트리들을 갖고 있을 수도 있으며, 이러한 엔트리 에러가 몇 차례의 `term`에 걸쳐 있을 수 있다.

Raft는 이러한 비일관성을 리더가 팔로워들의 로그에 자신의 로그를 붙여 넣게 함으로써 해결한다. 이것은 팔로워들에서 발생한 엔트리 충돌을 리더의 로그를 덮어 씌움으로써 해결한다는 것을 의미한다.

![](/img/posts/Paper-Review/2022-12-19-Raft-Review/4.png)

위 그림에서 이 작업은 한 가지의 제한을 가정하면 안전하게 이뤄질 수 있다.

팔로워의 로그와 리더 로그를 일치 시키기 위해, 리더는 두 개의 로그가 일치하는 가장 최신의 지점을 찾아야 한다. 팔로워는 해당 지점 이후의 모든 엔트리들을 삭제한다. 리더는 팔로워에게 리더가 갖고 있는 해당 지점 이후의 모든 엔트리들을 보낸다.

이 모든 작업들은 `AppendEntries` RPC에 의해 수행된 일관성 검사에 대한 응답으로 발생한다.

리더는 각 팔로워들에 대해 해당 팔로워에게 보낼 다음 로그 엔트리의 인덱스인, `nextIndex`를 유지한다.

리더는 자신이 리더로 선정되었을 때, 모든 `nextIndex` 값들을 서버의 마지막 엔트리의 다음 인덱스 값으로 설정한다.

팔로워의 로그가 리더와 비일관성을 갖는다면, `AppendEntries` 일관성 검사는 다음 `AppendEntries` RPC에서 실패할 것이다. RPC가 거절된 후, 리더는 `nextIndex` 값을 감소시키고, `AppendEntries`를 다시 시도하게 된다.

결과적으로 `nextIndex`는 리더와 팔로워의 로그가 매칭되는 지점에 도달하게 된다. 이 지점에 도달했을 때 `AppendEntries`는 성공하게 될 것이고, 팔로워에 존재하는 충돌을 일으키고 있는 모든 엔트리들이 제거되고 리더에 존재하는 엔트리들을 덧붙이게 될 것이다.

이런 메커니즘을 통해 리더는 별 다른 추가 조치 없이 로그 일관성을 복원할 수 있다.

리더는 Normal Operation으로 시작해서, `AppendEntries` 일관성 검사 실패에 대한 응답으로 자동으로 로그를 수렴시킨다. 리더는 절대로 자신의 로그를 삭제하거나 덮어쓰지 않는다.

Raft는 대부분의 서버들이 가용중인 한 새로운 로그 엔트리들을 받아들이고, 복제하고, 새로운 로그 엔트리들을 적용할 수 있다. 일반적인 경우 한 라운드의 RPC들을 통해 새로운 엔트리를 클러스터 내에 복제시킬 수 있으며, 소수의 느린 팔로워는 성능에 영향을 주지 못할 것이다.

#### AppendEntries 최적화

바람직하게는, 거절되는 `AppendEntries` RPC의 갯수를 줄이는 방향으로 프로토콜을 최적화할 수 있다.

예를 들어, `AppendEntries` 요청을 거절할 때, 팔로워는 충돌이 난 엔트리의 `term`과 그 `term`을 저장하는 첫 번째 인덱스를 포함시킬 수 있다.

이 정보들로, 리더는 `nextIndex`를 해당 `term`에 해당하는 모든 충돌 엔트리들을 건너뛰도록 조정 할 수 있다.

이런 최적화를 통해 한 개의 RPC로 한 개의 엔트리를 건너뛰는 것에서 해당 `term`에 해당하는 엔트리들을 모두 건너뛰는 방식으로 최적화 할 수 있다.

다만 실제 시스템에서 장애는 이렇게 빈번하게 일어나지 않고, 여러 엔트리에 비일관성이 나타날 가능성이 희박하기 때문에 이러한 최적화가 필요한 것인지는 의문이다.

## 안정성 보장

이전의 섹션들은 Raft가 어떻게 리더를 선출하고 로그 엔트리들을 복제하는지에 대해 다루었다. 그러나, 지금까지의 메커니즘들은 각각의 상태 머신들이 정확하게 같은 명령들을 같은 순서로 시행하는 것을 보장해주지 못한다.

예를 들어 리더가 여러 개의 로그 엔트리들을 커밋할 때, 한 팔로워가 불가용할 수 있다, 그리고 리더로 선출되고 커밋된 로그 엔트리들을 새 엔트리로 덮어씌워 버릴 수 있다.

결과적으로 다른 상태 머신들에서 서로 다른 명령어 시퀀스가 실행될 수 있다는 것을 의미한다.

### 선거 제한

어떤 리드 기반의 합의 알고리즘이든, 리더는 결과적으로 커밋된 모든 로그 엔트리들을 저장해야만 한다.

어떤 합의 알고리즘에서는 모든 커밋된 엔트리들을 포함하지 않는 서버를 리더로 선출할 수 있다.

불행하게도 이런 접근 방법은 상당한 복잡함을 유발하기 때문에 Raft는 이전 `term`들에서 커밋된 모든 엔트리들이 새 리더에 존재함을 보장한다. 이것은 이런 엔트리들을 팔로워에서 리더로 옮길 필요가 없게 만들어주고, 데이터의 흐름을 리더에서 팔로워의 단방향으로 흘라가게 만들어주며, 리더가 절대 기존의 엔트리들을 덮어씌우지 않게 만들어준다.

Raft는 모든 커밋된 로그들을 갖고 있지 않은 후보자가 선거에서 이기지 못하도록 만든다. 후보자는 클러스터 내 대다수의 서버들에게 접촉해야 한다. 이 중 적어도 하나의 서버는 모든 커밋된 엔트리들을 갖고 있을 것이므로, 이것은 후보자가 모든 커밋된 엔트리들을 갖고 있도록 만들어준다.

`RequestVote` RPC가 이러한 제한을 구현한다. 이 RPC는 후보자 로그의 정보를 포함하며, 투표자는 투표자의 로그가 후보자의 로그보다 더 최신일 때, 투표를 부정하게 된다.

Raft는 두 로그 중 어떤 로그가 최신인지를 index와 로그의 마지막 엔트리의 `term` 값을 비교함으로써 결정한다.

두 로그의 마지막 엔트리가 다른 `term` 값을 지닌다면 더 큰 `term` 값을 갖는 로그가 더 최신 로그가 된다.

두 로그의 마지막 엔트리가 같은 `term` 값을 지닌다면 두 로그 중 긴 것이 가장 긴 로그가 된다.

### 이전의 term들에서 엔트리 커밋하기

<!-- Log Replication 섹션에서 설명한 것과 같이, 리더는 current term으로 부터 해당 엔트리가 대다수의 서버에 저장되면, 어떤 엔트리가 커밋되었는지 알게 된다.

엔트리를 커밋하기 이전에 리더에 장애가 발생하는 경우, 미래의 리더들이 해당 엔트리의 복제를 끝내려고 시도하게 될 것이다. 그러나, 엔트리가 대다수의 서버에 저장되는 한 리더는 이전 term으로부터의 엔트리가 커밋되었는지 즉시 결론을 내릴 수 없다. -->

Raft는 오직 리더의 `current term` 값에서만 레플리카 갯수를 카운팅하는 식으로 로그 엔트리들을 커밋한다.

`current term`으로부터 로그 엔트리가 이 방식으로 커밋되고 나면, 모든 이전의 엔트리들은 `Log Matching Property`에 의해 간접적으로 커밋되게 된다.

리더가 안전하게 오래된 로그들이 커밋되었다고 결론 내릴 수 있는 몇 가지 상황들이 더 있다. 그러나 Raft는 단순성을 위해 보수적인 접근 방법을 취한다.

### 안전성 보장 관련 증명 (Safety Argument)

참고: 귀류법을 통해 `Leader Completeness`가 성립함을 5.4.3절에서 증명함.

`Leader Completeness`를 통해 `State Machine Safety Property`도 증명할 수 있다고 함.

### 팔로워, 후보자 서버 장애 핸들링

팔로워, 후보자에 발생하는 장애는 리더에 발생하는 장애보다 핸들링하기 쉽다.

둘은 같은 방식으로 핸들링 할 수 있는데 팔로워나 후보자에 장애가 발생하면 `RequestVote`나 `AppendEntries` RPC가 실패하게 될 것이다.

Raft는 이러한 장애를 계속해서 Retry를 시도하는 것으로 해결한다. 장애가 난 서버의 재시작이 끝나면 RPC가 성공할 것이다.

만약 팔로워, 후보자에 RPC가 성공했지만 응답 하기 전, 장애가 발생하면 서버의 재시작 후에 동일한 RPC를 받게 될 것이다. Raft RPC들은 *멱등성*을 지니므로(Idempotent, 연산을 여러 번 적용해도 같은 결과를 내는 성질) 이것은 문제가 되지 않는다.

### 타이밍과 가용성

Raft의 요구 사항 중 하나는 안정성이 타이밍에 의존하지 않아야 한다는 것이다. 시스템은 단지 특정 이벤트가 예상보다 빨리 일어났다거나 늦게 일어났다는 이유로 잘못된 결과를 내 놓지 않아야 한다.

그러나 가용성은 필연적으로 타이밍에 의존하게 된다. 예를 들어, 메세지 교환이 예상보다 오래 걸린다면 후보자들은 선출될 수 있을만큼 긴 기간 동안 살아 있을 수 없을 것이다. Raft는 안정적인 리더 없이 성립할 수 없다.

**리더 선출**은 Raft에서 타이밍이 가장 크리티컬한 측면이다.

Raft는 아래 타이밍 조건을 만족할 때 안정적인 리더를 선출하고 유지할 수 있다.

```
broadcastTime << electionTimeout << MTBF
```

위 부등식에서 `broadcastTime`은 서버가 매 다른 서버들에게 RPC 요청을 보내고 응답을 받기 까지 소요되는 평균 시간이며, `MTBF`는 단일 서버 장애에 발생하는 장애 간격의 평균 값이다.

`broadcastTime`과 `MTBF`가 근본적인 시스템의 성질인 반면, `electionTimeout`은 우리가 선택해야 하는 값이다.

Raft의 RPC들이 전형적으로 피호출자에게 정보를 **안정적 저장소(Stable storage)**에 저장하도록 요구하기 때문에, `broadcastTime`은 스토리지 기술에 따라 아마도 0.5ms에서 20ms 사이일 것이다.

결과적으로 `electionTimeout` 값은 *10ms - 500ms* 사이의 값 어딘가로 선정될 가능성이 높다.

또한 서버 `MTBF` 값은 전형적으로 몇 달 이상이기 때문에 위 조건을 쉽게 만족할 수 있다.

## Cluster Membership changes

지금까지는 클러스터의 설정이 고정되어 있다고 가정했지만, 실제로는 설정을 바꿔야 할 필요가 종종 생긴다. 이 경우 전체 클러스터 내 서버들을 모두 종료하고 설정 파일을 업데이트 하고 클러스터를 재시작하면 쉽지만 이렇게 하면 업데이트 동안 클러스터가 불가용해지며, 수동으로 설정 파일들을 업데이트 하기 때문에 리스크가 존재한다.

이런 이슈들을 피하기 위해 설정 파일 변경을 자동화를 Raft 합의 알고리즘의 일부로 포함시킬 수 있다.

설정 파일 업데이트 자동화를 안전하게 수행하기 위해, 업데이트가 이행되는 동안 두 개의 리더가 같은 `term`에서 선출되는 지점이 있으면 안 된다. 불행하게도, 설정 파일을 직접 업데이트하는 어떤 접근도 안전하지 못하다. 원자적으로 모든 서버들을 한 번에 변경하는 것은 가능하지 않고, 따라서 클러스터는 업데이트 기간 동안 잠재적으로 독립적인 두 부분(Majorities) 으로 나뉘게 된다.

안정성 보장을 위해, 설정 업데이트는 반드시 **Two-phase**에 걸쳐 이뤄져야 한다. 이런 **Two-phase**의 구현에는 다양한 방법이 있다.

예를 들어, 어떤 시스템은 첫 번째 페이즈를 구식 설정을 비활성화 하는데 사용해 클라이언트 요청을 처리할 수 없게 만들고, 두 번째 페이즈에서 새로운 설정을 활성화한다.

Raft에서 클러스터는 우선 `Joint consensus`라고 불리는 중간 단계의 설정으로 전환한 후, `Joint consensus`가 커밋된 후 새 설정으로 전이한다. `Joint consensus`는 구식 설정과 새 설정 양쪽을 합쳐서 구성된다.

* 로그 엔트리들은 모든 서버들에 양쪽 설정으로 모두 복제된다.

* 어느 쪽 설정의 서버든 리더로 선출될 수 있다.

* 리더 선출, 엔트리 커밋을 위해 구식 설정의 서버들과 새 설정의 서버들 각각은 별도의 다수파의 합의가 필요하다.

`Joint consensus`는 각각의 서버들이 설정 사이를 각각 다른 시간에 안전하게 전이할 수 있도록 해 준다. 뿐만 아니라, 설정 파일 업데이트를 서비스 중단 없이 진행할 수 있게 만들어준다.

클러스터 설정은 복제된 로그에서 특별한 엔트리들을 사용해 저장되고 통신된다.

리더가 설정 파일을 변경하라는 요청을 받게 되면, `Joint consensus`를 위한 설정을 로그 엔트리로 저장하고 이전 섹션들에서 설명한 메커니즘을 사용하여 해당 엔트리를 복제한다.

한 번 서버가 새 설정 엔트리를 로그에 추가하고 나면, 서버는 해당 설정을 모든 미래의 결정들에 사용한다. (서버는 해당 엔트리가 커밋 되었는지 여부에 상관 없이 항상 자신의 로그의 최신 설정을 사용한다.)

이것은 리더가 `Joint consensus`가 커밋될 시기를 결정하기 위해, `Joint consensus`의 설정을 사용한다는 것을 의미한다.

![](/img/posts/Paper-Review/2022-12-19-Raft-Review/5.png)

만약 리더에 장애가 발생하면 선출된 후보자가 C_old_new를 받았는지 여부에 따라 새로운 리더가 C_old 또는 C_old_new 내에서 선출될 수 있다.

어떤 케이스에도 C_new는 이 기간 동안 단독으로 의사 결정을 내릴 수 없다.

한 번 C_old_new가 커밋되고 나면, C_old, C_new 양쪽 모두 다른 쪽의 동의 없이 의사 결정을 내릴 수 없으며, `Leader Completeness Property`에 의해 오직 C_old_new 로그를 가지고 있는 서버만 리더로 선출될 수 있다. `joint_consensus`가 커밋되었으니 이제 리더가 안전하게 C_new를 로그 엔트리로 만들고, 클러스터에 복제할 수 있다. 해당 설정은 서버에서 발견되는대로 바로 적용된다.

C_new까지 커밋되고 나면 구식 설정은 무관한 설정으로 간주되며, 새 설정으로 업데이트 되지 않은 서버들은 종료될 수 있다.

위 그림에서 볼 수 있듯이 C_old와 C_new 모두 단독으로 의사 결정을 내릴 수 있는 시기는 존재하지 않으며, 이것이 안정성을 보장해준다.

설정 업데이트를 위해 다루어야 하는 몇 가지 이슈들이 더 있다.

### 첫 번째 이슈: 새 서버들의 로그 엔트리가 초기에 비어 있는 경우

첫 번째 이슈는 새로운 서버들이 초기에 어떤 로그 엔트리들도 저장하고 있지 않을 수 있다는 점이다.

만약 이들이 이 상태로 클러스터에 추가되게 되면, 이들이 동기화하는데 어느 정도 시간이 소요될 수 있으며, 이 시간 동안 새로운 로그 엔트리들을 커밋할 수 없게 된다.

불가용한 시간이 생기는 것을 막기 위해, Raft는 *설정 업데이트 이전에 추가적인 페이즈를 도입한다*.

이 시간 동안 새로운 서버들은 클러스터 **Non-voting 멤버**로서 참여하게 된다. 리더는 이들에게 로그 엔트리들을 복제하지만, 이들은 다수파로 간주되지 않는다. 새로운 서버들이 클러스터 내 다른 서버들을 따라잡고 나면, 설정 업데이트가 진행될 수 있다.

### 두 번째 이슈: 리더가 새 설정의 일부이지 않은 경우

두 번째 이슈는 클러스터 리더가 새 설정의 일부이지 않을 수 있다는 점이다.

이 경우 리더는 C_new 로그 엔트리를 커밋한 후 팔로워 상태로 돌아간다. 이것은 C_new를 커밋하는 동안 리더가 자기 자신을 포함하지 않는 클러스터를 관리하는 기간이 있을 것임을 의미한다.

리더는 로그 엔트리들을 복제하지만, 자기 자신을 다수파에 카운트 하지는 않는다. 리더의 상태 변화는, 새로운 설정이 독립적으로 운용되는 첫 번째 지점인 C_new가 커밋될 때 일어날 것이다.

이 지점까지 오직 C_old에서의 서버만 리더로 선출될 수 있을지도 모른다.

### 세 번째 이슈: 제거된 서버들이 클러스터를 방해하는 경우

세 번째 이슈는 C_new에 존재하지 않는 제거된 서버들이 클러스터를 방해할 수도 있다는 점이다.

이 서버들은 heartbeat를 받지 않을 것이기 때문에 타임아웃 된 후, 다음 선거를 시작할 것이다. 그리고 새로운 `term` 값으로 `RequestVote` RPC를 보낼 것이고 이것은 현재 리더가 팔로워 상태로 돌아가도록 만들 것이다. 결과적으로 새로운 리더가 선출되겠지만, 제거된 서버들은 다시 타임아웃을 일으킬 것이고, 이 과정이 반복되면서 가용성을 떨어뜨릴 것이다.

이 문제를 방지하기 위해, 서버들은 현재 리더가 존재한다고 판단될 때 `RequestVote` RPC를 무시한다. 특히나 서버가 현재 리더에게 `Minimum election timeout` 값 이전에 `RequestVote` RPC를 받으면 `term`값을 업데이트 하지도, 투표를 하지도 않는다.

이것은 각 서버들이 선거를 시작하기 전 적어도 `Minimum election timeout` 동안 기다리기 때문에, 일반적인 선거에 영향을 끼치지 않는다.

그러나 이것은 제거된 서버들로부터의 방해를 피하도록 도와준다: 만약 리더가 클러스터에 heartbeat를 가져올 수 있다면, 더 큰 `term` 값에 의해 제거되지 않을 것이다.

## Log compaction

WIP

## Raft 알고리즘 구현 요약

### 상태 (States)

#### 모든 서버들의 persisted 상태

아래 상태들은 RPC에 응답하기 전 안정적 저장소에 업데이트 된다.

- `currentTerm`: 서버가 관측했던 가장 최신의 term값.
- `votedFor`: 현재 term에서 투표한 후보자의 Id. 없는 경우 null.
- `log`: 로그 엔트리들의 배열. 각 로그 엔트리들은 상태 머신을 위한 명령어 및 엔트리가 리더에게 수신되었을 때의 term 값을 포함하고 있다.

#### 모든 서버들의 휘발성(volatile) 상태

- `commitIndex`: 커밋된 것으로 알려진 로그 엔트리들 중 가장 높은 인덱스 값.
- `lastApplied`: 상태 머신에 적용된 로그들 중 가장 높은 인덱스 값.

#### 리더의 휘발성(volatile) 상태

아래 상태들은 리더 선거 이후 재초기화 된다.

- `nextIndex`: 각각의 서버에 대해 갖고 있는, 다음에 보내줘야 하는 로그 엔트리의 인덱스 값.
- `matchIndex`: 각각의 서버들에 갖고 있는, 복제된 것으로 알려진 가장 높은 인덱스 값.

### AppendEntries RPC

로그 엔트리들을 복제하기 위해 리더에 의해 호출된다. 또한 heartbeat 목적으로도 사용된다.

#### 인자 (Arguments)

- `term`: 리더의 term 값.
- `leaderId`: 리더에게 요청을 리다이렉트 할 수 있도록 전달해주어야 하는 리더의 Id 값.
- `prevLogIndex`: 새 로그 엔트리 바로 앞 인덱스.
- `prevLogTerm`: 새 로그 엔트리 바로 앞 엔트리(prevLogIndex 엔트리)의 term 값.
- `entries`: 저장할 로그 엔트리들 (heartbeat인 경우 빈 배열.)
- `leaderCommit`: 리더의 커밋 인덱스.

#### 리턴 값 (Results)

- `term`: 리더가 자기 자신을 업데이트 하기 위한 currentTerm 값.
- `success`: 팔로워가 모든 매칭되는 prevLogIndex와 prevLogTerm을 포함할 경우 True.

#### 구현

1. term < currentTerm 인 경우 False를 리턴.
2. 로그에 prevLogTerm과 매칭되는 prevLogIndex에 엔트리를 포함하고 있지 않은 경우 False를 리턴.
3. 기존의 엔트리가 새로운 엔트리와 충돌(같은 인덱스를 갖지만 term이 다른 경우)을 일으키는 경우, 기존의 엔트리를 지운다.
4. 로그에 존재하지 않는 모든 엔트리들을 더한다.
5. leaderCommit > commitIndex인 경우, commitIndex 값을 `min(leaderCommit, 새 엔트리의 인덱스)`값으로 정한다.

### RequestVote RPC

투표를 위해 후보자에 의해 호출된다.

#### 인자 (Arguments)

- `term`: 후보자의 term 값.
- `candidateId`: 투표를 요청하는 후보자의 Id.
- `lastLogIndex`: 후보자의 마지막 로그 엔트리의 인덱스.
- `lastLogTerm`: 후보자의 마지막 로그 엔트리의 term 값.

#### 리턴 값 (Results)

- `term`: 후보자가 자기 자신을 업데이트 하기 위한 currentTerm 값.
- `voteGranted`: 후보자가 투표를 받았다면 True.

#### 구현

1. term < currentTerm 인 경우 False를 리턴.
2. 만약 votedFor이 null이거나 candidateId이라면, 후보자의 로그는 적어도 수신자의 로그와 업데이트된 상태이고, 투표한다.

### 서버들을 위한 규칙

#### 모든 서버들

- commitIndex > lastAppied인 경우, lastApplied를 증가시키고 상태 머신에 log[lastApplied]를 적용한다.
- RPC 요청이나 응답에 포함된 term 값이 currentTerm보다 높은 경우, currentTerm을 term 값으로 설정하고 팔로워 상태로 돌아간다.

#### 팔로워

- 후보자와 리더들의 RPC에 응답한다.
- election timeout 동안 리더와 후보자에게 AppendEntries 요청을 받지 못한 경우 후보자가 된다.

#### 후보자

- 후보자로 전환되면 아래 절차를 거쳐 선거를 시작한다
1. currentTerm을 증가시킨다.
2. 자기 자신에게 투표한다.
3. 선거 타이머를 초기화한다.
4. 다른 모든 서버들에게 RequestVote RPC를 보낸다.
- 다수의 서버들에게 득표를 받게 되면 리더가 된다.
- 리더에게 AppendEntries RPC를 받게 되면 팔로워가 된다.
- Election timer만큼 시간이 지나면 새 투표를 시작한다.

#### 리더

- 선거를 마치자마자 각 서버들에게 비어 있는 AppendEntries RPC를 보내고, Idle 상태일 동안 election timeout을 막기 위해 지속적으로 heartbeat을 보낸다.
- 만약 클라이언트에게 명령을 받게 되면, 엔트리를 로컬 로그에 추가하고, 상태 머신에 엔트리를 적용한 후 응답한다.
- nextIndex 배열을 순회하며, last log index >= nextIndex[i]인 팔로워들에 대해 nextIndex에서 시작하는 로그 엔트리들의 배열을 AppendEntries로 보낸다. 성공하는 경우, 해당 팔로워의 nextIndex와 matchIndex를 업데이트 한다. 만약 AppendEntries가 로그 비일관성으로 인해 실패하는 경우, nextIndex를 낮추고 Retry 한다.
- N > commitIndex, 대다수 서버들에 대해 matchIndex[i] >= N, log[N].term == currentTerm을 만족하는 N이 존재하는 경우, commitIndex에 N을 대입한다.