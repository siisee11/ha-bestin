# BESTIN Integration Tests - 종합 테스트 도구 문서

## 📌 개요

이 디렉토리는 BESTIN Home Assistant 통합 컴포넌트의 개발, 테스트, 디버깅을 위한 독립 실행형 도구 모음입니다. HDC 스마트홈 시스템의 프로토콜 분석, 통신 테스트, 디바이스 제어 테스트를 수행할 수 있습니다.

### 주요 용도
- **프로토콜 리버스 엔지니어링**: 새로운 디바이스 타입 분석
- **통신 테스트**: 시리얼/소켓 연결 검증
- **디바이스 제어**: 명령 송수신 테스트
- **실시간 모니터링**: 상태 변화 추적
- **디버깅**: 패킷 덤프 및 분석

## 📁 파일 구조 및 설명

```
tests/
├── client.py           # Smart Home 1.0 웹 API 클라이언트
├── dimming_light.py    # 조명 디밍 실제 패킷 데이터셋
├── dimming_test.py     # 실시간 디밍 모니터링 도구
├── packet_analyze.md   # 난방 패킷 프로토콜 문서
├── packet_extract.py   # 대량 패킷 추출/분류 도구
├── sse_client.py       # Server-Sent Events 클라이언트
└── test.py            # 범용 통신 테스트 프레임워크
```

---

## 🔧 각 테스트 도구 상세 설명

### 1. **client.py** - Smart Home 1.0 웹 API 클라이언트

HDC 스마트홈 1.0 버전의 웹 기반 API와 통신하는 클라이언트입니다. PHP 세션 기반 인증을 사용하며, 디바이스 상태 조회 및 제어가 가능합니다.

#### 핵심 기능
- **세션 인증 관리**: PHP 세션 쿠키 자동 처리
- **디바이스 상태 조회**: 전체 홈 디바이스 상태 확인
- **비동기 통신**: aiohttp 기반 효율적인 비동기 처리
- **에러 처리**: 인증 실패 및 네트워크 오류 처리

#### 코드 구조 및 동작 원리

```python
import aiohttp
import asyncio

async def _session():
    """웹앱 로그인 및 세션 생성"""
    base_url = "http://<server>/webapp/data/getLoginWebApp.php"
    params = {
        "login_ide": "",  # 사용자 ID
        "login_pwd": ""   # 비밀번호
    }
    
    # 브라우저 헤더 시뮬레이션
    headers = {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,...",
        "Accept-Encoding": "gzip, deflate",
        "Accept-Language": "ko,en-US;q=0.9,en;q=0.8,ko-KR;q=0.7",
        "Cache-Control": "max-age=0",
        "Connection": "keep-alive",
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."
    }
    
    async with aiohttp.ClientSession() as session:
        async with session.get(url=base_url, params=params, headers=headers) as response:
            # JSON 응답 파싱 (content_type="text/html" 주의)
            data = await response.json(content_type="text/html")
            
            # 응답 검증
            if "_fair" in data.get("ret", ""):
                print(f"로그인 실패: {data.get('msg')}")
                return
            
            # 쿠키 추출
            cookies = response.cookies
            phpsessid = cookies.get('PHPSESSID')
            user_id = cookies.get('user_id')
            user_name = cookies.get('user_name')
            
            # 새 쿠키 딕셔너리 생성
            new_cookie = {
                'PHPSESSID': phpsessid.value,
                'user_id': user_id.value,
                'user_name': user_name.value,
            }
            
            return new_cookie

async def _device_status(cookies):
    """디바이스 상태 조회"""
    status_url = "http://<server>/webapp/data/getDeviceStatus.php"
    
    async with aiohttp.ClientSession(cookies=cookies) as session:
        async with session.get(url=status_url) as response:
            devices = await response.json()
            return devices
```

#### 사용 예시

```python
import asyncio

async def main():
    # 1. 로그인 및 세션 생성
    cookies = await _session()
    print(f"세션 생성 완료: {cookies}")
    
    # 2. 디바이스 상태 조회
    devices = await _device_status(cookies)
    for device in devices:
        print(f"디바이스: {device['name']}, 상태: {device['status']}")

if __name__ == "__main__":
    asyncio.run(main())
```

#### 응답 데이터 형식

