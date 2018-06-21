---
layout: post
title: "자바스크립트 개발자를 위한 AST(번역)"
author: "Gyujin Cho"
description: "AST for JavaScript developers(by Bohdan Liashenko)의 번역글입니다."
image: "/assets/2018-06-19-AST/03.png"
---

이 글은 ITNEXT Medium에 [Bohdan Liashenko](https://www.linkedin.com/in/bohdan-liashenko-bb365854/)가 기고한 **[AST for JavaScript developers](https://itnext.io/ast-for-javascript-developers-3e79aeb08343)**의 번역입니다. 저자에게는 허락을 구하고 번역하였습니다. 혹시 이상하거나 어색한 부분이 있다면 [marina.gyujin.cho@gmail.com](mailto:marina.gyujin.cho@gmail.com) 으로 알려주세요.

-----------
<br>
> **TL;DR** 이 글은 제가 스톡홀름 ReactJS 밋업에서 최근에 발표한 내용입니다.<br> [여기(Slideshare)](https://www.slideshare.net/BohdanLiashenko/ast-for-javascript-developers)에서 슬라이드를 확인하실 수 있습니다. <br>
> [LinkedIn에서 원문 공유하기](https://www.linkedin.com/cws/share?url=https%3A%2F%2Fitnext.io%2Fast-for-javascript-developers-3e79aeb08343%3Futm_source%3Dmedium_sharelink%26utm_medium%3Dsocial%26utm_campaign%3Dbuffer)

## 왜 AST(Abstract Syntax Tree)를 알아야 할까요?

아무 모던 JS 프로젝트나 하나 골라서 `devDependencies`를 확인하면 지난 몇 년간 JS 툴들이 얼마나 발전했는지 알 수 있을 것입니다. JavaScript 툴링, 코드 압축(minification), CSS 전처리기, eslint, prettier 등등 문자 그대로 도구들이 떼를 지어 있습니다. 이들은 프로덕션 코드에는 포함되지 않는 JavaScript 모듈이지만 개발 과정에서 매우 중요한 역할을 합니다. 이 도구들은 모두 AST 처리를 기반으로 구축되었습니다.

![AST가 뭐죠](/assets/2018-06-19-AST/01.png "AST가 뭐죠")
_이 툴들이 다 AST 기반으로 만들어졌습니다._

이 글은 이렇게 전개됩니다. 먼저 AST가 무엇인지 그리고 어떻게 plain 코드에서 AST를 만들어내는지 알아봅니다. 그리고 AST 처리 위에 만들어진 도구 중 가장 많이 쓰이는 사례들을 살펴봅니다. 마지막으로 AST로 무엇을 만들 수 있는지에 대한 좋은 데모이자 제 프로젝트인 js2flowchart 에 대해서 소개하며 마무리할 예정입니다. 자, 그럼 시작하겠습니다.

![목차](/assets/2018-06-19-AST/02.png "목차")

## Abstract Syntax Tree가 뭘까요?

프로그래밍 언어의 문법에 따라 소스 코드 구조를 표시하는 계층적 프로그램 표현 (respresentation) 입니다. 각 AST 노드는 소스 코드의 항목(item)에 해당합니다.

![뭐라고요](/assets/2018-06-19-AST/03.png "뭐라고요")
_뭐라고 하신 거죠_

좋습니다. 예시를 보죠.

![아주 단순한 AST 예시](/assets/2018-06-19-AST/04.png "아주 단순한 AST 예시")
_매우 단순화된 예시입니다._

매우 단순하지만 이것이 핵심 개념입니다. 일반 텍스트로부터 트리와 같은 데이터 구조를 얻었습니다. 코드 항목(item)이 트리의 노드와 일치합니다.

### 어떻게 일반 코드에서 AST를 추출하나요? 

네, 아시다시피 **컴파일러**가 이미 그 일을 하고 있습니다. 일반적인 컴파일러를 확인해봅시다.

![컴파일러 겉핥기](/assets/2018-06-19-AST/05.png "컴파일러 겉핥기")
_컴파일러 겉핥기를 해봅시다._

다행히도, 우리는 고수준 언어로 된 코드를 비트(bit)로 변환하는 모든 단계를 살펴볼 필요는 없습니다. **어휘 분석 및 구문 분석(Lexical and Syntax Analysis)**만 보면 됩니다. 이 두 단계는 코드를 기반으로 AST를 생성하는 데 있어 주요 역할을 담당하고 있습니다.

![어휘 분석기](/assets/2018-06-19-AST/06.png "어휘 분석기")
_어휘 분석기(스캐너)는 코드 문자열을 토큰 목록으로 변환합니다._

첫 번째 단계입니다. **스캐너(scanner)라고도 하는 어휘 분석기**는 정의 된 규칙을 사용하여 문자 스트림(코드)을 읽고 이를 토큰으로 결합합니다. 공백 문자, 주석 등도 제거합니다. 결국 전체 코드 문자열이 토큰 목록으로 분할됩니다.

어휘 분석기는 소스 코드를 읽을 때 코드를 문자 단위로 하나하나 스캔하며 공백, 연산자 기호 또는 특수 기호를 발견하면 단어가 완성되었다고 봅니다.

![구문 분석기](/assets/2018-06-19-AST/06-2.png "구문 분석기")
_구문 분석기(파서)는 토큰 목록을 AST로 변환합니다._

두번째 단계인 **구문 분석기는 파서(parser)**라고도 하며, 어휘 분석 후 만들어진 플랫한 토큰 목록을 가져 와서 **언어 구문을 검증**하고 (구문 오류가 있다면 에러를 표시하는) **트리 구조로 변환**합니다.

일부 파서는 트리를 생성하면서 불필요한 토큰(예: 중복 괄호)을 생략하여 '추상 구문 트리(Abstract Syntax Tree)’를 만듭니다. 코드와 100% 일치하지는 않지만 일을 처리하기엔 충분하죠. 반면에, 모든 코드 구조를 완전히 커버하는 파서는 'Concrete Syntax Tree’라고 부르는 트리를 생성합니다.

![최종 AST](/assets/2018-06-19-AST/07.png "최종 AST")
_위 두 단계를 거쳐 뽑아낸 AST_

### 컴파일러에 대해 더 알아보고 싶다면?

![The-super-tiny-compiler](/assets/2018-06-19-AST/08.png "The-super-tiny-compiler")
_[https://github.com/jamiebuilds/the-super-tiny-compiler](https://github.com/jamiebuilds/the-super-tiny-compiler)_

**The-super-tiny-compiler** 레포를 살펴보는 것부터 시작할 수 있습니다. 컴파일러의 모든 주요 기능들을 아주 단순화하여 자바스크립트로 작성한 예시입니다. 실제 코드는 200줄 정도이며, Lisp를 C 언어로 컴파일하는 내용입니다. 모든 코드에 주석과 설명이 달려 있습니다.

![LangSandbox](/assets/2018-06-19-AST/09.png "LangSandbox")
_[https://github.com/ftomassetti/LangSandbox](https://github.com/ftomassetti/LangSandbox)_

**LangSandbox**라는 다른 좋은 프로젝트도 있습니다. 이 레포는 프로그래밍 언어를 작성하는 방법을 설명합니다. (더 선호한다면) 프로그래밍 언어를 어떻게 작성하는지에 대한 기사나 책의 목록도 있습니다. 여기서는 앞의 The-super-tiny-compiler와 같이 lisp를 C로 컴파일하는 대신에 직접 언어를 작성하고 C/바이트 코드로 컴파일하여 실행할 수 있기 때문에 좀 더 진도를 나갑니다.

### 그냥 라이브러리를 쓰면 안 되나요? 

물론 되죠, 많은 라이브러리가 있습니다. [AST Explorer](https://astexplorer.net/)에서 맘에 드는 것을 하나 고르면 됩니다. AST 파서를 실행할 수 있는 실시간 에디터입니다. JavaScript 외에도 다른 많은 언어를 지원합니다.

![AST Explorer](/assets/2018-06-19-AST/10.png "AST Explorer")
_[https://astexplorer.net/](https://astexplorer.net/)_

그 중에, 제 생각에 정말 좋은 라이브러리인 **Babylon**을 특히 강조하고 싶습니다.

![Babylon](/assets/2018-06-19-AST/11.png "Babylon")
_Babylon은 Babel에서 쓰이는 JS 파서입니다. JSX, TypeScript, Flow를 지원합니다._

Babylon은 Babel에서 사용하고 있고, 사실 그래서 인기가 많을 겁니다. _(역주: 최근 [`@babel/parser`](https://github.com/babel/babel/tree/master/packages/babel-parser)로 통합되었습니다.)_ Babel 프로젝트가 지원하기 때문에 지난 몇 년 간 꽤 자주 업데이트되어 온 새로운 JS 기능을 항상 최신 상태로 지원할 거라고 기대할 수 있습니다. 그러니까, ‘asynchronous iteration’(이건 뭐건 간에) 이런 새로운 기능이 추가되어도 이 파서는 `Unexpexted token`을 던지지 않을 거란 말이죠. 또, 꽤 좋은 API를 가지고 있으며 일반적으로 사용하기 쉽습니다.

좋습니다. 이제 AST를 생성하는 법을 알았으니 실제 사용 사례로 넘어가 볼까요.

## Use Cases

### Babel

제가 첫 번째로 이야기하고 싶은 케이스는 코드 **Transpiling**과, 말할 것도 없이, **Babel**입니다.

> Babel은 'ES6 지원 도구'가 아닙니다. 뭐 물론 그것도 맞지만, 그게 전부는 아닙니다.

![Babel](/assets/2018-06-19-AST/12.png "Babel")
_보통 트랜스파일러라고 부르는 소스를 소스로 변환하는 JS 컴파일러입니다.<br>[https://github.com/babel/babel](https://github.com/babel/babel)_

많은 이들이 Babel하면 ES6/7/8 기능의 지원을 떠올립니다. 그리고 사실, 그게 보통 우리가 Babel을 사용하는 이유입니다. 하지만 ES6/7/8 지원은 단지 플러그인 그룹 중 하나에 불과합니다. Babel은 코드 압축(minification)에도 사용할 수 있고 React와 관련된 문법(예를 들면 JSX 같은)을 트랜스파일하는 데에나 Flow를 위한 플러그인으로도 사용할 수 있습니다.

**Babel은 JS 컴파일러입니다.** 크게 보면 코드를 파싱, 변환 및 생성하는 3단계로 구성됩니다. 그럼 코드를 어떻게 변환할까요? 맞습니다! AST를 구축하고, 적용된 플러그인에 기반하여 AST를 탐색하며 수정한 후, 수정된 AST에서 새로운 코드를 생성하고 있습니다.

간단한 코드 예제를 보겠습니다.

![Babel Example](/assets/2018-06-19-AST/13.png "Babel Example")

앞에서 언급했듯이 Babel은 Babylon을 사용하므로 먼저 코드를 파싱합니다. 그 후 AST를 탐색하고 모든 변수 이름을 변환합니다. 마지막 단계로 코드를 생성합니다. 끝입니다. 보시면 아시겠지만 1단계(구문 분석)과 3단계(코드 생성)은 매번 수행할 작업, 공통 단계입니다. 그래서 이 단계는 Babel이 알아서 처리합니다. 우리가 정말 관심 있는 것은 **AST 변형**이기 때문입니다.

그래서 Babel 플러그인을 개발할 때는 AST를 변환할 노드 **"방문자(Visitor)"**만 작성하면 됩니다.

![Babel Plugin](/assets/2018-06-19-AST/14.png "Babel Plugin")
_AST를 변형하는 게 전부입니다.<br>이 플러그인을 Babel Plugin 목록에 넣고, Webpack Babel Loader를 설정하면 됩니다. 식은 죽 먹기죠._

Babel 플러그인을 만드는 방법에 대해 자세히 알고 싶으면 **[Babel-handbook](https://github.com/jamiebuilds/babel-handbook)**을 읽어보세요.

### jscodeshift

다음으로, **코드 리팩터링 자동화**와 **jscodeshift**에 대해 언급하고 싶습니다.<br>
모든 구식 익명 함수(anonymous function)를 짧고 멋진 화살표 함수(arrow function)로 바꾸고 싶다고 합시다.

![jscodeshift_Code_1](/assets/2018-06-19-AST/16.png "jscodeshift_Code_1")
_원클릭으로 코드를 정리할 수 있습니다._

대부분의 코드 에디터는 이 작업을 할 수 없습니다. 간단한 찾기 - 바꾸기 작업이 아니기 때문이죠.
바로 여기서 jscodeshift가 활약합니다.

jscodeshift를 언급할 때 대부분 **‘codemod'**와 함께 언급되기 때문에 처음에는 혼란스러울 수 있습니다.

 - jscodeshift는 ‘codemod'를 실행하기 위한 툴킷입니다.
 - ‘codemod’는 AST에 대해 실제로 어떤 변환이 이루어져야 하는지를 설명하는 코드입니다.

 Babel 및 그 플러그인과 아주 비슷한 개념이죠.

![jscodeshift_Code_2](/assets/2018-06-19-AST/17.png "jscodeshift_Code_2")
_Babel 플러그인과 거의 똑같이 생겼죠._

코드를 새로운 버전의 프레임워크로 자동 마이그레이션하고 싶다면 다음과 같이 하면 됩니다. 기존 React의 PropTypes를 React 16의 prop-types로 리팩토링한다고 해보죠.

![jscodeshift](/assets/2018-06-19-AST/18.png "jscodeshift")
_React 업데이트/Migration: App 코드 전체에서 PropTypes를 prop-types로 변환해 줍니다.<br>(모두 이미 16버전으로 마이그레이션했기를 바랍니다.)_

한 번 사용해보세요. 이미 많은 codemode가 만들어져 있고, 코드를 수동으로 변환하지 않으면 시간을 절약할 수 있습니다.

![jscodeshift](/assets/2018-06-19-AST/19.png "jscodeshift")
_[https://github.com/facebook/jscodeshift](https://github.com/facebook/jscodeshift)<br>[https://github.com/reactjs/react-codemod](https://github.com/reactjs/react-codemod)_

### Prettier

Prettier도 가볍게 짚고 넘어가고 싶은데요, 아마 모두들 매일 사용하고 계실 테니까요.

![Prettier](/assets/2018-06-19-AST/20.png "Prettier")

Prettier는 코드의 형식을 변환합니다. 긴 줄을 끊고, 공백과 괄호 등을 제거합니다. 그러려면 코드를 입력 받아서 수정 된 코드를 출력으로 반환해야 합니다. 어디서 들어본 것 같죠? 맞습니다.

![Prettier](/assets/2018-06-19-AST/21.png "Prettier")
_IR(Intermediate Representation)은 AST 노드가 어떤 형식으로 바뀔지 서술합니다.<br>[https://github.com/prettier/prettier](https://github.com/prettier/prettier)_

계속 같은 개념입니다. 먼저 코드를 가져 와서 AST를 생성합니다. 이제 Prettier가 마법을 부릴 차례입니다. **AST가 '중간 표현(Intermediate Representation)’또는 'Doc’으로 변환됩니다.** 크게 보면 AST 노드가 포맷/형식 측면에서 어떻게 서로 관련되는지에 대한 정보와 함께 확장되는 것입니다. 예를 들어, 함수에 대한 매개 변수 목록은 한 그룹으로 처리되어야 합니다.

이후 목록이 길고 한 줄에 들어 가지 않으면 각 매개 변수를 별도의 줄으로 나누는 등의 작업을 합니다. 그런 다음 **'printer'라는 주 알고리즘**이 IR을 통과하고 전체 그림을 기반으로 어떻게 코드를 포맷/작성할지 결정합니다.

역시나 Prettier의 printer 알고리즘에 대해 이론적으로 더 알아보고 싶다면 (보이는 것처럼 간단하진 않지만) 책을 한 권 추천합니다.

![Prettier](/assets/2018-06-19-AST/22.png "Prettier")
_[http://homepages.inf.ed.ac.uk/wadler/papers/prettier/prettier.pdf](http://homepages.inf.ed.ac.uk/wadler/papers/prettier/prettier.pdf)_

## js2flowchart

마지막으로 소개하고 싶은 것은 [js2flowchart](https://github.com/Bogdan-Lyashenko/js-code-to-svg-flowchart)라는 제 라이브러리입니다. (Github에 4.2k개의 별이 있습니다.)

![js2flowchart](/assets/2018-06-19-AST/23.png "js2flowchart")
_이름 그대로입니다. JS Code를 SVG 플로우챠트로 만드는 라이브러리입니다._

이것은 AST 코드 표현 방식을 사용하면 원하는 작업은 뭐든지 수행할 수 있다는 것을 보여 주는 좋은 예입니다. 꼭 AST를 다시 코드 문자열로 바꿀 필요는 없습니다. 그걸로 플로우차트를 그리든 뭐든 맘대로 할 수 있습니다.

그럼 이 라이브러리는 뭘 하는 걸까요? 플로우 차트로 코드를 설명/문서화하고, 다른 사람들이 쓴 코드를 시각적으로 이해하여 배우고, 유효한 JS 구문으로 되어 있기만 하다면 모든 프로세스의 플로우 차트를 작성할 수 있습니다. _(역주: 원 저자는 js2flowchart를 이용해 [Under the hood: React](https://github.com/Bogdan-Lyashenko/Under-the-hood-ReactJS) 프로젝트를 만들기도 했습니다. [한글 번역](https://github.com/Bogdan-Lyashenko/Under-the-hood-ReactJS/blob/master/stack/languages/korean/book/Intro.md)도 있으니 한 번 확인해보세요.)_

지금 바로 사용해 볼 수 있는 가장 간단한 방법은 [실시간 편집기](https://bogdan-lyashenko.github.io/js-code-to-svg-flowchart/docs/live-editor/index.html)를 사용하는 것입니다.

![js2flowchart](/assets/2018-06-19-AST/24.png "js2flowchart")
_[https://bogdan-lyashenko.github.io/js-code-to-svg-flowchart/docs/live-editor/index.html](https://bogdan-lyashenko.github.io/js-code-to-svg-flowchart/docs/live-editor/index.html)_

한 번 써보세요. 이 편집기 외에도 코드에서 사용할 수도 있고, CLI로 SVG 파일을 생성하려는 각 파일을 지정할 수도 있습니다. VSCode 익스텐션 _(역주: 못 찾았습니다..)_ 도 있습니다.

이 외에 또 뭘 할 수 있을까요? 먼저, 전체적인 코드 뼈대(scheme)를 작성하는 것 이외에, 구조를 얼마나 상세하게 작성해야 하는지 추상화 수준을 지정할 수 있습니다.

![js2flowchart](/assets/2018-06-19-AST/25.png "js2flowchart")

예를 들어 모듈이 내보내는(module exports) 것만 보여줄 수도 있고, 클래스 정의만 혹은 함수 정의 및 호출만 그릴 수도 있습니다. 이 기능을 사용하면 슬라이드가 진행될수록 더 자세한 코드 구조를 보여주는 식으로 프레젠테이션 문서를 만들 수 있습니다.

트리를 수정할 수 있는 유용한 도구도 많이 있습니다. 예를 들어  `.forEach` 메서드 호출은 메서드 호출일 뿐이지만, 우리 모두 이게 루프라는 걸 알고 있으므로 해당 메서드는 루프로 그려줘야 한다고 지정할 수 있습니다.

![js2flowchart](/assets/2018-06-19-AST/26.png "js2flowchart")

내부적으로는 어떻게 작동할까요?

![js2flowchart](/assets/2018-06-19-AST/27.png "js2flowchart")

먼저 AST 코드를 파싱한 다음 AST를 탐색하여 **FlowTree**라는 또 다른 트리를 생성합니다. 함수, 루프, 조건 등의 주요 블록을 조립하고, 작고 중요하지 않은 토큰들은 많이 생략합니다. 그 다음에는 FlowTree를 탐색하면서 **ShapesTree**를 생성합니다. ShapesTree의 각 노드는 시각적 유형, 위치, 연결 등에 관한 정보를 포함합니다. 마지막 단계에서는 모든 각각의 Shape에 대해 SVG 표현을 생성하고 모두 하나의 SVG 파일로 합칩니다.

프로젝트는 [Github Repo](https://github.com/Bogdan-Lyashenko/js-code-to-svg-flowchart) 에서 확인하실 수 있습니다.
