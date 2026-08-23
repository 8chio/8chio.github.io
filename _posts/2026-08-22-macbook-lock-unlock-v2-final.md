---
categories:
- macOS
- Automation
date: "2026-08-22 18:30:00 +0900"
tags:
- macOS
- iPhone
- Shortcuts
- SSH
- Tailscale
- AppleScript
- Swift
comments: true
title: MacBook 잠금해제 자동화 v2 --- 잠금 상태를 확인하고 안전하게
  토글하기
---

지난 글에서는 클램쉘 모드로 사용하는 MacBook을 iPhone 단축어와 SSH,
Tailscale을 이용해 잠금 해제하는 자동화를 만들었습니다.

처음에는 꽤 만족스러웠습니다. iPhone에서 단축어를 누르면 Mac의 화면이
깨어나고, 로그인 암호가 입력되면서 바로 잠금이 풀렸습니다.

그런데 실제로 사용하다 보니 **생각보다 위험한 문제가 하나 있었습니다.**

## v1에서 발견한 문제

v1의 잠금해제 스크립트는 Mac이 현재 잠겨 있는지 확인하지 않았습니다.

단축어를 실행하면 바로 `System Events`가 로그인 암호를 키보드 입력처럼
입력하고 Enter까지 눌렀습니다.

문제는 Mac이 이미 잠금 해제된 상태일 때였습니다.

``` text
Mac이 이미 열려 있음
        ↓
채팅창이나 입력창에 커서가 있음
        ↓
iPhone에서 단축어 실행
        ↓
로그인 암호 입력
        ↓
Enter
        ↓
⚠️ 암호가 그대로 입력되거나 전송될 수 있음
```

편의를 위해 만든 자동화가 오히려 로그인 암호를 노출시킬 수 있는
구조였습니다.

그래서 v2에서는 원칙을 하나 정했습니다.

> **Mac이 잠겨 있다는 것이 확인된 경우에만 로그인 암호를 입력합니다.**

그리고 Mac이 이미 열려 있다면 암호를 입력하는 대신 **잠금 버튼으로
동작하도록** 바꿨습니다.

## v2에서 원하는 동작

최종적으로 만들고 싶었던 동작은 다음과 같습니다.

``` text
iPhone 단축어 실행
        ↓
Mac의 현재 상태 확인
        ↓
┌───────────────────┬───────────────────┐
│ MAC_IS_LOCKED     │ MAC_IS_OPEN       │
│                   │                   │
│ 바로 잠금 해제       │ "Mac을 잠글까요?"    │
│        ↓          │        ↓          │
│        🔓         │   잠그기 / 취소      │
│                   │        ↓          │
│                   │        🔒         │
└───────────────────┴───────────────────┘
```

잠금 해제할 때는 기존처럼 빠르게 사용할 수 있고, Mac이 열려 있을 때는
iPhone에서 한 번 확인한 뒤 잠글 수 있도록 했습니다.

## 1. Mac의 잠금 상태 확인하기

먼저 SSH를 통해 Mac이 현재 잠겨 있는지 확인할 방법이 필요했습니다.

여기서는 macOS의 Core Graphics 세션 정보를 이용했습니다.

터미널에서 다음 Swift 코드를 실행하면 현재 세션 정보를 확인할 수
있습니다.

``` bash
swift -e '
import CoreGraphics

if let session = CGSessionCopyCurrentDictionary() as? [String: Any] {
    print(session)
} else {
    print("NO_SESSION")
}
'
```

Mac이 잠금 해제된 상태에서는 `CGSSessionScreenIsLocked` 항목이 보이지
않았습니다.

반대로 Mac을 잠근 상태에서 확인하니 다음 값이 추가됐습니다.

``` text
"CGSSessionScreenIsLocked": 1
```

이 값을 이용하면 현재 Mac이 잠겨 있는지를 구분할 수 있었습니다.

## 2. 상태 확인용 SSH 스크립트

iPhone 단축어의 첫 번째 `SSH를 통해 스크립트 실행`에는 다음 코드를
넣었습니다.

