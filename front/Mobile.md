모바일 앱 개발 관련 기록
(Flutter)

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