```json
{
    "ret": "success",
    "msg": "로그인 성공",
    "devices": [
        {
            "id": "light_1",
            "name": "거실 조명",
            "type": "light",
            "status": "on",
            "brightness": 80
        },
        {
            "id": "thermostat_1",
            "name": "거실 온도조절기",
            "type": "thermostat",
            "status": "heat",
            "current_temp": 23.5,
            "set_temp": 24.0
        }
    ]
}
```

---

### 2. **dimming_light.py** - 조명 디밍 패킷 데이터셋

실제 월패드와 통신 중 캡처한 조명 제어 패킷 데이터 모음입니다. 각 방의 조명 상태 요청/응답 쌍으로 구성되어 있습니다.

#### 데이터 구조

```python
TEST_CASE = [
    # Room 0x21 (방 1)
    "02 21 11 10 D6 00 00 00 0D 06 00 00 EB 00 00 00 ED",  # 요청 패킷
    "02 21 11 90 D6 00 00 00 0D 06 00 00 EB 00 FE 06 91",  # 응답 패킷
    
    # Room 0x22 (방 2)
    "02 22 11 10 D7 00 00 00 0D 06 00 00 FA 00 00 00 10",  # 요청 패킷
    "02 22 11 90 D7 00 00 00 0D 06 00 00 FA 01 0E 00 85",  # 응답 패킷
    
    # ... 더 많은 방 데이터
]
```

#### 패킷 구조 분석

##### 요청 패킷 (17 바이트)
```
02 21 11 10 D6 00 00 00 0D 06 00 00 EB 00 00 00 ED
│  │  │  │  │  └─────────────────────────────┘  │
│  │  │  │  │            DATA (11 bytes)        │
│  │  │  │  └─ SEQ: 시퀀스 번호 (0xD6)          │
│  │  │  └─ TYPE: 0x10 (요청)                   │
│  │  └─ LENGTH: 0x11 (17 바이트)               │
│  └─ ROOM_ID: 0x21 (방 1)                      │
└─ STX: 0x02 (시작 바이트)                      └─ CHECKSUM
```

##### 응답 패킷 (17 바이트)
```
02 21 11 90 D6 00 00 00 0D 06 00 00 EB 00 FE 06 91
│  │  │  │  │  └─────────────────────────────┘  │
│  │  │  │  │            DATA (11 bytes)        │
│  │  │  │  └─ SEQ: 시퀀스 번호 (0xD6)          │
│  │  │  └─ TYPE: 0x90 (응답)                   │
│  │  └─ LENGTH: 0x11 (17 바이트)               │
│  └─ ROOM_ID: 0x21 (방 1)                      │
└─ STX: 0x02 (시작 바이트)                      └─ CHECKSUM
```

#### 응답 데이터 해석

| 바이트 위치 | 필드명 | 설명 | 예시 값 |
|------------|--------|------|---------|
| 13 | 조명 개수 | 방의 조명 개수 | `0x00-0x0F` |
| 14 | 디밍 지원 | 디밍 가능 여부 | `0xFE`: 지원, `0x0E`: 미지원 |
| 15 | 조명 그룹 | 그룹 정보 | `0x06`: 그룹 ID |
| 16 | 상태 플래그 | 추가 상태 정보 | 비트 플래그 |

#### 활용 방법

```python
# 패킷 데이터 파싱 예시
def parse_light_response(packet_hex):
    """조명 응답 패킷 파싱"""
    bytes_data = bytes.fromhex(packet_hex.replace(" ", ""))
    
    room_id = bytes_data[1] & 0x0F
    seq_num = bytes_data[4]
    light_count = bytes_data[13]
    dimming_support = bytes_data[14] == 0xFE
    
    return {
        "room_id": room_id,
        "seq_num": seq_num,
        "light_count": light_count,
        "dimming_support": dimming_support
    }

# 사용 예시
packet = "02 21 11 90 D6 00 00 00 0D 06 00 00 EB 00 FE 06 91"
result = parse_light_response(packet)
print(f"방 {result['room_id']}: 조명 {result['light_count']}개, 디밍 {'지원' if result['dimming_support'] else '미지원'}")
```

---

### 3. **dimming_test.py** - 실시간 디밍 모니터링 도구

소켓 연결을 통해 실시간으로 조명 및 콘센트 상태를 모니터링하는 도구입니다. 디밍 레벨, 색온도, 전력 사용량 등을 실시간으로 확인할 수 있습니다.

#### 주요 클래스: `SocketClient`

