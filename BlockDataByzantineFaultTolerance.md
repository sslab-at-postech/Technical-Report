# 블록 데이터 비잔틴 장애 감내 보장 기법

# Block Data Byzantine Fault Tolerance

## 요약
  최근 블록체인 기술이 데이터를 신뢰성 있게 저장하는 데에 핵심적인 기술로 떠오르고 있다. 블록체인은 누구나 데이터 검증을 가능하도록 모든 노드들이 개별적으로 블록체인 정보를 저장해야 하는 스토리지 오버헤드가 존재한다. 이로 인해 제한된 스토리지 용량을 가지고 있는 노드를 지원하기 위해, 이더리움에서는 일정 구간동안의 블록체인 정보만을 개별 노드에 저장하도록 하고, 전체 블록체인 정보는 아카이브 노드로 규정하는 별도의 노드에 저장하고 있다. 아카이브 노드와 같은 블록체인 스토리지 서비스는 전체 블록체인 정보를 저장함으로써 정보 검색 및 검증을 효과적으로 지원하며, 또한 데이터 속성에 따른 법적 규정을 지원하는데 중요한 역할을 한다. 
  
  본 기술 문서는 클라우드 스토리지 서비스 등 기존의 외부 저장 장치를 활용하여 어떠한 블록체인 플랫폼에도 연동 가능한 블록체인 스토리지 서비스를 제안한다. 블록체인 스토리지 서비스는 비잔틴 장애 감내 블록 쓰기 기능과 블록 감사 (audit) 기능을 지원함으로써 외부 저장 장치를 연계한다.
  
## 1.서론
  최근 블록체인 기술이 데이터를 신뢰성 있게 저장하는 데에 핵심적인 기술로 떠오르고 있다. 블록체인이란 거래 내역과 같은 데이터를 네트워크에 참여하는 사용자들이 분산하여 저장하고, 처리하는 기술을 의미한다. 블록체인 기술을 실제 적용하고 있는 플랫폼들을 예시로 들자면, Bitcoin, Ethereum, EOS, Algorand, Hedera, Hyperledger Fabric, IOTA 등이 존재한다. 
  
  블록체인은 모든 네트워크 참여자들이 서로 다른 데이터를 공유하며 검증 가능하다는 장점을 가지고 있는 반면, 스토리지 측면에서의 문제점도 존재한다. 제한된 스토리지 용량을 가지고 있는 노드의 경우 각자 소유 중인 데이터베이스가 빠르게 포화된다 (i.e. storage overhead). 특히 Internet Of Things (IoT) 환경에서 블록체인 플랫폼을 운용하는 경우, 블록체인 플랫폼을 low-level, resource-constrained 기기들과 통합해야 하기 때문에 storage overhead 문제가 더 심해진다.
  
  Storage overhead를 해결하기 위해서 각 블록체인 플랫폼은 각자 검증 가능한 일부 데이터만을 저장한다. Ethereum의 경우 Full node는 블록체인 트랜잭션을 전송 및 처리 하며, 가장 최신의 128개 블록에 대한 state만을 저장한다 [1]. IOTA [2]는 local snapshot 기술을 사용하여 ledger의 resulting state만을 persist 한다. 이미 검증된 transaction을 고른 이후 해당 transaction을 참조하고 있는 모든 transaction 들을 삭제한다. Solana [3] 도 snapshot을 이용하여 ledger의 중간 state부터 유지한다.
  
  하지만, 블록 데이터에 대한 플랫폼 별 제도적 규정 기간이 존재하여 모든 데이터가 긴 기간 저장될 필요성이 있다. 실 예로, 결제 관련 데이터의 경우에는 특정 기간 동안 유지되어야 될 필요성이 존재하며, 신원 관련 데이터의 경우 해당 신원이 파기 될 때까지 유지되어야 한다는 요구사항이 존재한다. 
  
  각 블록체인 플랫폼은 저장 필요성에 따라 새로운 종류의 노드를 제시했다. Ethereum 의 경우 블록 데이터의 저장 요구사항에 따라 블록체인 트랜잭션을 전송 및 처리 하는 full node 이외 genesis block부터 모든 블록 저장 목적으로 archive node를 지원한다. IOTA는 트랜잭션 단위로 저장을 진행하며, Permanode [4]를 통하여 트랜잭션 데이터에 대한 저장을 진행하고, StoreTransaction API를 통하여 필요한 transaction을 선택적으로 저장한다. Solana 는 블록체인 노드에 참여하고 있는 모든 노드들이 모든 블록을 저장하고 있다. Solana는 블록 데이터 에 대한 분산 저장을 목적으로 archivers 라는 노드를 따로 운영한다. 
  
  하지만, 각 노드나 외부 저장소에 저장된 블록 데이터를 검증하기 위해서는 hash chain property 에 따라서 검증하고자 하는 블록 데이터 이전의 데이터들을 전부 받아와야 한다는 단점을 지닌다. 블록체인 네트워크가 전부 믿을 수 없는 (untrusted) 블록체인 노드들로 구성되었다는 점에서, 블록체인 노드가 비잔틴 장애 (Byzantine fault) 를 가질 때의 대응이 어렵다.
  
  이에, 본 기술 문서는 블록체인 플랫폼 상에서의 스토리지 관점의 문제점에 대한 해결책을 제시하면서도, 플랫폼 상 비잔틴 장애 감내 특성을 제공하는 대규모 트랜잭션 데이터 분산 저장 서비스 (Blockchain Storage Service 이하 BSS)를 제안한다. BSS안에서 적용되는 블록 데이터 비잔틴 장애 감내 보장에 대한 내용을 포함한다.
  
