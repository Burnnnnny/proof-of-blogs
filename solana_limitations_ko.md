<img 
  width="1000px"
 height="430px"
 src="./images/shinchan .jpg"
/>

# 🦀 심층 분석: Solana의 제약 사항들 (현재 작성 중)

GM GM 여러분 😁,

이 블로그에서는 스마트 컨트랙트를 개발하면서 마주칠 수 있는 다양한 유형의 Solana 리소스 제약 사항들을 살펴보겠습니다. "limit(제한)" 또는 "exceed(초과)"와 같은 단어가 포함된 에러를 만날 수 있습니다. 이러한 에러는 블록체인의 공정성과 성능을 유지하기 위해 Solana 프로그램이 미리 정의한 경계를 나타냅니다. 이러한 주제에 대해 더 알고 싶으시다면, 잘 찾아오셨습니다!

이 블로그에서 다룰 내용은 다음과 같습니다:

- CU 제약 사항 (Compute Unit Limitations)
- 트랜잭션 크기 제약 (Transaction size limitations)
- 스택 크기 (Stack size)

# Solana의 제약 사항:

Solana는 로직과 상태를 서로 다른 계정으로 분리하는 독특한 트랜잭션 처리 방식을 통해 Bitcoin이나 Ethereum 같은 전통적인 블록체인과 차별화되는 고성능 퍼블릭 블록체인입니다. 이를 통해 Solana는 많은 트랜잭션을 동시에 처리할 수 있습니다.

하지만 Solana에서 실행되는 프로그램은 몇 가지 리소스 제약 사항의 적용을 받습니다. 이러한 제약 사항은 프로그램이 시스템 리소스를 공정하게 사용하고 높은 성능을 유지하도록 보장합니다.

이러한 경계를 아는 것은 매우 도움이 되며, 프로그램 설계자로서 여러분은 저렴하고 빠르며 안전하고 기능적인 프로그램을 만들기 위해 이러한 제약 사항을 염두에 두어야 합니다.

## 1. 컴퓨트 유닛 제약 사항 (Compute Unit Limitations):

### 컴퓨트 유닛(Compute Unit)이란?

CU (Compute Unit)라는 이름 자체가 시사하듯이, CU는 Solana의 트랜잭션이나 명령어가 수행하는 연산 작업(CPU 사이클 및 메모리 사용량)의 기본 측정 단위입니다. Ethereum의 "Gas" 요금과 비슷하지만 훨씬 더 예측 가능하고 지연 시간이 짧습니다.

계정을 읽거나 쓰거나, 암호화 작업(예: zk-ElGamal)을 수행하거나, 서명 검증 및 직렬화/역직렬화와 같이 스마트 컨트랙트가 온체인에서 실행하는 모든 명령어는 일정량의 컴퓨트 유닛(CU)을 소비합니다. 이는 노드가 수행하는 작업량에 대략 비례합니다.

간단한 트랜잭션을 수행하면 노드가 스마트 컨트랙트를 효율적으로 처리할 수 있어 CU 소비량이 적습니다. 그러나 복잡한 수학 연산이나 무거운 루프를 수행하면 노드가 많은 양의 메모리와 CPU를 소비하고 프로그램을 실행하는 데 더 많은 시간이 걸리므로 CU 소비량이 높아집니다.

_예를 들어, 지갑 A에서 지갑 B로 SOL을 보내는 간단한 트랜잭션의 경우 약 3000 Compute Unit이 소요됩니다._

```rust

let transfer_amount_accounts = Transfer {
      from: ctx.accounts.signer.to_account_info(),
      to: ctx.accounts.recipient.to_account_info(),
    };

let ctx = CpiContext::new(
        ctx.accounts.system_program.to_account_info(),
        transfer_amount_accounts,
      );

transfer(ctx, amount * LAMPORTS_PER_SOL)?;

 // 약 3000 CU 소요
```

위 코드는 Anchor ⚓️로 작성되었으며, 간단한 SOL 전송을 수행하고 이를 위해 3000 CU를 사용합니다.

### 컴퓨트 유닛 예산 (Compute Unit Budget)

