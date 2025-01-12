---
title: RN 브릿지리스(Bridgeless)에 따른 React18 기능 도입
description: RN 0.76에 정식적으로 도입되는 React18 기능에 대해 알아보자
date: 2025-01-02
categories: [react-native]
image: /assets/img/default/react-native.png
tags: [React Native]
pin: false
---

## 리액트 네이티브 브릿지(Bridge)란?
---
브릿지는 리액트 네이티브의 Old Architecture에서 사용되는 하나의 통신 모드이다.

Old Architecture로 동작하는 리액트 네이티브는 네이티브 모듈을 호출하기 위해 자바스크립트를 사용했다. 그러나 서로의 언어가 다르기에 자바스크립트에서 네이티브 모듈을 직접 호출할 수 없었으며 두 엔진 간에 통신이 가능하게 도와주는 통로가 필요했는데 그것이 __"브릿지"__ 이다.

자세한 내용은 [React Native 브릿지(Bridge)와 렌더링 과정
](https://wonhyunkwon.github.io/posts/bridge/) 포스트를 참고하면 도움이 될 것이다.

## Transition
---
__"Transition"__ 란, 긴급한 업데이트와 그렇지 않은 업데이트를 구분하는 새로운 개념이다. 쉽게 설명하자면 __비동기 UI 업데이트를 보다 부드럽게 처리할 수 있도록 도와주는 기능__ 이며 React에서 화면을 업데이트할 때 __사용자 인터페이스의 상태를 관리하고 성능을 최적화하는 데 도움을 준다.__

일반적으로 생각하는 최상의 사용자 경험이란, __하나의 사용자 입력이 들어왔을 때 긴급 업데이트와 그렇지 않은 업데이트 모두를 트리거해야 한다.__ 

- __"긴급 업데이트(Urgent updates)"__ : 단순하면서 직관적인 작업 (예: press, onChange 등)
- __"전환 업데이트(Transition updates)"__ : UI를 한 뷰에서 다른 뷰로 전환 (Animation, 검색 결과 갱신 등)

즉, Transition은 덜 긴급한 업데이트를 처리하는 데 사용된다. UI가 여전히 응답성을 유지하면서 백그라운드에서 더 복잡한 작업을 처리할 수 있다. 특히 __"복잡한 UI"__ 나 __"대규모 데이터 처리"__ 가 필요한 부분에서 빛을 발한다.

#### 언제 사용해야 될까?
1. __UI가 느려질 때__ : 많은 데이터가 포함된 리스트나 검색 필터링을 처리할 때
2. __부드러운 전환 필요__ : 버튼 클릭, 텍스트 입력과 같은 직관적이면서 사용자 경험을 위해 바로 보여주는 게 필요한 긴급한 작업이 있을 경우. 즉, 작업 처리가 오래걸리는 작업과 분리하여 보여주고 싶은 경우. 이 경우 비긴급 업데이트 상태를 `isPending` 으로 로딩 상태를 표시해준다.
3. __성능 최적화__ : UI가 "끊기지 않고" 매끄럽게 작동하도록 원할 경우

#### `useTransition`
상태 업데이트를 __"전환 작업 으로 분류할 때"__ 사용하는 훅이다.

다음 두 가지 값을 반환한다.
1. `isPending` : 전환 작업이 진행 중인지 여부 (로딩 상태 관리 가능)
2. `startTransition` : 비긴급 업데이트를 수행하는 UI 상태 변경 함수. 상태 변화를 일으키는 콜백함수를 래핑하고, 래핑된 해당 콜백함수는 __낮은 우선순위__ 로 실행된다.

예시
```tsx
import React, { useState, useTransition } from 'react'

function SearchApp() {
  const [value, setValue] = useState('')
  const [results, setResults] = useState([])
  const [isPending, startTransition] = useTransition()

  const handleSearch = (e) => {
    const value = e.target.value

    // 긴급한 업데이트
    setValue(value)

    startTransition(() => {
      // 긴급하지 않은 상태 업데이트 > 작업을 백그라운드로 연기
      const filteredResults = fetchData(value) // 가상 검색 함수
      setResults(filteredResults)
    })
  }

  

  return (
    <div>
      <input
        type="text"
        value={value}
        onChange={handleSearch}
        placeholder="Search..."
      />
      {/* isPending 일 경우 Loading 상태임을 알림 */}
      {isPending && <p>Loading...</p>}
      <ul>
        {results.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>
    </div>
  )
}

function fetchData(data) {
  // 가상 데이터 필터링
  return ['apple', 'banana', 'cherry'].filter((item) =>
    item.includes(data)
  )
}

export default SearchApp
```

## 자동 배치(Automatic Batching)
---
리액트 컴포넌트에서는 하나의 상태만 가지고 있기 보단 두 개 이상의 상태를 가지고 있는 경우가 훨씬 많다.

이 경우 상태가 하나씩 업데이트 될 때 마다 컴포넌트는 리렌더링을 수행하게 된다. 만약에 트리가 깊고 복잡해질수록 다루는 상태가 많아지게 되고, 그 만큼 리렌더링 횟수가 많이 늘어나게 되는 문제가 발생한다.

이러한 문제를 해결하기 위해 리액트18부터 나온 기능이 바로 __"자동 배치(Automatic Batching)"__ 기능이다. 핵심은 __"하나의 컴포넌트 안에서 여러 개의 상태를 하나의 그룹으로 묶어서 업데이트를 해준다."__

이 기능은 따로 사용 방법이 있는 것이 아니라 말 그대로 자동으로 배치해주기 때문에 사용자는 크게 사용함에 있어 신경 쓸 필요는 없다.

예시
```tsx
function AutomaticBatching() {
  const [count, setCount] = useState(0)
  const [flag, setFlag] = useState(false)

  useEffect(() => {
    console.log('rendered')
  })

  const handleClick = () => {
    setCount(c => c+1)
    setFlag(f => !f)
  }

  return (
    <View>
      <Button onClick={handleClick} />
      <Text style={% raw %}{{ color: flag ? 'blue' : 'black' }}{% endraw %}>
        {count}
      </Text>
    </View>
  )
}
```

다음과 같이 `handleClick` 안에서 두 개의 상태를 업데이트 한다고 하자. 자동 배치가 없던 아키텍처에선 두 개의 상태인 `setCount`, `setFlag` 상태를 업데이트 하니 두 번의 리렌더링을 기대하게 된다. 그러나 __자동 배치 기능에 의하여 두 개의 상태를 업데이트 하더라도 리렌더링은 오직 한번만 일어난다.__

#### 만약 그룹화를 원하지 않고 상태 업데이트 마다 렌더링을 진행하고 싶다면?
`flushSync` 메서드를 사용하자. 각 상태가 바뀌길 기대하는 함수마다 작성해주면 된다.

예시
```ts
const handleClick = async () => {
  await flushSync(() => {
    setCount(c => c+1)
  })

  await flushSync(() => {
    setFlag(f => !f)
  })
}
```

## useLayoutEffect
---
이전 아키텍처에서는 레이아웃 정보를 읽기 위해 `onLayout` 이벤트로 비동기적으로 호출했다. 그러나 서로 다른 쓰레드에서 각각 처리하다 보니 레이아웃을 업데이트 할 때 사용자가 원하는 값에 벗어난 잘못된 레이아웃이 표시되는 문제가 발생했다. (아래 이미지 참고)

![](/assets/img/2024-12-29/4.gif){: style="display: center; width: 250px; margin: 0 auto;"}
_비동기화로 인한 렌더링의 문제 예시_

따라서 이벤트 루프와 동기적으로 레이아웃을 읽을 수 있는 기능을 출시했고 이 것이 바로 __"useLayoutEffect"__ 훅이다. useLayoutEffect 내에서 레이아웃 정보를 동기적으로 읽고 동일한 프레임에서 UI를 업데이트할 수 있으며, 최종적으로 위와 같이 레이아웃과 상태가 싱크가 맞지 않는 문제를 해결할 수 있다.

![](/assets/img/2025-01-02/1.gif){: style="display: center; width: 250px; margin: 0 auto;"}
_useEffectLayout로 레이아웃의 정확한 위치로 배치_

`useLayoutEffect`의 사용법은 `useEffect`와 동일하다. 다만, 두 훅의 차이점은 __"렌더링 후 layout과 paint 전에 동기적(useLayoutEffect)으로 실행되냐 비동기적(useEffect)으로 실행되냐 이다."__

## Suspense
---
__"Suspense"__ 는 __"컴포넌트의 렌더링을 일시 중지하고 데이터 로딩을 기다릴 수 있게 해주는 리액트 기능"이다.__

보통 REST API와 같은 비동기적 네트워크 호출 시 작업이 끝날 때 까지 잠시 렌더를 멈추고 indicator, skeleton UI 등을 보여준다.

Suspense의 핵심 개념으로는 다음과 같다.

1. __lazy__ : 동적 import를 통해 컴포넌트를 필요한 시점에 로드하는 기능. (FlatList처럼 실제 View 컨텐츠를 렌더가 필요한 시점에 load)
2. __fallback__ : 컴포넌트가 로딩 중일 때 보여줄 UI를 설정하는 prop. 일반적으로 fallback으로 로딩 스피너, 스켈레톤 UI 등이 있다.

#### 리액트 Suspense를 사용해야 하는 이유?
1. __깔끔한 데이터 로딩 처리__ :  데이터 로딩 상태를 __선언적__ 으로 처리할 수 있다. (즉, if 분기 처리가 생략된다.)
2. __초기 로딩 속도 개선__ : 개발자가 직접 상태를 관리하여 UI를 분기 처리하는 것 대비 번들 크기도 작고 로딩 속도도 다소 개선된다.
3. __에러 처리 용이__ : Error boundary와 함께 사용하여 독립적으로 피드백 받는 것이 가능하기에 어플리케이션의 다른 부분에 영향을 주지 않는다.
4. __UI의 waterfall 현상 방지__ : waterfall이란, 하나의 스크린에서 컴포넌트 내의 여러 개 비동기 처리 순서에 따라 부분적으로 UI가 나타나는 현상이다. 이를 Suspense로 래핑함으로써 해결 가능하다.

다음은 Suspense 사용 전의 코드이다.

```tsx
function User({ userId }) {
  const [isLoading, setIsLoading] = useState(false)
  const [user, setUser] = useState([])

  useEffect(() => {
    try {
      setIsLoading(true)

      const response = await fetchApi()
  
      if (!response) {
        setUser(response)
      }
    } catch (e) {
      console.log(e)
    } finally {
      setIsLoading(false)
    }
  }, [])

  if (isLoading) {
    return <Spinner />
  }

  return (
    <View>
      <Posts userId={userId} />
    </View>
  )
}

function Posts({ userId }) {
  const [isLoading, setIsLoading] = useState(false)
  const [posts, setPosts] = useState([])

  useEffect(() => {
    try {
      setIsLoading(true)

      const response = await fetchApi()
  
      if (!response) {
        setPosts(response)
      }
    } catch (e) {
      console.log(e)
    } finally {
      setIsLoading(false)
    }
  }, [])

  if (isLoading) {
    return <Spinner />
  }

  return (
    <View>
      {posts.map(post => (
        <Text>{post.id}</Text>
      )})}
    </View>
  )
}

function App() {
  return <User userid={1}>
}
```

`App` > `User` > `Posts` 순서로 부모-자식 간의 UI 호출을 하는 코드이다.

여기서 `User`, `Posts` 두 컴포넌트는 공통적으로 __1) 비동기 API 호출__ 을 하고 있고, __2) 데이터 수신 상태에 따라 알맞은 UI를 제공__ 하고 있다는 점이다.