```python
class SocketClient:
    def __init__(self, server_address, port):
        self.server_address = server_address
        self.port = port
        self.sock = None
        self.constant_packet_length = 10  # 고정 길이 패킷
```

#### 핵심 기능 구현

##### 1. 패킷 수신 로직 (`_receive_socket`)

```python
def _receive_socket(self):
    """소켓에서 완전한 패킷을 수신"""
    
    def recv_exactly(n):
        """정확히 n 바이트 수신"""
        data = b''
        while len(data) < n:
            chunk = self.sock.recv(n - len(data))
            if not chunk:
                raise socket.error("Connection closed")
            data += chunk
        return data
    
    packet = b''
    
    # 1. STX(0x02) 찾기
    while True:
        initial_data = self.sock.recv(1)
        packet += initial_data
        if 0x02 in packet:
            start_index = packet.index(0x02)
            packet = packet[start_index:]  # STX부터 시작
            break
    
    # 2. 헤더 읽기 (최소 3바이트)
    if len(packet) < 3:
        packet += recv_exactly(3 - len(packet))
    
    # 3. 패킷 타입 검증
    room_id = packet[1]
    if room_id not in [0x31, 0x32, 0x33, 0x34] and (room_id & 0xF0) != 0x50:
        return b''
    
    # 4. 패킷 길이 결정
    if (room_id == 0x31 and packet[2] in [0x00, 0x80]) or room_id == 0x61:
        packet_length = 10  # 고정 길이
    else:
        packet_length = packet[2]  # 가변 길이
    
    # 5. 나머지 데이터 수신
    packet += recv_exactly(packet_length - len(packet))
    
    return packet[:packet_length]
```

##### 2. 데이터 파싱 (`parse_data`)

```python
def parse_data(self, data):
    """패킷 데이터를 파싱하여 디바이스 정보 추출"""
    
    rid = data[1] & 0x0F  # Room ID 추출
    
    # 조명/콘센트 개수 파악
    if rid % 2 == 0:  # 짝수 Room
        light_count = data[10] & 0x0F
        outlet_count = data[11]
        base_count = outlet_count
    else:  # 홀수 Room
        light_count = data[10]
        outlet_count = data[11]
        base_count = light_count
    
    # 인덱스 계산
    light_start_idx = 18
    outlet_start_idx = light_start_idx + (base_count * 13)
    
    # 조명 정보 파싱
    for i in range(light_count):
        idx = light_start_idx + (i * 13)
        is_on = data[idx] == 0x01
        dimming_level = data[idx + 1]      # 0-255
        color_temperature = data[idx + 2]   # 0-255 (따뜻함-차가움)
        
        print(f"조명 {rid}/{i}:")
        print(f"  상태: {'켜짐' if is_on else '꺼짐'}")
        print(f"  밝기: {dimming_level * 100 // 255}%")
        print(f"  색온도: {color_temperature}")
    
    # 콘센트 정보 파싱
    for i in range(outlet_count):
        idx = outlet_start_idx + (i * 14)
        is_on = data[idx] == 0x21
        power_idx = idx + 8
        current_power = int.from_bytes(data[power_idx:power_idx+2], "big") / 10
        
        print(f"콘센트 {rid}/{i}:")
        print(f"  상태: {'켜짐' if is_on else '꺼짐'}")
        print(f"  전력: {current_power}W")
```

##### 3. 체크섬 검증

```python
def checksum(self, data):
    """XOR 기반 체크섬 검증"""
    checksum = 3
    for byte in data[:-1]:  # 마지막 바이트(체크섬) 제외
        checksum ^= byte
        checksum = (checksum + 1) & 0xFF
    return checksum == data[-1]
```

#### 실제 사용 예시

```python
if __name__ == "__main__":
    # 클라이언트 생성 및 연결
    client = SocketClient('192.168.0.27', 8899)
    client.connect()
    
    # 실시간 모니터링 시작
    client.receive_data()
```

#### 출력 예시

```
02 33 3B 91 E5 01 01 00 00 00 02 02 00 00 00 00 00 00 01 FF 80 00...
조명 3/0:
  상태: 켜짐
  밝기: 100%
  색온도: 128
조명 3/1:
  상태: 꺼짐
  밝기: 0%
  색온도: 128
콘센트 3/0:
  상태: 켜짐
  전력: 45.3W
콘센트 3/1:
  상태: 꺼짐
  전력: 0.0W
```