- 아시다시피, 무거운 수학 연산이나 루프는 많은 양의 컴퓨트 유닛을 소비합니다. 하지만 모든 트랜잭션이나 명령어에는 기본적으로 **200,000 CU의 예산**이 있습니다.

- 트랜잭션이나 명령어가 200,000 CU 한도를 모두 소진하면, 해당 트랜잭션이나 명령어는 단순히 되돌려지고(reverted), 모든 상태 변경이 취소되며, 트랜잭션을 호출한 서명자에게 수수료는 환불되지 않습니다. (이 메커니즘은 공격자가 노드에서 끝나지 않거나 계산 집약적인 프로그램을 실행하여 체인을 느리게 하거나 멈추게 하는 것을 방지합니다.)

```rust
Error: exceeded maximum number of instructions allowed (200000) compute units
```

- 개인적으로 Chaubet 프로젝트를 만들 때 이 에러를 겪었습니다. 베터(bettor)가 주식을 매수할 때, [이 함수](https://github.com/baindlapranayraj/Chaubet/blob/86b5c91de727dd173a0593a7378a0acb3eb25b2a/programs/chaubet/src/utils/helper.rs#L89C5-L132C6)는 꽤 무거운 수학적 계산과 확인을 수행합니다. 이로 인해 명령어가 200,000 CU 한도를 초과했고 전체 명령어가 되돌려졌습니다.

- `SetComputeUnitLimit`을 사용하여 계산 한도를 늘릴 수 있습니다. 트랜잭션에 명령어를 추가하여 특정 계산 유닛 한도를 요청할 수 있습니다. 하지만 🍑 CU 예산은 최대 140만 유닛까지만 늘릴 수 있습니다.

```rust
const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
  units: 500000,  // 200k에서 500k CU로 증가.
});
```

### 왜 제한된 Compute Unit 예산이 존재할까요?

짧고 간단하게 말하면, Solana는 공정한 리소스 할당을 보장하기 위해 CU 제약 사항을 두고 있습니다. 하지만 그게 무슨 뜻일까요?

Solana 검증자(validator)들은 블록체인 트랜잭션을 처리하고 네트워크 상태를 유지하는 개별 컴퓨터(또는 노드)입니다 (이건 알아야 할 기본 사항입니다 😒). 각 검증자는 일반 컴퓨터와 마찬가지로 제한된 CPU 파워와 메모리를 가지고 있습니다. 이러한 노드 내부에서 프로그램이 실행될 때, 모든 명령어를 처리하기 위해 약간의 메모리를 할당합니다.

이제 악의적인 사용자가 무한 루프가 포함된 트랜잭션을 보낸다면, 엄청난 양의 메모리를 사용하여 시스템을 느리게 하거나 심지어 충돌시킬 수 있습니다. 블록체인은 공유된 컴퓨터/노드들의 네트워크이므로, 한 명의 사용자가 엄청난 수의 CPU 작업을 수행하고 많은 CPU/메모리를 사용하면 시스템을 독점하여 다른 사용자들을 굶주리게(starving) 만듭니다.

CU 예산의 이러한 제약 덕분에 Solana 네트워크는 검증자에 대한 서비스 거부(DoS) 공격과 리소스 고갈을 방지할 수 있습니다.

## 2. 트랜잭션 크기 제약 (Transaction Size Limit)

트랜잭션 크기 제약으로 들어가기 전에, 트랜잭션이 무엇이고 무엇을 포함하는지 살짝 엿보겠습니다.
Solana의 트랜잭션은 사용자가 네트워크와 상호 작용하기 위해 보내는 요청이며, 일반적으로 계정 간에 램포트(Solana의 기본 토큰 단위)를 전송하는 등의 데이터 변경을 목적으로 합니다.

각 트랜잭션에는 블록체인에서 수행할 작업을 지정하는 하나 이상의 명령어가 포함되어 있습니다. 이 실행 로직은 프로그램 상태에 원시 바이트(raw bytes)로 저장됩니다.

<img
 width="1000px"
 height="550px"
 src="./images/trx_arch.png"
/>

### 트랜잭션의 구성 요소:

- **서명 배열 (Array of signatures)**: 트랜잭션에 포함된 서명들의 시퀀스입니다.
- **메시지 (Message)**: 트랜잭션의 핵심 부분입니다. 트랜잭션이 수행하고자 하는 작업을 설명하는 데 필요한 모든 것을 포함합니다.
  
  위 이미지에서 볼 수 있듯이, 메시지는 트랜잭션의 핵심 부분으로 다음을 포함합니다:

- 헤더 (Header): 서명자 및 읽기 전용 계정의 수를 나타냅니다 **(3 bytes)**.
- 계정 배열 (Array of Accounts): 클라이언트에서 제공한, 트랜잭션을 수행하는 데 필요한 모든 계정을 포함합니다 (각 Pubkey는 32 bytes).
- 최근 블록해시 (Recent Blockhash): 이 트랜잭션에 첨부된 최근 블록해시입니다 (32 bytes).
- 명령어 배열 (Array of Instruction): 서명자가 호출한 모든 명령어를 포함합니다 (크기는 명령어 코드의 복잡성에 따라 다름).

### 트랜잭션 크기 제약 사항:

| **구성 요소** | **크기** | **설명** |
| --- | --- | --- |
| Signature | 64 bytes | 트랜잭션에 포함된 각 서명 |
| Message Header | 3 bytes | 서명자 및 읽기 전용 계정 수 표시 |
| Account Pubkey | 32 bytes (계정당) | 트랜잭션에 관련된 각 계정의 공개 키 |
| Recent Blockhash | 32 bytes | 트랜잭션에 첨부된 최근 블록해시 |
| Instructions | 가변적 | 트랜잭션 내 명령어의 수와 복잡성에 따라 다름 |

<br/>

모든 Solana 트랜잭션에는 **1,232 바이트**의 크기 제한이 있습니다. 전체 트랜잭션 크기는 모든 서명, 관련된 계정, 블록해시, 그리고 명령어(해당 계정 및 데이터 포함)의 바이트 합계이며, 1,232 바이트 미만이어야 합니다. 이 제한 때문에 개발자는 모든 서명자, 계정 주소, 필요한 데이터를 1,232 바이트 내에 맞춰야 합니다.

클라이언트 측에서 너무 많은 계정을 보내거나 단일 트랜잭션에 너무 많은 명령어를 넣으려고 할 때 `transaction size limit exceeded` 에러를 만날 수 있습니다.

**왜 하필 1,232 바이트로 제한되는지 궁금하실 수 있습니다.** <br/>
간단히 말해서: Solana의 1,232 바이트 트랜잭션 크기 제한은 IPv6(인터넷 프로토콜 버전 6)를 사용하여 인터넷을 통해 데이터가 전송되는 방식과 밀접한 관련이 있습니다. 하지만 조금 더 자세히 분석해 봅시다.

**이유는 다음과 같습니다:**

#### 컴퓨터 네트워킹의 기초로 잠시 돌아가 봅시다.

그렇다면 컴퓨터 네트워킹이란 무엇일까요?

간단히 말해, 데이터 📦를 공유하기 위해 연결된 컴퓨터 또는 노드들의 그룹입니다. 이는 두 개의 로컬 노드/컴퓨터(LAN) 간의 통신일 수도 있고 전 세계 컴퓨터 간의 통신(인터넷)일 수도 있습니다.

데이터 📦나 정보를 잘못된 목적지 노드로 보내지 않고 효과적으로 노드와 통신하기 위해, 우리는 따라야 할 일련의 규칙이나 프로토콜이 필요합니다. **IP (Internet Protocol)**는 네트워크상의 노드를 위한 보편적인 주소 시스템입니다. IP는 모든 노드나 컴퓨터에 고유한 주소를 제공하고, 이 주소를 사용하여 데이터 패킷 📦을 올바른 목적지로 라우팅하고 전달합니다. 네트워크를 통해 전송되는 데이터는 한 번에 모두 전송되지 않습니다. 패킷이라고 하는 작은 조각으로 나뉩니다.

하지만 Solana는 패킷/데이터 📦를 더 빠르게 보내기 위해 IP 위에 UDP(User Datagram Protocol)도 사용합니다. (많은 확인 작업을 수행하고 패킷이 너무 크면 속도를 저해하는 TCP를 사용하는 이더리움과 달리). UDP는 패킷 📦이 도착한다는 보장 없이 가능한 한 빨리 발사합니다. 하지만 🍑 패킷이 크면(MTU보다 크면) 조각화(분할)될 수 있으며, 이 경우 일부 데이터를 잃어 신뢰성이 떨어질 수 있습니다.

<img
 width="1000px"
 height="550px"
 src="./images/trx_size_udp.png"
/>

자, 그런데 왜 이런 것들을 배우고 있을까요? 트랜잭션 크기 제한과 무슨 관련이 있을까요?
잠깐만요, 이제 곧 결론에 도달합니다. 점들을 연결해 봅시다 😌.

Solana 노드는 다른 네트워크 컴퓨터와 마찬가지로 인터넷 프로토콜(IP), 주로 최신 버전인 IPv6를 사용하여 통신합니다. 이들은 IP의 주소 지정 및 라우팅 규칙을 따라야 합니다. Solana 노드는 빠르고 효율적인 UDP 전송 프로토콜을 사용하여 직렬화된 트랜잭션을 서로에게 보냅니다.

그러나 트랜잭션(UDP 패킷)이 네트워크의 최대 전송 단위(MTU)보다 크면, UDP 자체가 아니라 IP 계층에 의해 조각화(분할)됩니다. UDP는 단순히 패킷을 보낼 뿐 조각화를 처리하지 않습니다. 만약 전송 중에 어떤 조각이라도 손실되면, UDP는 에러 수정이나 재전송을 제공하지 않으므로 전체 트랜잭션이 손실됩니다.

따라서 이러한 조각화는 Solana가 처리합니다. 조각화를 피하기 위해 패킷/트랜잭션 크기는 일반적으로 **1280 바이트**인 MTU(Maximum Transmission Unit)보다 작아야 합니다. 헤더(IP 및 UDP 헤더 = 48 바이트)를 제거한 후, 남은 1,232 바이트가 트랜잭션 크기에 할당됩니다.

### Solana의 새로운 업데이트: 트랜잭션 크기 제한을 4000 바이트로 증가

Solana가 QUIC(UDP처럼 엄격한 MTU 제약이 없으며, TCP의 조각화 처리와 UDP의 미친듯한 속도를 섞어놓은 듯한 현대적인 전송 프로토콜)과 같은 최신 전송 프로토콜을 채택함에 따라, 더 큰 메시지를 보다 안정적으로 전달할 수 있게 되었습니다.

트랜잭션 크기가 1,232 바이트로 제한되었던 이전 버전에서는 ZK 및 DeFi 애플리케이션을 위한 복잡한 트랜잭션을 생성하기가 어려웠습니다. 클라이언트 측에서 계정 수를 늘리면서(각 계정의 공개 키는 32 바이트) 모든 공개 키를 나열하면 트랜잭션 크기 제한에 금방 도달할 수 있었습니다.

이러한 제한 때문에 개발자들은 모든 계정과 사용자 서명을 작은 1,232 바이트 공간에 우겨넣는 최적화된 코드를 작성해야 했습니다.

<img
 width="1000px"
 height="550px"
 src="./images/meme_trx_size_limit.png"
/>

새로운 [SIMD-296 제안](https://github.com/solana-foundation/solana-improvement-documents/pull/296/commits/bbc29c909085589989ca5f258550ce4447e68a89)에 따라 트랜잭션 크기 제한이 **4k 바이트**로 증가했습니다. 이 변경으로 더 많은 명령어, 더 큰 데이터 페이로드, 그리고 특히 다중 상호작용이나 풍부한 메타데이터가 필요한 복잡한 dApp의 원활한 실행이 가능해졌습니다.

## 3. 스택 크기 제약 (Stack Size Limitations)

1. 일반 프로그래밍 언어의 메모리 할당에 대해 설명합니다.
2. 예제를 작성하고 이와 관련된 모든 것(예: 프레임, 스택, 힙, 데이터/전역 영구 데이터)을 설명합니다.
3. Solana가 메모리를 관리하는 방법을 설명합니다.
4. Solana BPF 가상 머신과 연결하여 설명합니다.
5. 스택 크기 제약을 어떻게 최적화할 수 있는지 설명합니다.

Solana 프로그램(스마트 컨트랙트)에는 스택 크기(지역 변수 및 함수 호출에 할당된 메모리 양)에 제한이 있습니다. 현재 스택 프레임 크기 제한은 **프레임당 4KB**입니다.

### 스택 메모리란 무엇인가요?

스택 메모리는 다음 용도로 사용됩니다:

- 함수의 지역 변수
- 함수 매개변수
- 반환 주소
- 임시 데이터

### 스택 크기 제약 조건

```rust
    // 이 함수 / 스택 프레임은 4KB 크기를 초과합니다
    pub fn handle(_ctx: Context<SendAmount>) -> Result<()> {
        // 스택에 1MB 버퍼 할당

        let mut buffer = [0u8; 50000]; // 50,000 bytes (각 u8은 1 byte이므로 50,000 * 1 = 50,000 bytes)

       // 버퍼를 어떤 패턴으로 채움
        for i in 0..buffer.len() {
            buffer[i] = (i % 256) as u8; // 0..255 반복으로 채움
        }

        // 모든 바이트의 합 구하기 (그냥 재미로)
        let sum: u64 = buffer.iter().map(|&b| b as u64).sum();
        println!("Sum of all bytes: {}", sum);

        Ok(())
    }
```

> ⚠️ **경고 예제:**
>
> 스택에 큰 버퍼나 배열을 할당하면 다음과 같은 에러를 만날 수 있습니다:
>
> ```
> Error: Function _ZN16cu_optimizations16cu_optimizations6handle17h60f6d64a7e5552edE Stack offset of 1048576 exceeded max offset of 4096 by 1044480 bytes, please minimize large stack variables. Estimated function frame size: 1048576 bytes. Exceeding the maximum stack offset may cause undefined behavior during execution.
> ```
>
> 이 경고는 함수의 스택 프레임이 Solana의 4KB 스택 제한을 초과했음을 나타냅니다. 이를 해결하려면 스택에 큰 배열이나 버퍼를 할당하는 것을 피하고, 힙 할당을 사용하거나 코드를 리팩토링하여 스택 사용량을 줄이세요.

```
error Access violation in program section at address 0x1fff02ff8 of size 8
```

### Solana BPF 메모리 영역

| **영역** | **시작 주소** | **목적** | **접근 규칙** |
| --- | --- | --- | --- |
| Program Code | 0x100000000 | 실행 가능한 바이트코드 (SBF 명령어) | 읽기 전용, 실행 전용 |
| Stack Data | 0x200000000 | 함수 호출 프레임 및 지역 변수 | 읽기/쓰기 (스택 프레임당 4KB) |
| Heap Data | 0x300000000 | 동적 메모리 할당 (기본 32KB) | 범프 할당자 (해제 불가) |
| Input Parameters | 0x400000000 | 직렬화된 명령어 데이터 및 계정 메타데이터 | 실행 중 읽기 전용 |

스택 제약을 우회하려면:

1. 필요할 때 힙 할당을 사용하세요.
2. 큰 함수를 작은 함수들로 나누세요.
3. 스택에 할당되는 큰 배열을 피하세요.
4. 큰 데이터 구조를 이동(move)하는 대신 참조(reference)를 사용하세요.

[solana_optimization_github](https://github.com/solana-developers/cu_optimizations) <br/>
[rare_skill_blog_post](https://www.rareskills.io/post/solana-compute-unit-price) <br/>
[solana github discussion SIMD-0296](https://github.com/solana-foundation/solana-improvement-documents/pull/296/commits/bbc29c909085589989ca5f258550ce4447e68a89)<br/>
[frank_castle tweet on solana transaction size limit](https://x.com/0xcastle_chain)<br/>
[프로그래밍의 메모리 관리를 이해하기 위한 훌륭한 영상](https://youtu.be/vc79sJ9VOqk?si=hTqpylYjBO88hvJ9)

<img
 width="1000px"
 height="430px"
 src="./images/solana_limitation_bye.gif"
/>