이렇게 한 페이지 상의 여러 컴포넌트에서 동시에 비동기 데이터를 읽어오는 경우, 마치 UI가 폭포수처럼 순차적으로 그려지게 되며 경우가 많아질수록 사용자 경험을 해칠 위험이 크다.

뿐만 아니라 비동기 통신은 반드시 요청한 순서대로 데이터가 응답된다는 보장이 없기 때문에 의도치 않게 싱크가 맞지 않은 상태의 데이터가 제공 될 가능성도 있다.

개발자 입장에선 각각 모두 `if` 절로 분기 처리를 해야하며 컴포넌트 안에 커플링이 되어 코드가 읽기 어려워지고 테스트 코드를 작성하기도 힘들어진다.

다음은 동일한 목적으로 Suspense를 사용했을 때의 코드다.

```tsx
function User({ userId }) {
  const [user, setUser] = useState([])

  useEffect(() => {
    try {
      const response = await fetchApi()
  
      if (!response) {
        setUser(response)
      }
    } catch (e) {
      console.log(e)
    }
  }, [])

  return (
    <Suspense fallback={<Spinner />}>
      <Posts userId={userId} />
    </Suspense>
  )
}

function Posts({ userId }) {
  const [posts, setPosts] = useState([])

  useEffect(() => {
    try {
      const response = await fetchApi()
  
      if (!response) {
        setPosts(response)
      }
    } catch (e) {
      console.log(e)
    }
  }, [])

  return (
    <View>
      <Text>
        {posts.map(post => (
          <Text>{post.id}</Text>
        ))}
      </Text>
    </View>
  )
}


function App() {
  // lazy를 통한 컴포넌트 동적 로딩
  const DataComponent = lazy(() => import('./User'))

  return (
    <Suspense fallback={<Spinner />}>
      <DataComponent userid={1} />
    </Suspense>
  )
}
```

이렇게 각각 Suspense로 래핑해주면 waterfall 현상이 사라지며, 코드 측면에서도 데이터 로딩과 UI 렌더링이 완전히 분리되어 코드 가독성과 유지 보수성이 향상된다.

## 도움글
---
[React Native 공식 문서](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here#uselayouteffect)

[자동 배치(Automatic Batching)](https://arc.net/l/quote/gpqugglc)