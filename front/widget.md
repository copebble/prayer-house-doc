# flutter widget 관련


## 📌 TextField

`TextField` 위젯에 지정하는 controller는 `TextEditingController`

해당 controller로 입력하고 있는 text를 받아볼 수 있다.

```dart
final controller = TextEditingController();
controller.text;
```
`controller.text` 통해서 입력하고 있는 텍스트와 관련해서 화면에 보여주고 싶은 것들이 있을 수 있다. (ex. 작성하고 있는 문자 길이, 미리보기 용도의 입력중 텍스트 등등)

그런데 아쉽게도 controller.text 직접가져와서 화면에 뿌려주면 반영이 안된다.

```dart
controller.text //
    .text
    .color(Colors.grey)
    .make();
```
위에 위젯을 화면에 보여주고 싶을 때 입력할 때마다 실시간으로 화면에 반영이 안되는 것을 볼 수 있다. (변경 감지가 안됨)

그래서 이를 그 즉시 화면에 반영되도록 조치가 필요

```dart
TextField(
    controller: controller,
    //...
    onChanged: (text) {
        //...
        setState(() {});
    }
)
```
위 방식은 편리하지만 제약이 있다. `StatefulWidget`에서만 사용 가능

hook을 사용하고 싶다면 다음과 같이 해야 할 듯
```dart
final editingText = useState<String>(initialText);
final controller = useTextEditingController();
useEffect(() {
    editingListener() {
        editingText.value = controller.text;
    }

    controller.addListener(editingListener);
    return () => controller.removeListener(editingListener);
}, const []);
// ...
```
즉 중요한 내용은 `TextEditingController`에서 직접가져온 text만으로 화면에 표시되는 부분을 실시간으로 변경할 수 없어서 상태 관리를 통해 구현해야 할 것 같다.