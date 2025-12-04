# **DashBumper: Linux Dedicated Server Architecture**

**Linux Dedicated Server 기반의 실시간 멀티플레이 서바이벌 게임** 
상용 엔진의 네트워크 한계를 극복하기 위해
직접 구현한 델타 압축(Delta Compression)과
**GC-Zero 메모리 구조**를 적용하여, 대규모 트래픽 상황에서도 안정적인 동기화를 보장합니다.

## **📖 Project Overview**

* **Project Name:** DashBumper  
* **Role:** 1인 개발 (Server Architecture, Network Logic, Client Prediction)  
* **Dev Period:** 2025.09.04 ~ 2025.12.04  
* **Video Demo:** [유튜브 영상 링크] (여기에 링크를 입력하세요)

### **🎯 Core Objectives**

* **Deterministic Environment:** Unity Physics의 비결정론적 특성을 배제하고, 서버 권한(Server Authority) 기반의 완벽한 동기화 구현.  
* **Optimization:** 30Hz 틱레이트 환경에서 GC Allocation 0Bytes 달성 및 대역폭 최적화.  
* **Secure Networking:** 신뢰성 있는 TCP 핸드셰이크와 빠른 UDP 통신의 하이브리드 구조 및 보안 적용.

## **🛠️ Tech Stack**

| Category | Technology |
| :---- | :---- |
| **Engine** | Unity 6000.0.62f1 |
| **Language** | C# (Server/Client Shared Logic) |
| **Server OS** | Linux (Ubuntu 20.04) on Google Cloud Platform (GCP) |
| **Architecture** | DOD (Data Oriented Design), ECS Pattern, VContainer (DI) |
| **Network** | TCP/UDP Custom Protocol, MessagePipe, UniRx |
| **Security** | AES Encryption, HMAC Authentication |

## **📂 System Architecture**

### **1. Server Loop & Physics Flow**

