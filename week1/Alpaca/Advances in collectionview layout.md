# Advances in collection view layout

현대의 앱은 복잡한 레이아웃을 제공해야 하기에 Custom Layout이 필요

## Current State-of-the-Art

Cutom Layout을 쉽게 제공하기 위해 새로운 Concrete Laout을 제공한다.

### 🆕 Compoisitional Layout

세 가지 특성을 가진다.

- Composable: 복잡한 레이아웃을 간단한 것을 조합하여 이룸
- Flexible: 어떠한 레이아웃도 만들 수 있음
- Fast: 성능 상의 최적화를 고려

결과적인 특징은 다음과 같다.

- 작은 레이아웃 그룹을 조합
- 레이아웃 그룹은 이전과 같이 line-based
-  Subclassing 대신 Composition을 사용해 레이아웃을 만든다.

#### Core Concepts

Item은 Group에 속하고 이 Group은 다시 Section에 속한다. 이 Section이 모여 최종적인 Layout을 이룬다.

> Group은 우리가 알고 있는 Row!

<img width="696" alt="스크린샷 2019-07-29 오후 1 05 24" src="https://user-images.githubusercontent.com/22453984/62021323-a4235080-b201-11e9-9ef4-430de6fb5b1c.png">

##### NSCollectionLayoutSize

모든 것은 Explicit한 사이즈를 가지고 있으며 Size는 다음과 같이 표현된다.

- Size = Width Dimension + Height Dimension

###### NSCollectionLayoutDimension

축에 독립적이며 네 가지 방법으로 정의할 수 있음

```swift
let widthDimension = NSCollectionLayoutDimension.fractionalWidth(0.5)
let heightDimension = NSCollectionLayoutDimension.fractionalHeight(0.3)
let absoluteHeightDimension = NSCollectionLayoutDimension.absolute(200)
let estimatedHeightDimension = NSCollectionLayoutDiem
```