## 2. 시스템 설계
### 2.1 시스템 구조
  BSS는 크게 S-node (저장 노드) 집합과 External Storage (외부 스토리지 저장소)로 이루어져 있다 (그림 1).

  가. S-node Set: 한 임의의 블록체인 노드 (i.e. BSS client) 로부터 블록체인 블록 를 받아온 이후, 해당 블록 데이터에 대한 BFT 합의를 진행하게 된다. 각 S-node 들 간 블록 합의 메시지 를 주고받음으로써 BFT 합의를 수행한다. 하나의 블록 데이터에 대한 BFT 합의가 끝난 이후 각 S-node는 합의 메시지를 모아 BFT 블록 메타데이터를 생성한다.
  
  나. External Storage: S-node 집합으로부터 블록 와 메타데이터 을 받아서 저장하는 외부 저장소이다.
  
  블록체인 노드 (BSS 클라이언트) 에게 두 가지 인터페이스를 지원한다: AddBlock() 인터페이스는 블록 를 BSS에 기록한다. AddBlock(B) 는 BSS로부터 합의 결과를 반환 받는다. GetBlockByNumber(h) 는 BSS 상 저장되어 있는 높이 의 블록 을 읽어온다.
  
  각 S-node 들은 블록  에 대한 global predicate P 를 만족 하는지 검증 목적으로 메타데이터 M 을 생성한다. 본 기술 문서에서는, 를 “n-f개 이상의 S-node에 의해서 블록에 대한 합의 결과를 도출 할 수 있는가?”로 설정하였다.
  
  각 S-node 들은 다섯 가지 연산을 지원 한다: 키 는 블록 높이로 정의한다. write() 는 블록 와 합의에 사용된 메타데이터 을 external storage에 기록하고, 메타데이터 을 반환한다. valid()은 이 유효한 정보를 가지고 있으면, true를 반환한다. read(는 write()이 선행되었으면, 블록 를 external storage로부터 받는다. read_causal()는 블록  이전 높이의 모든 블록 을 external storage로부터 받는다. 마지막으로 audit()은 external storage로부터 와 연관된 메타데이터 를 수신한다.
  
  ### 2.2 시스템 모델
   우리는 악의적인 노드 (i.e. replica 혹은 블록체인 노드) 가 일반적인 노드 장애를 가질 수 있다고 가정한다. 다만, 악의적인 노드가 암호학적 기술 (i.e. collision-resistant hashes, signatures) 을 위반할 수 없다고 가정한다.
   
  우리 시스템은 두 가지 보안 특성 [5] 을 만족한다. 최대 f 개의 replica 가 악의적인 노드일 때까지 safety와 liveness 특성을 만족한다. 추가로, 우리는 유한한 개수의 블록체인 노드가 악의적임을 가정한다.
  
  • Safety: 모든 정상적인 노드는 같은 높이에 대해서 서로 다른 블록을 commit 할 수 없다.
  • Liveness: 특정 높이 상 적어도 한 블록의 합의 결과는 언젠가 반드시 도출된다.
  
  우리 시스템의 safety와 liveness 특성은 노드들이 일반적인 노드 장애 (Byzantine fault) (i.e. 메시지를 보내지 않거나, 이상한 (conflicting) 메시지를 보내는 경우)를 가질 수 있는 partially synchronous 한 네트워크를 가정한다. Partially synchronous 네트워크 상 전파되는 메시지의 경우 고정되어 있으나, 알려지지 않은 time bound 안에 최종적으로 전송된다.
  
   이에, BSS는 아래와 같은 추가적인 특성을 만족한다.
   
  • Integrity: 정상적인 S-node에 의해서 호출된 서로 다른 두 개의 read() 는, external storage가 정상적이라면, 같은 를 반환 한다.
  • Block Availability: 정상적인 primary S-node에 의해서 write() 이후 정상적인 S-node로부터 read() 가 호출되면, read() 는 최종적으로 실행되고, external storage가 정상적이라면, 를 반환한다.
  • Containment: read_causal() 로부터 반환된 블록 집합  에 대해서 (), read_causal() 로부터 반환된  ()은 를 만족한다.
  • Causality: read_causal()는 commit된 에 대한 write() 가 수행되기 이전, 최소  개 honest S-node들에 의해 commit 된 블록을 전부 포함한다.
  • External validity: 만일 정상적인 S-node가 블록 에 대한 합의 결과를 도출하고 메타 데이터  를 생성했다면,  는 참이다.
  
  [6]에서 block availability, containment, causality 특성을 시스템 특성으로 제시했다. 또한,  [9]에서 external validity를 처음 제안하였고, [7, 8]에서 external validity를 그들 시스템의 특성으로 언급하였다. 우리는 BSS에 적용 가능하도록 각 특성의 정의를 수정하였다.
  
  ### 2.3 합의 상태와 메시지 형식
  각 S-node 들은 BFT 합의를 통해서 특정 높이에 대한 블록의 합의 상태 (Consensus State) 를 세 단계로 유지한다. 높이  에 대한 블록  의 합의 상태는 Level-0, Level-1, Level-2, 총 세 단계로 나뉜다.
  
가) Level-0: 블록체인 노드로부터 블록  및 블록체인 노드 이 서명한 블록 해시 를 전달 받은 상태이다.
나) Level-1: Level-0에서 S-node가  개 이상의 합의 메시지 를 수신하였을 때의 상태이다.
다) Level-2: Level-1에서 S-node가  개 이상의 합의 메시지 를 수신하였을 때의 상태이다.

  S-node 가 블록에 대한 합의 상태를 유지하고, BFT 합의를 진행하기 위해서 블록체인 노드로부터 블록을 수신한 이후 audit protocol 을 수행한다. Audit Protocol 상에서는 두 가지 메시지: 1) 합의 메시지 (Consensus Message)  와 2) 메타데이터 (Metadata)  에 대하여 아래와 같이 정의한다.
  
  h는 블록 높이, H(Bh)는 블록 해시, v는 현재 블록을 수신하는 블록체인에 대한 번호 (view number), i는 S-node ID, L_{alpha_h}는 Bh 에 대한 합의 상태를 Level-2 로 변화 시킨 합의 메시지 리스트, Hprev는 이전 높이 블록 해시, sigma_{BN}은 블록체인 노드의 서명, sigma_{i}는 S-node 서명을 의미한다.
  
  Audit protocol 에 의해서 블록체인 노드의 악의적인 행동이 발견된 경우 BN (Blockchain Node) change protocol을 시작하고, BN change protocol 상에서는 세 가지 메시지: (1) BN Change Message mv=<r,h1, (Hh1)_{sigmaBN}, QCh1, v, vh1, i>_{sigmai} , (2) New BN Proposal beta_{v}=<<r,h1,(Hh1)_{sigmaBN},v,vh1>,Lmv,i>_{sigmai} , 그리고 (3) Vote gamma_{v}=<r, beta_{v},i,LBG>_{sigmai} 로 정의한다. 
  
  r은 BN change protocol 상 메시지를 수합 및 전파하는 primary 결정 라운드 번호, h1은 가장 높은 Level-1 로 전이된 블록 높이,  H1은 h1 높이의 블록 Bh1 해시, QCh1 은 h1 높이의 alpha_{h1} 집합, v 는 새로운 view number, vh1은 높이 h1에서의 view number, Lmv 는 n-f 개 이상의 mv 집합, LBG 는 Lmv에서 가장 h1 낮은 부터 가장 높은 h1 높이 까지 블록 리스트를 의미한다.
  
  ### 2.4 프로토콜
  블록체인 노드, 각 S-node, 그리고 external storage service는 audit protocol, 그리고 view change protocol을 수행한다 [그림 2]. 각 노드는 블록 합의 및 쓰기 기능을 위해 audit protocol을 수행한다. 2-phase에 걸친 audit protocol 수행 이후, 각 S-node는 블록 B에 대한 합의 결과를 도출한다.
  
  가) Primary S-node 선정
  Primary S-node의 역할은 S-node 들 간 BFT 합의된 블록을 External Storage에 쓰고, 합의 메시지를 수합하여 전파하는 역할을 한다. 매 합의 라운드 round-robin 방식으로 하나의 S-node를 선정한다.
  
  나) 블록 수신
  각 S-node는 블록체인 노드들 중 하나의 블록체인 노드로부터 블록 B 와 블록체인 노드가 서명한 블록 해시를 수신한다.
  
  다) 블록 검증 및 합의 메시지 공유
  각 S-node는 블록 서명 검증 및 해시 체인 검사 이후 각 S-node는 primary S-node에게 합의 메시지를 전송한다. Primary S-node는 합의 메시지를 수합한다. 같은 블록 B에 대한 합의 메시지를 n-f 개 이상 수합한 경우, primary S-node는 합의 메시지 집합을 S-node 들에게 전파한다.   
  
