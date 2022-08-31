# Suspense on the server

[https://github.com/reactwg/react-18/discussions/37](https://github.com/reactwg/react-18/discussions/37)

---

**리액트의 서버 사이드 렌더링**

1. 서버에서 전체 앱에 대한 데이터를 가져옴.
2. 서버에서 전체 앱을 HTML로 렌더링하고 응답으로 보냄.
3. 클라이언트에서 전체 앱에 대한 Javascript 코드를 로드함.
4. Javascript 로직을 전체 앱을 대상으로 서버에서 생성한 HTML에 연결함. (Hydration)
    1. Dry한 HTML에 수분(Javascript)을 공급하는 것. (`<div id=”root”></div>` 만 있으니까!)

---

**리액트 서버 사이드 렌더링 과정의 문제점**

다음 단계가 시작되기 전에 각 단계가 전체 앱에 대해 *한번에 완료*되어야 한다는 것. (동기 방식 및 Waterfall 방식)

거의 모든 앱의 경우와 같이 앱의 일부가 다른 부분보다 느린 경우 이는 효율적이지 않다.

왜냐면 한 페이지에 여러 컴포넌트가 있을 때 HTML 생성이 오래걸리는 컴포넌트가 있고, 그렇지 않은 컴포넌트가 있는데 *빨리 생성된 컴포넌트가 다른 컴포넌트를 위해 계속 기다려야 하기 때문에* 비효율적이게 됨.

---

**React 18에서 Suspense로 문제 해결하기**

React 18을 사용하면 `<Suspense />`를 사용하여 *앱을 더 작은 독립 단위*로 나눌 수 있음.

이 단위는 서로 독립적인 단계를 거치며 앱의 나머지 부분을 차단하지 않음.

결과적으로 앱 사용자는 콘텐트를 더 빨리 보고 훨신 더 빠르게 상호작용할 수 있음.

앱에서 가장 느린 부분이 빠른 부분에 영향을 끼치지 않음.

이러한 개선 사항은 자동이며 작동하기 위해 특별한 조정 코드를 작성할 필요가 없음.

---

**서버 렌더링의 장점**

서버 렌더링을 하지 않을 때는 자바스크립트가 로드되기 전까지는 아무것도 볼 수가 없음. 하지만 서버 렌더링을 하면 자바스크립트가 로드되기 전에 버튼을 클릭하거나 상호작용할 수 있는 부분은 사용하지 못하지만, 자바스크립트가 로드되는 과정에서도 정적인 파일들을 화면에 보여줄 수가 있음. 그래서 네트워크가 느린 컴퓨터도 빠르게 화면을 볼 수가 있음.

이것뿐 아니라 index 생성이 쉽고 속도가 빨라 검색 엔진 순위에 도움이 됨.

---

**오늘날 서버 렌더링의 단점은?**

- 어떠한 것을 가져오기 위해서 모든 것을 fetch 해줘야 함.
- Hydrate를 하기 전에 모든 것을 로드해야 함.
- 어떤 것과도 상호 작용하기 전에 모든 것을 Hydrate 해야 함.

결국 All or Nothing…???

---

**이러한 방법을 해결하기 위해서 React 18에서 하는 것은?**

- Streaming HTML (Server)
    
    : 모든 데이터를 가져오기 전에 HTML 스트리밍 하는 방법. renderToString 대신에 renderToPipeableStream 메서드를 사용함.
    
    - 예로, `<Suspense>` 로 `<Comments>`를 감싸면 React가 페이지의 나머지 부분에 대해 HTML 스트리밍을 시작할 때까지 기다릴 필요가 없다고 React에게 알림. 대신 React는 `<Comments>` 대신 스피너를 보냄.
        
        이후에 댓글 데이터가 서버에서 준비되면 React는 추가 HTML을 동일한 스트림에 보내고, 해당 HTML을 “올바른 위치"에 넣기 위한 *최소한의 인라인 <script> 태그*를 보냄.
        
        결과적으로, React 자체가 클라이언트에 로드되기 전에 뒤늦은 `<Comments>` HTML이 ‘팝업' 됨.
        
    - 이것은 우리의 첫 번째 문제를 해결함. 이제 아무것도 표시하기 전에 모든 데이터를 가져올 필요가 없음. *화면의 일부가 초기 HTML을 지연시키는 경우 모든 HTML을 지연하거나 HTML에서 제외할지 선택할 필요가 없음.* 나중에 HTML 스트림에서 해당 부분이 “Pop in” 되도록 허용할 수 있음.
    - 기존 HTML 스트리밍과 달리 Top-Down 순서로 발생할 필요가 없음.

- Selective Hydration (Client)
    
    : 모든 코드가 로드되기 전에 Hydrate 하는 방법. 
    
    - 현재는 초기 HTML을 더 일찍 보낼 수 있지만, 여전히 문제가 있음. `<Comments>` 위젯에 대한 Javascript 코드가 로드될 때까지 클라이언트 앱에서 Hydrate를 시작할 수가 없음. 코드 크기가 크면 시간이 걸릴 수 있음.
        
        큰 번들을 피하기 위해 일반적으로 “코드 분할”(Code Splitting)을 사용함. *코드 조각을 동기적으로 로드할 필요가 없도록 지정하면* 번들러가 이를 별도의 <script> 태그로 분할함.
        
        `React.lazy`로 코드 분할을 사용하여, 기본 번들에서 `<Comments>` 코드를 분할할 수 있음.
        
        ```jsx
        import { lazy } from 'react';
        const Comments = lazy(() => import('./Comments.js'));
        
        // ...
        
        <Suspense fallback={<Spinner />}>
        	<Comments />
        </Suspense>
        ```
        
        이전 버전에서 이 방법은 서버 렌더링에서 작동하지 않았음.
        
    - `React.lazy`
        
        : 클라이언트 사이드 렌더링 단계에서 큰 번들의 JavaScript 코드들을 작은 Chunk들로 나누어 로드될 수 있게 해주는 역할. (리액트 18부터는 서버 사이드에서도 가능)
        
    - 서버 렌더링에서 순차적으로 Hydrating 하는 순서
        - HTML이 순차적으로 스트리밍되고 렌더링하는 비용이 큰 컴포넌트들은 `<Suspense>`로 감쌈으로 인해 해당 부분이 여전히 Fallback Element를 내보내고 있어도 그와 상관없이 페이지 다른 부분의 Hydrating을 진행함.
        - 나머지 부분까지 HTML 스트리밍 된 후 JS 번들이 로드된 컴포넌트들은 그 부분 또한 Hydrating 해줌.
        - *사용자의 인터랙션에 따라* 어떤 것을 먼저 hydration 시킬지에 대한 *우선순위*를 정할 수 있게 되었음. React는 클릭이 발생했음을 기록하고 더 긴급하기 때문에 댓글에 우선 순위를 부여함.
            
            React will record that the click happened, and prioritize hydrating the comments instead