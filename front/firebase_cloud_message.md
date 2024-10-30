# firebase cloud messaging 적용(Flutter)

- [flutterfire doc](https://firebase.flutter.dev/docs/overview)

<br>

## :pushpin: firebase setup

macos silicon chip 환경

```shell
brew install firebase-cli
firebase --version

firebase login
# google login
```
firebase 부터 설치 (brew 사용)

<br>

## :pushpin: flutterfire 구성

```shell
fvm dart pub global activate flutterfire_cli
cd [project_dir]
flutterfire configure --project=[firebase_project_name]
```

```
Exception: <internal:/Users/beanie.joy/.rbenv/versions/3.3.5/lib/ruby/3.3.0/rubygems/core_ext/kernel_require.rb>:136:in `require': cannot load such file -- xcodeproj (LoadError)
	from <internal:/Users/beanie.joy/.rbenv/versions/3.3.5/lib/ruby/3.3.0/rubygems/core_ext/kernel_require.rb>:136:in `require'
	from -e:1:in `<main>'
```
flutterfire configure 과정에서 오류 발생 가능성이 존재하는데 아래 링크 참고
(참고: https://github.com/invertase/flutterfire_cli/issues/127)

```shell
sudo gem install cocoapods && pod install
```
만약 ruby version 문제가 있다면 최신버전 적용 필요
(본인은 `rbenv` 설치해서 최신 버전 ruby install 적용)
