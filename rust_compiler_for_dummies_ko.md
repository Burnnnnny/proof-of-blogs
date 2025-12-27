<img 
  width="1500px"
 height="460px"
 src="./images/header_rust_compiler2.jpg"
/>

# 🍊 초보자를 위한 Rust 컴파일러 (Rust Compiler For Dummies)

## 배경 설정 (Setting Up the Stage):

이미 아시겠지만, 작성된 모든 프로그램은 결국 이진 명령어(그 화려한 0과 1들)로 변환됩니다. 왜냐하면 CPU는 우리의 소스 코드를 이해할 수 없고, 오직 0과 1만 이해하기 때문입니다. 우리가 0과 1을 작성하여 메모리와 상호 작용하거나 프로그램을 작성할 수는 없으므로, 사람이 읽을 수 있는 방식으로 프로그램을 작성하기 위해 몇 가지 추상화를 만들어야 했습니다.

메모리와 상호 작용하는 코드를 작성하기 위한 가장 얇은 추상화 계층은 **어셈블리(Assembly)**입니다. 이것은 우리가 사람이 읽을 수 있는 코드를 작성하도록 돕고, 메모리와 직접 작동하며, 컴퓨터 연산에 대한 세밀한 제어를 제공하는 저수준 언어입니다.
어셈블리는 높은 효율성과 속도를 제공하지만, 메모리 문제에 더 취약하고 코드 작성이 복잡하기 때문에 일반적으로 사용되지 않습니다. 그렇다면 어떻게 메모리와 안전하게 상호 작용할 수 있을까요?

두 번째 추상화 레벨은 우리에게 **고수준 프로그래밍 언어**(Rust, C, Go 등)를 제공합니다. 이러한 언어들은 어셈블리 언어보다 사람이 읽기 쉬우며, 프로그래머로부터 복잡성과 메모리 관리를 숨기는 문법적 설탕(syntactic sugar)을 특징으로 합니다. 이를 통해 프로그래머는 더 적은 복잡성과 더 적은 오류로 프로그램을 작성할 수 있습니다. 언어마다 메모리 할당 및 해제 방식이 다르며, 어떤 언어는 효율적으로 관리하지만 다른 언어는 그렇지 않을 수 있고, 각각의 장단점이 있습니다. 하지만 잠깐만요... 제가 방금 CPU는 0과 1(이진수) 이외에는 아무것도 이해할 수 없다고 하지 않았나요?

우리는 고수준 프로그래밍 언어 프로그램을 CPU에서 직접 실행할 수 없으므로, 이러한 문법적 설탕이 가미된 프로그램을 이진 코드로 변환해야 합니다. 이 변환은 **컴파일러(compilers)**에 의해 수행됩니다. 우리는 Rust 코드가 어떻게 이진 코드로 변환되는지, 그리고 컴파일 중에 어떤 내부 검사가 발생하는지에 초점을 맞출 것입니다. 우리는 한 겹 한 겹 벗겨내며 모든 단계를 이해해 볼 것입니다. 이 점을 염두에 두고 시작해 봅시다 😄.

## 컴파일 계층의 고수준 개요

개발자가 작성한 코드는 사람이 읽을 수 있어 다른 사람들이 쉽게 읽고 이해할 수 있습니다. 그러나 컴파일러는 사람이 작성한 코드를 이해할 수 없습니다. 우리는 소스 코드를 CPU가 이해할 수 있는 유일한 형식인 이진 코드로 변환해야 합니다.

이진 코드를 생성하는 데는 여러 단계가 필요하지만, 높은 수준에서 보면 이러한 단계는 **프론트엔드(Frontend), 미들(Middle), 백엔드(Backend)**라는 세 가지 주요 기둥으로 나뉩니다. 이것은 소스 코드를 기계어 코드로 변환하는 복잡한 과정을 세분화합니다.

<img 
  width="1500px"
  height="560px" 
  src = "./images/compiler_three_pillers.png" 
