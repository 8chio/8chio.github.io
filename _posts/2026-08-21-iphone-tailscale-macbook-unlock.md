---
title: "iPhone 단축어와 Tailscale로 클램쉘 MacBook 잠금 해제하기"
date: 2026-08-21 17:40:00 +0900
categories: [macOS, Automation]
tags: [macOS, iPhone, Shortcuts, SSH, Tailscale, AppleScript]
image:
  path: /assets/img/posts/macbook-unlock-tailscale.png
  alt: iPhone 단축어와 Tailscale을 이용한 MacBook 잠금 해제 구성
---

MacBook을 외장 모니터에 연결해 **클램쉘 모드**로 사용하다 보면 한 가지 불편한 점이 생긴다.

**MacBook 본체의 Touch ID를 사용할 수 없다는 것.**

외장 키보드에서 매번 로그인 암호를 입력하는 대신, 평소 옆에 두고 사용하는 iPhone으로 Mac을 깨우고 잠금을 해제하도록 자동화해봤다.

이번 구성에는 **iPhone 단축어 + SSH + Tailscale + AppleScript**를 사용했다.

> 이 방법은 macOS의 공식 잠금 해제 기능이 아니라 개인적인 편의를 위한 자동화다.  
> 로그인 암호를 자동으로 입력하는 방식이므로 보안상 주의사항을 반드시 확인해야 한다.

![MacBook 잠금 해제 전체 구조](/assets/img/posts/mac-unlock-architecture.png)

## 전체 구조

```text
iPhone 단축어
      ↓
   Tailscale
      ↓
 MagicDNS
(macbook-pro)
      ↓
 SSH / ED25519
      ↓
   MacBook
      ↓
  caffeinate
      ↓
   osascript
      ↓
 System Events
      ↓
암호 입력 + Enter
      ↓
   🔓 Unlock
```

각 구성요소의 역할은 단순하다.

| 구성요소      | 역할                                 |
| ------------- | ------------------------------------ |
| iPhone 단축어 | 잠금 해제 명령 실행                  |
| Tailscale     | 네트워크가 달라도 Mac에 접근         |
| MagicDNS      | IP 대신 `macbook-pro` 같은 이름 사용 |
| SSH           | iPhone에서 Mac으로 명령 전달         |
| ED25519       | SSH 공개키 인증                      |
| `caffeinate`  | Mac 화면 깨우기                      |
| `osascript`   | AppleScript 실행                     |
| System Events | 키보드 입력 자동화                   |

## 1. Mac에서 SSH 활성화

macOS에서 다음 메뉴로 이동한다.

**시스템 설정 → 일반 → 공유 → 원격 로그인**

원격 로그인을 활성화한 뒤 터미널에서 사용자 이름을 확인한다.

```bash
whoami
```

예를 들어 다음처럼 나온다면:

```text
hok
```

iPhone 단축어의 SSH 사용자 이름도 `hok`를 사용하면 된다.

## 2. iPhone 단축어에서 SSH 설정

단축어 앱에서 **`SSH를 통해 스크립트 실행`** 동작을 추가한다.

내 설정은 다음과 같다.

```text
호스트 : macbook-pro
포트   : 22
사용자 : hok
인증   : SSH 키
키     : ED25519
```

SSH 연결이 정상인지 먼저 확인하려면 스크립트 칸에 다음 명령을 넣고 실행한다.

```bash
whoami
```

Mac 사용자 이름이 반환되면 SSH 연결은 정상이다.

### ED25519 공개키 등록

iPhone 단축어의 SSH 키 설정에서 **공개 키 복사**를 선택한다.

Mac에서:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

복사한 공개키를 한 줄 그대로 붙여 넣고 저장한다.

그다음 권한을 설정한다.

```bash
chmod 600 ~/.ssh/authorized_keys
```

## 3. 왜 Tailscale을 사용했는가?

처음에는 Mac의 로컬 IP 주소로 SSH에 접속했다.

예를 들면:

```text
192.168.0.10
```

집에서는 잘 동작했지만 장소가 바뀌면 IP 주소도 달라졌고 휴대폰 와이파이가 꺼져있는 경우, 모바일데이터를 사용하는경우 문제가 생겼다.

```text
집 Wi-Fi        → 192.168.x.x
출장지 Wi-Fi     → 172.x.x.x
다른 네트워크    → 또 다른 IP
```

출장이 잦다 보니 매번 iPhone 단축어의 SSH 호스트를 수정하는 것이 번거로웠다.

그래서 iPhone과 Mac에 **Tailscale**을 설치하고 같은 Tailnet에 연결했다.

MagicDNS를 사용하면 Tailscale IP를 직접 기억할 필요 없이:

```text
macbook-pro
```

처럼 기기 이름으로 접속할 수 있다.

