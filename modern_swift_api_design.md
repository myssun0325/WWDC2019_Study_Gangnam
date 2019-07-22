> Modern Swift API Design

- 앞서 binary 와 module stability를 소개했었다.

- 2016년에 Swift API Design Guidle line 발표
- 지향점은 결국 Clarity at the point of use.
	- 사용하는 측면에서의 명료함
	- 적용하기: 회사에서 코드를 짜면서 메서드를 네이밍하거나 할때 구현하는 객체의 입장에서 설계를 할 것인지 사용처에서의 편의성을 위해 설계할 때 항상 고민하는데 결국 사용처에서의 명료함이 더 중요한듯.
	- 뷰모델 짤때 참고하자.
- 무엇을 하는지가 중요 (역할)
- No Prefixes in Swift-only freameworks
 - 스위프트 모듈 시스템은 disamgiuation을 허용한다.(by prepending the module name in front of the type.)
 - 이런 이유 때문에 standard library는 절때 prefix를 가지지 않는다.
 
 
#### Values and references and protocols and generics.

- value 타입을 사용할 때 장점은 위에서 설명한것과 같은 방향이다.
	- 결국 어디서 왔건 뭘했건 앞내용은 중요치않다. 결국 사용시점의 clarity의 이득을 취할 수 있다.
	 

#### 레퍼런스와 값 타입 중 어떤걸 사용해야할까? 

- class 보단 struct를 사용해라.
	- 참조가 중요한 경우에만 사용해라
- class는 리소스에 대한 레퍼런스 카운팅과 deinitialization이 필요한 경우에 사용해라.
- where there is a separate notion of 'identity' from 'equality'
- 사용처의 오기까지 여러과정이 있다면(참조로인한 Side effect) 이를 방지하기 위해 값타입을 사용하는것이 좋다.
- use case별로 나누는건 중요한듯..(moya가 떠오른다.)

#### Key Distinction between Value and References Types

- Value and Reference Semantics
	- how the type behaves.
- 만약에 mutable reference type을 public API의 한 부분으로 포함하고 있는 것은 자연스러운 일이다.
- 첫번째 질문 : 만약에 값 타입 처럼 행동하길 원한다면?
- final class 활용들 하시나요?
- mutable subclass problem이 뭘까요?
- isKnownUniquelyReferenced 쓰는 부분 > copy on write 기법

#### Protocol and Generics

- 대부분 스위프트에선 서로 다른 타입간의 코드 공유가 필요할 경우 class 상속이 아니라 프로토콜로 시작하는데 이게 반드시 프로토콜로 무조건 하라는 뜻은 아니다.
- In Swift API Design
	- 먼저 구체적인 타입의 유즈 케이스를 파악해라. (first explore the use case with concrete types)
	- 여러 함수들을 서로 다른 타입에서 복붙하고 있을 때 어떤 부분에서 공통으로 사용할 수 있는 코드가 있는지 파악해라. (Understand what code it is that you want to share when you find yourself repeating multiple functions on different types.)
	- 그 다음 제네릭을 사용해서 공통된 코드를 뽑아내고 분석해라.
	- And then factor that shared code out using generics. Now, that might mean creating new protocols. 이 뜻이 무엇일까요? 제가 이해하기론 코드를 그렇게 하고나서 새로운 프로토콜을 만드는 것을 의미할지도 모른다?
	- 이미 존재하는 프로토콜에서 필요한 것을 구성하는 것을 고려해라.(first consider composing what you need out of existing protocols.) -> Try to compose solutions from existing protocols first 이말인듯.
	- 프로토콜을 생성하는 대안으로 제네릭 타입을 만드는 것을 고려해라.
	- `protocol GeometricVector: SIMD` is that right?~
		- is `GeometricVector` a `SIMD` Typ이라고 말할 수 있을까..? 맞는 말이긴할 수 있는데 예를 들면 두 GeometricVector를 multiply하거나 add the number 1 to a vector 같은걸 할 수 없다. 
		- is-a relationship, has-a relationship
		- 아래와 같이 제네릭을 활용해서 만들 수 있다.
		
		```swift
			struct GeometricVector<Storage: SIMD> where Storage.Scalar: FloatingPoint {				typealias Scalar = Storage.Scalar				var value: Storage
				init(_ value: Storage) { self.value = value }
			}
		```

- 발표자 코드 정말 어렵고 우아하게 쓴다..부럽


#### Key Path Member Lookup

- write a single scbscript operation that exposes multiple different computed properties on a type all in one go.
- `@dynamicMemberLookup`
- Swift5에서와의  다른점은 `type safe`, a lot more of it is done at compile time.
- 두번째 예제에서 Material 에서 texture의 모든 프로퍼티를 노출하고 싶다면? dynamic member lookup을 쓰면된다.

#### Property Wrappers

- 목적 : effectively get code reuse out of the computed properties you write.
- 질문 : out of ~~어쩌구는 그거를 활용해서라는 뜻인가
- `lazy`의 사용용도
- @LateInitialized
- Provides similar benefits to the built-in `lazy`
	- Elminiates boilerplate
	- Documents semantics at the point of definition
- 내부적으로 어떻게 동작하는지 보여줬음.
- DefensiveCopying Property Wrapper
- 이 부분은 아예 예제를 뜯어보고 직접 실습해보면서 익혀야 들어올듯하다.