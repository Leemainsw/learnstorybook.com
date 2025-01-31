---
title: '깃허브 액션으로 UI 테스트(UI Test) 자동화하기'
tocTitle: '자동화(Automate)'
description: '작업 흐름(workflow)을 더 빠르게 만들고, 더 고품질의 코드를 전달하기'
commit: 'c1c7372'
---

개발자는 평균적으로 버그를 고치는데 [매주 4-8 시간 정도](https://www.niss.org/sites/default/files/technicalreports/tr81.pdf)를 사용한다고 합니다. 버그가 production에 몰래 끼어들기 시작하면, 상황은 나빠지기만 합니다. 버그를 고치는데에는 [5-10배](https://www.cs.umd.edu/projects/SoftEng/ESEG/papers/82.78.pdf) 더 많은 시간이 듭니다. 이게 UI테스팅이 질 높은 사용자 경험을 전달하는데 필수적인 이유지만, 테스트에 많은 시간을 잡아먹히기도 합니다. 코드를 변경할 때마다 모든 테스트를 수작업으로 하나하나 하려하면 일이 지나치게 커집니다.

이런 작업흐름(workflow)를 자동화해서 개발자가 code를 push할 때마다 테스트가 작동하게 만들 수 있습니다. 테스트들은 background에서 실행되고 완료되면 결과를 보고합니다. 이는 회귀(regressions) 에러를 자동으로 감지할 수 있게 해줍니다.

이 챕터에서는 어떻게 이런 workflow를 [Github Actions](https://github.com/features/actions)을 이용해 구축할 수 있는지 보여줍니다. 더불어, 테스트 실행을 최적화하는 방법도 짚어보겠습니다.

## 지속적인 UI 테스트

코드 리뷰는 개발자가 되는데 중요한 부분입니다. 여러분이 버그를 조기에 발견하고 높은 코드 품질을 유지할 수 있게 돕습니다.

Pull request(PR)가 production 코드를 망가트리지 않는다는 걸 보장하기 위해서, 여러분은 보통 코드를 pull 해와서 테스트 suite를 로컬에서 실행해볼 겁니다. 이런 작업은 workflow 흐름을 끊고 시간도 오래 걸립니다. 지속적 통합(CI)을 통해서, 손으로 하나하나 개입하지 않고도 테스트의 이득은 모두 챙길 수 있습니다.

여러분은 UI를 바꿀 수도, 새로운 기능을 만들 수도, 의존성을 최신으로 업데이트할 수도 있다. Pull request를 열었을 때, CI 서버는 자동적으로 포괄적인 UI테스트를-시각, 구성, 접근성, 인터렉션, 사용자 흐름- 실행해줍니다.

테스트 결과로 PR뱃지를 받게 되고, 이를 통해 모든 검사 결과를 개략적으로 확인할 수 있습니다.

![](/ui-testing-handbook/image-19.png)

pull request가 모든 품질 검사를 통과했는지 한눈에 알 수 있습니다. 그 답이 "네(yes)"라면 실제 코드를 리뷰하는 단계로 넘어가면 됩니다. 그렇지 못한 경우에는, 로그를 살펴보면서 무엇이 잘못됐는지 찾습니다.

> "테스트는 의존성 최신화를 자동화해도 저에게 확신을 줍니다. 테스트가 통과하면, 최신화된 사항을 merge 시키죠."
>
> — [Simon Taggart](https://github.com/SiTaggart), Twilio 수석 엔지니어(Principal Engineer)

## 튜토리얼

지난 다섯 챕터들에서는 Taskbox UI의 다양한 면을 어떻게 테스트할 수 있는지 보여주었습니다. 여기에 더해서 우리는 이제 Github Actions를 이용해 지속적 통합(CI)을 구축해볼 것입니다.

### CI 구축하기

먼저 repository에 `.github/workflows/ui-tests.yml`파일을 만들면서 시작해봅시다. 한 [**작업흐름(workflow)**](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#workflows)은 자동화하고 싶은 여러 [**작업(jobs)**](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#jobs)으로 이루어집니다. workflow는 commit을 push한다던가 pull request를 만드는 것 같은 [**events**](https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows)와 함께 시작됩니다.

우리가 만들 workflow는 코드를 repository의 branch에 push하는 순간 실행하고, 3가지 작업(job)이 있습니다.

- 상호작용 테스트와 접근성 검사를 Jest와 Axe로 실행합니다.
- 시각 그리고 구성 테스트를 Chromatic으로 실행합니다.
- 사용자 흐름(user flow) 테스트를 Cypress로 실행합니다.

```yaml:title=.github/workflows/ui-tests.yml
name: 'UI Tests'

on: push

jobs:
  # 상호작용 테스트와 접근성 검사를 jest와 Axe로 실행
  interaction-and-accessibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dependencies
        run: yarn
      - name: Run test
        run: yarn test
  # 시각적 요소와 구성 테스트를 Chromatic으로 실행
  visual-and-composition:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0 # Required to retrieve git history
      - name: Install dependencies
        run: yarn
      - name: Publish to Chromatic
        uses: chromaui/action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # 이 토큰은 Chromatic 관리 페이지(manage page) 에서 가져온다
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
  # 사용자 흐름 테스트를 Cypress로 실행
  user-flow:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install dependencies
        run: yarn
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          start: npm start
```
기억해둡시다, Chromatic을 실행하려면, `CHROMATIC_PROJECT_TOKEN`이 필요합니다. 이걸 Chromatic의 관리 페이지(manage page)에서 가져올 수 있고 여러분의 repository secrets에 [넣어둬야 합니다](https://docs.github.com/en/actions/reference/encrypted-secrets). 반면에 `GITHUB_TOKEN`은 기본적으로 이용할 수 있게 되어 있습니다.

<figure style="display: flex;">
  <img style="flex: 1 1 auto; min-width: 0; margin-right: 0.5em;" src="/ui-testing-handbook/get-token.png" alt="get project token from Chromatic" />
  <img style="flex: 1 1 auto; min-width: 0;" src="/ui-testing-handbook/add-secret.png" alt="add secret to your repository" />
</figure>

마지막으로, 새 커밋을 만들고, 변경사항을 Github에 push하면, workflow가 실행되는 걸 볼 수 있어야 합니다!

![](/ui-testing-handbook/image-21.png)

### Cache dependencies 의존성 캐시하기

각 작업은 독립적으로 실행되고, 이는 CI 서버가 의존성을 세 작업 모두에서 설치해야 한다는 뜻입니다. 때문에 테스트 실행은 느려집니다. 이를 피하기 위해서 의존성을 캐시해놓고, 오직 lock file이 변경되었을 때에만 `yarn install`을 실행하도록 할 수 있습니다. 그러면 workflow를 `install-cache` 작업(job)을 포함해서 수정해봅시다.

```yaml:title=.github/workflows/ui-tests.yml
name: 'UI Tests'

on: push

jobs:
  # npm 의존성을 설치하고 캐시합니다.
  install-cache:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Commit
        uses: actions/checkout@v2
      - name: Cache yarn dependencies and cypress
        uses: actions/cache@v2
        id: yarn-cache
        with:
          path: |
            ~/.cache/Cypress
            node_modules
          key: ${{ runner.os }}-yarn-v1-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-v1
      - name: Install dependencies if cache invalid
        if: steps.yarn-cache.outputs.cache-hit != 'true'
        run: yarn
  # 상호작용 테스트와 접근성 검사를 jest와 Axe로 실행
  interaction-and-accessibility:
    runs-on: ubuntu-latest
    needs: install-cache
    steps:
      - uses: actions/checkout@v2
      - name: Restore yarn dependencies
        uses: actions/cache@v2
        id: yarn-cache
        with:
          path: |
            ~/.cache/Cypress
            node_modules
          key: ${{ runner.os }}-yarn-v1-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-v1
      - name: Run test
        run: yarn test
  # 시각적 요소와 구성 테스트를 Chromatic으로 실행
  visual-and-composition:
    runs-on: ubuntu-latest
    needs: install-cache
    steps:
      - uses: actions/checkout@v2
        with:
          fetch-depth: 0 # Required to retrieve git history
      - name: Restore yarn dependencies
        uses: actions/cache@v2
        id: yarn-cache
        with:
          path: |
            ~/.cache/Cypress
            node_modules
          key: ${{ runner.os }}-yarn-v1-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-v1
      - name: Publish to Chromatic
        uses: chromaui/action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          # Grab this from the Chromatic manage page
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
  # 사용자 흐름 테스트를 Cypress로 실행
  user-flow:
    runs-on: ubuntu-latest
    needs: install-cache
    steps:
      - uses: actions/checkout@v2
      - name: Restore yarn dependencies
        uses: actions/cache@v2
        id: yarn-cache'
        with:
          path: |
            ~/.cache/Cypress
            node_modules
          key: ${{ runner.os }}-yarn-v1-${{ hashFiles('**/yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-v1
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          start: npm start
```

나머지 세 작업도 `install-cache`작업이 끝나고 캐시된 의존성을 사용할 수 있을 때까지 기다리도록 약간 고쳤습니다. 이 workflow를 다시 실행해보기 위해 다른 commit을 push해봅시다.

성공! 여러분은 테스팅 workflow를 자동화했습니다. 이제 PR을 열면 알아서 Jest, Chromatic 그리고 Cypress를 병렬로 실행하고 PR page에 그 결과를 보여줄 것입니다.

![](/ui-testing-handbook/image-22.png)

## UI 테스팅 workflow를 정복하기

우리가 만든 testing workflow는 Storybook을 이용해 독립된 컴포넌트로 시작합니다. 코드를 짜는 동안 검사를 실행하면서 더 빠른 피드백 고리(feedback loop)를 만들 수 있습니다. 마지막으로 여러분의 모든 테스트 suite를 지속적 통합(Continuous Integration)을 이용해서 실행해보도록 합시다.
Chapter 8은 이러한 완전한 작업 흐름(workflow)을 실제로 보여줍니다. 이제 새 기능을 production에 올리기 전에 어떻게 테스트할 수 있는지 보겠습니다.