이후에는 집 Wi-Fi, 출장지 Wi-Fi, iPhone 5G 등 네트워크가 바뀌어도 단축어를 수정할 필요가 없어졌다.

## 4. macOS 손쉬운 사용 권한

SSH에서 실행한 AppleScript가 키보드 입력을 발생시키려면 macOS에서 관련 프로세스에 **손쉬운 사용(Accessibility)** 권한이 필요하다.

내 환경에서는 다음 위치에서:

**시스템 설정 → 개인정보 보호 및 보안 → 손쉬운 사용**

`sshd-keygen-wrapper`를 허용하니 정상적으로 동작했다.

> macOS 버전에 따라 표시되는 프로세스 이름이나 동작은 달라질 수 있다.

## 5. 잠금 해제 스크립트

최종적으로 iPhone 단축어의 **SSH를 통해 스크립트 실행**에 넣은 내용은 다음과 같다.

```bash
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

> `YOUR_MAC_PASSWORD`에는 실제 Mac 로그인 암호가 들어간다.  
> 블로그나 GitHub에 실제 암호가 포함된 단축어나 스크립트를 올리지 않도록 주의해야 한다.

스크립트의 동작은 다음과 같다.

```text
caffeinate
    ↓
Mac 화면 깨우기
    ↓
1초 대기
    ↓
← 방향키 입력
    ↓
로그인 화면 활성화
    ↓
1초 대기
    ↓
로그인 암호 입력
    ↓
0.3초 대기
    ↓
Enter
    ↓
🔓 Unlock
```

### `caffeinate`

```bash
caffeinate -u -t 3
```

사용자 활동이 발생한 것처럼 macOS에 알려 디스플레이를 깨운다.

### `key code 123`

```applescript
key code 123
```

macOS에서 **← 방향키**를 의미한다.

잠금 화면이 키보드 입력을 받을 수 있도록 활성화하기 위해 사용했다.

### `keystroke`

```applescript
keystroke "YOUR_MAC_PASSWORD"
```

로그인 암호를 실제 키보드 입력처럼 입력한다.

### `key code 36`

```applescript
key code 36
```

**Return(Enter)** 키다.

암호 입력 후 Enter를 눌러 로그인을 시도한다.

## 6. 실제 사용

MacBook을 클램쉘 상태로 사용하다 잠기면 iPhone에서 단축어를 한 번 실행한다.

```text
📱 iPhone
    ↓
Mac Unlock 단축어
    ↓
Tailscale
    ↓
SSH / ED25519
    ↓
💻 MacBook
    ↓
화면 깨우기
    ↓
🔓 잠금 해제
```

Tailscale 덕분에 iPhone과 MacBook이 서로 다른 네트워크에 있어도 동작한다.

예를 들면:

```text
iPhone  → 5G
MacBook → 호텔 Wi-Fi
```

같은 상황에서도 두 기기 모두 Tailscale에 연결되어 있다면 동일한 `macbook-pro` 호스트명으로 SSH 접속할 수 있다.

## 보안상 주의할 점

이 방법은 **Touch ID나 Apple Watch처럼 macOS가 공식 제공하는 인증 기능을 대체하는 방법이 아니다.**

실제로는 로그인 화면에 **Mac 로그인 암호를 자동으로 타이핑하는 방식**이다.

따라서 다음 사항은 꼭 지키는 것이 좋다.

- SSH 인증에는 암호 대신 **ED25519 공개키 인증** 사용
- SSH 22번 포트를 인터넷에 직접 노출하지 않기
- Tailscale과 같은 사설 네트워크를 통해 접근
- 실제 로그인 암호가 들어 있는 단축어 공유 금지
- 단축어 설정 화면이나 암호가 포함된 스크린샷 공개 금지
- Mac이 이미 잠금 해제된 상태에서는 단축어 실행 주의

> **이 자동화는 편의를 위한 개인용 구성이다. Touch ID나 Apple Watch의 보안 인증을 대체하지 않는다.**

## 마무리

시작은 단순했다.

**“클램쉘 모드에서도 Touch ID처럼 편하게 Mac을 열 수 없을까?”**

처음에는 iPhone 단축어와 SSH만 사용했지만, 네트워크가 바뀔 때마다 IP 주소를 수정해야 하는 불편함이 생겼다.

여기에 Tailscale과 MagicDNS를 추가하면서 최종적으로:

```text
클램쉘 Touch ID 불편
        ↓
iPhone 단축어
        ↓
SSH 자동화
        ↓
네트워크 변경 문제
        ↓
Tailscale + MagicDNS
        ↓
어디서든 같은 방식으로 실행
```

이라는 구조가 완성됐다.

작은 불편함에서 시작했지만, SSH 공개키 인증과 AppleScript, Tailscale까지 직접 연결해보면서 꽤 재미있는 자동화 프로젝트가 됐다.
