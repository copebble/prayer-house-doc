# Riverpod

flutter의 상태 관리 플러그인 중 하나
provider로 구현할 수 있는 상태관리 모듈 중에 최근에 떠오르고 있는 패키지라 선택하게 되었음

<br>

## :pushpin: provider build 시점

riverpod provider는 watch를 통해서 처음 notifier가 build되고 해당 state를 계속 트래킹하는 것인줄 알았는데 read를 통해 읽어도 최초 build되지 않았으면 build를 먼저 진행하고 read로 불러온 notifier의 function을 호출하는 것을 확인

```dart
// 적절한 context 사용
final container = ProviderScope.containerOf(context);

// 1. provider.notifier 호출 후 function 실행
container.read(errorPageMsgStateProvider.notifier).setMessage(message);

// ...
@override
Widget build(BuildContext context, WidgetRef ref) {
    // 2. watch로 provider call
    final message = ref.watch(errorPageMsgStateProvider);

    return Scaffold(
        resizeToAvoidBottomInset: false,
        body: CommonAlertMsg(
        message: message ?? '내부 서버 오류가 발생하였습니다.',
        icon: Icons.error,
        color: Colors.red.shade300,
        ),
    );
}
```
1번 시점에서 `errorPageMsgStateProvider`가 build되지 않은 상태여도 read한 순간 build를 먼저 수행 후
`setMessage` function이 동작한다고 보면 된다.