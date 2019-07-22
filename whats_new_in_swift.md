> What's New in Swift

## What's New in Swift

### Binary Frameworks

#### ABI and Module Stability

- ABI란? Application Binary Interface
- specifies detils of a program's representation at runtime
	- 런타임 시 프로그램의 repsentation의 세부사항을 구체화하는 것
- 여기서 말하는 detail은 함수호출(function call) 같은거다.
- ABI 호환(Compatible ABI)은 separately compiled code to interact at runtime.
- What is the available metaedata?
	- 프로그램이 실행될 때 excuatable한 프로그램은 framework(Swift로 작성된 프레임워크)를 사용한다. 
	- the executable(실행파일) is using APIs from the framework and they have to be able to talk to each other at runtime.

- ABI Stability 전에는 서로 같은 컴파일러에서 컴파일을 했어야했는데(Swift 5 이전에), ABI Stability는 서로 다른 독립적인 컴파일러에서 컴파일해서 사용할 수 있다.
	- 어떻게 이게 가능하냐? 스위프트의 core fundamentlas을 evolving (진화되게) 때문에 이런게 되는거다.. 결국 자기들이 잘 만들어서 이런게 가능하다.
	- make sure all the building blocks that we wanted to have in place to build upon in the future were in the right place.
	- 이게 가져다 주는 의미 중 하나는 더 이상 같은 컴파일러에서 빌드할 필요가 없다는 얘기다. Swift5 이상버전에선.
	
- 두번째로 중요한 것은 Module Stability
	- 스위프트 라이브러리와 API는 컴파일러에 의해 만들어진 "Moules"이라고 불리는 파일들을 추출한다(export). (Swift libraries and the APIs they export are known as "
	modules" Module files are created (and used) by the compiler)
	- this is a compiled time concept.
	- shared namespace called module.
	- Swift Moudle file.
	- framework를 빌드하기 위해서 Swift Compiler가 사용되고, 결과로 모든 APIs의 manifest를 생성한다. a manifest of all APIs in that framework that can be then consumed by clients of that framework.
		- 이 manifest가 바로 `Swift module file`

- 예를 들어서 과정을 살펴보자.
	- 스위프트 소스파일이 framework를 참조한다. (reference), Foo.swift에 import Framework하면 이 프레임웤를 참조하는거.
	- Foo.swift -> import MyFramework -> .swiftModule
- SwiftMoudle을 한번 더 추상화시켜서 Swift Moduel Interface를 만들었다. 
- Stable interface를 제공하기 위해
- Module + ABI stability = Binary frameworks
	- Swift Frameworks = can be deployed and shared with others
	
- [ABI Stability and More](https://swift.org/blog/abi-stability-and-more/)

- Binary framework는 API Story에 있어서 중요한 부분이다.
- Swift package
- integration of the swift package manager

- later swift 5.0, shared swift runtime
- 결국 OS에 내장되어있는걸 쓰기 때문에 기존처럼 다 import할 필요 없다. 런타임때 되는게 좋다
- OS의 최적화도 좋은 점.
- Launch time overhaed가 주는 것도 이점.
- bridging 속도도 빨라졌다.
- NSString에서 String으로 브릿징하는게 15배 빨라졌다니깐 엄청 좋아보인다.
- SwiftNIO는 크로스플랫폼 (비동기 event-driven network application framework)... 하지만 서버에 아직 관심없으로 패스
- SourceKit
- sourcekitd stress tester
- reproducible
- Layered service provider

#### Language and Standard Library (in Swift5, Swift5.1)

- 이번년도에 몇개 더 생겼다.
	- Property wrappers
	- key path member lookup
	- dynamically "callable" types
	- Library evolution for stable ABIs
	- String interpolation improvements
	- Opaque result types
	
- Swift Evloution
	- Implicit return from single expressions: return을 생략할 수 있게됐음.
	- Synthesized default values for the memberwise initializer: 이니셜라이저에 모든인자를 반드시 다 줘야했다. 하지만 Swift5부터는 멤버와이즈의 일부만 초기값을 줄 수 있다.(멤버와이즈를 사용하고 프로퍼티가 많을 경우 유용하게 사용할 수 있을거같음.)
	- Single Insturection Mutiple Data (low level performance에 민감한 프로그래밍에 쓰임), 그래픽이라든지 이미지를 다루는 작업이라든지 AR과 같은.. SIMD타입이라는게 있음.벡터를 다룸 : 표현을 좀더 편하게 할 수 있다. ex) String(format)을 쓸 필요 없이 interporlation으로 해결할 수 있다. SwiftUI에 유용..왠지 SwiftUI때문에 나온느낌이 강하다. String으로 안다루고 Text라는 데이터로 다루기 위해서 그래서 Multiple Data라는걸했다
		- 내가 느끼기엔 뭔가 다 데이터로 다루기 위함. Text에선 String이 아니라 LocalizedStringKey 타입을 이용.
		- 내부적으로 어떻게 동작하는지 보여줬음
	- New Design for String Interpolation

- Reason to Abstract a Return Type
	- API는 같은 프로토콜을 따르는 따른 별개의 타입을 반환한다.
	- API는 같은 타입을 반환하지만 구현의 세세한 것들이 빠져있다? (API returns the same type but is leaking implementation details)
	- 이건 예제보는게 더 이해가 잘되는듯 하다.
	- 제네릭타입의 프로토콜로 묶게되면 결국 프로토콜을 채택한 타입의 detail할 다른 인터페이스들을 볼 수 없어져서 굉장히 불편해진다.
	- 프로토콜 타입을 리턴하는 것에 대해 제한이 걸림
	- 제네릭시스템과 결합해서 쓰면안좋음 왜?
		- `==`로 적용할 수 없음.
		- 리턴타입이 어떤 associated type도 갖을 수 없음
		- 리턴타입이 `self`를 포함한 요구사항을 가질 수 없음
		- Disables optimizations
		- 아래와 같이 쓸 수 있음 (use opaque result type)

		```swift
		struct EightPointStart {
			// ...
			
			var shape: some Shape {
				return Union(Square(), Transformed(Square(), by: .fortyFiveDegrees)
			}
		}
		```
		
		```swift
		struct EightPointedStar {		//...			var shape: some Shape {				if symmetrical {					return Union(Square(), Transformed(Square(), by: .fortyFiveDegrees))				} else {					return Transformed(Square(), by: .twentyDegrees)				}
			}		}
		```
		
	- backward deploying이 뭘까요
	- 이전버전에서 사용하려면 static availbility check로 나눠서 사용을 체크해야한다. 
	
#### Property Wrapper Types

- 이것도 예제가 좋음, UserDefault에서 코드 중복이 생길경우 처리하는 예제

#### Domain이란 무엇일까용? (DSL)

- 나눠보면 좋을듯 : syntax와 semantic에 대해 정의를 내려볼까요?
- Define Embedded DSLs in Swift하는거 좀 좋아보임 > 웹영역의 어쩌구저쩌구 할 때 좋을거같음. 크롤같은거 만들때도 좋을거 같음...
- HTML 코드 인상적
- 느낀점 : 점점 플랫폼의 장벽이 허물어져 가고있구나
- 애플에서 하고있는 비젼은 : HTML 코드안에다가 심지어 Swift의 조건문 같은걸 직접 넣는걸 생각하고있다.

### 빌더패턴 써보신분? 
	- 빌더패턴은 언제 어떻게 쓰는건가요