그림  전체 프로토콜
 

  단, 한 쌍의 합의 메시지라도 같은 높이 블록에 대한 서로 다른 블록 해시값을 가진다면, 해당 합의 메시지 쌍을 S-node 들에게 전파한다. 추가로, primary S-node가 다른 노드들이 제출한 합의 메시지 상 블록체인 노드가 서명한 블록 해시값 을 기준으로 합의 메시지를 생성하는 것을 방지하기 위하여, primary S-node는 background 상 자신이 받은 블록을 broadcast한다. 일정 시간이 지나도 합의 메시지가 도달하지 않는 문제는, (1) primary S-node를 변경하고, (2) 합의 메시지를 기다리는 시간을 두 배 늘림으로써 해결한다.
  
  각 S-node가 primary S-node로부터 해당 블록 높이에 대한  개 이상의 유효한 합의 메시지를 확인하면, 자신이 가지고 있는 블록에 대한 합의 상태를 변경한다. 더 높은 높이에 대한  개 이상의 유효한 합의 메시지를 확인하면 해당 높이의 블록 보다 더 낮은 높이의 블록 모두에 대한 합의 상태를 변경한다. 모든 블록이 해시 포인터로 연결되어 있기 때문에, 더 높은 높이의 블록에 대한 합의가 하위 높이 블록의 합의를 보장한다.
  
  각 S-node는 (1) 서로 다른 블록체인 노드 서명이 있는 블록 해시 값을 가지는 합의 메시지 쌍을 수신하거나, (2) primary S-node가 broadcast 한 B 를 기준으로 생성한 블록 해시 값과 합의 메시지 내 블록 해시 값이 다르거나, (3) 블록체인 노드로부터 일정 시간이 지나도 블록이 오지 않으면, BN change message를 다른 할당된 Primary S-node 에게 제출한다. Primary S-node는 자신이 수신한 BN change message 들을 기반으로 가장 큰 h1을 결정하고, new BN proposal을 생성, 전파한다. 다른 S-node들은 new BN proposal에 대한 vote를 진행한다. S-node가 n-f 개 vote를 모으면, 다음 BN으로 넘어간다.
  
  ## 3. 보안성 분석
  Theorem 1. (Safety) 모든 정상적인 노드는 같은 높이에 대해서 서로 다른 블록을 commit 할 수 없다.
  
  Proof. 우리는 두 가지 경우를 고려한다. 첫 번째, primary S-node가 비잔틴 노드이라고 가정하자. 각 S-node는 블록 B 에 대한 합의 인스턴스를 종료하지 않는다. Primary S-node가 정상 노드가 될 때 까지 B 에 대한 합의 결과를 얻는 것을 지연한다. 두 번째, primary S-node가 정상 노드라고 가정하자. h'이 충분히 클 때, Primary S-node는 성공적으로 블록 Bh'에 대한 n-f 개의 합의 메시지를 모으고 다른 s-node들에게 전파한다. 각 S-node는 B를 commit 한다. 각 S-node는 블록체인의 hash chain property 에 따라서 인 모든 블록 B 에 대한 합의 결과를 가진다.
  
  Theorem 2. (Liveness) 특정 높이 상 적어도 한 블록의 합의 결과는 언젠가 반드시 도출된다.
  
  Proof. h'이 충분히 클 때, 각 S-node는 에 Bh'에 대한 합의 결과를 가진다. h<h' 인 모든 블록 Bh 에 대하여 각 S-node는 블록체인의 hash chain property에 따라 적어도 한 번 Bh를 관찰하였다. 따라서, Bh'에 대한 합의 결과를 가진 이후, Bh에 대한 합의 결과를 가질 수 있다. 일관성을 잃지 않고, 각 S-node는 모든 블록 B에 대한 합의 결과를 도출한다.
  
  Theorem 3. (Integrity) 성공적인 write(h, B, M) 이후, 정상적인 S-node에 의해서 호출된 서로 다른 두 개의 read(h) 는, external storage가 정상적이라면, 같은 B 를 반환 한다.
  
  Proof. 정상적인 S-node 라면, B에 대한 같은 M 내 블록체인 노드가 서명한 블록 해시 정보를 가진다.  메타데이터 M을 통해 같은 블록 B 반환을 검증 가능하다.
 
  Theorem 4. (Block Availability) 성공적인 write(h, B, M) 이후 정상적인 S-node로부터 read(h) 가 호출되면, read(h) 는 최종적으로 실행되고, external storage가 정상적이라면, B를 반환한다.
  
  Proof. S-node가 가진 M 을 통해 B가 external storage에 쓰여졌는지 검증 가능하다. External storage가 정상이라면, S-node의 read(h) 호출 시, 이미 쓰인 B를 반환한다.
 
  Theorem 5. (Containment) read_causal(h) 로부터 반환된 블록 집합 {Bh} 에 대해서, read_causal(h') 로부터 반환된 {Bh'} (h'<h)은 {Bh'} in {Bh} 를 만족한다.
  
  Proof. Theorem 1에 따라 같은 높이에 대해서 두 개 다른 블록을 commit하지 않는다. 즉, S-node들은 항상 같은 이전 해시 포인터를 가진 고유 블록들에 대해서 commit한다.
  
  Theorem 6. (Causality) read_causal(h)는 commit된 에 대한 write(h, B, M) 이 수행되기 이전, 최소 n-f 개 honest S-node들에 의해 commit 된 블록을 전부 포함한다.
  
  Proof. B commit 시점에 그 이전 높이 블록 Bh' (h'<h) 는 Theorem 5에 따라 무조건 commit 된다. 즉, commit 된 블록 이전 높이 블록들은 전부 commit 된 상태이다.
   
  Theorem 7. (External validity) 만일 정상적인 S-node가 블록 에 대한 합의 결과를 도출하고 메타 데이터  를 생성했다면,  는 참이다.
  
  Proof. 모든 정상적인 S-node는 external storage에 저장된 각 블록 B 에 대한 메타데이터 M 을 가지고 있다. 정상적인 S-node의 경우 메타데이터에 포함된 합의 메시지 리스트 를 기준으로 는 P(B, M)이 참임을 검증할 수 있다.
  
  ## 4. 결론
  본 논문에서는 Amazon S3와 같은 대규모 클라우드를 활용한  블록체인 스토리지 시스템인 BSS를 제안한다. 외부 저장소는 신뢰할 수 없기 때문에 audit protocol이 BSS에 적용된다. 우리는 BSS가 7가지 보안 속성 들을 충족함을 보여주었다.
  
  ## Reference
  [1] Fjl (Feb 2018), Iceberg [Online], Available: https://github.com/ethereum/go-ethereum/releases/tag/v1.8.0
  [2] IOTA (2022) [Online], Available: https://iota.org
  [3] Snapshot Verification (2022) [Online], Available: https://docs.solana.com/proposals/snapshot-verification
  [4] Stefano Della Valle, IOTA Permanode As a Service (Apr 2020) [Online], Available: https://medium.com/things-lab/iota-permanode-as-a-service-b907216c6931
  [5] Y. Amir, B. Coan, J. Kirsch and J. Lane, "Prime: Byzantine Replication under Attack," in IEEE Transactions on Dependable and Secure Computing, vol. 8, No. 4, pp. 564-577, Juli-Aug. 2011, doi: 10.1109 / TDSC.2010.70.
  [6] Danezis, G., Kokoris-Kogias, L., Sonnino, A., & Spiegelman, A. (2022, March). Narwhal and Tusk: a DAG-based mempool and efficient BFT consensus. In Proceedings of the Seventeenth European Conference on Computer Systems (pp. 34-50).
  [7] J. Sousa and A. Bessani, "From Byzantine Consensus to BFT State Machine Replication: A Latency-Optimal Transformation," in Ninth European Dependable Computing Conference, 2012, pp. 37-48, doi: 10.1109/EDCC.2012.32..
  [8] S. Duan and H. Zhang, "PACE: Fully Parallelizable BFT from Reproposable Byzantine Agreement." Cryptology ePrint Archive (2022).
  [9] C. Cachin, K. Kursawe, F. Petzold, and V. Shoup. Secure and efficient asynchronous broadcast protocols. In Proc. of the 21st Annual Int. Cryptology Conf. on Advances in Cryptology – CRYPTO’01, 2001
  