---

### 4. **packet_analyze.md** - 난방 패킷 프로토콜 문서

난방 시스템의 패킷 구조를 리버스 엔지니어링한 상세 분석 문서입니다.

#### 문서 내용 요약

##### 16바이트 난방 상태 패킷
- **용도**: 각 방의 난방 상태 정보 전송
- **주요 필드**: 현재 온도, 설정 온도, 켜짐/꺼짐 상태
- **온도 변환**: 16진수 → 10진수 → ÷10 = °C

##### 14바이트 난방 설정 패킷
- **용도**: 난방수/온수 온도 설정값 전송
- **주요 필드**: 최소/최대/설정 온도
- **에러 코드**: 시스템 오류 상태

#### 상세 분석 예시

```markdown
## 패킷 상세
02 28 10 91 A8 02 02 58 00 F2 00 25 00 00 00 8E

| 필드명 | 위치 | 길이 | 값 | 설명 |
|--------|------|------|-----|------|
| PREFIX | 0 | 1 | `02` | 패킷 시작 |
| HEADER | 1 | 1 | `28` | 난방 패킷 |
| LEN | 2 | 1 | `10` | 16바이트 |
| TYPE | 3 | 1 | `91` | 응답 타입 |
| SEQ | 4 | 1 | `A8` | 시퀀스 |
| ROOM_ID | 5 | 1 | `02` | 방 ID |
| 상태 | 6 | 1 | `02` | OFF |
| SET_TEMP | 7 | 1 | `58` | 8.8°C |
| CUR_TEMP | 8-9 | 2 | `00F2` | 24.2°C |
```

---

### 5. **packet_extract.py** - 대량 패킷 추출/분류 도구

장시간 캡처한 통신 로그에서 특정 패턴의 패킷을 추출하고 분류하는 도구입니다.

#### 주요 기능

##### 멀티라인 패킷 파싱

```python
CONTROL_PACKET_LIST = [
    """
    # 여러 줄에 걸친 패킷 데이터
    02 61 00 99 00 00 00 00 00 02  # 환기 OFF
    02 61 80 99 80 00 01 00 03 04  # 환기 응답
    
    02 28 06 21 9A 8B  # 난방 요청
    02 28 19 A1 9A 10 32 04 02 19 01 13 02 58 00 F2...  # 난방 상태
    
    02 D1 07 02 9C 00 4D  # 에너지 요청
    02 D1 22 82 9C 80 05 01 02 18 24 37 00 32 80 02...  # 에너지 데이터
    """
]
```

##### 패킷 분류 및 추출

```python
def extract_packets(packet_list):
    """패킷 타입별 분류 및 추출"""
    
    classified = {
        "light": [],      # 0x31-0x36
        "thermostat": [], # 0x28
        "fan": [],        # 0x61
        "energy": [],     # 0xD1
        "doorlock": [],   # 0x41
    }
    
    for packet in packet_list:
        lines = packet.strip().splitlines()
        
        for line in lines:
            # 주석 제거
            if "#" in line:
                line = line.split("#")[0].strip()
            
            if not line:
                continue
            
            # 패킷 타입별 분류
            if line.startswith("02 31"):
                classified["light"].append(line)
            elif line.startswith("02 28"):
                # 난방 상태 응답만 추출
                if "02 28 19 A1" in line:
                    parts = line.split("02 28")
                    for part in parts:
                        if part.strip().startswith("19 A1"):
                            classified["thermostat"].append(f"02 28 {part.strip()}")
            elif line.startswith("02 61"):
                classified["fan"].append(line)
            elif line.startswith("02 D1"):
                classified["energy"].append(line)
            elif line.startswith("02 41"):
                classified["doorlock"].append(line)
    
    return classified
```

##### 통계 생성

```python
def generate_statistics(classified_packets):
    """패킷 통계 정보 생성"""
    
    stats = {}
    for device_type, packets in classified_packets.items():
        if packets:
            stats[device_type] = {
                "count": len(packets),
                "unique": len(set(packets)),
                "sample": packets[0] if packets else None
            }
    
    # 통계 출력
    print("=== 패킷 통계 ===")
    for device, stat in stats.items():
        print(f"{device}:")
        print(f"  총 개수: {stat['count']}")
        print(f"  고유 패킷: {stat['unique']}")
        print(f"  샘플: {stat['sample'][:50]}...")
    
    return stats
```

