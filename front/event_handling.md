# 여러 event 관련

<br>

## 키보드 on/off 여부

```dart
bool _isKeyboardVisibility = false;
late StreamSubscription<bool> _keyboardSubscription;

@override
void initState() {
    super.initState();
    final keyboardVisibilityController = KeyboardVisibilityController();

    // 키보드 on/off 여부 체크
    _keyboardSubscription = keyboardVisibilityController.onChange.listen((bool visible) {
        setState(() => _isKeyboardVisibility = visible);
    });
}

@override
  void dispose() {
    _keyboardSubscription.cancel();

    super.dispose();
  }
```