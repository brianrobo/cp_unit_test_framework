# cp_unit_test_framework


일단 문제점을 정의할께.



CP modem은 Shannon DM 인데..

실제 이슈를 재현하려면 실제 필드에서 해당망에서 테스트를 해야함.



그럼 해당 망을 잡고 특정 이슈 재현 시나리오대로 모뎀과 망간의 시그널링을 할껀데.

이때 이 시그널링 하는 필드 로그가 있다면,



이를 기반으로 실제 필드에 안 있더라고,  현재 단말에  RF 및 RRC 또는 NAS signalling을 가짜 mock input으로 주어서  단말이 그렇게 동작하게끔 유도하고자 해.



최종적인 목적은이런  특정 이슈 시나리오에 대한 mock을 만들고,, unit test set을 만들어서,



과거 핵심 이슈들이 차기 모델들에서 발생안하거나,, 특정 패치를 릴리즈 했을 씨 side effect이 없다는것을 필드 검증을 안통해서도, 바로 빠르게 검증할 수 있는 시스템을 만드는 것이지





정확하게 이해했습니다. 말씀하신 접근 방식은 모뎀 개발 및 QA 프로세스에서 **"Log-based Traffic Replay"** 또는 **"Network Digital Twin"**이라고 불리는 매우 고도화된 테스트 방법론입니다.

비용이 많이 드는 필드 테스트(Field Test)를 실험실 환경(Lab)으로 가져와 **Regression System(회귀 테스트 시스템)**을 구축하는 것이 핵심 목표시군요. 이를 실현하기 위한 **아키텍처와 구현 전략**을 제안해 드립니다.

---

## 1. 전체 시스템 아키텍처: "Virtual Network Simulation based on DM Logs"

핵심은 **'모뎀이 망(Network)과 통신한다고 착각하게 만드는 것'**입니다. 이를 위해 기존 Shannon DM 로그를 **'시나리오 스크립트'**로 변환하고, 이를 모뎀의 하위 레이어(PHY 또는 IPC 레벨)에 주입(Injection)하는 구조가 필요합니다.

### 3단계 핵심 모듈

1. **Log Parser & Converter (로그 변환기):**
* 필드에서 캡처한 Shannon DM 로그(Binary)를 분석하여 **Downlink 메시지(Network  Modem)**를 추출합니다.
* 추출된 RRC, NAS 메시지를 사람이 읽을 수 있고 편집 가능한 포맷(JSON, XML, or Custom Script)으로 변환합니다.


2. **Virtual Network Simulator (가상 기지국/코어):**
* 실제 RF 장비(Call Box) 없이, 소프트웨어적으로 기지국 역할을 하는 모듈입니다.
* 변환된 스크립트를 읽어 들여, 모뎀이 특정 Uplink 메시지를 보냈을 때, 로그에 기록된 Downlink 메시지를 순서대로 주입합니다.


3. **Test Runner & Verifier (검증기):**
* 테스트를 자동 실행하고, 모뎀의 상태(State)가 예상대로 변했는지, 크래시(Crash)가 없는지 판별합니다.



---

## 2. 기술적 구현 전략 (How-to)

### A. Injection Point (어디에 가짜 신호를 넣을 것인가?)

실제 RF 신호를 쏘는 것은 고가 장비가 필요하므로, SW 레벨에서 인터셉트해야 합니다.

* **Target:** Modem Protocol Stack (L1/L2/L3)
* **Method:** **L1 Bypass (PHY-API Injection)** 또는 **Virtual PHY**
* 모뎀의 Protocol Stack은 L1(PHY) 계층에서 데이터를 받아옵니다.
* 이 L1 계층의 입력을 실제 안테나가 아닌, **Host PC(또는 테스트 보드)의 메모리 버퍼**에서 읽어오도록 드라이버를 수정하거나 **Hooking** 합니다.
* 즉, `PHY_Receive(Data)` 함수가 호출될 때, 실제 전파 데이터 대신 **Log에서 추출한 RRC/NAS 패킷**을 리턴해줍니다.



### B. Synchronization (타이밍과 순서 맞추기)

단순히 로그를 시간순으로 뿌리는 것은 실패할 확률이 높습니다. 모뎀은 **State Machine**이기 때문입니다.

* **Event-Driven Replay:**
* 단순 Time-based가 아닌, 모뎀의 **Uplink(Request)**를 트리거로 삼아야 합니다.
* *(예시)* 모뎀이 `ATTACH_REQUEST`를 보내면  (감지)  시뮬레이터가 로그에 있던 `ATTACH_ACCEPT`를 주입.


* **State Check:**
* 로그 상의 타임스탬프 간격(Delta Time)을 유지하되, 모뎀이 해당 상태(예: `RRC_CONNECTED`)에 진입했는지 확인 후 다음 메시지를 주입해야 합니다.



### C. Test Case 생성 (Log to Scenario)

필드 로그 전체를 다 쓸 필요는 없습니다. 이슈 재현에 필요한 핵심만 추출합니다.

1. **Golden Log 선정:** 이슈가 발생한 로그 확보.
2. **Filter:** `SIB(System Information Block)`, `RRC Signaling`, `NAS Signaling` 메시지만 필터링.
3. **Parameterization:**
* 단말이 바뀌면 IMSI, IMEI 등의 값이 바뀔 수 있습니다.
* 로그 파일 내의 고정된 ID 값들을 변수화(Variable)하여, 어떤 단말을 연결해도 테스트 가능하도록 스크립트를 일반화합니다.



---

## 3. 기대 효과 및 검증 시나리오 예시

이 시스템이 구축되면 다음과 같은 **Unit Test Set** 구성이 가능합니다.

### 시나리오 1: 특정 기지국 호환성 이슈 (Handover Fail)

* **상황:** 특정 벤더 기지국에서 Handover Command 수신 시 모뎀이 처리 못 하고 Drop 됨.
* **Mocking:** 해당 기지국이 보내는 비표준(또는 특이한) `RRCConnectionReconfiguration` 메시지를 로그에서 추출하여 주입.
* **검증:** 패치된 펌웨어가 이 메시지를 받고 정상적으로 Handover Complete를 보내는지 확인.

### 시나리오 2: 네트워크 거절 (Attach Reject) 무한 루프 방지

* **상황:** 망에서 특정 Cause Code로 `Attach Reject`를 계속 보낼 때 모뎀이 배터리를 광탈시키며 무한 재시도하는지 확인.
* **Mocking:** `Attach Reject (Cause #XX)` 메시지를 반복적으로 주입.
* **검증:** 모뎀이 Back-off 알고리즘(대기 시간)을 정상적으로 수행하며 재시도 횟수를 제한하는지 확인.

---

## 4. 바로 실행 가능한 Next Step

이 시스템은 규모가 크기 때문에 작게 시작하는 것이 좋습니다.

1. **가장 먼저:** 보유하고 계신 Shannon DM 로그 중 가장 간단한 `Attach` 성공 로그 하나를 골라 텍스트(JSON 등)로 디코딩 가능한지 확인해보십시오. (이미 파서가 있다면 절반은 성공입니다)
2. **질문:** 현재 개발 환경이 **실제 타겟 보드(단말)** 위에서 테스트하는 구조인가요, 아니면 PC 상에서 모뎀 스택을 돌리는 **가상 환경(SIL - Software In the Loop)**이 구축되어 있나요?

환경을 알려주시면 **'어떤 인터페이스를 Hooking 해야 하는지'** 더 구체적인 조언을 드릴 수 있습니다.
