[https://github.com/reactwg/react-18/discussions/41](https://github.com/reactwg/react-18/discussions/41)

[https://github.com/reactwg/react-18/discussions/65](https://github.com/reactwg/react-18/discussions/65)

---

**startTransition**

React 18에서는 업데이트 중에도 앱의 응답성(!)을 유지하는 데 도움이 되는 새로운 API를 도입함.

이 새로운 API를 사용하면 특정 업데이트를 “Transition”으로 표시하여 사용자 상호 작용을 크게 개선할 수 있음.

React를 사용하면(?), 상태 전환 중에 시각적 피드백을 제공하고 전환이 발생하는 동안 브라우저의 응답성을 유지할 수 있음.

이 기능은 리액트에서 어떠한 업데이트가 Urgent 하며 어떠한게 그러하지 않은지 알려줌. 그래서 상태 업데이트를 하는데 우선순위를 주게 됨.

대표적으로 검색 기능을 구현할 때 검색하는 input은 이벤트에 따라서 리렌더링이 해당화면에 업데이트 되어야 함.

하지만 그 아래 검색결과도 이에 따라 업데이트가 되는데,

검색 결과 리스트가 많지 않더라도 내부적으로 검색 결과를 가져오는데 많은 작업을 진행할 수 있기에 검색창에 타이핑을 하는 것에 따라 바로 바로 검색 결과도 업데이트를 하면 성능에 문제가 생길 수 있음.

그러하기에 이 부분은 검색 창과 결과 창 두 부분으로 나눌 수 있으며, 유저가 타이핑하는 것에 따라 즉각 반영되기를 기대하는 검색창, 그리고 검색 창보다는 UI 업데이트가 느린 것에 자연스럽게 받아들여져야 하는 결과 창으로 나눌 수 있음.

- Urgent Updates 검색창 → 버튼 클릭, 키보드 입력과 같이 직관적으로 보았을 때 업데이트가 즉각적으로 일어나는 것을 기대하는 상태 값들을 대상으로 함.
- Transition Updates 결과 창 → 사용자의 상태 값의 변화에 따른 모든 업데이트가 뷰에 즉각적으로 일어나는 것을 기대하지 않음.

```jsx
// Urgent: Show what was typed
setInputValue(input);

// Not urgent: Show the results
setSearchQuery(input);
```

---

**어떻게 이러한 문제점을 개선할 수 있음?**

새로운 startTransition API는 업데이트를 “Transition”으로 표시할 수 있는 기능을 제공하여 이 문제를 해결함.

이 API로 리액트에게 상태 업데이트 하는데 우선 순위를 정해주는 것임.

```jsx
import { startTransition } from "react";

// Urgent: Show what was typed
setInputValue(input);

startTransition(() => {
  // Secondary: Show the results
  setSearchQuery(input);
});
```

startTransition에 래핑된 업데이트는 *긴급하지 않은 것으로 처리*되며 클릭이나 키 누름과 같은 더 긴급한 업데이트가 들어오는 경우 중단됨. Transition이 사용자에 의해 중단되면 (ex. 여러 문자를 연속으로 입력) React는 다음을 throw 함. 완료되지 않은 오래된 렌더링 작업을 제거하고 최신 업데이트만 렌더링함.

Transition을 사용하면 UI가 크게 변경되더라도 대부분의 상호 작용을 빠르게 유지할 수 있음. 또한 더 이상 관련이 없는 콘텐츠를 렌디렁하는 데 시간을 낭비하지 않아도 됨.

---

**Transition이 보류 중인 동안(?) 어떻게 해야함?**

검색 창에 타이핑을 했을 때 startTransition API로 인해 결과 창에는 UI 업데이트 우선순위가 밀려서 업데이트 보류가 일어날 때는 아래와 같이 isPending이 true로 되기에 isPending이 true일 시에 Spinner 같은 컴포넌트를 보여주면 됨.

```jsx
import { useTransition } from "react";

const [isPending, startTransition] = useTransition();
```

```jsx
{
  isPending && <Spinner />;
}
```

---

**리액트18 이전에는 어떻게 이러한 문제를 처리함?**

startTransition이 없을 때는

- State를 두 개로 나눠서 따로 처리를 해주거나(?)
  - state가 두 개니 업데이트 처리 방법을 다르게 해줌.
- debounce를 이용해서 처리하거나
- setTimeout을 이용해서 처리함.
  - debounce를 이용하거나 setTimeout을 이용하는 것은 결국 모든 이벤트가 Schedule 되어 있고 _뒤로 밀리는 것이기 때문에_ 이벤트가 끝나도 계속 결과를 표출하게 됨.

```jsx
onChange = {(e, nextValue) => {
	// Update slider
	setValue(nextValue);

	setTimeout(() => {
		// Update results
		onChange(nextValue);
	}, 0)
}}
```

---
