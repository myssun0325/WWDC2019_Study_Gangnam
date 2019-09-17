# Advances in Networking, Part 1 - WWDC 2019 712

URLSession과 Network.framework를 통해 네트워크에 대해 설명할 것.

## Low Data Mode

사용자가 데이터의 사용량을 최소화할 수 있는 설정

- 네트워크 데이터의 사용량을 줄이겠다는 명시적인 신호
- Wi-Fi와 셀룰러 네트워크마다 옵션이 존재

시스템 정책적으로는

- Discretionary한 작업은 미뤄짐
- Background App Refresh는 비활성화됨

### App Adoption

애플은 "사용자 경험에 지장이 없다면 항상 네트워크 데이터를 아껴라"라고 권장한다.

실제로 적용할 수 있는 방안은 다음과 같다.

- 이미지 퀄리티 낮추기
- Pre-fetching을 줄이기
- Synchronize 주기 늘리기
- Task들을 discretionary로 지정하기
- Auto-play의 비활성화
- **그러나 사용자가 시작한 작업은 막으면 안 됨!**

#### Low Data Mode APIs

##### URLSession

- 큰 데이터를 받거나 Prefetch를 할 때는 `allowsConstrainedNetworkAcess = false`  로 하자.
- 만약 네트워크 제한으로 인해 에러를 받는다면, 그 때 Low Data Mode를 시도하자.

즉, 데이터가 많이 들 것 같거나 유저 경험에 이득이 되는 작업을 할 때는 `allowsConstrainedNetworkAcess` 를 false로 두고 리퀘스트를 하자. 하지만 네트워크 상황이 안 좋아 실패하여 해당 에러를 받는다면, 그 때 Low Data 일 때 해당하는 작업 또는 리퀘스트를 하자.

##### Network.framework

마찬가지로 네트워크 Path 별로 제한 설정을 하는 프로퍼티를 만지는 것 같은데 무슨 소리인지 잘 모르겠다...

#####Constrained and Expensive

- Constrained - Low Data Mode
- Expensive - 셀룰러와 개인 핫스팟을 통한 연결
  - URLSession의 `allowsExpensiveNetworkAccess` 프로퍼티로 제어가 가능
  - Network.framework에서는 `NWParameters` 의 `prohibitExpensivePaths` 로 제어가 가능

Expensive 프로퍼티는 시스템이 관리하는 프로퍼티로 현재는 보통 가격이 비싼 셀룰러 네트워크와 관련되어 있지만 새로운 네트워크가 나오게 되고 이것의 비용이 클 경우 이것을 제어하는 프로퍼티가 될 수 있다. 시스템이 알아서 해줄 것이기 때문에 이 프로퍼티를 사용할 것을 애플은 권장한다.

## Combine in URLSession

### DataTaskPublisher

Single Value Publisher로 기존에 있던 `URLSession.dataTask(with:completionHandler)` 메서드와 비슷하다.

```swift
public struct DataTaskPublisher: Publisher {
		public typealias Output = (data: Data, response: URLResponse) 
  	public typealias Failure = URLError
}
```

다음과 같이 사용한다.

예를 들어 이미지를 표현하는 TableViewCell이 있다고 하면, `AnyCancellable` 을 따르는 subscriber를 가지고 있는다.

```swift
class MenuItemTableViewCell: UITableViewCell {
    @IBOutlet weak var itemImageView: UIImageView!
  	
  	// subcriber를 가지고 있어 데이터 리퀘스트의 구독을 함
  	var subscriber: AnyCancellable?
  
  	override func prepareForReuse() {
        super.prepareForReuse()
      	itemImageView.image = nil
      	
      	// subcriber를 cancel시켜줌으로써
        // 셀에 맞지 않는 이미지가 로드되지 않도록 함
      	subscriber?.cancel()
    }
}
```

그러면 cellForRowAt 메서드에서는 해당 subscriber를 tryCatch 문을 통해 DataTask를 발행하도록 한다.

```swift
var request = URLRequest(url: menuItem.highResImageURL)
request.allowConstrainedNetworkAccess = false
cell.subscriber = pubSession.dataTaskPublisher(for: request)
		// 네트워크 제약으로 인한 에러일 경우
    // 낮은 화질의 이미지를 로드하는 DatataskPublisher를 반환한다.
    .tryCatch { error -> URLSession.DataTaskPublisher in
       guard error.networkUnvaliableReason == .constrained else {
           throw error
       }
       return pubSession.dataTaskPublisher(for: menuItem.lowResImageURL)
      }
    .tryMap { data, response -> UIImage in
       // 고화질이든, 저화질이든 상관없이
       // 성공적으로 받은 리스폰스의 statusCode를 검증하고(서버의 에러라면 throw!)
       guard let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 200,
           let image = UIImage(data: data) else {
             throw PubSocketError.invalidServerResponse
       }
       // 데이터를 UIImage의 형태로 Mapping한다.
       return image
     }
		// 에러 이미지를 보여주기 전 retry를 시도하게끔!(대박...)
		.retry(1)
		.replaceError(with: placeHolderImage)
		.receive(on: DispatchQueue.main)
		.assign(to: \.itemImageView.image, on: cell)
}
```