/>
<br/>

프론트엔드에는 Rust 코드가 있습니다. 백엔드에는 타겟 머신에서 직접 실행되는, **LLVM (Low Level Virtual Machine)**에 의해 생성된 이진 기계어 코드가 있습니다. 그 중간에서는 Rust의 모든 소유권(ownership) 및 빌림(borrowing) 검사가 발생합니다.

우리는 각 계층을 벗겨내며 Rust 컴파일이 어떻게 작동하는지 이해할 것입니다. Rust 컴파일의 세 가지 기둥을 조금 더 확대해 보면, 아래 이미지와 같은 모습을 볼 수 있습니다.

<img 
  width="1500px"
  height="560px" 
  src = "./images/compiler_all_stages.png" 
/>
<br/>

위 그림에 나오는 모든 용어를 알 필요는 없지만, 걱정하지 마세요. 이 블로그가 끝날 때쯤이면 모두 이해하게 될 것입니다. 코드를 컴파일하는 동안 무슨 일이 일어나는지 이해하기 위해 각 계층을 벗겨내며 단계별로 살펴보겠습니다.

이 큰 그림을 염두에 두고, 오렌지 🍊 껍질을 벗겨 봅시다.

### 1단계: 렉싱, 파싱, 그리고 AST (Lexing, Parsing, and AST)

소스 코드 예제를 들어보겠습니다:

```rust
fn main() {
    // 조사 좀 해보자 :)
    let some = String::from("chinna");
    println!("Say my name:");
    println!("{}", some);

    time_pass(&some);
}

fn time_pass(pass: &String) {
    println!("Time passing with this guy: {}", pass);
}
```

이 단계는 컴파일이 시작되는 첫 번째 단계입니다. 컴파일러는 먼저 `.rs` 파일을 일반 텍스트로 읽은 다음, 이 선형 텍스트를 `fn`, `some`, `{`와 같은 **토큰(Tokens)**으로 분해합니다. 이것을 **렉싱(Lexing)**이라고 합니다.

