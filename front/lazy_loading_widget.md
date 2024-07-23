# Flutter Widget lazy loading(build) 적용

flutter widget 개발시 화면 최초 진입 때 열리지 않은 화면에 대해서 build가 발생하는 경우가 많다.
이렇게 되면 문제인 것이 사용자가 아직 보지도 않는 화면에 대해서 build하면서 riverpod provider도 build가 발생, 그 안에 최초 api를 호출하게 된다.

즉 불필요한 서버 api request가 발생한다는 점에서 모바일, 서버 둘 다 불필요한 리소스 낭비 발생

lazy loading 하는 방법에 대해 알아보고 적용을 하게 되었다.

<br>

## 📌 bottom navigation bar에 의한 lazy loading

> 현재는 사용 X,,, go router StatefulShellRoute.indexedStack 사용하여 구현

앱 맨 처음 진입 지점에 bottom navigation bar 기능을 적용하게 된다. 바텀에 있는 탭을 터치하면 해당 화면이 그려지는 형태다.
하지만 lazy loading을 적용하지 않으면 모든 탭에 대해 build를 하게 되고 각각에 적용되어 있는 api 호출이 불필요하게 발생하게 된다.

[lazy_load_indexed_stack](https://pub.dev/packages/lazy_load_indexed_stack) plugin을 적용

```dart
lazy_load_indexed_stack: ^1.1.0
```

### LazyLoadIndexedStack 원리

lazy loading 기능을 적용하기 위해 IndexedStack를 확장한 widget

LazyLoadIndexedStack 원리는 간단하다.

```dart
// LazyLoadIndexedStack
List<Widget> _initialChildren() {
  return widget.children.asMap().entries.map((entry) {
    final index = entry.key;
    final childWidget = entry.value;

    if (index == widget.index || widget.preloadIndexes.contains(index)) {
      return childWidget;
    } else {
      return widget.unloadWidget;
    }
  }).toList();
}
```

최초 `LazyLoadIndexedStack`의 `_children`을 생성할 때 최초 index와 같은 위치의 widget만 build해서 가져오고, 나머지는 unloadWidget을 설정(default로 empty Container)

즉 최초 진입한 탭에 대해서만 위젯을 빌드해서 화면을 구성하고 나머지 위젯에 대해서는 비어있는 Container로 채움

```dart
// LazyLoadIndexedStack
@override
void didUpdateWidget(final LazyLoadIndexedStack oldWidget) {
  super.didUpdateWidget(oldWidget);

  if (widget.children.length != oldWidget.children.length) {
    _children = _initialChildren();
  }

  _children[widget.index] = widget.children[widget.index];
}
```
만약 LazyLoadIndexedStack를 감싸고 있는 상위 위젯에서 rebuild가 발생하면 위의 메소드가 실행된다.

여기서 widget.index가 최초 index와 다른 경우 설정해두었던 widget.children에서 해당 index의 위젯을 가져오게 된다.

상위 위젯에 setState 등으로 tab의 `currentIndex`를 변경하면 rebuild가 일어나면서 LazyLoadIndexedStack에서는 바뀐 tab의 위젯을 가져오게 되고 이 때 build가 되면서 api 호출 발생
`
이런 방식으로 하단 탭이 이동되는 시점에 바뀐 tab의 위젯이 build가 되면서(lazy loading) 화면에 그려지게 된다.

> 현재 프로젝트에서 Go Router로 bottom navigation bar tab 구성하였기에 `lazy_load_indexed_stack` 관련 내용은 걷어내고 플러그인 삭제하였습니다.

<br>

## 📌 TabBar & TabBarView lazy loading

```dart
final tabListViews = _userPrayerTabTypes
  .mapIndexed(
    (tabType, index) => KeepAliveWrapper(
      child: UserPrayerLapsesTabView(_userPrayerTabTypes[index]),
    ),
  )
  .toList();

DefaultTabController(
  length: _userPrayerTabTypes.length,
  child: Column(
    children: [
      TabBar(...),
      Expanded(
        child: TabBarView(
          children: tabListViews,
        )
      )
    ]
  )
)
```
TabBarView에 들어갈 children을 생성하는데 따로 StatelessWidget으로 UserPrayerLapsesTabView 위젯을 만들어 적용. (그 안에 실제 내역 리스트 조회하고 화면 구성)

이렇게만 해도 tab 이동 간에 lazy loading으로 동작하게 되는 듯 싶다.

여기에 추가로 KeepAliveWrapper를 구성했는데 이는 탭 간 이동시 비활성화된 탭에 대해 dispose하지 않고 그 상태 그대로 캐싱하기 위해 사용한다고 보면 된다.

```dart
class _KeepAliveWrapperState extends State<KeepAliveWrapper> with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context); // Important: call super.build
    return widget.child;
  }
}
```
`wantKeepAlive` overriding getter를 true로 하고 build 안에서 
`super.build(context);` 호출하는 내용이 있어야 한다.

이렇게 되면 비활성화된 탭이 dispose되어 다시 api 호출하지 않아도 이전 상태 그대로 유지하게 된다. 

scroll 위치까지 저장되어서 `TabBarView` 각 위젯에 `PageStorageKey(...)`를 따로 설정할 필요가 없는 것 같다.