다만, 중요한 것은 네트워크 요청은 비싼 작업이기 때문에 무작정 retry를 넣지는 말아야 한다는 것이다. 비교적 간단한 작업, 그리고 해야 할 때만 넣자! 

- 서버의 장애가 났거나 이럴 때는 retry를 해도 소용이 없기 때문에 미리 throw로 거르자...!
- Payment 같은 경우 retry는 치명적일 수 있기 때문에 지양하자.

다음은 지금까지 설명한 것들의 정리 코드이다.

<img width="1401" alt="스크린샷 2019-08-10 오후 11 58 50" src="https://user-images.githubusercontent.com/22453984/62823312-d1410d00-bbca-11e9-8150-9a440b8aaba1.png">

## WebSocket

#### WebSocket에 대한 간략한 조사

기존에 HTTP 응답으로부터 웹을 그리는 방식은 기존 브라우저의 화면을 지우고 받은 내용을 다시 그리는 방식이었다. 지우고 다시 그리기 때문에 깜박임이 발생한다. 이러한 깜박임 없이 원하는 부분만 다시 그리며 실시간으로 사용자와 상호작용 할 수 있게 하는 방식이 발달했고, 이를 RIA(Rich Internet Application) 기술이라 한다.

기존 ''브라우저의 HTTP 요청 -> 웹 서버의 HTTP 응답''의 구조는 단방향 메시지 교환 규칙을 따르고 있으며 이는 Long Polling, Stream 등의 상호작용 웹 기술이 채택한 방식이었다. 그렇기 때문에 웹 페이지의 구현이 복잡하고 어려웠다.

보다 쉽게 만들기 위해 양방향 메시지 송수신(bi-directional full-duplex communication)이 필요했는데 이를 지원한 것이 WebSocket API이다.

![WebsocketSo1](https://d2.naver.com/content/images/2015/06/helloworld-1336-1-1.png)

### 🆕 URLSessionWebSocketTask

해당 Task는 기존 URLSession과 같이 동작하기 때문에 기존의 Configuration을 그대로 이용할 수 있다.

서버와 클라이언트가 WebSocket을 구현한다면, 명시적인 요청없이도 한 번 연결이 이루어지는 것으롤 지속적인 상호 업데이트가 가능하다.

## Mobility Improvements

집의 WiFi로부터 멀어지면 WiFi를 끄고 셀룰러 네트워크만 쓰는 경험을 한 번쯤 해봤을 것이다. 애플은 사용자가 이렇게 하는 것을 바꾸고 싶어한다.

Wi-Fi는 중앙으로부터 멀어질 수록 약해지는 네트워크 신호이고 원의 형태로 나타난다고 쉽게 생각할 수 있지만 현실은 많은 장애물에 가로 막혀 Wi-Fi가 어떤 지 예측하기 힘들어진다.

### 🆕 Wi-Fi Assists in iOS 13

- Cross-layer Mobility Detection: Wi-Fi Assist 객체가 상위의 Network.framework, URLSession 그 밖의 daemon 프로세스에서 정보를 얻어 mobility에 대한 더 나은 판단을 내릴 수 있게 하여 더 나은 네트워크를 사용하도록 한다.
- Improved Flow Recovery: 첫 번째 리퀘스트에 대한 Wi-Fi 신호가 중간에 약해지면 두 번째 리퀘스트는 셀룰러로 이동하도록 하는 유동적인 정책을 취함.

이것의 이점을 앱에 적용하려면,

- URLSession이나 Network.framework 같은 고차원 API를 사용
- `SCNetworkReachablity` 의 사용을 재고려: 미리 보내서 확인하는 것은 실제 보내는 것과 시점이 다르기 때문에
- 셀룰러 네트워크의 사용의 제한은 `allowExpensiveNetworkAcess` 를 false로 하자.

### Multipath Transports

Siri에는 이미 적용이 되었지만 iOS 13부터는 Apple Map, Apple Music에서도 지원이 된다.

#### 앱에 적용하려면(물론 서버가 지원되어야 함...)

- URLSessionConfiguration의 `multipathServiceType` 를 지정