#### 사용 예시

```python
# 패킷 추출 및 분류
classified = extract_packets(CONTROL_PACKET_LIST)

# 특정 타입만 출력
for packet in classified["thermostat"]:
    print(f"난방 패킷: {packet}")

# 통계 생성
stats = generate_statistics(classified)
```

---

### 6. **sse_client.py** - Server-Sent Events 클라이언트

실시간 이벤트 스트림을 수신하는 SSE 클라이언트입니다. 엘리베이터 호출, 비상벨 등의 실시간 이벤트를 처리합니다.

#### 구현 방식

##### 동기 방식 (requests + sseclient)

```python
import requests
import sseclient

def sync_sse_client(url):
    """동기 방식 SSE 클라이언트"""
    
    headers = {
        "Accept": "text/event-stream",
        "Cache-Control": "no-cache",
    }
    
    try:
        with requests.get(url, stream=True, headers=headers) as response:
            client = sseclient.SSEClient(response)
            
            for event in client.events():
                # 이벤트 타입별 처리
                if event.event == "elevator_call":
                    handle_elevator_call(event.data)
                elif event.event == "emergency":
                    handle_emergency(event.data)
                else:
                    print(f"Unknown event: {event.event}")
                    print(f"Data: {event.data}")
                
    except KeyboardInterrupt:
        print("중단됨")
    except Exception as e:
        print(f"오류 발생: {e}")
    finally:
        client.close()

def handle_elevator_call(data):
    """엘리베이터 호출 이벤트 처리"""
    import json
    event_data = json.loads(data)
    
    floor = event_data.get("floor")
    direction = event_data.get("direction")
    timestamp = event_data.get("timestamp")
    
    print(f"[{timestamp}] {floor}층에서 {direction} 방향 호출")
```

##### 비동기 방식 (aiohttp)

```python
import aiohttp
import asyncio
import json

async def async_sse_client(url):
    """비동기 방식 SSE 클라이언트"""
    
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            async for line in response.content:
                # SSE 프로토콜 파싱
                if line.startswith(b"event:"):
                    event_type = line.decode().split(":", 1)[1].strip()
                
                elif line.startswith(b"data:"):
                    data = line.decode().split(":", 1)[1].strip()
                    
                    # 이벤트 처리
                    await process_event(event_type, data)
                
                elif line == b"\n":
                    # 이벤트 종료
                    pass

async def process_event(event_type, data):
    """이벤트 비동기 처리"""
    
    event_data = json.loads(data)
    
    if event_type == "elevator_arrival":
        # 엘리베이터 도착 알림
        await notify_arrival(event_data)
    
    elif event_type == "maintenance":
        # 유지보수 알림
        await log_maintenance(event_data)
    
    # 데이터베이스 저장
    await save_to_db(event_type, event_data)
```

#### SSE 프로토콜 형식

```
event: elevator_call
data: {"floor": 5, "direction": "up", "timestamp": "2024-01-01T12:00:00Z"}

event: elevator_arrival
data: {"floor": 5, "car_number": 1, "timestamp": "2024-01-01T12:00:30Z"}

event: emergency
data: {"type": "fire", "location": "B1", "timestamp": "2024-01-01T12:01:00Z"}
```

#### 실행 예시

```python
if __name__ == "__main__":
    url = "https://center.hdc-smart.com/v2/admin/elevators/sse"
    
    # 동기 실행
    # sync_sse_client(url)
    
    # 비동기 실행 (권장)
    asyncio.run(async_sse_client(url))
```

---

### 7. **test.py** - 범용 통신 테스트 프레임워크

시리얼 및 소켓 통신을 위한 종합 테스트 프레임워크입니다. 다양한 디바이스 타입을 자동으로 인식하고 파싱합니다.

#### 핵심 클래스 구조

##### `Communicator` - 통신 관리자

```python
class Communicator:
    """범용 통신 관리 클래스"""
    
    def __init__(self, conn_str):
        self.conn_str = conn_str  # "192.168.0.1:8899" 또는 "COM3"
        self.is_serial = False
        self.is_socket = False
        self.connection = None
        self.reconnect_attempts = 0
        self.chunk_size = 1024
        
        self._parse_conn_str()
    
    def _parse_conn_str(self):
        """연결 문자열 파싱"""
        if re.match(r'COM\d+|/dev/tty\w+', self.conn_str):
            self.is_serial = True
        elif re.match(r'\d+\.\d+\.\d+\.\d+:\d+', self.conn_str):
            self.is_socket = True
        else:
            raise ValueError(f"Invalid connection string: {self.conn_str}")
```

