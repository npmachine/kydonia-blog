---
layout: post
cover: 'assets/images/cover2.jpg'
title: 레일즈 API 서버 처음 구축시 고려할 점
author: npmachine
date: 2016-12-17 00:00:00
tags: rails
subclass: 'post tag-fiction'
categories: 'npmachine'
navigation: false
---

> 2016 대림절을 위해 ~~날려~~ 쓰는 ~~짜집기~~ 글입니다. 최대한 출처를 밝혔습니다만 혹시 누락한게 있을 수 있습니다. 어디선가 비슷한 내용을 읽은 기억이 있다면, 조용히 기억을 지워주세요.

---

### API 서버?  

전통적으로 웹은 서버에서 HTML 페이지를 생성하고 전송하면 브라우저는 이를 수신하여 표현하는 형태였다. 그렇지만 최근에는 자바스크립트와 웹 브라우저의 발전으로 웹브라우저에서 동적인 표현이 비약적으로 향상되면서 이전과 다르게 웹 앱을 네이티브 앱처럼 클라이언트 앱으로 개발하는 추세이다. 이 경우에 네이티브 앱과 웹 클라이언트 앱을 동일 선상에 놓인 클라이언트로 개발하고 이 앱들의 백엔드를 API 서버로 처리한다.  

레일즈도 API 모드를 지원하니 살펴보기로 하자.

### 레일즈 기본 서버와 API 서버의 차이점  

그렇다면 API 서버는 레일즈 기본 서버와 비교해서 어떤 점이 다를까?  
API 서버는 레일즈 기본 서버에서 클라이언트, 즉 브라우저와 관련된 기능을 제거하였다.  
세션, 쿠키, 애셋 등등을 제거하였고 뷰, 헬퍼, 애셋을 생성하지 않도록 제네레이터를 변경하였다.

이는 설치한 미들웨어 목록을 보여주는 명령어, `rake middleware`를 수행하면 알 수 있다. 레일즈 기본 서버에는 있지만 API 서버에는 없는 기능 목록은 다음과 같다.

- ActiveSupport::Cache::Strategy::LocalCache::Middleware
- Rack::MethodOverride
- WebConsole::Middleware
- ActionDispatch::Cookies
- ActionDispatch::Session::CookieStore
- ActionDispatch::Flash

또한 레일즈 기본 서버의 ApplicationController가 ActionController::Base에서 상속받는데 비해서 API 서버는 ActionController::API를 상속받는다.

