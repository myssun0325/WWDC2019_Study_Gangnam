# Advances in App Background Execution - WWDC 2019 / 707

## Overview of Background Execution

사용자에게 보여줄 필요가 없지만 앱이 실행되고 있는 상태를 백그라운드 상태로 일컫는다.

##### 백그라운드로 상태로 들어가는 이유

- App request: 정기적인 백그라운드 fetch, 포그라운드에서의 작업의 마무리 등을 하기 위해 앱이 백그라운드 상태로 앱을 이동
- Event trigger: 사용자가 특정 액션을 통해 백그라운드에서의 실행을 야기

##### 백그라운드 실행에서 고려해야 할 점

- Power
  - 작업이 완료되면 빠르게 completionHandler를 호출하여 작업이 소모하는 배터리를 줄여야 한다.
- Performance
  - 포그라운드와 백그라운드 모두에서 앱이 실행되기 때문에 CPU와 메모리 사용량을 똑똑하게 고려해야 한다. 따라서 백그라운드 API를 사용하여 앱을 실행할 때는 이러한 자원에 대한 한계를 유념해야 하며 그렇게 함으로써 시스템이 앱을 종료시키지 않을 것이다.
- Privacy
  - 사용자는 포그라운드보다 백그라운드에서 자신의 데이터가 어떻게 쓰이는 지 민감하지 않다. 따라서 어떻게 데이터가 쓰일 지 투명하게 공개하라.

## Backgrond Execution Best Practices

메시지 앱을 통해 각각의 경우를 설명한다.

### Send Messages

- 사용자는 즉각적인 완료(메시지가 보내짐)를 기대
- 이 완료는 네트워크 오류 등의 상황을 거쳐도 이루어진다는 것을 보장해야 함

#### Background Task Completion

앱이 일시정지 되기 전 백그라운드에서의 추가적인 실행시간을 보장한다.