서버의 메인 루프는 물리 연산(Velcro)과 게임 로직을 엄격하게 분리하여 순차적으로 처리합니다. 물리 엔진의 콜백을 이벤트로 변환하여 로직단에서 처리함으로써 결합도를 낮췄습니다.
```mermaid
sequenceDiagram
    autonumber
    
    box rgb(100, 100, 251) Server Loop
    participant SGFM as ServerGameFlowManager
    end

    box rgb(40, 40, 60) Physics Engine (Deterministic)
    participant SPM as ServerPhysicsManager
    participant Velcro as Velcro World (Core)
    participant Wrapper as VelcroBodyWrapper
    end

    box rgb(222, 222, 220) Event System
    participant MsgPipe as MessagePipe
    participant CM as CollisionManager
    end

    box rgb(255, 200, 200) Game Logic
    participant IS as ImpactService
    end

    Note over SGFM: 1. 물리 시뮬레이션 시작
    SGFM->>SPM: Step(fixedDeltaTime)
    activate SPM
        
        SPM->>Velcro: Step(dt)
        activate Velcro
            Note right of Velcro: 물리 연산 수행<br/>(위치 갱신, 충돌 감지)
            
            %% Velcro 내부에서 충돌 발생 시 콜백 호출
            Velcro-->>SPM: OnBodyCollision(FixtureA, FixtureB)
            activate SPM
                Note right of SPM: Unity 의존성 없는<br/>순수 데이터 변환
                SPM->>MsgPipe: Publish(PhysicsCollisionEvent)
                activate MsgPipe
                    
                    MsgPipe->>CM: HandlePhysicsCollision(Event)
                    activate CM
                        
                        Note right of CM: GC-Zero 필터링<br/>(NativeHashSet)
                        CM->>CM: Check Duplicate Pair
                        
                        CM->>IS: ResolveImpact(BodyA, BodyB)
                        activate IS
                            
                            Note right of IS: 충돌 타입 판별<br/>(Player vs Player)
                            
                            IS->>IS: Calculate Knockback & Stun
                            
                            Note right of IS: 반동(Force) 적용
                            
                            %% 다시 물리 바디에 힘 적용 (순환 구조)
                            IS->>Wrapper: ApplyLinearImpulse(Force)
                            activate Wrapper
                                Wrapper->>Wrapper: Vector2 변환 (Unity -> Velcro)
                                Wrapper->>Velcro: Body.ApplyLinearImpulse()
                            deactivate Wrapper
                            
                            IS->>Wrapper: ApplyStun() (Update Context)
                        deactivate IS

                    deactivate CM
                deactivate MsgPipe
            deactivate SPM
        
        deactivate Velcro
        
    deactivate SPM
    Note over SGFM: 2. 물리 결과 동기화 (SyncToTransform)
    end

### **2. Network Handshake & Security**

TCP로 안전하게 세션 키를 교환한 후, UDP 통신으로 전환하는 하이브리드 핸드셰이크 구조입니다.

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client
    participant Photon as Photon Lobby
    participant ServerTCP as TcpSocketManager
    participant ServerLogic as SecurityManager
    participant ServerUDP as UdpSocketManager

    Note over Client, ServerTCP: [Phase 1] TCP Connection & Token
    Client->>Photon: 1. Join Room
    Client->>ServerTCP: 2. Connect (Port 7777)
    Client->>ServerTCP: 3. Send Code 30 (TcpHandshake) + Token(Photon)
    ServerTCP->>ServerLogic: 4. Verify Tcp Token
    ServerLogic-->>Client: 5. Send UDP Token (via Photon RPC) & Req AES Key

    Note over Client, ServerUDP: [Phase 2] UDP Punch & Security
    loop UDP Retry (ClientHandshakeService)
        Client->>ServerUDP: 6. Send Code 10 (UdpHandshake) + UDP Token
    end
    ServerUDP->>ServerLogic: 7. Verify UDP Token & Register EndPoint
    
    Note over Client, ServerLogic: [Phase 3] AES Key Exchange (TCP Payload)
    Client->>Client: 8. Generate AES Key & Encrypt with Server RSA PubKey
    Client->>ServerTCP: 9. Send Encrypted AES Key (TCP Packet)
    ServerTCP->>ServerLogic: 10. Decrypt AES Key (RSA PrivKey) & Store
    ServerLogic->>ServerLogic: 11. Check All Ready (TCP+UDP+AES)
    ServerLogic-->>Client: 12. Send Ack (Security Established)

    rect rgb(200, 500, 200)
    Note over Client, ServerUDP: [Phase 4] In-Game (Encrypted UDP)
    Client->>ServerUDP: 13. PlayerInputData (AES Encrypted + HMAC)
    ServerUDP->>ServerUDP: 14. Verify HMAC & Decrypt
    ServerUDP->>Client: 15. DeltaSnapshot (Tick 100)
    end

### **3. Server Connecting Cycle**

클라이언트가 해당 게임의 포톤 서버와 관제탑 매칭 요청을 통한 상태 전환과 접속 흐름입니다.
```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Lobby as LobbyPanel
    participant NM as NetworkManager
    participant MM as MatchmakingManager
    participant HTTP as Control Tower (Web Server)
    participant Photon as Photon Cloud
    participant GameServer as Dedicated Server

    Note over User, Photon: [Phase 1] 초기화 및 마스터 서버 접속
    User->>NM: Start Game
    NM->>Photon: PhotonNetwork.ConnectUsingSettings()
    Photon-->>NM: OnConnectedToMaster()
    NM->>Photon: JoinLobby()
    Photon-->>NM: OnJoinedLobby()
    NM-->>Lobby: Activate "Find Match" Button

    Note over User, HTTP: [Phase 2] 관제탑 매칭 요청 (HTTP)
    User->>Lobby: Click "Find Match"
    Lobby->>MM: FindRandomMatch()
    MM->>HTTP: POST /findMatch (Region, Rank...)
    activate HTTP
    Note right of MM: Retry Logic (Max 3 times)<br/>Backoff applied
    HTTP-->>MM: 200 OK (MatchInfo: RoomName, Region)
    deactivate HTTP

    Note over MM, GameServer: [Phase 3] 방 입장 및 서버 접속
    MM->>Photon: PhotonNetwork.JoinRoom(MatchInfo.RoomName)
    Photon-->>MM: OnJoinedRoom()
    
    rect rgb(220, 2230, 520)
        Note right of MM: 여기서부터 TCP/UDP 핸드셰이크 시작
        MM->>GameServer: PacketRouteManager.SendSecurityReadySignal()
        GameServer-->>MM: (Handshake Process...)
    end

    


### **4. Server Tick Cycle**

