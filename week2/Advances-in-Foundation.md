> 영상 : [Advances in Foundation](https://developer.apple.com/videos/play/wwdc2019/723)

### Ordered Collection Diffing

------

[[SE-0240 : Ordered Collection Diffing]](https://github.com/apple/swift-evolution/blob/master/proposals/0240-ordered-collection-diffing.md)을 통해 제안된 것이 적용된 것 같다. 

콜렉션 간의 차이를 계산하고 코드로 작성하며. 적용할 수 있는 API다. 이에 대해서 `bear`와 `bird` 두 문자열을 예로 들어 설명하고 있다. 

```swift
let bird: String = "bird"
let bear: String = "bear"

let diff = bird.difference(from: bear)
let newBird = bear.applying(diff) // [b,i,r,d]
```

기존에 하나의 콜렉션에서 기준이 되는 콜렉션으로 변경하려면 인덱스를 사용해 `remove`와 `insert`를 진행해주었어야 했다. 하지만 이제 위의 코드에서 보여지는 것처럼 이러한 diffing 작업을 쉽게 수행할 수 있게 되었다. 

`difference` 메소드는 `String`의 메소드가 아니라 `BidirectionalCollection` 프로토콜 메소드이다. `String`은 이 프로토콜을 따르고 있다.  이 프로토콜의 정의를 문서를 통해 살펴보면 다음과 같이 설명하고 있다. 

```
순방향 및 역방향 순회를 지원하는 콜렉션

양방향 콜렉션은 startIndex를 제외한 어떠 유효한 인덱스에서도 역방향 순회 기능을 제공한다. 그러므로 양방향 콜렉션은 마지막 요소에 효율적으로 접근할 수 있는 last 프로퍼티와 요소를 역방향으로 보여주는 reversed() 메소드와 같은 연산자를 제공한다. 게다가 양방향 콜렉션은 suffix(_:)와 같은 시퀀스, 콜렉션 메소드를 보다 효율적으로 구현할 수 있다.
```

 `difference` 메소드의 최악 시간 복잡도는 `O(n * m)`으로 `n`은 메소드를 호출하는 콜렉션의 길이,  `m`은 인자로 넘어가는 콜렉션의 길이를 의미한다. 그러므로 두 콜렉션이 비슷할 수록 더 좋은 성능을 보인다. 

반환값은 인자로 넘긴 콜렉션에서 메소드를 호출한 콜렉션을 만들기 위해 필요한 차이로 그 타입은 `CollectionDifference` 타입이다. `CollectionDifference` 타입을 다음과 같이 설명하고 있다.

```
A collection of insertions and removals that describe the difference between two ordered collection states.

삽입과 삭제의 콜렉션으로 두 콜렉션의 차이를 표현한다.
```

위의 `diff`를 `forEach`를 통해 출력해보면 그 모습은 다음과 같다. 

```
Swift.ColletionDifference<Swift.String>.Change.insert(offset: 1, element: "I", associatedWith: nil)
Swift.ColletionDifference<Swift.String>.Change.insert(offset: 3, element: "D", associatedWith: nil)
Swift.ColletionDifference<Swift.String>.Change.insert(offset: 1, element: "E", associatedWith: nil)
Swift.ColletionDifference<Swift.String>.Change.insert(offset: 2, element: "R", associatedWith: nil)
```

여기서 `offset`에 주목해보자.

`insert`의 `offset`은 변경 목적의 요소가 변경 대상에 들어가야 할 인덱스를 의미한다. 즉 `bear`를 `bird`로 만드려면 `i`를 `e` 위치에 넣어야 하기 때문에 그 인덱스가 `1`이 된다. 

`remove`의 `offset`은 변경 대상에서 삭제해야 할 요소의 인덱스를 의미한다. `bear` 를 `bird`로 만드려면 `e`를 제거해야 하기 때문에 인덱스가 `1`이 된다. 

그리고 우리는 이렇게 `difference`의 결과물을 사용해 `applying(_:)` 메소드를 호출한다. `applying(_:)` 메소드의 시간 복잡도는 `O(n + c)`로 `n`은 변경 대상의 길이를 의미하고 `c`는 `CollectionDifference`의 길이를 의미한다.

이렇게 우리는 **Collection Diffing**을 통해 콜렉션을 보다 쉽게 조작할 수 있게 되었다. 



### Data

------

**Contiguity**

로컬 디스크에 사진이나 파일을 저장할 때는 연속적인 메모리 공간을 사용한다. 하지만 네트워크를 통해 데이터를 받아올 때는 다수의 데이터 청크가 각기 다른 시간에 도착하기 때문에 메모리에 비연속적으로 공간을 차지하게 된다. 그리고 Swift 5 이전에는 `struct`가 이러한 연속적이고 비연속적인 공간을 모두 표현하고 있었다. 

![DataContiguity](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/DataContiguity.png)

이렇게 서로 다른 메모리 공간 사용을 하나의 인터페이스로 제공함으로써 사용하기엔 쉬웠지만, 비연속적인 공간에 저장된 데이터에 접근하기 위해 데이터가 저장되어 있지 않은 공간을 포함한 버퍼를 사용해야 했다. (공간의 낭비)

그렇기 때문에 성능 또한 예측이 불가능했다. 

>  [This meant that the performance ](https://developer.apple.com/videos/play/wwdc2019-723/?time=173)[may sometimes be unpredictable.](https://developer.apple.com/videos/play/wwdc2019-723/?time=174) [In fact, we have look at ](https://developer.apple.com/videos/play/wwdc2019-723/?time=177)[real-world usage of data, and ](https://developer.apple.com/videos/play/wwdc2019-723/?time=178)[every discontiguous data gets ](https://developer.apple.com/videos/play/wwdc2019-723/?time=182)[flattened ](https://developer.apple.com/videos/play/wwdc2019-723/?time=184)[sometime during its lifecycle.](https://developer.apple.com/videos/play/wwdc2019-723/?time=184) 라고 언급했는데 그중에서 Discontiguous data gets flattened sometime during its lifecycle 이 부분을 명확히 이해하지 못했다. 
>
> 비연속적인 공간이 연속적으로 평평해질 수 있고, 기대하던 위치에 데이터가 없을 수 있는 부작용(?)을 말하는 건가…?

Swift 5부터 `Data`는 데이터의 연속적인 할당을 보장한다. 그리고 이를 표현하기 위해 `ContiguousBytes` 프로토콜을 제공한다. 연속적으로 저장된 공간에 직접 접근하는 타입은 이 프로토콜을 따른다. 

연속성을 보장하지 않는 타입을 위해 `DataProtocol`과 여기에 가변성을 더한 `MutableDataProtocol`이 제공된다. 

![ProtocolConforms](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/ProtocolConforms.png)

위에 나열된 타입들이 위의 프로토콜들을 따르고 있고 `DataProtocol`을 제네릭 타입 제약으로 사용할 수 있다.



**Compression**

데이터를 압축을 한 줄의 코드를 통해 쉽게 수행할 수 있게 되었다. 

```swift
let compressed = try data.compressed(using: BUILT_IN_COMPRESSION_ALGORITHM_ENUMS)
```



**Units**

새로운 데이터 단위가 추가되었다. 

- `UnitDuration`
  - 시간의 양을 표현하는 타입
- `UnitFrequency`
  - 빈도를 표현하는 타입
  - Frequency is a quantity of occurrences for a repeating event over time.



**Displaying Date or Time**

상대적인 시간 표현 방식이 추가. 

`2분전 포스팅`, `결제까지 2일 남음`과 같은 상대적인 시간을 포맷팅할 수 있는 클래스가 추가되었다. 

- [`RelativDateTimeFormatter`](https://developer.apple.com/documentation/foundation/relativedatetimeformatter)

![RelativeDateTimeFormatter](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/RelativeDateTimeFormatter.png)

위와 같이 매우! 편하게 사용할 수 있게 되었다. 사실 이 영상 중에 제일 만족스러운 부분이었다. 😅

이와 유사하게 [`ListFormatter`](https://developer.apple.com/documentation/foundation/listformatter)도 생겼다.

![RelativeDateTimeFormatter](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/RelativeDateTimeFormatter.png)

`ListFormatter`와 `DateFormatter`를 사용해 날짜 표기 나열 방식도 지역에 따라 다르게 표현할 수 있다. 

![ListDateFormatter](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/ListDateFormatter.png)



### OperationQueue 

------

`OperationQueue`에도 새로운 기능이 추가되었는데, 만일 백그라운드에서 동시에 진행 중인 작업들이 모두 종료되고 이후의 상태를 저장하려면 어떻게 할 수 있을까? 또한 상태를 저장하는 작업에는 그 작업만 수행할 수 있도록 다른 작업의 시작을 막으려면 어떻게 해야 할까? 

![BeforeBarrier](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/BeforeBarrier.png)

위와 같이 동작 중인 작업의 개수를 검사하는 것은 옳지 않다. 그래서 새로 등장한 메소드가 `addBarrierBlock` 메소드다. 

![AfterBarrier](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/AfterBarrier.png)

우리는 이 메소드를 통해 특정 작업만을 유일하게 실행할 수 있게 된다. 특정 시간에 특정 작업만을 할 수 있도록 보장한다.



**Progress Reporting**

또한 동시에 진행중인 작업의 진행 상태를 추적할 수 있게 되었다. 

![ProgressReporting](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/ProgressReporting.png)



### Be Prepared for USB and SMB on iOS

------

USB와 SMB 지원에 따라 이를 위한 새 기능들이 추가되었다. 아토믹하고 안전하게 파일을 저장할 수 있도록 하며, USB를 추출하거나 네트워크가 끊어졌을 때를 대비할 수 있는 기능들이 추가되었다. 

로컬 저장소가 아닌 외부 저장소의 데이터에 접근하는 것은 느릴 수 있기 때문에 메인 스레드가 아닌 스레드에서 진행해야 한다. 

![USBSMB](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/USBSMB.png)

### Swift Updates

------

**Scanner**

`Scanner`에 새 API가 추가되었고, 이모지까지 다룰 수 있게 되었다. 

![Scanner](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/Scanner.png)

> 사실 `Scanner`를 사용해보지 않아서 새로 생긴 API가 무엇이 어떻게 왜 좋아진 것인지 와닿지가 않았다. 😅



**Error-based API**

호출 시점에 즉시 에러를 다룰 수 있게 되었다. 또한 데이터를 쓰는 API는 `DataProtocol`과 함께 사용되기 때문에 비연속적인 데이터를 사용할 수 있도록 최적화되어 있다. 

 ![ErrorBasedAPI](https://ehdrjsdlzzzz.github.io/2019/08/04/Advances-in-Foundation/ErrorBasedAPI.png)