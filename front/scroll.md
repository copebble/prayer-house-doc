# Scroll 관련

<br>

## 📌 특정 위젯으로 이동

```dart
Scrollable.ensureVisible(
    _globalKey.currentContext!,
    duration: 1000.ms,
)
```
`Scrollable`을 사용해 특정 위젯의 key를 가지고 scroll 이동 가능