##### `PacketParser` - 패킷 분석기

```python
class PacketParser:
    """패킷 자동 분석 클래스"""
    
    HEADER_BYTES = {
        0x28: DeviceType.THERMOSTAT,
        0x31: DeviceType.GEC,
        0x41: DeviceType.DOORLOCK,
        0x61: DeviceType.FAN,
        0xC1: DeviceType.ELEVATOR,
        0xD1: DeviceType.ENERGY,
    }
    
    def parse_single_packet(self, start_idx):
        """단일 패킷 파싱"""
        
        # 1. STX 확인
        if self.data[start_idx] != 0x02:
            return None
        
        # 2. 디바이스 타입 결정
        header = self.data[start_idx + 1]
        device_type = self.get_device_type(header)
        
        # 3. 패킷 길이 계산
        length = self.get_packet_length(start_idx, device_type)
        
        # 4. 패킷 데이터 추출
        packet_data = self.data[start_idx:start_idx + length]
        
        # 5. 체크섬 검증
        is_valid = self.check_checksum(packet_data)
        
        # 6. 패킷 객체 생성
        return Packet(
            device_type=device_type,
            packet_type=self.get_packet_type(packet_data),
            seq_number=self.get_seq_number(packet_data),
            data=packet_data,
            is_valid=is_valid
        )
```

##### `DevicePacketParser` - 디바이스별 파서

```python
class DevicePacketParser:
    """디바이스 특화 파싱 클래스"""
    
    def parse(self):
        """디바이스 타입에 따른 자동 파싱"""
        
        # 동적 메서드 호출
        method_name = f"parse_{self.device_type.value}"
        parse_method = getattr(self, method_name, None)
        
        if parse_method:
            return parse_method()
        else:
            logging.error(f"Parser not found for {self.device_type}")
            return None
    
    def parse_thermostat(self):
        """난방 데이터 파싱"""
        return DevicePacket(
            trans_key=DeviceType.THERMOSTAT,
            room_id=self.data[5] & 0x0F,
            sub_id=None,
            device_state={
                "state": bool(self.data[6] & 0x01),
                "set_temp": self._parse_temperature(self.data[7]),
                "cur_temp": int.from_bytes(self.data[8:10], 'big') / 10.0,
                "heating_water_temp": self.data[11],
                "hot_water_temp": self.data[12],
            }
        )
    
    def parse_energy(self):
        """에너지 사용량 파싱"""
        device_packets = []
        start_idx = 7
        repeat_cnt = (len(self.data) - 8) // 8
        
        for _ in range(repeat_cnt):
            energy_type = self._get_energy_type(self.data[start_idx])
            
            device_packets.append(DevicePacket(
                trans_key=DeviceType.ENERGY,
                room_id=energy_type.value,
                sub_id=None,
                device_state={
                    "today_usage": self.data[start_idx + 1],
                    "yesterday_usage": self.data[start_idx + 2],
                    "monthly_usage": self.data[start_idx + 3],
                    "yearly_usage": self.data[start_idx + 4],
                    "realtime_usage": int.from_bytes(
                        self.data[start_idx + 6:start_idx + 8], "big"
                    ),
                }
            ))
            
            start_idx += 8 if (self.data[start_idx] & 0xF0) != 0x80 else 10
        
        return device_packets
    
    def _parse_temperature(self, byte_value):
        """온도 값 파싱 (0.5도 단위 지원)"""
        temp = byte_value & 0x3F
        if byte_value & 0x40:  # 0.5도 플래그
            temp += 0.5
        return temp
```

#### 실제 통합 사용 예시

