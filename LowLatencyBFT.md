# 이레이저 코드 기반 전파 기법을 적용한 저지연 비잔틴 장애 감내 합의 프로토콜

# A Low-Latency BFT Consensus Protocol utilizing Erasure Code Based Broadcast

## 요약
    
    BFT 기반 합의 프로토콜을 채택하는 블록체인 플랫폼에서는 트랜잭션 전파, 블록 생성, 블록 전파, BFT 합의, 트랜잭션 수행, 그리고 트랜잭션에 대한 응답 등 크게 여섯가지 단계를 거쳐 클라이언트 트랜잭션을 처리한다. 대부분의 BFT 기반 합의 프로토콜은 클라이언트 트랜잭션 처리 과정상 BFT 합의 과정에 집중하고 있으며, 블록 생성, 전파 및 합의 과정이 하나로 묶여서 구성되어 있는 Monolithic한 합의 구조를 가진다. 따라서 이로 인해 트랜잭션 전파 과정은 합의 과정과 별도로 처리되어야 함에 따라, 트랜잭션 전파 및 블록 전파가 중복적으로 진행되어야 하는 네트워크 오버헤드가 존재한다. 본 논문에서는 트랜잭션 전파와 블록 전파를 통합하고, 또한 블록 전파시 이레이저 코드를 적용하여 블록 전파에 대한 지연 시간을 줄이는 저지연 BFT 합의 프로토콜을 제안한다. 제안한 합의 프로토콜의 성능 검증을 위해 허가형 블록체인 플랫폼 Audit Chain [4]에 적용하고, 결과적으로 네트워크 성능에 크게 영향을 받지 않음을 보였다.

  
## 1.서론
  BFT 기반 합의 프로토콜을 채택하는 블록체인 플랫폼에서는 트랜잭션 전파, 블록 생성, 블록 전파, BFT 합의, 트랜잭션 수행, 그리고 트랜잭션에 대한 응답을 거쳐 클라이언트 트랜잭션을 처리한다.
  현재 대부분의 BFT 기반 합의 프로토콜에 관련된 연구는 그림 1의 transaction lifecycle 상 BFT Consensus 부분에 집중하고 있다. 실제, Hotstuff [1], Fast Hotstuff [2], DiemBFT [3] 와 같은 연구들은 합의 프로토콜 오버헤드를 고려하여 리더 중심의 합의 프로토콜을 적용해 합의 알고리즘 복잡도를 O(n)으로 줄였다. 추가로, 합의 라운드 개수를 줄임으로 합의 과정의 latency를 낮춘다.
  