더 자세한 사항은 [레일즈 한글 가이드 문서](http://guides.rorlab.org/api_app.html)에서 확인할 수 있다.

### JSON Output

위에서 언급하였듯이 API 서버를 생성하면 view layer가 없다.  
그렇다면, api 요청이 오면 보통 json으로 응답하는데, render를 이용해 일일히 그려줄게 아니라면(...) 당연히 json을 생성해주는 무엇인가가 필요하다. 크게 3가지 선택이 있다.

- [jbuilder](https://github.com/rails/jbuilder)
- [active-model-serializer](https://github.com/rails-api/active_model_serializers)
- [rabl](https://github.com/nesquena/rabl)

이 중에서 jbuilder, active-model-serializer를 주로 사용한다.  
rabl은 전체적으로 jbuilder와 유사하지만 jbuilder가 더 배우기 쉽고 사용하기 편하다고 한다.

둘 중에서는 무엇을 써야할까?  
jbuilder, active-model-serializer의 속도 차이에 관한 [Kirill Platonov의 글](http://kirillplatonov.com/2014/11/04/active_model_serializer_vs_jbuilder/)에 따르면 active-model-serializer의 압승이다.  
대략 10배 차이난다.

다만 Kirill이 지적하듯이 이 차이는 render method에서 기인한다.  
많은 이들이 jbuilder의 기본 파서를 optimized json gem인 [oj gem](https://github.com/ohler55/oj)을 이용하면 속도에 개선이 있다고 말하고 있다. jbuilder github 저장소의 README.md에서도 이 점을 언급하고 있다. 다만 여기서는 기본 파서 대체제로 [yajl gem](https://github.com/brianmario/yajl-ruby)을 권하고 있다.

그 밖에 [reddit에 올라온  글](https://www.reddit.com/r/ruby/comments/2lcldk/activemodelserializers_vs_jbuilder/)의 realntl는 active-model-serializer는 모델을 있는 그대로 던져주는 경우에 유용하고, json을 가공하게 되면 jbuilder가 조금 더 낫다고 말한다.

실제로 써봤을 때, realntl 말에 수긍하게 되는게 active-model-serializer는 모델을 확장하는 느낌이고 jbuilder는 erb와 유사해서, 편하게 사용할 수 있다. 달리 말하자면 jbuilder가 레일즈에 기본으로 포함된 이유가 이해간다.

### JSON Response Format

처음 json 응답을 작성할 때는 원하는 데이터를 { key: value } 형식으로 작성하면 되는 줄 알았다.  
하지만 늘 그렇듯, 잘 하는 사람들이 만든 권장 형식 혹은 규약이 있다.  
~~개발자들의 성지~~ [스택오버플로우 글](http://stackoverflow.com/questions/12806386/standard-json-api-response-format)을 정리하였다.

- [JSON API](http://jsonapi.org/)
- [JSend](https://labs.omniti.com/labs/jsend)
- [Google JSON Style Guide](https://google.github.io/styleguide/jsoncstyleguide.xml)
- [RFC 7807: Problem Details for HTTP APIs](https://datatracker.ietf.org/doc/rfc7807/)
- [OData JSON Protocol](http://docs.oasis-open.org/odata/odata-json-format/v4.0/errata02/os/odata-json-format-v4.0-errata02-os-complete.html#_Toc403940655)
- [HAL](http://stateless.co/hal_specification.html)
- 그외 Swagger, WADL in JSON, RAML 같은 형식도 있다

JSON API, JSend가 간단하게 쓰기 좋다.  
[Leigh Halliday의 글](https://blog.codeship.com/building-a-json-api-with-rails-5/)에 따르면 레일즈, 엠버 커뮤니티에서는 JSON API 형식을 많이 사용한다고 한다.

### Token

API 서버는 OAuth 기반 토큰 인증 시스템을 사용한다.

- [doorkeeper](https://github.com/doorkeeper-gem/doorkeeper)
  - 가장 많이 사용한다.  
  - 완성도 높고, 문서화가 잘 되어 있다.  
  - grape, devise 같은 다른 프레임워크 연동도 잘 지원한다.
- [devise_token_auth](https://github.com/lynndylanhurley/devise_token_auth)
  - devise 기반 토큰 모듈이다.  
  - devise가 내장되어 있다.  
  - 이미 사용하는 별도 devise와도 연동 가능하다.  
  - 그러나, 직접 사용해 본 경험으로는 연동이 매끄럽지 않았다.
- [ruby-jwt](https://github.com/jwt/ruby-jwt)
  - [RFC 7519 OAuth JSON Web Token](https://tools.ietf.org/html/rfc7519)의 루비 구현이다.
  - 표준이어서 레일즈 외에도 많이 쓰는듯 하다.

### API Versioning

API 버전 관리를 해야하는 이유는 시간이 지나면서 API를 업그레이드 하기 때문이다.
이때 여러가지 이유로 기존 API를 사용하는 클라이언트에게 서비스 영속성을 보장해야 한다.
API 버전 관리 전략은 다음처럼 나눌 수 있다.

- HTTP header
  - 헤더에 버전 정보를 포함한다.
  - `Accept: application/vnd.mycompany.com; version=1,application/json`
  - 더 자세한 내용은 [Steve Klabnik의 토막글](http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned)을 참조하자.
- URL 경로
  - URL 경로에 버전 정보를 포함한다.
  - `https://example.com/api/v2/users`
  - 많은 예제가 이 형식을 따르고 있지만, 찾아보면 하지 말라고 한다.  
  - 이유 하나를 꼽자면, 버전이 증가하면서 같은 리소스에 대해 반복되는 라우팅을 만든다.  
  - 예를 들어, user 모델, 콘트롤러를 만들고 v1, v2, v3 api를 만드는 경우를 가정해 보자.  
  - 이 때 `rails routes` 명령어를 수행해보면 바로 상황이 이해간다.
- 쿼리 파라미터
  - URL 쿼리로 버전 정보를 전달한다.
  - `https://example.com/users?version=v3`
- 요청 파라미터
  - 요청 본문에 버전 정보를 포함한다.

길게 썼지만, 버전 정보는 헤더에 포함하자는 말이다.

다음과 같은 버전 관리 젬이 있다.

- [versionist](https://github.com/bploetz/versionist)
- [versioncake](https://github.com/bwillis/versioncake)

### API Framework

레일즈 5의 API 기능은 [rails-api](https://github.com/rails-api/rails-api)라는 별도의 프로젝트를 기본 레일즈 설정으로 병합한 모듈이다. rails-api 외에도 눈여겨 볼 API 프레임워크가 있다.

- [grape](https://github.com/ruby-grape/grape)
  - 이 중에서 grape를 가장 많이 사용한다.  
  - grape는 rails-api 보다 먼저 개발을 시작한, 레일즈와 독립적인 API 프레임워크이다.  
  - 가볍고 완성도 높고 문서화가 잘되어 있다.  
  - 레일즈 API 서버는 레일즈에 익숙한 개발자가 바로 사용할 수 있다는 장점을 제외하고는, 둘 중 어느 것을 사용해도 무방해 보인다.
- [rocket_pants](https://github.com/Sutto/rocket_pants)
  - rocket_pants는 자신들의 프로젝트가 grape와 유사하지만 rails와 같이 사용하기 위해 개발하였다고 소개하고 있다.
- [acts_as_api](https://github.com/fabrik42/acts_as_api)
  - acts_as_api는 다른 acts_as 젬 시리즈 처럼 액티브 레코드 모델에 사용할 수 있는 API 젬이다.

---

레일즈로 API 서버를 구축하기 전에 사전 작업으로 조사한 자료를 정리하였다.  
CORS, Rack::Attack 처럼 설정해야 하지만 특별히 고려할 내용이 없는 경우는 제외하였다.  
API 서버를 처음 구축하는 사람에게 도움이 되었으면 한다.