``` bash
swift -e '
import CoreGraphics

if let session = CGSessionCopyCurrentDictionary() as? [String: Any] {
    if session["CGSSessionScreenIsLocked"] != nil {
        print("MAC_IS_LOCKED")
    } else {
        print("MAC_IS_OPEN")
    }
} else {
    print("UNKNOWN")
}
'
```

결과는 세 가지 중 하나입니다.

``` text
MAC_IS_LOCKED  → Mac이 잠겨 있음
MAC_IS_OPEN    → Mac이 열려 있음
UNKNOWN        → 상태 확인 실패
```

처음에는 `LOCKED`와 `UNLOCKED`라는 문자열을 사용했습니다.

그런데 단축어 조건을 `LOCKED`를 **포함**하는지 확인하도록 설정하니
문제가 생겼습니다.

``` text
LOCKED
UNLOCKED
  └────┘
   LOCKED
```

`UNLOCKED`에도 `LOCKED`라는 문자열이 들어 있기 때문에 두 상태 모두 잠금
상태로 판단됐습니다.

그래서 서로 겹치지 않는 `MAC_IS_LOCKED`와 `MAC_IS_OPEN`으로
변경했습니다.

작은 부분이지만 실제로 자동화를 만들면서 꽤 재미있었던 디버깅
포인트였습니다.

## 3. iPhone 단축어에서 조건 분기하기

상태 확인 SSH 바로 아래에 **조건문**을 추가했습니다.

조건은 다음과 같습니다.

``` text
셸 스크립트 결과
    [다음을 포함]
MAC_IS_LOCKED
```

이 조건이 참이면 Mac이 잠긴 상태이므로 기존 잠금해제 SSH 스크립트를 바로
실행합니다.

``` bash
caffeinate -u -t 3 </dev/null &>/dev/null &
sleep 1

osascript <<'APPLESCRIPT'
tell application "System Events"
    key code 123
    delay 1
    keystroke "YOUR_MAC_PASSWORD"
    delay 0.3
    key code 36
end tell
APPLESCRIPT
```

이제 이 스크립트는 **Mac이 잠겨 있다고 판별된 경우에만 실행됩니다.**

따라서 Mac이 열려 있는 상태에서 채팅창이나 브라우저 입력창에 커서가
있더라도 로그인 암호가 입력되는 것을 막을 수 있습니다.

## 4. Mac이 열려 있다면 잠금 메뉴 표시하기

조건이 거짓이라면 `MAC_IS_OPEN` 상태입니다.

이 경우에는 바로 Mac을 잠그지 않고 iPhone에서 한 번 확인하도록 했습니다.

단축어의 **메뉴에서 선택** 동작을 추가하고 다음과 같이 구성했습니다.

``` text
Mac을 잠글까요?

🔒 잠그기
취소
```

`취소`를 선택하면 아무 작업도 하지 않습니다.

`🔒 잠그기`를 선택했을 때만 SSH를 통해 Mac 잠금 명령을 실행합니다.

## 5. Mac 잠그기

처음에는 다음과 같이 `keystroke "q"`를 이용했습니다.

``` applescript
keystroke "q" using {control down, command down}
```

하지만 제 환경에서는 가끔 제대로 동작하지 않았습니다.

그래서 문자 입력 대신 **Q 키의 key code를 직접 보내는 방식**으로
변경했습니다.

``` bash
osascript -e 'tell application "System Events" to key code 12 using {control down, command down}'
```

`key code 12`는 Q 키이므로 결과적으로 macOS의 화면 잠금 단축키인:

``` text
Control + Command + Q
```

를 실행하는 것과 같습니다.

제 환경에서는 이 방식이 더 안정적으로 동작했습니다.

여기에 한 가지를 더 추가했습니다. Mac을 잠근 뒤 외장 모니터도 바로 꺼지도록 **디스플레이 절전**까지 이어지게 했습니다.

최종 잠금용 SSH 스크립트는 다음과 같습니다.

```bash
osascript -e 'tell application "System Events" to key code 12 using {control down, command down}'
sleep 1
pmset displaysleepnow
```

동작 순서는 간단합니다.

```text
Control + Command + Q
        ↓
Mac 잠금 🔒
        ↓
1초 대기
        ↓
pmset displaysleepnow
        ↓
디스플레이 절전 🌙
```

