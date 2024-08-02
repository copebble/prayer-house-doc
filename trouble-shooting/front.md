# front 관련 trouble shooting

추후 주제별로 분리 가능

## 📌 dependOnInheritedWidgetOfExactType

`InheritedWidget` 사용과 관련된 부분

```dart
@override
void initState() {
    super.initState();

    /// [RootBottomNavBarController]는 [InheritedWidget]
    context.dependOnInheritedWidgetOfExactType<RootBottomNavBarController>()!.turnOff();
}
```
initState 부분에서 `InheritedWidget` 구현한 클래스를 가져오려고 했는데 에러 발생

```
dependOnInheritedWidgetOfExactType<RootBottomNavBarController>() or 
dependOnInheritedElement() was called before 
_PrayerLapseDetailScreenState.initState() completed.
```

다음 에러로그에 힌트가 있음

```
When an inherited widget changes, for example if the value of Theme.of() changes, its dependent widgets are rebuilt. 
If the dependent widget's reference to the inherited widget is in a constructor or an initState() method, 
then the rebuilt dependent widget will not reflect the changes in the inherited widget.

Typically references to inherited widgets should occur in widget build() methods. 
Alternatively, initialization based on inherited widgets can be placed in the didChangeDependencies method, 
which is called after initState and whenever the dependencies change thereafter.
```
initState에서 `dependOnInheritedWidgetOfExactType`를 통해 InheritedWidget 가져와 사용하는 것은 안되는 듯하다.
build 과정에서 inherited widget의 변화를 제대로 반영해줄 수 없다고 나오는데 완벽하게 이해는 되지 않았다.

해결책은 간단한데 build 된 이후에 호출하는 것이다.
`AfterLayoutMixin` mixin 사용해서 구현(`afterFirstLayout` overriding)
(단 `StatefulWidget`에서 사용해야 함)

```dart
@override
FutureOr<void> afterFirstLayout(BuildContext context) {
    // 기본 bottom navigation bar 제거
    context.dependOnInheritedWidgetOfExactType<RootBottomNavBarController>()!.turnOff();
}
```