- [`UIApplication.beginBackgroundTask(expirationHandler:)`](https://developer.apple.com/documentation/uikit/uiapplication/1623031-beginbackgroundtask)
- [`ProcessInfo.performExpiringActivity(withReason:using:)`](https://developer.apple.com/documentation/foundation/processinfo/1617030-performexpiringactivity)(Extension에서 실행될 경우)

이 API를 통해 포그라운드에서 시작된 작업을 완료하면 된다.

```swift
func send(_ message: Message) {
    let sendOperation = SendOperation(message: message)
    var identifier: UIBackgroundTaskIdentifier!
    identifier = UIApplication.shared.beginBackgroundTask(expirationHandler: {
        sendOperation.cancel()
      	// 네트워크 상황 등 어쩔 수 없이 실패하는 상황에서 유저에게 알려줌
        postUserNotification("Message not sent, please resend")
        // Background task will be ended in the operation's completion block below
    })
    sendOperation.completionBlock = {
      	// 작업이 완료됐을 때 시스템에게 작업이 끝남을 알려줌으로써
      	// 배터리 소모를 줄이고 미래의 성능에 영향을 미치는 것을 막음
        UIApplication.shared.endBackgroundTask(identifier)
    }
    operationQueue.addOperation(sendOperation)
}
```

### Phone Calls

- VoIP push notifications: 앱을 실행하는 특별한 타입의 푸시

```swift
func registerForVoIPPushes() {
    self.voipRegistry = PKPushRegistry(queue: nil) 
    self.voipRegistry.delegate = self
    self.voipRegistry.desiredPushTypes = [.voIP]
}
```

#### 🆕 추가된 부분

iOS 13 이후부터는 Incoming VoIP 전화가 오면 PushKit의 [`pushRegistry(_:didReceiveIncomingPushWith:for:completion:)`](https://developer.apple.com/documentation/pushkit/pkpushregistrydelegate/2875784-pushregistry) 를 통해서 처리해줘야 하며 그렇지 않다면 시스템이 앱을 강제 종료시킨다. 통화 받음의 반복적인 실패는 시스템이 앱을 더이상 통화를 받지 못하게 한다.

```swift
let provider = CXProvider(configuration: providerConfiguration)
func pushRegistry(_ registry: PKPushRegistry, didReceiveIncomingPushWith payload: PKPushPayload, for type: PKPushType, completion: @escaping () -> Void) {
    if type == .voIP {
        if let handle = payload.dictionaryPayload["handle"] as? String {
            let callUpdate = CXCallUpdate()
            callUpdate.remoteHandle = CXHandle(type: .phoneNumber, value: handle)
            let callUUID = UUID()
            provider.reportNewIncomingCall(with: callUUID, update: callUpdate) { _ in
                comletion()
            }
            establishConnection(for: callUUID)
        }
    }
}
```

### Muted Threads

- 다양하고 많은 threads
- 기기에는 알리지만 유저에게는 알리지 않아야 함

#### Background Pushes

- 유저에게 알림이 가지 않은 채 기기에게 새로운 데이터가 있다고 알려줄 수 있는 기법
- Push를 보낼 때 `"alert"` , `"sound"`, `"badge"` 없이  `"content-avaliable: 1"` 를 보냄
- 시스템이 스스로 알아서 언제 데이터를 받을 지 최적의 결정

#### 🆕 추가된 부분

- `"apns-priority = 5"` 를 설정해줘야 하며 그렇지 않으면 앱이 launch가 안 됨
- `"apns-push-type = background"` 를 설정해줘야 함. 이는 WatchOS에서만 필수이지만 **모든 플랫폼에서 하기를 권장!**

### Download Past Attachments

- 과거의 컨텐츠를 굳이 포그라운드에서 받아 불필요한 성능을 낭비할 필요가 없음
- 최신 컨텐츠를 포그라운드에서, 과거 컨텐츠를 **백그라운드에서 받도록 미루기(defer)**

#### Discretionary Background URL Session

- 시스템이 더 좋은 시간까지 다운로드를 미룰 수 있도록 허용

```swift
// Set up background URL session
let config = URLSessionConfiguration.background(withIdentifier: "com.app.attachments") 
let session = URLSession(configuration: config, delegate: ..., delegateQueue: ...)

// Set discretionary
config.isDiscretionary = true
```

- 시스템이 최적의 스케줄링을 할 수 있도록 정보를 제공

``` Swift
// Set timeout intervals
config.timeoutIntervalForResource = 24 * 60 * 60
config.timeoutIntervalForRequest = 60

// Create request and task
var request = URLRequest(url: url)
request.addValue("...", forHTTPHeaderField: "...")
let task = session.downloadTask(with: request)

// Set time window
task.earliestBeginDate = Date(timeIntervalSinceNow: 2 * 60 * 60)

// Set workload size
task.countOfBytesClientExpectsToSend = 160
task.countOfBytesClientExpectsToReceive = 4096
task.resume()
```

## New BackgroundTasks Framework

#### 미룰 수 있는 유지보수 작업

- Syncing
- DB Cleanup
- Backups

이러한 작업은 백그라운드로 미뤄서 하면 시스템의 성능에 도움이 됨

### 🆕 BackgroundTasks

백그라운드 작업을 스케줄링하는 데 사용하는 새 프레임워크로 전 플랫폼에 사용 가능

#### Background Processing Tasks

시스템의 최적 시간에 수 분의 실행 시간을 제공

- 미룰 수 있는 유지보수 작업
- Core ML 학습 / 추론

큰 작업을 할 때 CPU 모니터를 끌 수 있는 능력을 제공하여 하드웨어의 성능을 최대로 이용 가능

포그라운드에서 요청했거나 최근에 앱이 사용된 경우 이 Task를 사용할 수 있음

### 🆕 Background App Refresh Task

새로운 API지만 기존과 같은 정책임

- 30초의 실행 시간
- 앱을 최신 상태로 유지

이 Task의 사용은 유저의 과거 앱 사용 패턴에 기반함

과거의 UIAppplication fetch API는 dprecated되며 Mac에서 지원되지 않으므로 새 API를 사용해야 함

#### BGTask 과정의 High level Overview

1. App이나 Extension에서 BGAppRefreshTaskRequest나 BGAppProcessingTaskRequest를 생성하고 이는 BGTaskSchduler에 큐잉된다.
2. 시스템이 BGTask를 실행시킬 시간이 되었다고 판단되면, 스케줄러에서 BGTaskRequest를 빼내 App에 BGTask를 전달한다. Extension에서 요청된 Task도 Main App에서 실행한다.
3. App은 깨어나 이 Task를 실행하고 마치면 Complete Task를 호출한 뒤 다시 Suspended 상태로 이동한다.

#### BGTask 과정

1. 백그라운드 Task 실행을 가능하도록 Capabilities에서 세팅
2. Launch Handler에서 Task를 등록
3. Task를 스케줄링

#### BGTask를 디버깅하는 방법

- Launch a Task

  - Task가 제출된 후 BreakPoint를 찍은 뒤 Console에 다음 라인을 실행한 후 다시 시작한다.

  - ```
    e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"TASK_IDENTIFIER"]
    ```

- Force Early Termination of a Task

  - Task 실행 중 BrakdPoint를 찍은 뒤 Console에 다음 라인을 실행한 후 다시 시작한다.

    ```
    e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateExpirationForTaskWithIdentifier:@"TASK_IDENTIFIER"]
    ```

#### Additional Considerations

- `earliestBeginDate` 를 너무 먼 미래로 설정하지 말자: 한 주 아래로 설정하는 것을 권장
- 기기가 잠겨있어도 files에 접근할 수 있도록 해야 작업이 가능하기 때문에 이것을 고려
- UIScene 앱은 `UIApplication.requestSceneSessionRefresh(_:)` 을 호출하여 앱이 바뀌었다는 것을 알려야 함
- 성능이 많이 소모되는 launch 단계에서 작업을 해야 하는 경우, `BGTaskScheduler.submit(_:)` 의 호출을 백그라운드 큐에서 하는 것을 고려하자