그런 다음 컴파일러는 이 토큰들을 AST(Abstract Syntax Tree, 추상 구문 트리)라고 하는 트리 구조로 변환합니다. 이 AST는 여전히 소스 코드와 매우 유사하지만 트리 구조로 되어 있습니다. 이것을 **파싱(Parsing)**이라고 합니다. 우리가 가져온 코드 예제의 AST 버전은 [여기](https://github.com/baindlapranayraj/rektoff/blob/main/rektoff-office-hour/AST.txt)에서 볼 수 있습니다. 이 단계에서 모든 매크로가 확장됩니다.

<img src="./images/first_step.png" />

AST는 모든 구문 코드를 트리와 같은 구조로 캡처합니다. 왜 이렇게 해야 하는지 물을 수 있습니다.

글쎄요, 컴파일러는 이 선형 소스 코드를 직접 이해할 수 없습니다. 소스 코드는 컴파일러가 아닌 사람이 읽기 쉽도록 설계된 문법으로 포장되어 있습니다. AST(Abstract Syntax Tree)는 특정 세부 사항을 추상화하며, 소스 코드의 구문 구조를 가장 잘 나타내는 트리 데이터 구조입니다. AST에 대한 자세한 내용은 이 [위키피디아 페이지](https://en.wikipedia.org/wiki/Abstract_syntax_tree)에서 확인할 수 있습니다.

### 2단계: AST 낮추기 (HIR 및 THIR)

토큰을 파싱하고 AST로 변환한 후, 다음 계층이 시작됩니다. 이 시점에서 AST는 소스 코드와 매우 유사하며, `for`나 `match`와 같은 많은 문법적 설탕(syntactic sugar)을 여전히 포함하고 있습니다.

우리는 AST를 단순화하기 위해 이 문법적 설탕을 벗겨내야 합니다. 이 "desugaring(설탕 제거)" 과정의 결과는 **HIR (High-Level Intermediate Representation, 고수준 중간 표현)**이라고 알려진 AST의 형태입니다. HIR은 여전히 사용자가 원래 작성한 것과 가깝지만, 문법적 설탕을 제거합니다. 예를 들어, `for` 루프를 반복 로직이 있는 `loop`로 변환합니다. 모든 불필요한 부분을 제거한 후, HIR은 이제 AST보다 더 컴파일러 친화적인 추상화 표현이 됩니다.

문법적 설탕을 제거하여 AST를 단순화하거나 변환하는 이 과정을 **낮추기(lowering)**라고 하며, 그건 그렇고 다음 명령어를 실행하여 AST의 HIR 표현을 확인할 수 있습니다: `rustc +nightly -Z unpretty=hir-tree src/main.rs`.

HIR을 더 낮추고 코드의 모든 타입이 올바르게 사용되었는지 확인합니다. 예를 들어 정수와 문자열을 더할 수 없습니다. 물론 **_JavaScript_**에서는 이런 짓을 할 수 있죠 🤡.

<div align="center">
 <img  src="./images/javascript_meme.jpg"/>
</div>

모든 타입 검사를 수행한 후 이제 **THIR(Typed High-Level Intermediate Representation, 타입화된 고수준 중간 표현)**이 AST 형태로 표현됩니다. 이름에서 알 수 있듯이, THIR은 타입 검사가 완료된 후 가능한 모든 타입이 채워진 HIR의 낮춰진(lowered) 버전입니다.

HIR을 THIR로 낮춘 후 그리고 MIR 이전에, unsafety 검사기(unsafety checker)는 THIR 표현을 훑어보며 모든 표현식에서 안전하지 않은 작업(raw 포인터 역참조, unsafe 함수 호출, static mut 접근, union 필드 접근 등)을 확인하고 이것들이 오직 unsafe 컨텍스트(unsafe 블록이나 함수) 내부에서만 나타나는지 검증합니다.

`unsafeck`는 타입 표현식이 필요하기 때문에, 타입 검사와 THIR 구성 후, 하지만 MIR 빌드 이전에 배치됩니다. 이러한 위치 선정은 Rust의 안전성 보장을 조기에 정확하게 시행할 수 있게 하여, 모든 unsafe 코드가 명시적인 unsafe 컨텍스트 안에 적절히 포함되도록 보장합니다.

### 3단계: 중간 중간 표현 (MIR - Middle Intermediate Representation)

낮추기(lowering) 후, HIR은 AST의 컴파일러 친화적인 추상화가 됩니다. 여기서부터 다음 추상화 계층이 시작되며, 이것이 Rust 컴파일러의 심장부입니다. MIR (Middle Intermediate Representation)은 많은 고전적인 메모리 버그(경쟁 상태 및 use-after-free 오류 등)를 감지할 수 있는 단계입니다. 이러한 버그가 발견되면 컴파일러는 단순히 에러를 발생시킵니다.

MIR은 여러분의 코드를 **제어 흐름 그래프(CFG, Control Flow Graph)**로 나타냅니다. 이것을 상세한 순서도로 생각하세요. 모든 `if`, `loop`, `match`는 기본 블록과 그들 사이의 명시적인 "go-to" 점프들로 분해됩니다.

안전성을 보장하기 위해 컴파일러는 단순히 "행복한 경로(happy path)"만 확인할 수 없습니다. 코드가 취할 수 있는 모든 가능한 경로를 분석해야 합니다. 이 `if`가 참이면 어떻게 될까요? 거짓이면요? 이 루프가 0번 실행되면요?

CFG는 이 모든 경로를 명시적으로 만듭니다. 빌림 검사기(borrow checker)는 이 순서도를 체계적으로 따라가며 모든 변수의 상태(소유됨, 빌려짐, 이동됨)를 모든 가능한 분기와 루프를 통해 추적하여 어떤 규칙도 위반되지 않도록 합니다. [여기](https://en.wikipedia.org/wiki/Control-flow_graph)에서 CFG에 대해 더 알아볼 수 있으며, 우리 코드 예제의 MIR 버전을 보려면 [여기](https://github.com/baindlapranayraj/rektoff/blob/main/rektoff-office-hour/MIR.txt)를 확인할 수 있습니다.

<div align="center">
 <img width=800 height=500 src="./images/CFG.png"/>
</div>

요약하자면, MIR은 컴파일러가 모든 가능한 경로를 철저히 확인하고 Rust의 안전성 불변 조건이 충족됨을 보장할 수 있는 구조를 제공합니다. 이 시스템은 매우 견고하며 Rust 안전성의 기초입니다.

`unsafe` 키워드는 중요한 예외를 만듭니다. 이것은 프로그래머가 Rust의 안전 규칙을 수동으로 지키겠다는 약속입니다. 그 약속이 깨지면 컴파일러의 기본 가정이 무효화됩니다. 이는 주변 코드가 아무리 "안전해" 보일지라도 전체 프로그램의 동작을 위태롭게 할 수 있습니다.

궁극적으로, Rust 프로그램은 정의되지 않은 동작(undefined behavior)이 없을 때 **건전(sound)**하다고 간주됩니다. 컴파일러의 검사를 통과한 안전한(safe) 코드는 자동으로 이러한 건전성을 달성합니다.

### 4단계: LLVM 코드 생성

우리는 컴파일의 마지막 단계에 접어들고 있습니다. MIR 단계에서 모든 Rust 빌림 및 소유권 검사를 수행한 후, 컴파일러는 죽은 코드(dead code) 제거나 제어 흐름 단순화와 같은 최적화를 MIR 레벨에서 적용합니다.

그런 다음 MIR은 LLVM 백엔드에서 사용하는 플랫폼 독립적인 중간 표현인 LLVM IR (Low Level Virtual Machine Intermediate Representation)로 변환됩니다. **LLVM IR은 어셈블리와 비슷하지만**, 조금 더 고수준이며 사람이 읽기 쉽습니다.

Rust 컴파일러의 코드 생성 단계는 주로 **LLVM (Low Level Virtual Machine)**에 의해 수행됩니다. LLVM은 컴파일러를 구축하기 위한 도구 모음으로, C/C++ 컴파일러 Clang에서 가장 잘 알려져 있습니다. 마지막으로, 모든 엄격한 최적화 후, LLVM IR은 타겟 플랫폼(예: x86_64 또는 ARM64)을 위한 기계어 코드로 변환됩니다. 결과적으로, 하나의 아키텍처를 위해 컴파일된 바이너리는 에뮬레이션이나 변환 없이는 다른 아키텍처에서 실행될 수 없습니다. 즉, x86_64 바이너리 코드는 ARM64 프로세서에서 실행할 수 없으며 그 반대도 마찬가지입니다.

<div align="center">
 <img width=800 height=500 src="./images/llvm.png"/>
</div>

## 참고 자료 (References):

[daniela의 rust_compiler_video](https://youtu.be/Ju7v6vgfEt8?si=-ElYGBaXZvUwt98m)<br/>
[stack_overflow_discussion](https://stackoverflow.com/questions/43385142/how-is-rust-compiled-to-machine-code)<br/>
[llvm_video](https://youtu.be/3WojCM9r0Ls?si=kfTtzQ-BrsPvs-5q)<br/>
[llvm에_대한_stack_overflow_discussion](https://stackoverflow.com/questions/2354725/what-exactly-is-llvm)
[정의되지_않은_동작에_대한_rust_book](https://google.github.io/learn_unsafe_rust/undefined_behavior.html#unsoundness)

<img 
  width="1500px"
 height="460px"
 src="./images/rust_compiler_footer.png"
/>