게임의 상태 갱신, 물리 시뮬레이션, 네트워크 동기화가 이루어지는 메인 루프입니다.
```mermaid
flowchart TD
    Start(("Server Start")) --> Init["Initialize Managers<br/>(VContainer)"]
    Init --> Loop{"Game Loop<br/>(FixedUpdate / 30Hz)"}
    
    subgraph Tick_Execution [Server Tick Cycle]
        Loop -->|Tick Start| Time["TimeManager.Tick"]
        Time --> Unreg["Process Unregister Queue"]
        
        subgraph Logic [Simulation]
            Unreg --> Input["Process Player Inputs<br/>(Apply Velocity)"]
            Input --> Physics["Simulate World<br/>(Velcro Physics Step)"]
            Physics --> Sync["Sync Physics to Transform"]
            Sync --> Context["Update Player Contexts"]
        end

        subgraph Network_Sync [Snapshot & Replication]
            Context --> LagComp["Record Lag Compensation"]
            LagComp --> SnapGen["Create Full Snapshot<br/>(WriteBuffer)"]
            SnapGen --> DeltaCheck{"Has Ack?"}
            
            CheckAck -- Yes --> Delta["Create Delta Compression"]
            CheckAck -- No --> Full["Send Full Snapshot"]
            
            Delta --> Send["UDP Send"]
            Full --> Send
        end

        subgraph Memory [GC-Zero Optimization]
            Send --> ClearEvents["Clear Events"]
            ClearEvents --> Swap["Swap Read/Write Buffers"]
        end
    end

    Swap --> Loop
    end
## **🚀 Key Features & Solutions**

### **1. Zero Allocation & GC Optimization**

**File:** NetworkDataConverter.cs, GameManager.cs

* **Problem:** 초당 30회 발생하는 패킷 직렬화(Serialization) 과정에서 BinaryWriter의 내부 문자열 처리 등으로 인해 지속적인 GC Spike가 발생, 서버 프레임 드랍 유발.  
* **Solution:**  
  * RecyclableMemoryStream (Microsoft)과 ArrayPool<byte>를 도입하여 힙 메모리 할당을 원천 차단.  
  * 제네릭 (TryDeserializeInto<T> where T : struct)을 활용하여 Boxing/Unboxing 제거.  
* **Result:** **In-Game Loop 분당 GC Allocation 0 Bytes 달성.**

### **2. Custom Delta Compression**

**File:** DeltaCompressionManager.cs

* **Problem:** 매 틱(Tick)마다 전체 상태(Full Snapshot)를 전송할 경우 대역폭 낭비가 심각하여 동시 접속자 수 확장에 한계.  
* **Solution:**  
  * **Double Buffering:** 현재 프레임(Write)과 이전 프레임(Read) 버퍼를 XOR 비트 연산하여 변경된 필드만 추출.  
  * 변경된 데이터에만 비트 플래그(Bitmask)를 세워 전송하는 로직 구현.  
* **Result:** **패킷 사이즈 평균 40~60% 절감.**

### **3. Server Authority Physics (Deterministic)**

**File:** ServerPhysicsManager.cs

* **Architecture:** Unity Physics(PhysX)는 비결정론적(Non-deterministic)이므로 서버-클라이언트 간 미세한 위치 오차가 누적되는 문제 발생.  
* **Solution:**  
  * 순수 C# 기반의 결정론적 물리 엔진인 **VelcroPhysics**를 래핑하여 사용.  
  * RootInstaller.cs를 통해 물리 시스템을 DI 컨테이너에 등록하고, 게임 로직과 물리 연산을 분리.  
* **Benefit:** 모든 클라이언트와 서버에서 동일한 입력에 대해 **완벽하게 동일한 물리 결과 보장.**

### **4. Robust Security Architecture**

**File:** SecurityManager.cs

* **Protocol:** TCP(신뢰성) → AES Key 교환 → UDP(HMAC 서명) 순서의 3단계 핸드셰이크 구현.  
* **Features:**  
  * **RSA/AES Hybrid:** RSA로 세션키를 안전하게 교환하고, 성능을 위해 AES로 실시간 패킷 암호화.  
  * **HMAC Authentication:** 비연결성 UDP 패킷의 변조(Man-in-the-Middle) 방지 서명 포함.

## **📝 Troubleshooting Log**

### **Issue 1: GC Spike during Packet Serialization**

* **Situation:** 프로파일링 결과 BinaryWriter.Write(string) 호출 시마다 내부적으로 임시 버퍼를 생성하여 주기적인 GC Spike가 발생함을 확인.  
* **Analysis:** C# 기본 라이브러리의 인코딩 방식이 고성능 리얼타임 서버에는 적합하지 않은 메모리 패턴을 보임.  
* **Action:** NetworkDataConverter.cs에 ArrayPool을 사용하는 커스텀 직렬화 메서드를 직접 구현하고, 전송되는 모든 패킷 구조체를 class가 아닌 struct로 변경하여 참조 비용 제거.  
* **Result:** 패킷 처리 과정의 GC Alloc **0KB** 달성 및 프레임 안정화.

### **Issue 2: Race Condition in Security Handshake**

* **Situation:** 클라이언트의 UDP 패킷이 서버의 AES 키 생성 및 등록 로직보다 먼저 도착하여, 암호화 해제 실패(Decryption Failed) 오류가 간헐적으로 발생.  
* **Action:** SecurityManager.cs에 _armedPlayers 상태와 PendingQueue를 도입. 보안 채널이 완전히 확립되기 전 도착한 패킷은 폐기하지 않고 큐에 보관했다가, 핸드셰이크 완료 즉시 순차적으로 처리하도록 변경.  
* **Result:** 네트워크 지연(Latency)이 있는 환경에서도 핸드셰이크 성공률 **100%** 보장.

## **📜 Source Code Highlights**

*본 프로젝트는 Private Repository로 관리 중입니다. 핵심 로직의 구조는 아래 설명을 참고해 주세요.*

* **RootInstaller.cs**: VContainer 기반의 의존성 주입(DI) 및 시스템 생명주기 관리.  
* **GameManager.cs**: 결정론적 서버 루프(Tick) 및 전체 게임 흐름 제어.  
* **ServerPhysicsManager.cs**: Unity 의존성을 제거한 결정론적 물리 엔진 래퍼.  
* **DeltaCompressionManager.cs**: 비트 연산 기반의 스냅샷 델타 압축 로직.  
* **SecurityManager.cs**: RSA/AES 암호화 및 핸드셰이크 상태 머신.
* **NetworkDataConverter.cs**: GC 최적화 기반 제네릭 공용 메서드 로직.