![Figure 0 일반적인 BFT 기반 블록체인에서의 transaction lifecycle](https://user-images.githubusercontent.com/23546909/235391635-f8bcca05-6494-4959-aab7-90325394db65.png)
그림 1. BFT 합의 프로토콜 채택 블록체인 플랫폼 transaction lifecycle

  하지만 일반적인 BFT 기반 합의 프로토콜의 경우, 블록 생성, 전파 및 합의 과정이 하나로 묶여서 구성되어 있는 Monolithic 합의 구조를 가지고 있다. Monolithic한 합의 구조로 블록 생성 및 전송에서 오는 오버헤드가 존재한다. 그림 1 에서 보이듯, 각 노드들이 BFT 합의를 진행하기 위해서는 앞선 과정에서 블록을 생성하고 전파하는 과정이 수반되어야 한다. 블록 생성 및 전파 과정에서의 블록 크기가 BFT Consensus 과정에서 주고 받는 합의 메시지보다 크기 때문에, 리더 기반 BFT 합의 알고리즘에서의 leader bottleneck이 존재한다.
  Audit Chain [4] 은 Monolithic한 합의 구조에서 오는 문제점을 해결하기 위하여, 블록 생성 및 전파를 담당하는 Blockchain Service Provider (BSP) 와 BFT 합의 및 BSP 감시/교체를 담당하는 Auditor로 구성된 BFT 합의 프로토콜이다 (그림 2). 크게 세 가지 아이디어를 기반으로 합의 구조에 대한 문제점을 해결하였다. (1) O(n) 메시지 복잡도를 가지는 선형화된 BFT 합의 프로토콜이다. 블록 전파 대신 블록 해시 전파로 BFT 합의를 시작하는 overlapping 최적화 기법, 합의 과정을 중첩하여 진행하는 pipelining 기법을 적용하여 블록 생성 및 전파에서 오는 오버헤드 및 합의 오버헤드를 줄였다. (2) 블록 생성과 합의 과정을 분리함으로써, 각 과정이 독립적으로 수행될 수 있도록 하였다. 이는 블록 생성 과정, 블록 합의 과정을 위해 구성된 P2P 오버레이 네트워크의 효과를 확대한다. 또한, 전체 트랜잭션 처리에 대한 워크로드를 다른 노드에 분담함으로써, 블록 생성, 블록 전파, 그리고 합의에 대한 오버헤드를 줄인다. (3) Speculative하게 트랜잭션을 BSP 단에서 수행하도록 함으로써, BSP 수준에서의 트랜잭션 처리 혹은 auditor 단에서의 트랜잭션 처리를 지원한다.
  
  ![Audit Chain 1 0](https://user-images.githubusercontent.com/23546909/235391614-12ff29bf-fca8-4721-abe7-81367640b1e5.png)
  그림 2. Audit Chain transaction lifecycle

  본 논문에서는 트랜잭션 전파와 블록 전파를 통합하고, 블록 전파 단에서 이레이저 코드를 적용하여 블록 전파에 대한 지연시간을 줄이는 저지연 BFT 합의 프로토콜을 제안한다. 제안한 합의 프로토콜의 검증을 위해 허가형 블록체인 플랫폼 Audit Chain [4]에 적용한다. (이하 적용 버전을 Audit Chain 2.0으로 일컬음.) Audit Chain 2.0에 대한 주요 contribution은 아래와 같다.
  • 블록 생성 및 전파 단에서 이레이저 코드를 적용하여 블록 전파에 대한 지연시간을 줄이고, 블록 저장 오버헤드를 감소시켰다.
  • Block Availability Layer와 Block Consensus Layer를 분리하여, 블록 생성 및 전파와 블록 합의 과정이 독립적으로 수행될 수 있도록 하였다.
  • Block Availability Layer에서 블록에 대한 causality와 availability를 보장하고, Block Consensus Layer 상에서는 total order를 위한 별도의 communication overhead 없이 각자 auditor 노드가 total order를 결정하고, 트랜잭션을 수행한다.
  
  ![표 0 transaction batch 당 보내는 데이터 양 및 전송 시간 비교표](https://user-images.githubusercontent.com/23546909/235391666-0a6612c0-2444-46d9-ab6c-04d2a1965245.png)
  표 1. transaction batch 당 보내는 데이터 양 및 전송 시간 비교

  지금부터, Audit Chain [4] 을 Audit Chain 1.0 이라고 일컬을 때, 하나의 트랜잭션 batch에 대하여 BSP로부터 각 auditor에게 보내는 데이터 양은 Audit Chain 2.0 이 Audit Chain 1.0 보다 더 작다. 이에, 한 트랜잭션 batch를 보내는 시간이 Audit Chain 2.0 이 Audit Chain 1.0 보다 더 짧다 (표 1).
  
## 2. 시스템 설계
### 2.1 시스템 모델 및 가정
  메시지 통신에 사용되는 네트워크는 partially synchronous network를 가정한다. 즉, global stabilization time (GST) 이전 전송된 메시지에 대해서는 adversary가 유한한 시간 만큼 딜레이 시킬 수 있고, GST 이후에는 모든 메시지가 유한한 time bound 안에 도달한다. 단, 어느 누구도 언제 GST에 도달했는지 알 수 없다.                             
  우리는 클라이언트로부터의 트랜잭션 batch를 physical block으로 정의하고, physical block을 이레이저 코딩하여 생긴 chunk에 대응되는 block을 logical block으로 정의한다. 
  Physical block 생성, physical block encoding 그리고 encoding 결과 생성된 chunk 전파를 담당하는 하나의 Blockchain Service Provider (BSP) 노드와 logical block 전파와 physical block의 causality와 availability 확인을 위한 투표를 담당하는  개의 auditor 노드를 가정한다. 최대 개의 auditor가 byzantine fault를 일으킬 수 있다.
  우리 시스템은 일반적인 BFT 합의 프로토콜이 만족하는 세 가지 보안 특성 [5] 을 만족한다. 현재 우리 시스템에 맞게 세 가지 특성을 재구성하여 기술하였다.
  • Validity: 만일 특정 logical block에 대하여 n-f 개 이상의 auditor 노드가 합의 하였다면, 그 logical block은 특정 auditor 노드에 의해서 propose된 logical block이다.
  • Agreement: 만일 한 honest한 auditor가 최종적으로 특정 높이의 physical block을 commit 하고, 다른 honest한 auditor도 해당 높이의 physical block에 대하여 commit한다면, 그 block은 같다.
  • Termination: 각 physical block에 대한 합의 결과는 언젠가 반드시 도출된다.
  클라이언트, BSP, auditor단 트랜잭션 제출 및 처리를 위한 인터페이스들이 존재한다. 
  클라이언트는 트랜잭션을 제출하기 위한 목적의 한 가지 인터페이스가 있다. SubmitTransaction()를 통하여, 클라이언트는 BSP에게 트랜잭션을 제출한다.
  BSP는 physical block의 인코딩 결과로 생성된 chunk set 을 auditor 들에게 보내기 위한 목적의 한 가지 인터페이스가 존재한다. BroadcastChunks()를 통하여 n명의 auditor 들에게 physical block 의 chunk를 전파한다.
  각 Auditor는 chunk를 BSP로부터 수신받고, 수신받은 chunk를 기준으로 각자 logical block 을 propose 한다. Propose한 block에 대하여 각 auditor는 voting rule에 따라 투표하고, physical block에 대한 commit 유무를 결정한다.
  각 Auditor는 유지하고 있는 logical block을 기준으로 physical block을 재구성하기 위한 두 가지 인터페이스가 존재한다. 하나는 특정 높이의 physical block 를 재구성하는 Read(), 다른 하나는 특정 높이로부터 그 이전 모든 높이의 physical block  를 재구성하는 ReadCausal() 이다. 두 인터페이스를 통하여 각 auditor는 auditor network 상 분배되어 있는 chunk 정보를 기준으로 physical block을 reconstruct할 수 있다.
  Physical block, logical block에 대한 정의와 위 인터페이스를 기준으로 생각하였을 때, Audit Chain 2.0은 아래의 세 가지 추가적인 특성을 만족한다. 아래 세 가지 특성은 DAG-based BFT Consensus (e.g. [6], [7], [8]) 상에서 일반적으로 만족하게 되는 세 가지 특성을 재구성한 것이다.
  모든 physical block 와 모든 honest한 auditor 를 기준으로 생각했을 때, 모든 commit된 logical block 에 대하여, 만일 한 honest auditor 가 최종적으로 logical block을 commit 하였다면,
  • Availability:  가 Read() 한다면, 그 auditor는 최종적으로 physical block 을 재구성할 수 있다.
  • Integrity: 다른 모든 honest auditor 가 invoke한 모든 Read()는 동일한 physical block 를 반환한다.   
  • Total Order: 다른 모든 honest auditor 가 invoke한 모든 ReadCausal()는 동일한 순서의 physical block  를 반환한다.
  
  ![Audit Chain 2 0 Architecture](https://user-images.githubusercontent.com/23546909/235391660-cc0cfe48-5d17-462e-ba95-df3ec408b931.png)
  그림 4. Audit Chain 2.0 구조
  
  ### 2.2 시스템 구조
 BSP는 클라이언트로부터 트랜잭션을 받는다. BSP는 수신한 트랜잭션을 기준으로 트랜잭션 batch를 구성하여, physical block을 구성한다. Physical block은 정해진 순서가 있는 linear chain 형태로 구성된다. BSP는 구성된 physical block에 Erasure Coding을 적용한다. 이 때, 를 통해서 Erasure Coding을 적용하는데, 그 뜻은 전체의 데이터를 개로 나누고,  개의 chunk 만을 가지고도 다시 원래의 데이터를 복구할 수 있다는 뜻이다. 해당 encoding 된 데이터를 chunk라고 하고, BSP는 각 chunk를 auditor 들에게 전파하게 된다. 
  Auditor는 logical block proposal과 vote를 round 단위로 진행한다. Logical block proposal은 각 auditor는 하나의 chunk를 BSP로부터 전파받는다. Auditor는 전파받은 chunk를 기준으로 logical block을 생성한다. Auditor는 logical block을 생성할 경우, 다른 auditor들에게 내가 BSP로부터 chunk를 받았다는 사실을 알리기 위해 logical block을 propose한다. 또, logical block vote는 다른 auditor가 propose한 logical block이 valid하다고 판단할 경우, logical block에 대하여 vote한다. physical block에 대한 chunk가 auditor network상 충분히 분배되어 있는지 확인하기 위하여 vote를 진행한다. 한 auditor가 특정 logical block에 대하여 n-f 개 이상의 vote를 모았을 경우, vote를 기준으로 certificate를 생성하고, 생성된 certificate를 다른 auditor들에게 전파한다. 각 auditor는 logical block과 certificate를 모두 가지고 있는 경우, logical block과 certificate으로 해당 round의 한 DAG 노드를 생성한다.
  
  ### 2.3 메시지 형식
  
  <img width="1327" alt="Audit Chain 2 0 메시지 형식" src="https://user-images.githubusercontent.com/23546909/235393883-c795bd66-d96e-487c-961f-4b28a5f333c5.png">
  그림 5. Audit Chain 2.0 메시지 형식. 메시지 형식 상 표시되어 있는 block은 전부 logical block을 의미한다.
  
   그림 5는 Audit Chain 2.0 상 BSP와 Auditor가 주고 받는 메시지 형식을 나타낸다. 한 auditor는 하나의 physical block에 대하여 하나의 chunk를 담당하여 저장하고 전파한다. 이에 chunk 에는 physical block에 대하여 어떤 chunk에 대응되는지 확인 목적으로 chunk id, 어떤 physical block에 대하여 대응되는지 확인 목적으로, block num을 가지고 있다. 또한, BSP는 해당 chunk가 physical block에 대응되는 chunk라는 사실을 증명하기 위하여 chunk들을 leaf node로 하는 merkle tree를 구성하고, merkle root와 merkle path를 chunk에 대한 metadata로 구성하여 포함한다. Certificate 상 포함된 개의 signature는 해당 logical block에 대한 vote 인증으로 볼 수 있다.
  
  ### 2.4 프로토콜
  각 auditor 입장에서 round 진행 규칙은 아래와 같다. 각 auditor는 라운드 에서 아래 세 가지 중에 하나만 만족하면, 다음 라운드 로 넘어간다. (1) round r 에 대해서, 개 이상의 logical block에 대한 certificate를 가지고 있을 경우, (2) 특정 round 에 대한 chunk를 수신한 시점에 timer를 시작하고, 에 대한 logical block certificate가 도착한 시점에 timer를 다시 초기화 한다고 하자. Timer 최대 대기 시간을 초과하는 경우. 만일 최대 대기 시간을 초과하는 경우, 최대 대기 시간을 두 배로 변경한다. 만일 최대 대기 시간 이전 라운드가 진행된다면, 다시 최대 대기 시간을 초기화한다. (3) 현재 라운드보다 더 큰 라운드에 대한 logical block proposal 이나 certificate를 수신한 경우.
  각 auditor의 proposal 규칙은 아래와 같다. 두 개 규칙을 모두 만족해야만 propose할 수 있다. (1) 이전 라운드에 대한 개 이상의 certificate를 가지고 있는 경우, (2) 해당 라운드에 대한 physical block의 chunk를 수신한 경우, 두 개를 모두 만족하면 logical block proposal을 낼 수 있다.
  각 auditor의 voting rule은 아래와 같다. 다섯 가지 조건을 모두 만족하면, 해당 logical block proposal에 대하여 투표할 수 있다. (1) block proposal이 올바른 signature를 포함하고 있는 경우, (2) block proposal이 현재 내가 유지하고 있는 round값과 같은 round를 포함하고 있는 경우, (3) block proposal 이 이전 라운드의 n-f 개 이상의 certificate를 포함하고 있는 경우, (4) block proposal 이 해당 round의 proposal 내 auditor로부터 처음 수신받는 block proposal일 경우, (5) block proposal이 유효한 Merkle root를 포함하고 있는 경우 투표한다.
  Physical block에 대한 commit rule은 아래와 같다. 각 auditor는 두 가지 조건 중 하나를 만족할 때, 현재 라운드의 physical block에 대하여 commit 한다. (1) 현재 라운드 중 적어도 하나의 logical block에 대한 certificate를 가지고 있는 경우, (2) 이후 라운드 중 적어도 하나의 logical block에 대한 certificate를 가지고 있는 경우, physical block에 대하여 commit 한다.
  Physical block commit 이후 각 auditor는 해당 블록에 대한 chunk에 대해서 저장한다. 
  
  ## 3. 분석
  우리는 표 1을 기준으로 트랜잭션 batch (logical block) 전파 시간 측면에서 Audit Chain 1.0과 Audit Chain 2.0을 비교 분석하였다.
  
  <img width="583" alt="latency vs  number of auditors" src="https://user-images.githubusercontent.com/23546909/235391684-d4ffffd7-aeed-466b-ab29-f30a8105fdc4.png">
  그림 6 트랜잭션 batch 전파 시간 vs. auditor 개수
  
  평균 트랜잭션 크기를 2.82KB (대표 Permissioned Blockchain Hyperledger Fabric 기준으로 할 때), batch size를 100이라고 가정한다. Audit Chain 2.0의 경우 auditor의 개수가 증가하는 것 (i.e. 네트워크 통신 오버헤드 증가)에 큰 영향을 받지 않고, 비슷한 수준의 트랜잭션 batch 전송 시간을 가진다. 
  
  ## 4. 결론 및 향후 방향
    본 논문에서는 트랜잭션 전파와 블록 전파를 통합하고, 블록 전파 단 이레이저 코드를 추가하여 블록 전파 지연시간을 낮춘 Audit Chain 2.0을 제안했다. 향후 추가 연구로는 복수 개의 BSP를 세팅함으로 BSP에 대한 공격을 고려하고, 더 높은 처리량을 낼 수 있는 확장성 있는 설계를 진행하고자 한다. 
  
  ## Reference
  [1] Yin, Maofan, et al. "HotStuff: BFT consensus with linearity and responsiveness." Proceedings of the 2019 ACM Symposium on Principles of Distributed Computing. 2019.
[2] Jalalzai, Mohammad M., et al. "Fast-hotstuff: A fast and resilient hotstuff protocol." arXiv preprint arXiv:2010.11454 (2020).
[3] Pierro, G. Antonio, and Roberto Tonelli. "A Study on Diem Distributed Ledger Technology." Proceedings http://ceur-ws. org ISSN 1613 (2022): 0073.
[4] 오하늘, 박찬익, Audit Chain: O(n) BFT 합의 프로토콜 기반 고성능 허가형 블록체인, 한국정보과학회 학술발표논문집 Vol.2022 No.06 [2022]
[5] Antoniadis, Karolos, et al. "Leaderless consensus." Journal of Parallel and Distributed Computing 176 (2023): 95-113.
[6] Keidar, Idit, et al. "All you need is dag." Proceedings of the 2021 ACM Symposium on Principles of Distributed Computing. 2021.
[7] Danezis, G., Kokoris-Kogias, L., Sonnino, A., & Spiegelman, A. (2022, March). Narwhal and Tusk: a DAG-based mempool and efficient BFT consensus. In Proceedings of the Seventeenth European Conference on Computer Systems (pp. 34-50).
[8] Spiegelman, Alexander, et al. "Bullshark: Dag bft protocols made practical." Proceedings of the 2022 ACM SIGSAC Conference on Computer and Communications Security. 2022.

    
  ## Appendix 특성 증명
  Theorem 1. (Validity) 만일 특정 logical block에 대하여 n-f개 이상의 auditor 노드가 합의 하였다면, 그 logical block은 특정 auditor 노드에 의해서 propose된 logical block이다.
  
  Proof. 그림 5 메시지 형식에서 보면, logical block proposal 내 signature of sender가 포함되어있다. 특정 logcial block에 대하여 n-f개 이상의 auditor 노드가 합의 하였다는 뜻은, 해당 logical block의 경우 모든 honest한 노드가 logical block 내 signature of sender를 검증하고 vote한 logical block이다. Q.E.D.
  
  Theorem 2. (Agreement) 만일 한 honest한 auditor가 최종적으로 특정 높이의 physical block을 commit 하고, 다른 honest한 auditor도 해당 높이의 physical block에 대하여 commit한다면, 그 block은 같다.
  
  Proof. Voting rule (5)를 살펴보았을 때, 각 auditor는 valid한 merkle root가 포함되어 있는지 검증한 이후에 vote 한다. Physical block에 대한 commit 조건 두 가지로 경우를 나누어 생각해보자. (1) 현재 라운드 중 적어도 하나의 certificate를 가지고 있는 경우, voting rule (5)에 의하여 같은 라운드의 서로 다른 auditor로부터의 certificate는 전부 같은 physical block을 가리킨다. 만일 다른 physical block을 가리킬 경우 voting rule (4), (5)에 위배되어 각 auditor가 투표하지 않음. (2) 이후 라운드 중 적어도 하나의 logical block에 대한 certificate를 가지고 있는 경우, 미래의 적어도 하나의 logical block에 대한 certificate가 나온 라운드를 r'이라고 하자. r' certificate에 속해 있는 r'-1 의 n-f 개 certificate는 고정 되어 있다. 즉, r'-1의 n-f개 certificate는 n-f 개 이상의 auditor 노드에 의하여 합의된 이전 라운드의 certificate이다. 같은 방식으로 r'-2, r'-3..., r 의 certificate 는 모든 honest 한 auditor 노드에 의해서 각 라운드의 logical block이 합의되었다는 것을 의미한다. 결국 이전 모든 라운드에 대해서도 같은 physical block에 대해서 commit한다.
  
  Theorem 3. (Termination) 각 physical block에 대한 합의 결과는 언젠가 반드시 도출된다.
  
  Proof. Partially synchronous network를 assume 하고 있다. 즉, 노드와 노드가 교환하는 메시지는 global stabilization time (GST) 이후에 언젠가 도착하게 된다. BSP로부터의 physical block에 대한 chunk는 반드시 각 auditor한테 도달하고, honest한 모든 n-f 개의 auditor 노드는 수신한 chunk를 기준으로 logical block proposal을 생성하게 된다. 각 auditor 노드로부터 생성된 logical block proposal은 모든 honest한 auditor 노드한테 도달하기 때문에, 모든 honest한 auditor 노드로부터 해당 logical block proposal에 대한 vote를 반드시 받을 수 있다. Physical block에 대한 합의 결과는 logical block에 대한 certificate를 수신 여부로 결정되기 때문에, 각 physical block에 대한 합의 결과는 언젠가 반드시 도출된다.
  
  아래 세 가지 Theorem에 대해서도 증명 가능하다.
  
  모든 physical block Bh 와 모든 honest한 auditor Ai 를 기준으로 생각했을 때, 모든 commit된 logical block Bhi' 에 대하여, 만일 한 honest auditor Ai 가 최종적으로 logical block Bhi'을 commit 하였다면,
 
  Theorem 4. (Availability) Ai 가 Read(h) 한다면, 그 auditor는 최종적으로 physical block Bh 을 재구성할 수 있다.
  
  Proof. logical block Bhi'을 commit 하였다는 뜻은 반드시 logical block Bhi'에 대한 certificate가 존재한다는 의미이다. 즉, physical block Bh에 대한 chunk n-f개 이상이 auditor network에 분배되어 있다는 뜻이다. 설령, malicious 한 auditor 최대 f개의 chunk가 vote 이후 유실된다고 하더라도, EC(n, n-2f) 인코딩 scheme을 적용하고 있기 때문에, n-f-f = n-2f 개의 chunk 만 모으더라도, physical block Bh를 재구성할 수 있다.
 
  Theorem 5. (Integrity) 다른 모든 honest auditor Ai 가 invoke한 모든 Read(h)는 동일한 physical block Bh를 반환한다. 
  
  Proof. Theorem 2에 의해서 높이가 고정되어 있는 상황에서는 commit 된 특정 높이의 physical block은 전부 같다. 즉, commit된 logical block Bhi'에 포함되어 있는 vote에 대응되는 auditor 들의 chunk를 통하여 physical block Bh를 재구성할 수 있다.
  
  Theorem 6. (Total Order) 다른 모든 honest auditor Ai 가 invoke한 모든 ReadCausal(h)는 동일한 순서의 physical block B1, B2, ..., Bh 를 반환한다.
  
  Proof. Voting rule (3)에 의해서 logical block Bhi'은 이전 라운드 h-1의 n-f 개 이상의 logical block을 참조한다. 즉, 이전 라운드 h-1의 n-f개 이상의 logical block을 통해서 physical block Bh-1 을 재구성할 수 있다. 마찬가지로 h-1의 logical block은 h-2 라운드의 n-f 개 이상의 logical block을 참조한다. 같은 원리로, Bh-2, Bh-3, ..., B1을 순서대로 재구성할 수 있다.
  

