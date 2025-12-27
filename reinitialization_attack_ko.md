<img
 width="1000px"
 height="500px"
 src="./images/reintialization_attack.jpg"
/>

# 🌙 재초기화 공격 (Reinitialization Attack)

프로그램 예제 (참고로 저는 Neovim을 사용합니다 🙂): [github link](https://github.com/baindlapranayraj/reinitialization-attack-demo)  
재초기화 공격(Reinitialization attacks)이 어떻게 작동하는지 이해하려면, 먼저 Anchor ⚓️의 `init`과 `init_if_needed` 제약 조건의 내부 동작을 이해해야 합니다.

## **`init`:**

간단히 말해, `init`은 주어진 주소(seeds와 bump로 결정됨)에 대해 계정이 **단 한 번만** 생성되고 초기화되도록 보장합니다. `init`은 내부적으로 다음 두 가지 명령어 세트를 수행합니다:

### **시스템 레벨 (System Level):**

- System Program의 `create_account` 명령어를 사용합니다.
- 온체인(on-chain)에 계정을 위한 공간을 할당합니다.
- `Rent::minimum_balance`를 사용하여 렌트(rent) 면제에 필요한 램포트(lamports)를 채워 넣습니다.
- 소유자 프로그램(예: 여러분의 커스텀 프로그램)을 할당합니다.

### **계정 초기화 (Initialization of an Account):**

- 여러분의 프로그램은 `#[account]` 구조체를 사용하여 계정 데이터 구조를 정의합니다.
- **8바이트 지정자(discriminator)를 작성하고 데이터에 첨부합니다 (이것은 타입 체커로 사용됩니다).**
- 아래 PDA 계정 구조체 예제에 나온 대로 초기 데이터를 저장합니다.

```rust
#[account]
#[derive(InitSpace)]
pub struct User {
  pub user_pubkey: Pubkey,
  #[max_len(30)]
  pub user_name: Option<String>,
  pub balance: u64,
  pub user_vault_bump: u8,
  pub user_bump: u8,
}
```

```rust
#[account(
    init,
    payer = USER,
    space = 8 + User::INIT_SPACE,
    seeds = [
              USER,
              user.key().to_bytes().as_ref(),
              chau_config.key().to_bytes().as_ref()
            ],
    bump
)]
pub user_profile: Account<'info, User>,
```

### `init` 제약 조건의 보안성 🛡️:

`init` 제약 조건의 보안은 `init_if_needed`와 달리 엄격합니다. 가짜 PDA를 생성하여 프로그램에 전달할 수 없으며, 동일한 계정을 두 번 초기화할 수도 없습니다. 그렇게 하면 에러가 발생합니다.

**Anchor는 다음 조건들을 평가하여 계정이 초기화되었는지 확인합니다:**

1. 계정에 램포트가 일부 있고 System Program이 소유하고 있다면, Anchor는 이를 초기화되지 않은 계정으로 간주하고 에러를 발생시킵니다.
2. 계정에 램포트가 0이라면, Anchor `init`은 계정을 생성하고 초기화할 수 있습니다 (새로운 계정이기 때문입니다).

**Anchor가 PDA를 확인하는 방법 🔍:**

1. 계정을 직렬화할 때, Anchor는 계정 이름(예: 위 PDA의 경우 `User`)의 SHA256 해시의 처음 8바이트를 사용하여 계정 이름을 저장하는 추가적인 지정자(discriminator, 또는 태그)를 추가합니다.
2. 역직렬화 중에 이 지정자는 타입 체커 역할을 합니다. 악의적인 계정이 전달되면, Anchor는 해당 계정 데이터의 지정자와 해당 계정 타입에 대해 예상되는 지정자를 비교합니다. 일치하지 않으면 Anchor는 계정을 거부하여 악의적인 계정이 사용되는 것을 방지합니다.
3. Solana에서는 모든 데이터가 raw-binary 데이터로 저장되므로, Anchor는 계정이 `User`인지 아닌지 알 수 없습니다. 이 지정자는 라벨/태그 역할을 하여 Anchor가 올바른 구조체로 다시 역직렬화할 수 있게 해줍니다.

### **공격 시나리오 🔪:**

- `User` PDA가 필요한 명령어가 있다고 가정해 봅시다(첫 번째 예제 코드 참조). 공격자가 `User` 주소로 악의적인 PDA를 보내려고 시도합니다.
- 여기서 Anchor는 먼저 `Fake` 계정의 지정자를 확인하고 이를 `User`의 지정자와 비교합니다. 지정자가 일치하지 않으면 Anchor는 계정을 거부하고 에러를 발생시킵니다.

여러분은 Anchor의 진가를 알아주어야 합니다 (그리고 그래야만 합니다). Anchor는 계정의 직렬화 및 역직렬화(그리고 다른 여러 잠재적 위험 요소들)뿐만 아니라, 보이지 않는 곳에서 중요한 보안 검사를 많이 처리해 줍니다.

참고: 여기서 Anchor가 생성한 Account는 AccountInfo(아래 참조)를 감싸는 래퍼(wrapper)이며, 이는 Anchor가 프로그램 소유권을 검증하는 데 도움을 줍니다.

```rust
pub struct AccountInfo<'a> {
     /// Public key of the account
     pub key: &'a Pubkey,
     /// The lamports in the account.  Modifiable by programs.
     pub lamports: Rc<RefCell<&'a mut u64>>,
     /// The data held in this account.  Modifiable by programs.
     pub data: Rc<RefCell<&'a mut [u8]>>,
     /// Program that owns this account
     pub owner: &'a Pubkey,
     /// The epoch at which this account will next owe rent
     pub rent_epoch: u64,
     /// Was the transaction signed by this account's public key?
     pub is_signer: bool,
     /// Is the account writable?
     pub is_writable: bool,
     /// This account's data contains a loaded program (and is now read-only)
     pub executable: bool,
}
```

## `init_if_needed`:

`init`과 달리, `init_if_needed`는 엄격하지 않고 사용하기 유연합니다. `init_if_needed`는 `init`과 유사하지만, 그만큼 엄격하지는 않습니다. `init` 대신 `init_if_needed`를 사용할 때 Anchor는 다음 확인 절차를 따릅니다:

1. 계정이 존재하고 초기화되었는가? 아니라면 생성 및 초기화하고 진행합니다.
2. 계정은 존재하지만 초기화되지 않았는가? 그렇다면 초기화하고 데이터를 리셋한 뒤 진행합니다.
3. 계정이 존재하고 초기화되었는가? 그렇다면 초기화를 건너뛰고 허가 없이 기존 계정을 수정할 수 있게 합니다 (적절한 검증 없이는 위험합니다).

### **공격 시나리오 (재초기화 공격) 🔪:**

> **참고:**
>
> Anchor는 다음과 같을 때 계정을 "**초기화되지 않음(uninitialized)**"으로 간주합니다:
>
> - 8바이트 지정자가 누락되었거나 유효하지 않음 (예상 해시와 일치하지 않음).
> - 계정 소유권이 예상 프로그램과 일치하지 않음.
> - 계정의 램포트가 0임 (사실상 온체인에 존재하지 않음).

### 공격자 흐름 (Attacker Flow):

### **정상적인 코드 흐름:**

일반적인 시나리오에서 `init_if_needed`는 지연 초기화(lazy initialization)가 필요할 수 있는 계정에 사용됩니다. 예를 들어:

```rust
#[account(
  init_if_needed,
  payer = user,
  space = 8 + User::INIT_SPACE,
  seeds = [b"user", user.key().as_ref()],
  bump
)]
pub user_account: Account<'info, User>,
```

여기서 계정은 존재하지 않거나 초기화되지 않은 경우에만 안전하게 초기화됩니다.

### **악의적인 사용자가 이를 악용하는 방법:**

공격자는 다음과 같은 방법으로 `init_if_needed`를 악용할 수 있습니다:

1. 램포트를 모두 빼내거나(잔액을 0으로 설정) 소유권을 변경하여 **계정을 초기화 해제(Uninitializing)**합니다.
2. `init_if_needed`를 사용하는 명령어에 동일한 계정을 전달하여 **재초기화를 강제**함으로써, 프로그램이 계정 상태를 리셋하도록 속입니다.

### **계정 초기화 해제 방법:**

1. **램포트 인출 (Draining Lamports):**
   - 공격자는 계정에서 모든 램포트를 인출하여 Anchor가 보기에 "초기화되지 않은" 상태로 만들 수 있습니다 (램포트가 0인 계정은 존재하지 않는 것으로 취급되므로).
2. **소유권 변경 (Changing Ownership):**
   - 공격자가 계정의 소유자를 다른 프로그램으로 변경하면, Anchor는 이를 원래 프로그램에 대해 초기화되지 않은 것으로 간주합니다.
3. **지정자 손상 (Corrupting the Discriminator):**
   - 공격자가 계정 데이터의 처음 8바이트(지정자)를 수정하면, Anchor는 이를 초기화되지 않은 것으로 취급합니다.

### **공격 후에는 무슨 일이 일어날까요?**

- 계정 데이터가 **기본값으로 리셋**됩니다 (예: balance = 0, user_name = None).
- 공격자는 이를 악용하여 다음을 수행할 수 있습니다:
  - 대출 프로토콜에서 부채를 리셋함.
  - 잠긴 계정에 대한 접근 권한을 다시 얻음.
  - 계정 상태에 의존하는 로직을 악용함.

### **`init_if_needed` 사용 시 보안 수칙 🛡️:**

1. **중요한 상태 계정에는 `init_if_needed` 사용을 피하세요:**
   - 계정이 중요한 데이터(예: 사용자 잔액, 프로토콜 설정)를 저장한다면, 재초기화를 막기 위해 `init`을 사용하세요.
2. **명시적인 확인 절차 추가:**
   - `init_if_needed`를 사용하기 전에 계정이 이미 초기화되었는지 수동으로 확인하세요.

```rust
#[account]
#[derive(InitSpace)]
pub struct User {
   pub user_pubkey: Pubkey,
   #[max_len(30)]
   pub user_name: Option<String>,
   pub balance: u64,
   pub is_initialized: bool,
   pub user_vault_bump: u8,
   pub user_bump: u8,
}
```

```rust
require!(user_account.discriminator.is_empty(), AlreadyInitialized);
```

### **결론:**

`init_if_needed`는 유연성을 제공하지만, 잘못 사용하면 위험을 초래합니다. **보안상 중요한 계정에는 항상 `init`을 선호**하고, `init_if_needed`는 절대적으로 필요한 경우에만 적절한 안전장치와 함께 사용하세요. Anchor의 지정자 확인은 일부 공격을 방지하는 데 도움이 되지만, 개발자는 재초기화 악용으로부터 프로그램을 완벽하게 보호하기 위해 추가적인 확인 절차를 구현해야 합니다.

## 참고 자료 (References):

[solana developer course](https://solana.com/developers/courses/program-security/reinitialization-attacks) <br/>
[rare skill blog](https://www.rareskills.io/post/init-if-needed-anchor ) <br/>
[init vs init_if_needed](https://medium.com/@calc1f4r/init-vs-init-if-needed-a-deep-dive-d33fe59e4de5)  <br/>
[security concern in init_if_needed](https://syedashar1.medium.com/program-security-in-anchor-framework-solana-smart-contract-security-b619e1e4d939  
)<br/>
[stackoverflow_discussion](https://solana.stackexchange.com/questions/4948/what-is-anchor-8-bytes-discriminator) <br/>

## Source Code:

https://github.com/baindlapranayraj/reinitialization-attack-demo
