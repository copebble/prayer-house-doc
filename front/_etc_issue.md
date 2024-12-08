# 기타 이슈 기록

모바일 앱 개발 관련 기록
(Flutter)

<br>

## 📌 Form 관련

`FormField`는 가로로 공간을 차지하기 때문에 width 고정되어 있는 `Row`와 같이 사용하는 경우 `Expanded`로 Field를 감싸서 설정해야 함 

<br>

## 📌 NestedScrollView 관련
```
Build scheduled during frame.
While the widget tree was being built, laid out, and painted, a new frame was scheduled to rebuild the widget tree.
This might be because setState() was called from a layout or paint callback.
```
NestedScrollView에서 반복해서 스크롤 up or down을 하면 위와 같은 에러메시지가 나오게 됨

link: https://github.com/flutter/flutter/issues/104798

### RefreshIndicator

NestedScrollView에서 RefreshIndicator를 바로 사용할 수 없는 것 같다. (에러 발생)  

```dart
RefreshIndicator.adaptive(
  // TODO 알아보기             
  notificationPredicate: (notification) {
    if (notification is OverscrollNotification || Platform.isIOS) {
      return notification.depth == 2;
    }
    return notification.depth == 0;
  },
  onRefresh: () async {
    //임의의 새로고침 지연시간
    await Future.delayed(500.ms, () => {});
  },
  color: Colors.grey,
  child: NestedScrollView(...)
```
[RefreshIndicator.adaptive](https://api.flutter.dev/flutter/material/RefreshIndicator/RefreshIndicator.adaptive.html) 활용

<br>

## 📌 StatefulShellRoute.indexedStack & OffStage (미해결)

- 현재 상황

Go Router `StatefulShellRoute.indexedStack`를 가지고 앱의 최초 시작격인 `RootScreen` 위젯에서 bottom navigation bar에 의한 여러 tab들을 구현  

```
RootScreen - Tab Page Widgets(각각 riverpod provider watch 중)
```

`RootScreen`의 `Scaffold` body 부분을 `OffStage` 적용해서 탭이 활성화 되지 않은 페이지들에 대해서 위젯 트리에서는 안보이도록 하려고 시도했으나 잘 안됨

`OffStage`로 각 Tab Page를 감싸서 적용했고 위젯 트리에도 비활성화된 페이지에 대해 제외된 것을 확인할 수 있었지만 한 번 방문했던 tab page에 대해 다시 재방문을 하게 되면 riverpod provider가 rebuild 되면서 데이터 조회를 위해 api 호출을 하게 된다.

`KeepAliveWrapper`, `PageStorageBucket` 등을 사용해보았지만 원하던 방식대로 동작하지 않음

<br>

## 📌 flutter 버전 관련 이슈

Android Studio `settings` 내에 `Languages & Frameworks` > `Android SDK` 메뉴
`SDK Tools` 탭에 버전 업데이트 하면 된다.