처음에는 Mac 자체를 잠자기 상태로 만드는 것도 생각했습니다. 하지만 이 자동화는 iPhone에서 Tailscale과 SSH를 통해 다시 Mac에 접근해야 합니다. Mac 전체를 잠자기 상태로 만들면 네트워크 연결이나 SSH 접근이 끊길 수 있기 때문에, 여기서는 **Mac은 깨어 있게 두고 디스플레이만 절전 상태로 만드는 방식**을 사용했습니다.

다시 iPhone에서 단축어를 실행하면 잠금 상태가 확인되고, 기존 잠금해제 분기의 `caffeinate`가 디스플레이를 깨운 뒤 잠금 해제를 진행합니다.

## 최종 단축어 구조

v2의 전체 구조를 정리하면 다음과 같습니다.

``` text
[SSH] Mac 상태 확인
        ↓
MAC_IS_LOCKED 포함?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
   ↓         ↓
 잠금해제     메뉴에서 선택
 SSH        "Mac을 잠글까요?"
   │         │
   ↓       ┌─┴──────┐
   🔓    잠그기     취소
           │        │
           ↓        └→ 종료
          잠금 SSH
           ↓
           🔒
           ↓
       디스플레이 절전 🌙
```

이제 iPhone의 단축어 하나가 **Mac Lock / Unlock 토글 버튼**처럼
동작합니다.

Mac이 잠겨 있으면 별도의 질문 없이 바로 잠금 해제를 시도하고, Mac이 이미
열려 있으면 로그인 암호는 입력하지 않고 잠글 것인지 먼저 물어봅니다.
잠그기를 선택하면 화면 잠금 후 디스플레이까지 절전 상태로 전환됩니다.

## v1과 v2 비교

| 항목              | v1                 | v2                     |
| ----------------- | ------------------ | ---------------------- |
| Mac 상태 확인     | 없음               | 잠금 상태 확인         |
| 잠금 상태         | 바로 잠금 해제     | 상태 확인 후 잠금 해제 |
| 열린 상태         | 암호가 입력될 위험 | 잠금 여부 확인         |
| 잠금 기능         | 없음               | 지원                   |
| 잠금 후 화면 절전 | 없음               | 지원                   |
| 오입력 방지       | 낮음               | 개선                   |
| 사용 방식         | Unlock 버튼        | Lock / Unlock 토글     |

## 완성된 단축어 공유

v2를 완성한 뒤에는 직접 구성하지 않아도 사용할 수 있도록 **배포용 단축어**도 따로 만들었습니다.

다만 이 단축어에는 SSH 접속 정보와 Mac 로그인 암호가 들어갈 수 있기 때문에, 실제 사용 중인 단축어를 그대로 공유하지 않고 먼저 복제한 뒤 개인 정보를 제거했습니다.

배포용에서는 다음 값을 자신의 환경에 맞게 설정하도록 했습니다.

```text
YOUR_MAC_HOSTNAME  → 자신의 Mac 호스트 이름
USER_NAME          → 자신의 macOS 사용자 이름
YOUR_MAC_PASSWORD  → 자신의 Mac 로그인 암호
SSH Key            → 설치한 기기에서 생성된 SSH 키
```

특히 **실제 Mac 로그인 암호가 들어 있는 원본 단축어는 공유하지 않는 것이 중요합니다.**

### SSH 키는 공유하면 어떻게 될까?

`SSH를 통해 스크립트 실행` 동작에는 ED25519 SSH 키가 선택되어 있기 때문에, 처음에는 이 키까지 공유되는 것이 아닌지 걱정됐습니다.

그래서 배포용 단축어를 iCloud 링크로 공유한 뒤 다른 Apple 기기에서 직접 가져와 확인해봤습니다.

테스트한 환경에서는 공유본을 가져왔을 때 **SSH 키가 원본의 키를 그대로 사용하는 것이 아니라 새 키가 생성되는 동작**을 확인했습니다.

```text
원본 단축어
   └─ SSH Key A 🔑
          ↓
      iCloud 공유
          ↓
가져온 단축어
   └─ 새 SSH Key B 🔑
```