```python
import asyncio
import logging

# 로깅 설정
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

async def monitor_devices():
    """디바이스 모니터링 메인 루프"""
    
    # 통신 연결
    comm = Communicator("192.168.0.27:8899")
    # comm = Communicator("COM3")  # 시리얼 사용 시
    
    comm.connect()
    
    # 통계 변수
    stats = {
        "thermostat": 0,
        "light": 0,
        "fan": 0,
        "energy": 0,
        "errors": 0
    }
    
    try:
        while True:
            # 데이터 수신
            raw_data = comm.receive()
            if not raw_data:
                await asyncio.sleep(0.1)
                continue
            
            # 패킷 파싱
            packets = PacketParser.parse(raw_data)
            
            for packet in packets:
                # 체크섬 검증
                if not packet.is_valid:
                    stats["errors"] += 1
                    logging.warning(f"Invalid checksum: {packet.data.hex()}")
                    continue
                
                # 응답 패킷만 처리
                if packet.packet_type != PacketType.RES:
                    continue
                
                # 디바이스별 파싱
                parsed = DevicePacketParser(packet).parse()
                
                if parsed:
                    # 통계 업데이트
                    device_type = packet.device_type.value
                    stats[device_type] = stats.get(device_type, 0) + 1
                    
                    # 데이터 출력
                    print(f"[{device_type}] {parsed}")
                    
                    # 데이터 처리 (예: 데이터베이스 저장)
                    await process_device_data(parsed)
            
            # 주기적 통계 출력
            if sum(stats.values()) % 100 == 0:
                print(f"통계: {stats}")
    
    except KeyboardInterrupt:
        print("모니터링 중단")
    finally:
        comm.close()

async def process_device_data(device_packet):
    """디바이스 데이터 처리"""
    
    # 예: Home Assistant 업데이트
    # await update_home_assistant(device_packet)
    
    # 예: 데이터베이스 저장
    # await save_to_database(device_packet)
    
    # 예: 이벤트 트리거
    # if device_packet.trans_key == "thermostat":
    #     if device_packet.device_state["cur_temp"] > 30:
    #         await trigger_alert("High temperature detected")
    
    pass

if __name__ == "__main__":
    asyncio.run(monitor_devices())
```

---

## 🧪 테스트 실행 가이드

### 환경 설정

#### 1. 의존성 설치

```bash
# 기본 패키지
pip install pyserial
pip install aiohttp
pip install aiofiles
pip install sseclient-py

# 선택적 패키지
pip install xmltodict  # XML 파싱용
pip install colorama   # 컬러 출력용
```

#### 2. 연결 설정

```python
# 설정 파일 (config.py)
CONFIG = {
    # 연결 설정
    "connection": {
        "type": "socket",  # "serial" 또는 "socket"
        "socket": {
            "host": "192.168.0.27",
            "port": 8899
        },
        "serial": {
            "port": "COM3",  # Windows
            # "port": "/dev/ttyUSB0",  # Linux
            "baudrate": 9600,
            "timeout": 30
        }
    },
    
    # API 설정
    "api": {
        "base_url": "http://your-server.com",
        "username": "your_id",
        "password": "your_password"
    },
    
    # SSE 설정
    "sse": {
        "url": "https://center.hdc-smart.com/v2/admin/elevators/sse"
    },
    
    # 디버그 설정
    "debug": {
        "packet_logging": True,
        "save_dumps": True,
        "dump_dir": "./packet_dumps"
    }
}
```

### 실행 명령어

#### 기본 테스트

```bash
# 1. 통신 테스트
python test.py

# 2. API 테스트
python client.py

# 3. 디밍 모니터링
python dimming_test.py

# 4. SSE 이벤트 수신
python sse_client.py

# 5. 패킷 분석
python packet_extract.py
```

#### 고급 사용법

```bash
# 디버그 모드 실행
python test.py --debug

# 특정 디바이스만 모니터링
python test.py --device thermostat

# 패킷 덤프 저장
python test.py --dump packets.log

# 실시간 통계
python test.py --stats
```

---

## ⚠️ 주의사항 및 트러블슈팅

### 보안 주의사항

#### 1. 실제 디바이스 제어 위험
```python
# ⚠️ 주의: 실제 디바이스를 제어할 수 있음
# 테스트 전 확인사항:
# - 테스트 환경 분리
# - 중요 디바이스 제외
# - 백업 계획 수립
```

#### 2. 인증 정보 보안
```python
# ❌ 하지 마세요
params = {
    "login_ide": "admin",
    "login_pwd": "password123"
}

# ✅ 이렇게 하세요
import os
from dotenv import load_dotenv

load_dotenv()
params = {
    "login_ide": os.getenv("BESTIN_ID"),
    "login_pwd": os.getenv("BESTIN_PW")
}
```

### 트러블슈팅 가이드

#### 연결 문제

