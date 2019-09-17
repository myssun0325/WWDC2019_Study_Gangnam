# Architecting Your App for Multiple Windows - WWDC 2019 258

## Lifecycle의 변화

iOS 12까지는 UI를 하나만 가지는 구조였기 때문에 App Delegate가 앱의 생명 주기를 모두 관리했었다. 하지만 iOS 13부터는 하나의 프로세스를 공유하며 다양한 UI를 가질 수 있어 새롭게 Scene Delegate가 추가되어 각각의 UI를 관리한다.

이러한 UI는 **Scene Session** 이라고도 불린다.

<img width="1139" alt="스크린샷 2019-08-10 오후 8 06 20" src="https://user-images.githubusercontent.com/22453984/62821109-57992700-bbaa-11e9-94de-b2646154aa89.png">

iOS 13에서 만약 앱이 새로운 Scene 생명 주기를 적용한다면 UIKit은 더 이상 UI 상태에 대한 App Delegate 메서드를 호출하지 않을 것이다. 대신, Scene Delegate의 메서드를 호출할 것이기 때문에 대응되는 메서드를 적용해야 한다.

``` swift
// App Delegate
application:willEnterForeground 
application:didEnterBackground
application:willResignActive
application:didBecomeActive

// Scene Delegate
scene:willEnterForeground
scene:didEnterBackground
scene:willResignActive
scene:didBecomeActive
```

이러한 새로운 메서드를 적용하는 것이 iOS 12 이하에서 적용되는 메서드를 버리라는 뜻은 아니다. **이전 버전을 지원하는 경우 UIKit이 런타임 시에 스스로 판단하여 맞는 메서드를 호출할 것**이다.

### Session Lifecycle

앱이 처음 실행되었을 때부터 생명 주기를 관찰한다면,

1. App Delegate의 `didFinishLaunching` 이 호출되며 프로세스가 시작됨
2. App Delegate의 `configurationForSession` 이 호출되며 어떤 session이 나와야 하는지 결정

#### Configuring New Sessions

```swift
func application(_ application: UIApplication,
								configurationForConnecting connectingSceneSession: UISceneSession,
								options: UIScene.ConnectionOptions) -> UISceneConfiguration
```

- 새로운 Scene을 만들 때 App Delegate에서 호출 
- 적절한 `UISceneConfiguration` 을 반환하여 새 Scene Session을 구성. 예를 들어 메인 세션인지, 부 세션인지를 판단하여 그에 맞게 Session을 구성하게 시스템에 요청함.
- `UserActivity`, URL 등을 받아 적절한 Session을 구성할 수 있음
- Info.plist에서 Static하게 생성하거나 dynamic하게 코드로 구성이 가능

3. Scene Delegate에서 `willConnectToSession` 이 호출되어 곧 Session에 연결될 것임을 알려줌. 이 때는 연결이 되지 않았기 때문에 UI가 보이지 않는다.

``` swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate { 
    var window: UIWindow?
  
    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: .ConnectionOptions)
        window = UIWindow(windowScene: scene as! UIWindowScene)
        if let activity = options.userActivities.first ?? session.stateRestorationActivity {
            configure(window: window, with: activity)
        }
    } 
}
```

`willConnectToSession` 에서 window에 인스턴스를 넣어주고 새 Session이 화면에 보일 준비를 한다. 이 때 사용자가 **기존에 작업하던 것이 있다면 복원해주는 절차가 중요**하다.

여기서, 만약 사용자가 앱을 스와이프하여 홈으로 간다면 어떻게 될까?

4. Scene Delegate에서 `willResignActive`, `didEnterBackground` 가 호출된다.
5. Scene Delegate에서 최종적으로 `didDisconnect` 가 호출되어 Scene Session과의 연결이 끊어질 수도 있다.

### Scene Disconnection

``` swift
func sceneDidDisconnect(_ scene: UIScene)
```

- 시스템이 scene을 메모리에서 해제함. 또한 여기서 해제될 scene과 관련된 자원을 해제할 좋은 기회가 되기도 함.
- 언제든지 불릴 수 있음
- Scene은 다시 돌아올 수 있음! -> 상태 복원이 필요

하지만 만약, 사용자가 단순히 앱을 나가는 것이 아니라 앱을 명시적으로 꺼버린다면 어떻게 될까?

6. App Delegate에서 `didDiscardSceneSession` 이 호출되어 해당 세션을 버린다.

### Discard Session

```swift
func application(_ application: UIApplication, 
                 didDiscardSceneSessions sceneSessions: Set<UISceneSession>)
```

- 영구적으로 session이 삭제될 때 호출
- 해당 Scene과 관련된 데이터를 삭제함
- 만약 사용자가 앱이 *Running* 상태가 아닐 때 종료한다면, 시스템이 이를 저장해놨다가 다음 Launch 시에 이 메서드를 호출하여 session이 사라졌다는 것을 알려줌

## Architecture

### State Restoration

iOS 13에서는 상태 복원이 그저 바람직한 것이 아니라 Scene 기반의 앱이라면 적용해야 하는 핵심 사항이다. 만약 상태 복원을 하지 않는다면 사용자가 앱 스위처에서 전환을 하여 사용할 경우 입력해 놓은 데이터가 날아가는 등의 좋지 못한 사용자 경험을 겪게 될 것이다.

#### Per-Scene State Restoration

iOS 13에서는 Scene 마다의 상태 복원을 할 수 있게 새 메서드를 제공한다.

```swift
func stateRestorationActivity(for scene: UIScene) -> NSUserActivity?
```

- Scene의 백그라운드에서 호출됨
- `NSUserActivity` 를 통해 상태를 암호화하며 앱의 보호 상태에 맞게끔 해줌(... 잘 모르겠다)

실제 사용되는 모습은 다음과 같다.

``` swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    // 적절한 Activity를 찾아 반환해줌
    func stateRestorationActivity(for scene: UIScene) -> NSUserActivity? {
        let currentActivity = fetchCurrentUserActivity(for: self.window)
        return currentActivity
    }
  	// scene이 새로 연결될 때 stateRestoartionActivity가 있다면 그것으로 복원 및 구성
    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: .ConnectionOptions)
        if let restorationActivity = session.stateRestorationActivity {
            self.configure(window: window, with: restorationActivity)
        }
}
```

### Keeping Scenes in Sync

두 가지 UI가 같은 화면을 보여줄 수도 있기 때문에 데이터의 Sync를 맞추는 것이 중요하다.

따라서 다음과 같은 구조 같이 Model을 업데이트하면 그곳에서 UI를 업데이트하도록 시키는 형태가 되어야 한다.

<img width="1185" alt="스크린샷 2019-08-10 오후 9 00 12" src="https://user-images.githubusercontent.com/22453984/62821564-dc3b7380-bbb1-11e9-8273-3d2fefcfaff3.png">