따라서 단축어를 받은 사용자는 새로 생성된 키의 **공개 키**를 자신의 Mac에 등록해야 합니다.

> 이 동작은 제가 사용한 환경에서 직접 확인한 결과입니다. Shortcuts나 macOS/iOS 버전에 따라 동작이 달라질 수 있으므로, 공유하거나 가져온 뒤에는 SSH 키 정보를 직접 확인하는 것을 권장합니다.

### 받은 사람이 해야 할 설정

공유된 단축어를 가져왔다고 바로 SSH 연결이 되는 것은 아닙니다.

먼저 단축어의 SSH 키를 열어 **`공개 키 복사`**를 선택합니다.

그다음 Mac의 `~/.ssh/authorized_keys`에 해당 공개 키를 등록합니다.

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

복사한 공개 키를 한 줄로 붙여넣고 저장한 뒤 권한을 설정합니다.

```bash
chmod 600 ~/.ssh/authorized_keys
```

그리고 macOS의 **시스템 설정 → 일반 → 공유 → 원격 로그인**이 활성화되어 있고, 단축어에서 사용하는 macOS 계정의 SSH 접속이 허용되어 있어야 합니다.

마지막으로 배포용 단축어의 다음 항목을 자신의 환경에 맞게 수정합니다.

```text
호스트     → Mac의 Tailscale/MagicDNS 호스트 이름
사용자     → macOS 사용자 이름
SSH 키     → 가져온 기기에서 생성된 ED25519 키
로그인 암호 → 자신의 Mac 로그인 암호
```

설정을 마치면 먼저 SSH 연결이 되는지 확인한 뒤 Lock / Unlock 동작을 테스트하는 것이 좋습니다.


## 단축어 받기

아래 단축어는 배포용 템플릿입니다.

> ⚠️ 실제 로그인 암호는 포함되어 있지 않습니다.
> 설치 후 자신의 Mac 호스트, 사용자 이름, SSH 키, 로그인 암호를 설정해야 합니다.

[📱 Mac Lock / Unlock v2 단축어 받기](https://www.icloud.com/shortcuts/827d4dc0a3f94b089cb831426b913f60)


## 그래도 남아 있는 한계

v2가 v1보다 안전해진 것은 맞지만, 이 방식이 Touch ID나 Apple Watch와
같은 macOS의 공식 인증 기능을 대체하는 것은 아닙니다.

잠금 해제를 위해서는 여전히 iPhone 단축어 안에서 Mac 로그인 암호를
사용하고, 최종적으로 `System Events`가 해당 암호를 잠금 화면에
입력합니다.

이번 v2의 목적은 이 구조 자체를 없애는 것이 아니라,

**"Mac이 잠겨 있다는 것을 확인한 경우에만 암호 입력 코드에 도달하도록
만드는 것"**

입니다.

또한 `CGSSessionScreenIsLocked`는 제 환경에서 정상적으로 동작하는 것을
확인했지만, macOS 버전이나 환경에 따라 동작이 달라질 가능성도 고려해야
합니다.

## 마무리

v1을 만들었을 때는 단순히 **"iPhone으로 Mac을 열 수 있다"**는 편리함에
집중했습니다.

하지만 실제로 사용해보니 자동화는 **잘 동작하는 것만큼 잘못 동작했을 때
어떻게 되는지도 중요하다**는 것을 알게 됐습니다.

그래서 v2에서는 기능을 더 많이 추가하기보다 먼저 현재 상태를 확인하고,
위험한 동작은 특정 조건에서만 실행하도록 바꿨습니다.

``` text
v1
iPhone → SSH → 암호 입력 → Unlock

v2
iPhone
  ↓
Mac 상태 확인
  ↓
잠겨 있음 → Unlock
열려 있음 → 확인 → Lock → Display Sleep
```

작은 자동화지만 직접 사용하면서 문제를 발견하고 다시 구조를 고치는
과정이 꽤 재미있었습니다.

다음 버전이 생긴다면 그때도 실제로 사용하면서 발견한 문제부터 하나씩
개선해볼 생각입니다.