##### 문제: Connection refused
```python
# 원인: 포트가 열려있지 않음
# 해결:
# 1. 포트 번호 확인
# 2. 방화벽 설정 확인
# 3. 서비스 실행 상태 확인

# 진단 스크립트
import socket

def check_port(host, port):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(5)
    result = sock.connect_ex((host, port))
    sock.close()
    return result == 0

if check_port("192.168.0.27", 8899):
    print("포트 열림")
else:
    print("포트 닫힘")
```

##### 문제: Timeout
```python
# 원인: 네트워크 지연 또는 차단
# 해결:
# 1. ping 테스트
# 2. 타임아웃 값 증가
# 3. 네트워크 경로 확인

# 진단 명령
import subprocess

def ping_test(host):
    result = subprocess.run(
        ["ping", "-n", "4", host],  # Windows
        # ["ping", "-c", "4", host],  # Linux/Mac
        capture_output=True,
        text=True
    )
    print(result.stdout)

ping_test("192.168.0.27")
```

#### 패킷 파싱 오류

##### 문제: Invalid packet length
```python
# 원인: 게이트웨이 타입 불일치
# 해결: 게이트웨이 타입별 처리

def get_packet_length(data, gateway_type):
    if gateway_type == "AIO":
        if data[1] == 0x31 and data[2] in [0x00, 0x80]:
            return 10  # 고정 길이
    elif gateway_type == "Gen2":
        return data[2]  # 가변 길이
    else:
        # General 타입 처리
        return data[2] if data[2] > 0 else 10
```

##### 문제: Checksum error
```python
# 원인: 데이터 손상 또는 잘못된 계산
# 해결: 체크섬 알고리즘 확인

def debug_checksum(data):
    """체크섬 디버깅"""
    
    calculated = calculate_checksum(data[:-1])
    received = data[-1]
    
    print(f"데이터: {data.hex()}")
    print(f"계산된 체크섬: {calculated:02X}")
    print(f"수신된 체크섬: {received:02X}")
    print(f"일치: {calculated == received}")
    
    # 단계별 계산 과정 출력
    checksum = 3
    for i, byte in enumerate(data[:-1]):
        old_checksum = checksum
        checksum ^= byte
        checksum = (checksum + 1) & 0xFF
        print(f"Step {i}: {old_checksum:02X} ^ {byte:02X} + 1 = {checksum:02X}")
```

#### 디바이스 미인식

##### 문제: Unknown device type
```python
# 원인: 새로운 디바이스 타입
# 해결: 헤더 매핑 추가

# 1. 미인식 패킷 로깅
def log_unknown_packet(data):
    with open("unknown_packets.log", "a") as f:
        f.write(f"{datetime.now()}: {data.hex()}\n")

# 2. 패턴 분석
def analyze_unknown_packets():
    packets = []
    with open("unknown_packets.log", "r") as f:
        for line in f:
            hex_data = line.split(": ")[1].strip()
            packets.append(bytes.fromhex(hex_data))
    
    # 헤더 통계
    headers = {}
    for packet in packets:
        header = packet[1]
        headers[header] = headers.get(header, 0) + 1
    
    print("헤더 통계:")
    for header, count in sorted(headers.items()):
        print(f"  0x{header:02X}: {count}회")

# 3. 새 디바이스 추가
HEADER_BYTES[0xXX] = "new_device_type"
```

---

## 📚 참고 자료

### 프로토콜 문서
- [패킷 구조 분석](packet_analyze.md)
- [BESTIN Integration 메인](../README.md)
- [설치 가이드](../guide/install.md)

### 관련 코드
- [커스텀 컴포넌트](../custom_components/bestin/)
- [Hub 구현](../custom_components/bestin/hub.py)
- [Controller 구현](../custom_components/bestin/controller.py)

### 외부 리소스
- [HDC 스마트홈](https://www.hdc-smart.com)
- [Home Assistant 문서](https://www.home-assistant.io/docs/)
- [GitHub Issues](https://github.com/lunDreame/ha-bestin/issues)

---

## 🤝 기여 가이드

### 테스트 데이터 제공
1. 새로운 디바이스 패킷 캡처
2. `packet_extract.py`로 정리
3. Issue에 첨부하여 제출

### 코드 개선
1. Fork 후 브랜치 생성
2. 테스트 코드 작성
3. Pull Request 제출

### 문서 개선
- 오타 수정
- 설명 보완
- 예제 추가

---
