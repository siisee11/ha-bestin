BESTIN Custom Component 상세 동작 설명

  1. 컴포넌트 구조 및 초기화

  BESTIN 컴포넌트는 HDC 아파트의 홈네트워크 시스템을 Home Assistant와 연동하는 통합 컴포넌트입니다.

  주요 구성 요소:
  - 도메인: bestin
  - 버전: 1.1.9
  - 지원 플랫폼: Climate(온도조절기), Fan(환기), Light(조명), Sensor(센서), Switch(스위치)
  - 필수 라이브러리: pyserial-asyncio, aiofiles, xmltodict

  2. 연결 방식 및 초기화 프로세스

  컴포넌트는 두 가지 연결 방식을 지원합니다:

  2.1 시리얼/소켓 연결 (월패드 직접 연결)

  # __init__.py:20-29
  if CONF_SESSION not in entry.data:
      await hub.connect()  # 시리얼 또는 소켓 연결
      await hub.async_initialize_serial()  # 게이트웨이 모드 결정 및 컨트롤러 시작

  - ConnectionManager 클래스가 연결 관리 (hub.py:33-208)
  - COM 포트나 IP:PORT 형식으로 연결
  - 자동 재연결 기능 (지수 백오프 적용)

  2.2 센터 API 연결 (클라우드 연결)

  # __init__.py:31-32
  else:
      await hub.async_initialize_center()  # Smart Home 1.0/2.0 API 사용

  3. 게이트웨이 모드 결정

  시리얼 연결 시 자동으로 게이트웨이 타입을 감지합니다 (hub.py:295-337):

  - AIO 모드: 패킷 길이 20-22, 명령 0x91/0xB2
  - Gen2 모드: 패킷 길이 59-150, 명령 0x91
  - General 모드: 기본 모드

  4. 통신 프로토콜

  4.1 패킷 구조

  # 기본 패킷 구조
  [0x02, room_id, packet_length, timestamp, ...data, checksum]

  - STX: 0x02 (시작 바이트)
  - Room ID: 방/기기 식별자
  - Length: 패킷 길이
  - Timestamp: 시간 정보
  - Checksum: XOR 기반 체크섬 (controller.py:135-152)

  4.2 디바이스별 패킷 생성

  각 디바이스 타입별로 전용 패킷 생성 함수가 있습니다:

  - 조명: make_light_packet() (controller.py:159-196)
    - AIO: room_id 0x50+n, 명령 0x12
    - Gen2: room_id 0x30+n, 명령 0x21
    - 밝기/색온도 제어 지원
  - 콘센트: make_outlet_packet() (controller.py:198-235)
    - 일반/대기전력차단 제어
    - 위치별 개별 제어
  - 온도조절기: make_thermostat_packet() (controller.py:237-254)
    - 온도 설정 (정수 + 0.5도 단위)
    - 켜기/끄기 제어
  - 환기: make_fan_packet() (controller.py:276-295)
    - 속도 조절 (퍼센트)
    - 자연환기 모드

  5. 데이터 처리 흐름

  5.1 수신 데이터 처리

  # controller.py:104-105
  async def start(self):
      self.tasks = [
          self.process_incoming_data(),  # 수신 데이터 처리
          self.process_queue_data()      # 송신 큐 처리
      ]

  5.2 소켓 수신 최적화 (hub.py:143-199)

  - 청크 단위 수신 (64바이트)
  - STX(0x02) 기반 패킷 시작 감지
  - 가변 길이 패킷 처리

  6. 엔티티 생성 및 관리

  6.1 디바이스 발견 및 등록

  # hub.py:351-374
  def async_add_device_callback(self, device_type, device):
      # 중복 체크
      if unique_id in self.entity_groups.get(domain, set()):
          return

      # 디스패처로 새 디바이스 신호 전송
      async_dispatcher_send(
          self.hass,
          self.async_signal_new_device(device_type),
          [device]
      )

  6.2 플랫폼별 엔티티 생성 (climate.py:29-57)

  async def async_setup_entry():
      @callback
      def async_add_climate(devices):
          entities = [
              BestinClimate(device, hub)
              for device in devices
              if device.unique_id not in hub.entity_groups[CLIMATE_DOMAIN]
          ]
          async_add_entities(entities)

  7. 디바이스 베이스 클래스

  BestinBase (device.py:14-50)
  - 디바이스 정보 관리
  - 명령 큐잉
  - Device Registry 정보 제공

  BestinDevice (device.py:53-100)
  - Home Assistant Entity 통합
  - 콜백 관리
  - 폴링/푸시 모드 지원

  8. 주요 특징

  1. 자동 재연결: 연결 실패 시 지수 백오프로 재연결
  2. 다중 게이트웨이 지원: AIO, Gen2, General 모드 자동 감지
  3. 비동기 처리: asyncio 기반 완전 비동기 구조
  4. 큐 기반 명령 처리: 명령 순서 보장 및 재시도
  5. 체크섬 검증: 모든 패킷의 무결성 확인
  6. 동적 엔티티 생성: 실시간 디바이스 발견 및 등록

  이 컴포넌트는 HDC 스마트홈 시스템의 다양한 버전과 호환되며, 시리얼/네트워크/클라우드 연결을 모두
  지원하는 유연한 구조로 설계되었습니다.
