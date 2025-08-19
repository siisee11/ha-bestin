# BESTIN 패킷 분석 명세서

## 목차
1. [개요](#개요)
2. [공통 패킷 구조](#공통-패킷-구조)
3. [조명(Light) 패킷 명세](#조명light-패킷-명세)
4. [콘센트(Outlet) 패킷 명세](#콘센트outlet-패킷-명세)
5. [난방(Thermostat) 패킷 명세](#난방thermostat-패킷-명세)
6. [환기(Fan) 패킷 명세](#환기fan-패킷-명세)
7. [가스(Gas) 패킷 명세](#가스gas-패킷-명세)
8. [에너지(Energy) 패킷 명세](#에너지energy-패킷-명세)

---

## 개요

BESTIN 스마트홈 시스템의 패킷 프로토콜 분석 문서입니다. 각 디바이스 타입별 패킷 구조와 명령어를 상세히 설명합니다.

### 게이트웨이 타입
- **AIO (All-In-One)**: 통합형 월패드, 0x50-0x5F 헤더 사용
- **Gen2 (2세대)**: 신형 월패드, 0x30-0x3F 헤더 사용  
- **General (일반형)**: 구형 월패드, 0x31 고정 헤더 사용

### 체크섬 계산
```python
def calculate_checksum(packet):
    checksum = 3
    for byte in packet[:-1]:
        checksum ^= byte
        checksum = (checksum + 1) & 0xFF
    return checksum
```

---

## 공통 패킷 구조

### 기본 구조
```
[STX][HEADER][LENGTH][TYPE][SEQ][DATA...][CHECKSUM]
```

| 필드 | 위치 | 크기 | 설명 |
|------|------|------|------|
| STX | 0 | 1 | 시작 바이트 (0x02) |
| HEADER | 1 | 1 | 디바이스 타입 식별자 |
| LENGTH | 2 | 1 | 패킷 전체 길이 |
| TYPE | 3 | 1 | 명령 타입 (Query/Response/Control) |
| SEQ | 4 | 1 | 시퀀스 번호 |
| DATA | 5~ | 가변 | 데이터 영역 |
| CHECKSUM | 마지막 | 1 | XOR 체크섬 |

### 명령 타입
- **Query (조회)**: 0x01, 0x02, 0x11, 0x21
- **Response (응답)**: 0x80, 0x82, 0x91, 0xA1, 0xB1
- **Control (제어)**: 0x01, 0x12, 0x22

---

## 조명(Light) 패킷 명세

### 1. 디바이스 식별자
- **AIO Gateway**: 0x50 + room_id (0x51~0x5F)
- **Gen2 Gateway**: 0x30 + room_id (0x31~0x36)
- **General Gateway**: 0x31 (고정)

### 2. 조명 제어 패킷 (Command)

#### 2.1 AIO Gateway 조명 제어
```
패킷 구조: [0x02][0x5X][0x0A][0x12][SEQ][DATA(5)][CHECKSUM]
총 길이: 10 바이트
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 0 | STX | 0x02 |
| 1 | Room ID | 0x50 + room_id (0x51~0x5F) |
| 2 | Length | 0x0A (10 bytes) |
| 3 | Type | 0x12 (Control) |
| 4 | Sequence | 타임스탬프 & 0xFF |
| 5 | On/Off | 0x01(ON) / 0x00(OFF) |
| 6 | Position | 1 << pos_id (비트마스크) |
| 7-8 | Reserved | 0x00 |
| 9 | Checksum | 계산된 체크섬 |

**예시: Room 1, Light 0 켜기**
```
02 51 0A 12 A5 01 01 00 00 3C
```

#### 2.2 Gen2 Gateway 조명 제어
```
패킷 구조: [0x02][0x3X][0x0E][0x21][SEQ][DATA(9)][CHECKSUM]
총 길이: 14 바이트
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 0 | STX | 0x02 |
| 1 | Room ID | 0x30 + room_id (0x31~0x36) |
| 2 | Length | 0x0E (14 bytes) |
| 3 | Type | 0x21 (Control) |
| 4 | Sequence | 타임스탬프 & 0xFF |
| 5 | Command | 0x01 (고정) |
| 6 | Reserved | 0x00 |
| 7 | Position | pos_id + 1 (1~4) |
| 8 | On/Off | 0x01(ON) / 0x02(OFF) |
| 9 | Brightness | 0xFF(무시) / 0~100(디밍값) |
| 10 | Color Temp | 0xFF(무시) / 0~10(색온도) |
| 11 | Reserved | 0x00 |
| 12 | Reserved | 0xFF |
| 13 | Checksum | 계산된 체크섬 |

**예시: Room 2, Light 1 켜기 (밝기 50%, 색온도 5)**
```
02 32 0E 21 B3 01 00 02 01 32 05 00 FF 4E
```

#### 2.3 General Gateway 조명 제어
```
패킷 구조: [0x02][0x31][0x0D][0x01][SEQ][DATA(8)][CHECKSUM]
총 길이: 13 바이트
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 0 | STX | 0x02 |
| 1 | Header | 0x31 (고정) |
| 2 | Length | 0x0D (13 bytes) |
| 3 | Type | 0x01 (Control) |
| 4 | Sequence | 타임스탬프 & 0xFF |
| 5 | Room ID | room_id & 0x0F |
| 6 | Position | (1 << pos_id) \| 0x80(ON) / 0x00(OFF) |
| 7-10 | Reserved | 0x00 |
| 11 | On/Off Flag | 0x04(ON) / 0x00(OFF) |
| 12 | Checksum | 계산된 체크섬 |

**예시: Room 3, Light 2 켜기**
```
02 31 0D 01 C7 03 84 00 00 00 00 04 B5
```

### 3. 조명 상태 응답 패킷 (Response)

#### 3.1 AIO/Gen2 Gateway 응답
```
패킷 구조: [0x02][HEADER][LENGTH][0x91][SEQ][DATA...][CHECKSUM]
```

**데이터 구조 (가변 길이)**
- 바이트 5-9: 헤더 정보
- 바이트 10: 조명 개수 (하위 4비트)
- 바이트 11: 콘센트 개수
- 바이트 12~: 디바이스 상태 데이터

**조명 상태 데이터 (13바이트/조명)**
| 오프셋 | 설명 | 값 |
|--------|------|-----|
| +0 | 상태 | 0x01(ON) / 0x00(OFF) |
| +1 | 밝기 | 0~255 (0~100% 매핑) |
| +2 | 색온도 | 0~255 (3000K~5700K 매핑) |
| +3~12 | Reserved | 0x00 |

#### 3.2 General Gateway 응답
```
패킷 구조: [0x02][0x31][LENGTH][0x91][SEQ][DATA...][CHECKSUM]
```

**예시: Room 1, 2개 조명 상태**
```
02 31 27 91 A5 01 00 00 00 00 02 00 00 00 00 00 00 00
01 FF 80 00 00 00 00 00 00 00 00 00 00  // Light 0: ON, 100%, 중간색온도
00 00 00 00 00 00 00 00 00 00 00 00 00  // Light 1: OFF
7C
```

### 4. 스마트 조명 (디밍/색온도 지원)

#### 4.1 밝기 변환
```python
def convert_brightness(value, reverse=False):
    if reverse:  # HA -> Device (0~255 -> 0~10)
        value = round(value / 2.55)
        return (value // 10) * 10 // 10  # 10단계로 변환
    else:  # Device -> HA (0~10 -> 0~255)
        return round(value * 10 * 2.55)
```

#### 4.2 색온도 변환
```python
COLOR_TEMP_LEVELS = [3000, 3300, 3600, 3900, 4200, 4500, 4800, 5100, 5400, 5700]

def convert_color_temp(value, reverse=False):
    if reverse:  # Kelvin -> Device Level (3000K~5700K -> 0~10)
        for i, temp in enumerate(COLOR_TEMP_LEVELS):
            if value < temp:
                return i
        return 10
    else:  # Device Level -> Kelvin (0~10 -> 3000K~5700K)
        index = max(value - 1, 0)
        return COLOR_TEMP_LEVELS[index]
```

### 5. 조명 제어 시나리오

#### 5.1 단순 ON/OFF
1. 제어 패킷 전송 (Type: 0x01/0x12/0x21)
2. ACK 패킷 수신 (Type: 0x82)
3. 상태 조회 패킷 전송 (Type: 0x11)
4. 상태 응답 패킷 수신 (Type: 0x91)

#### 5.2 디밍 제어 (Gen2/스마트조명)
1. 디밍값 포함 제어 패킷 전송
   - Brightness 필드: 0~100 (0%~100%)
2. ACK 패킷 수신
3. 상태 확인

#### 5.3 색온도 제어 (Gen2/스마트조명)
1. 색온도값 포함 제어 패킷 전송
   - Color Temp 필드: 0~10 (3000K~5700K)
2. ACK 패킷 수신
3. 상태 확인

### 6. 오류 처리

#### 6.1 체크섬 오류
- 재전송 요청 (최대 3회)
- 실패 시 오류 로그 기록

#### 6.2 타임아웃
- AIO/General: 120ms
- Gen2: 220ms
- 타임아웃 시 재전송 (최대 2회)

#### 6.3 잘못된 응답
- 시퀀스 번호 불일치: 무시하고 대기
- 헤더 불일치: 오류 로그 후 재시도

### 7. 실제 패킷 예시

#### 7.1 Room 1, Light 0 켜기 (General)
```
TX: 02 31 0D 01 A5 01 81 00 00 00 00 04 93
RX: 02 31 82 A5 1A  // ACK
TX: 02 31 00 11 A6 26  // 상태 조회
RX: 02 31 27 91 A6 01 00 00 00 00 02 00 ... // 상태 응답
```

#### 7.2 Room 2, Light 1 디밍 50% (Gen2)
```
TX: 02 32 0E 21 B3 01 00 02 01 32 FF 00 FF DD
RX: 02 32 82 B3 2B  // ACK
```

#### 7.3 Room 3, 모든 조명 끄기 (AIO)
```
TX: 02 53 0A 12 C7 00 0F 00 00 4F  // 비트마스크 0x0F = 모든 조명
RX: 02 53 82 C7 3D  // ACK
```

---

## 콘센트(Outlet) 패킷 명세

### 1. 콘센트 제어 패킷

#### 1.1 General Gateway
```
패킷 구조: [0x02][0x31][0x0D][0x01][SEQ][DATA(8)][CHECKSUM]
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 5 | Room ID | room_id & 0x0F |
| 7 | Position | (1 << pos_id) \| 0x80(ON) |
| 8 | Standby Cut | 0x83(ON) / 0x03(OFF) |
| 11 | On/Off Flag | 0x09 << pos_id (ON) / 0x00 |

### 2. 콘센트 상태 응답

**콘센트 상태 데이터 (14바이트/콘센트)**
| 오프셋 | 설명 | 값 |
|--------|------|-----|
| +0 | 상태 | 0x21(ON) / 0x20(OFF) |
| +1~7 | Reserved | 0x00 |
| +8~9 | 전력 사용량 | Big-endian, W*10 |
| +10~13 | Reserved | 0x00 |

---

## 난방(Thermostat) 패킷 명세

### 1. 난방 제어 패킷
```
패킷 구조: [0x02][0x28][0x0E][0x12][SEQ][DATA(9)][CHECKSUM]
총 길이: 14 바이트
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 5 | Room ID | room_id & 0x0F |
| 6 | On/Off | 0x01(ON) / 0x02(OFF) |
| 7 | 설정 온도 | 정수부 \| 0x40(0.5도) |

### 2. 난방 상태 응답 (16바이트)
```
패킷 구조: [0x02][0x28][0x10][0x91][SEQ][DATA(11)][CHECKSUM]
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 5 | Room ID | room_id & 0x0F |
| 6 | 상태 | 0x01(ON) / 0x00(OFF) |
| 7 | 설정 온도 | 온도값 & 0x3F, +0x40(0.5도) |
| 8-9 | 현재 온도 | Big-endian, °C*10 |
| 11 | 난방수 온도 | °C |
| 12 | 온수 온도 | °C |

### 3. 난방 설정 응답 (14바이트)
```
패킷 구조: [0x02][0x28][0x0E][0x91][SEQ][DATA(9)][CHECKSUM]
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 6-8 | 난방수 | 최소/최대/설정 온도 |
| 9-11 | 온수 | 최소/최대/설정 온도 |

---

## 환기(Fan) 패킷 명세

### 1. 환기 제어 패킷
```
패킷 구조: [0x02][0x61][TYPE][SEQ][0x00][DATA(4)][CHECKSUM]
총 길이: 10 바이트
```

**제어 타입**
- 0x00: 상태 조회
- 0x01: ON/OFF 제어
- 0x03: 풍량 설정
- 0x07: 자연환기 모드

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 5 | 상태 | 0x01(ON) / 0x00(OFF) / 0x10(자연환기) |
| 6 | 풍량 | 1~3 (약/중/강) |

### 2. 환기 상태 응답
```
패킷 구조: [0x02][0x61][0x80][SEQ][DATA(5)][CHECKSUM]
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 4 | Room ID | 설치 위치 |
| 5 | 상태 | bit0: ON/OFF, bit4: 자연환기 |
| 6 | 풍량 | 1~3 |

---

## 가스(Gas) 패킷 명세

### 1. 가스 차단 제어
```
패킷 구조: [0x02][0x31][0x02][SEQ][0x00][DATA(4)][CHECKSUM]
총 길이: 10 바이트
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 4 | 명령 | 0x00(차단) |

### 2. 가스 상태 응답
```
패킷 구조: [0x02][0x31][0x80][SEQ][DATA(5)][CHECKSUM]
```

| 바이트 | 설명 | 값 |
|--------|------|-----|
| 5 | 상태 | 0x00(차단) / 0x01(열림) |

---

## 에너지(Energy) 패킷 명세

### 1. 에너지 조회
```
패킷 구조: [0x02][0xD1][0x07][0x02][SEQ][0x00][CHECKSUM]
총 길이: 7 바이트
```

### 2. 에너지 응답
```
패킷 구조: [0x02][0xD1][LENGTH][0x82][SEQ][0x80][DATA...][CHECKSUM]
```

**에너지 데이터 (8바이트/타입)**
| 오프셋 | 설명 | 값 |
|--------|------|-----|
| +0 | 타입 ID | 0x01(전기) / 0x02(수도) / 0x03(온수) / 0x04(가스) / 0x05(난방) |
| +1 | 금일 사용량 | 단위값 |
| +2 | 전일 사용량 | 단위값 |
| +3 | 월 사용량 | 단위값 |
| +4 | 년 사용량 | 단위값 |
| +5 | Reserved | 0x00 |
| +6-7 | 실시간 사용량 | Big-endian, W |

**확장 데이터 (10바이트, ID & 0xF0 == 0x80)**
- 추가 2바이트의 누적 사용량 포함

---

## 부록

### A. 디바이스 타입 매핑
```python
DEVICE_TYPES = {
    0x28: "thermostat",  # 난방
    0x31: "light/outlet/gas",  # 조명/콘센트/가스
    0x41: "doorlock",  # 도어락
    0x61: "fan",  # 환기
    0xC1: "elevator",  # 엘리베이터
    0xD1: "energy",  # 에너지
    # AIO Gateway
    0x51-0x5F: "room_control",
    # Gen2 Gateway  
    0x31-0x36: "room_control"
}
```

### B. 패킷 타입 식별
```python
def identify_packet_type(packet):
    if len(packet) < 4:
        return None
    
    header = packet[1]
    length = packet[2]
    type_byte = packet[3]
    
    # 10바이트 고정 길이
    if length == 10 or length in [0x00, 0x80, 0x02, 0x82]:
        if type_byte == 0x00: return "QUERY"
        elif type_byte == 0x80: return "RESPONSE"
        else: return "ACK"
    
    # 가변 길이
    else:
        if type_byte in [0x02, 0x11, 0x21]: return "QUERY"
        elif type_byte in [0x82, 0x91, 0xA1, 0xB1]: return "RESPONSE"
        else: return "ACK"
```

### C. 실시간 모니터링 예제
```python
import socket
import time

def monitor_packets(host, port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect((host, port))
    
    while True:
        data = sock.recv(1024)
        if data and data[0] == 0x02:
            hex_str = ' '.join(f'{b:02X}' for b in data)
            print(f"[{time.strftime('%H:%M:%S')}] {hex_str}")
            
            # 패킷 파싱
            if len(data) > 3:
                device_type = identify_device(data[1])
                packet_type = identify_packet_type(data)
                print(f"  Device: {device_type}, Type: {packet_type}")

monitor_packets("192.168.0.27", 8899)
```

---

*문서 버전: 1.0.0*  
*최종 수정: 2024년*