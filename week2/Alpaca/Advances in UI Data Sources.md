#  Advances in UI Data Sources - WWDC 2019 220 

## Current State of the Art

`UITableViewDataSource`, `UICollectionViewDataSource`에서 비교적 간단하고 직관적인 API를 전달해주는 것과 달리 지금의 앱은 점점 복잡해지고 있다.

- UI data source가 controller에 의해 지원되는데 Controller는 하는 일이 점점 많아짐
- Web Services
- DB

이러한 복잡한 앱을 만들다 보면(예: Controller가 데이터가 바뀌었다고 UI한테 알려줘야 하는 경우) 업데이트가 잘못되었을 경우, 다음과 같은 에러를 꽤 자주 보게 된다.

``` swift
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Invalid update: invalid number of sections. The number of sections contained in the collection view after the update (10) must be equal to the number of sections contained in the collection view before the update (10), plus or minus the number of sections inserted or deleted (0 inserted, 1 deleted).'
***
```

문제를 해결하기 위해서 가장 확실한 방법은 `reloadData()` 를 호출하는 것이지만 이 방법은 애니메이션이 없어 유저에게 좋은 인상을 주지 못한다.

### Where is Our Truth?

- Data source와 현재 UI 상태는 항상 동일해야 함
- 현재의 방법은 에러가 일어나기 쉬운 상태
- 중앙 집중된 Truth가 없음

## 🆕 Diffable Data Source

- ❌ `performBatchUpdates()` - 크래시를 유발하고 귀찮으며 복잡함
- ✅ `apply()` - 쉽고 자동으로 diffing

### Snapshots

- UI 상태의 truth
- 섹션과 아이템의 Indexpath 대신 Unique identifier를 사용

### 네 가지 요소

- iOS, tvOS
  - `UICollectionViewDiffableDataSource`
  - `UITableViewDiffableDataSource`
- MacOS
  - `NSCollectionViewDiffableDataSource`
- All Platform
  - `NSDiffableDataSourceSnapshot`

### DiffableDataSource를 사용하는 세 가지 주요 과정

1. 새로운 Data로 DataSource를 업데이트한다.
2. 새로운 Snapshot을 만든다.
3. `apply()` 메서드를 통해 변화를 제출한다.

#### 새 Snapshot 만드는 법

```swift
let mountains = mountainsController.filteredMountains(with: filter).sorted { $0.name < $1.name }

// NSDiffableDataSourceSnapshot은 SectionIdentifierType과 ItemIdentiferType을 가지는 제네릭 타입
let snapshot = NSDiffableDataSourceSnapshot<Section, MountainsController.Mountain>()
// .main은 Section enum의 case
snapshot.appendSections([.main])
// ItemIdetifierType의 배열을 전달해야 하지만
// Mountain은 해당 타입의 구현체이기 때문에 그대로 전달
snapshot.appendItems(mountains)
dataSource.apply(snapshot, animatingDifferences: true)
```

####  ItemIdentiferType의 정의

```swift
// Hashable을 따르기 때문에 DiffableDatasource가
// Unique한 identifier를 가지고
// 무엇이 추가 / 삭제되어야 하는지 알 수 있음
struct Mountain: Hashable {
    ...
    let identifier = UUID()
    func hash(into hasher: inout Hasher) {
        hasher.combine(identifier)
    }
    static func == (lhs: Mountain, rhs: Mountain) -> Bool {
        return lhs.identifier == rhs.identifier
    }
    ...
}
```

####DataSource의 구성

```swift
// 원래 cellForItemAt, cellForRowAt에서 하던 일을
// 클로저로 전달하여 cell을 구성
self.dataSource = UITableViewDiffableDataSource
    <Section, Item>(tableView: tableView) { [weak self]
        (tableView: UITableView, indexPath: IndexPath, item: Item) -> UITableViewCell? in
    guard let self = self, let wifiController = self.wifiController else { return nil }

    let cell = tableView.dequeueReusableCell(
        withIdentifier: WiFiSettingsViewController.reuseIdentifier,
        for: indexPath)
   // Cell의 IdentifierType에 따라 Cell을 구성
    // network cell
    if item.isNetwork {
        cell.textLabel?.text = item.title
        cell.accessoryType = .detailDisclosureButton
        cell.accessoryView = nil

    // configuration cells
    } else if item.isConfig {
        cell.textLabel?.text = item.title
        if item.type == .wifiEnabled {
            let enableWifiSwitch = UISwitch()
            enableWifiSwitch.isOn = wifiController.wifiEnabled
            enableWifiSwitch.addTarget(self, action: #selector(self.toggleWifi(_:)), for: .touchUpInside)
            cell.accessoryView = enableWifiSwitch
        } else {
            cell.accessoryView = nil
            cell.accessoryType = .detailDisclosureButton
        }
    } else {
        fatalError("Unknown item type!")
    }
    return cell
}
```

### Considerations

- `performBatchUpdates()`, `insertItems()` 대신 `apply()` 를 호출하자.
- Snapshot을 구성하는 두 가지 방법

```swift
// Empty snapshot
let snapshot = NSDiffableDataSourceSnapshot<Section, UUID>()

// Current data source snapshot copy
let snapshot = datasource.snapshot()
```

- `append` 뿐만 아니라 `insert`, `move` 메서드도 Snapshot이 제공
- Identifier는 unique해야 하며 Hashable을 준수해야 함.

### IndexPath-based API는 어떻게 되는 것인가?

```swift
func collectionView(_ collectionView: UICollectionView, didSelectItemAt indexPath: IndexPath) {
	if let identifier = dataSource.itemIdentifier(for: indexPath) {
        // Identifier를 얻어 필요한 작업을 해주면 됨
    }
}
```

#### Performance

- 가능한 한 빠르게 만들었음
- 하지만 diffing은 linear한 동작이기 때문에 개발 단계에서 성능을 측정해보는 것이 매우 중요!
- Background 큐에서의 `apply()` 호출이 안전하기 때문에 성능을 고려해서 적용
  - main, background 큐 중 하나를 정해서 호출하자.

