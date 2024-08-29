# Github Actions

## 구성

```
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
```
- `actions/checkout@v4`
github repository > CI server로 checkout
