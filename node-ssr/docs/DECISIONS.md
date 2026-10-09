# 결정 사항 기록 (2단계 - Node.js 로 Classic SSR 구현하기)

구현 전에 정한 결정 사항을 기록한다. 결정마다 **근거** 칸은 직접 작성한다.

## 요구사항 요약

- `/` 와 `/detail/:id` 에서 react-csr 과 동일한 화면을 SSR 로 반환한다.
- 사용자 인터랙션은 동작하지 않아도 된다.
- `/public` 의 html, css 리소스를 기반으로 구현한다.
- express 로 서버를 만들고, TMDB 데이터를 동적으로 받아 HTML 을 동적으로 생성한다.
- Railway 등으로 배포해서 `/detail/:id` 의 og tag 동작과 `/` 의 FCP 개선을 확인한다.

## 목차

| #   | 결정                      | 선택                              |
| --- | ------------------------- | --------------------------------- |
| D1  | HTML 생성 방식            | 템플릿 리터럴 함수                |
| D2  | 파일 구조                 | 라우트 + 뷰 분리                  |
| D3  | `/detail/:id` 화면 구성   | 홈 + 모달                         |
| D4  | og 태그 범위              | OG 기본 + 홈에도 적용             |
| D5  | 이미지 URL 기준           | react-csr 기준                    |
| D6  | 사용자 인터랙션           | 링크로 최소 이동                  |
| D7  | 에러 처리                 | 상태코드 + 에러 HTML              |
| D8  | HTML 이스케이프           | escapeHtml 유틸 적용              |
| D9  | og:url 절대 URL           | 요청 호스트에서 계산              |
| D10 | 포트                      | `process.env.PORT ?? 8080`        |
| D11 | 모달 '내 별점' 영역       | 초기 상태로 렌더                  |
| D12 | 로컬 실행 준비            | 토큰 복사 + npm install           |
| D13 | '자세히 보기' 이동 방식   | `<a>` 로 `<button>` 감싸기        |
| D14 | og:image                  | 포스터 w500                       |
| D15 | 공통 유틸 위치            | `src/utils/`                      |
| D16 | 헤더 평점 표기            | react-csr 그대로                  |
| D17 | 홈 화면 조합 위치         | `views/MovieHomePage.ts`          |
| D18 | 에러 페이지 뷰 위치       | `views/ErrorPage.ts`              |
| D19 | 정의되지 않은 경로        | express 기본 404                  |
| D20 | 뷰 함수 네이밍            | `render` 접두사                   |
| D21 | og:type 과 `<title>` 포맷 | website / video.movie, `{제목} \| 영화 리뷰` |
| D22 | 인기 목록이 비어 있을 때  | 200 + 문구 (react-csr)            |
| D23 | 홈 description 문구       | 지금 인기 있는 영화               |
| D24 | public / react-csr 마크업 차이 | public HTML (`<img>`)        |
| D25 | 에러 페이지 구성          | 문구만                            |

---

## D1. HTML 생성 방식

**배경**

서버에서 TMDB 데이터를 받아 HTML 문자열을 만들어 응답해야 한다. react-csr 은 JSX 로 화면을 그리지만 node-ssr 에는 React 가 없다.

**선택지**

- **템플릿 리터럴 함수**: 데이터를 받아 HTML 문자열을 반환하는 TS 함수. 기존 `server.ts` 스켈레톤(`/*html*/`)과 같은 방식이고 의존성 추가가 없다.
- **public/index.html 치환**: `public/index.html` 을 읽고 `<!--MOVIE_LIST-->` 같은 placeholder 를 `replace` 한다. 반복 요소는 결국 문자열 함수가 필요하다.
- **EJS 템플릿 엔진**: express view engine 으로 `views/*.ejs` 를 렌더한다. `ejs` 의존성이 추가된다.

**결정**

템플릿 리터럴 함수

**근거**

> - 의존성을 추가하지 않고 TS 만으로 구현할 수 있다.
> - 템플릿 엔진처럼 별도 방식을 익히지 않아도 되어, 미션 규모에 맞게 가볍게 시작할 수 있다.

**트레이드오프 / 영향**

- React 와 달리 자동 이스케이프가 없어서 직접 처리해야 한다. → D8
- 마크업이 문자열이라 에디터의 HTML 문법 검사·자동완성을 받기 어렵다.

---

## D2. 파일 구조

**배경**

라우트 처리(데이터 fetch), HTML 조각 생성, TMDB 호출 코드를 어떻게 나눌지 정해야 한다.

**선택지**

- **라우트 + 뷰 분리**: 라우터와 뷰(HTML 조각)를 나눈다. react-csr 컴포넌트와 1:1 로 대응된다.
- **server.ts 하나에**: 라우트와 렌더 함수를 한 파일에 모두 둔다.
- **페이지 단위 분리**: 라우트는 `server.ts` 에, 페이지별 렌더 함수만 `pages/` 에 둔다.

**결정**

라우트 + 뷰 분리

```
src/
├─ server.ts
├─ routes/
│  └─ movies.ts            (/, /detail/:id)
├─ views/
│  ├─ layout.ts            (html/head/og)
│  ├─ MovieHomePage.ts     (D17)
│  ├─ Header.ts
│  ├─ MovieList.ts
│  ├─ MovieDetailModal.ts
│  ├─ Footer.ts
│  └─ ErrorPage.ts         (D18)
├─ utils/                  (D15)
└─ service/                (기존 tmdbApi.ts, types.ts)
```

**근거**

> - 데이터 fetch·상태코드 결정(라우트)과 마크업 생성(뷰)을 나눠서 각 파일의 역할이 분명하다.
> - 마크업을 고칠 때는 뷰만, 요청 흐름을 고칠 때는 라우트만 건드리면 되어 변경 범위가 좁다.

**트레이드오프 / 영향**

- 미션 규모에 비해 파일 수가 많아진다.

---

## D3. `/detail/:id` 화면 구성

**배경**

react-csr 의 `MovieDetailPage` 는 `MovieHomePage` 를 그대로 렌더한 뒤 그 위에 상세 모달을 띄운다. `public/modal.html` 도 같은 구조다.

**선택지**

- **홈 + 모달**: 홈 화면 위에 모달이 열린 상태로 렌더한다. 인기 목록과 상세 정보를 `Promise.all` 로 병렬 요청한다.
- **모달만 단독**: 배경 목록 없이 상세 모달만 렌더한다. TMDB 요청이 1번으로 줄지만 react-csr 화면과 달라진다.

**결정**

홈 + 모달 (`Promise.all` 병렬 요청)

**근거**

> 요구사항이 react-csr 과 동일한 화면을 반환하는 것이고, react-csr 의 `MovieDetailPage` 가 홈 위에 모달을 띄우는 구조다.

**트레이드오프 / 영향**

- `/detail/:id` 응답 시간은 두 요청 중 더 느린 쪽에 맞춰진다.
- 두 요청 중 하나라도 실패하면 페이지 전체가 실패한다. → D7

---

## D4. og 태그 범위

**배경**

요구사항은 `/detail/:id` 에서 og tag 가 정상 동작하는 것이다. CSR 은 크롤러가 JS 를 실행하지 않으면 빈 HTML 만 보게 되지만, SSR 은 응답 HTML 에 메타 태그를 넣을 수 있다.

**선택지**

- **OG 기본 + 홈에도 적용**: `/detail/:id` 에 og:type/title/description/image/url, `/` 에는 사이트 기본 og 를 넣고 `<title>`, `meta description` 도 페이지별로 설정한다.
- **OG + Twitter Card**: 위 범위에 `twitter:card(summary_large_image)/title/description/image` 까지 추가한다.
- **`/detail/:id` 만 최소**: `/detail/:id` 에만 og:title/description/image 를 넣는다.

**결정**

OG 기본 + 홈에도 적용

**근거**

> - 요구사항인 `/detail/:id` 에 더해, `/` 링크를 공유할 때도 미리보기가 나오게 한다.
> - 크롤러가 JS 를 실행하지 않아도 페이지별 메타 정보를 볼 수 있다는 것이 SSR 의 이점이므로, 모든 페이지에 적용한다.
> - `<title>` 을 페이지별로 다르게 해야 탭·검색 결과에서 페이지가 구분된다.
> - X(트위터)도 og 태그로 미리보기를 만들 수 있어서 Twitter Card 전용 태그까지는 넣지 않는다.

**트레이드오프 / 영향**

- og:type 값, `<title>` 포맷은 D21 에서 정한다.
- Twitter Card 전용 태그가 없어서 X(트위터)는 og 태그로 대체해 미리보기를 만든다.

---

## D5. 이미지 URL 기준

**배경**

react-csr 과 `public/index.html` 의 이미지 URL 형식이 서로 다르다.

**선택지**

- **react-csr 기준**
  - 썸네일: `https://image.tmdb.org/t/p/w500{poster_path}`
  - 모달: `https://image.tmdb.org/t/p/original{poster_path}`
  - 헤더 배경: `https://image.tmdb.org/t/p/w1920_and_h800_multi_faces/{poster_path}`
  - 포스터 없음: `/images/no_image.png`
- **public HTML 샘플 기준**
  - 썸네일: `https://media.themoviedb.org/t/p/w440_and_h660_face{poster_path}`
  - 모달: `https://image.tmdb.org/t/p/original{poster_path}`
  - 헤더 배경: `https://image.tmdb.org/t/p/w1920_and_h800_multi_faces{backdrop_path}`

**결정**

react-csr 기준

**근거**

> 요구사항이 react-csr 과 동일한 화면이므로 이미지 URL 도 react-csr 을 기준으로 한다.

**트레이드오프 / 영향**

- 헤더 배경에 세로형 `poster_path` 를 쓰므로 가로형 배너보다 잘려 보일 수 있다 (react-csr 과 동일).
- 모달 이미지가 `original` 이라 용량이 크다 (react-csr 과 동일).

---

## D6. 사용자 인터랙션

**배경**

요구사항상 인터랙션은 동작하지 않아도 된다. 클라이언트 JS 를 보내지 않는 Classic SSR 이다.

**선택지**

- **링크로 최소 이동**: JS 없이 `<a>` 태그만 쓴다. 영화 아이템·자세히 보기 → `/detail/:id`, 모달 닫기·로고 → `/`. 페이지를 이동할 때마다 서버가 새 HTML 을 내려준다. 별점 주기는 동작하지 않는다.
- **인터랙션 없음**: 정적 마크업만 반환한다.
- **바닐라 JS 로 일부 복원**: 링크 이동 + 별점 클릭(sessionStorage 저장) 등을 클라이언트 스크립트로 복원한다.

**결정**

링크로 최소 이동

**근거**

> - 클라이언트 JS 없이 요청마다 서버가 새 HTML 을 내려주는 Classic SSR 방식을 그대로 보여준다.
> - 링크가 있어야 목록에서 `/detail/:id` 로 이동해 SSR 결과(모달, og 태그)를 확인할 수 있다.
> - 인터랙션은 요구사항이 아니므로, 별점 같은 기능을 바닐라 JS 로 복원하는 것은 범위 밖이다.

**트레이드오프 / 영향**

- 모달 열기·닫기마다 전체 페이지를 다시 받는다 (TMDB 요청도 다시 발생).
- 별점 클릭은 동작하지 않는다. → D11

---

## D7. 에러 처리

**배경**

TMDB 요청이 실패하거나 존재하지 않는 영화 id 로 접근하는 경우를 처리해야 한다. react-csr 은 상태코드 개념 없이 "영화 정보를 불러오는데 실패했습니다." 문구만 보여준다.

**선택지**

- **상태코드 + 에러 HTML**: 숫자가 아닌 id·TMDB 에서 404 인 id → 404 + "영화 정보를 찾을 수 없습니다.", 그 외 TMDB 실패 → 500 + "영화 정보를 불러오는데 실패했습니다."
- **react-csr 처럼 문구만**: 상태코드는 200 그대로 두고 실패 문구만 렌더한다.
- **express 기본 처리**: 따로 처리하지 않고 express 5 기본 에러 핸들러(500)에 맡긴다.

**결정**

상태코드 + 에러 HTML

**근거**

> - 없는 페이지를 200 으로 응답하면 크롤러·검색엔진이 정상 페이지로 받아들인다. 상태코드로 결과를 정확히 알린다.
> - 없는 영화(404)와 서버·TMDB 문제(500)를 구분할 수 있다.

**트레이드오프 / 영향**

- TMDB 404 를 구분하기 위해 axios 에러의 응답 상태를 확인하는 분기가 필요하다.
- 에러 페이지 뷰 위치는 D18, 정의되지 않은 경로(`/foo`) 처리는 D19 에서 정한다.

---

## D8. HTML 이스케이프

**배경**

템플릿 리터럴은 React 와 달리 자동 이스케이프가 없다. 줄거리·제목에 `"`, `<`, `&` 가 들어가면 마크업이나 og `content` 속성이 깨질 수 있다.

**선택지**

- **escapeHtml 유틸 적용**: `& < > " '` 를 치환하는 함수를 만들어 텍스트·속성값에 적용한다.
- **이스케이프 없이**: 그대로 넣는다.

**결정**

escapeHtml 유틸 적용

**근거**

> - 제목·줄거리에 `"`, `<`, `&` 가 들어가도 마크업과 og `content` 속성이 깨지지 않는다.
> - TMDB 는 외부 데이터라 그대로 HTML 에 넣으면 스크립트가 삽입될 수 있다(XSS).
> - react-csr 에서 JSX 가 자동으로 해주던 이스케이프를 함수 하나로 대신할 수 있다.

**트레이드오프 / 영향**

- 데이터를 넣는 모든 지점에서 빠뜨리지 않고 호출해야 한다.

---

## D9. og:url 절대 URL

**배경**

og:url 은 절대 URL 이어야 한다. 로컬(`http://localhost:8080`)과 배포 도메인이 다르다. Railway 는 프록시 뒤에서 앱을 실행하므로 앱이 직접 받는 요청은 http 다.

**선택지**

- **요청 호스트에서 계산**: `req.protocol + req.get('host')` 로 만들고 `app.set('trust proxy', true)` 로 프록시의 `X-Forwarded-Proto` 를 신뢰한다. 환경변수 추가가 없다.
- **BASE_URL 환경변수**: 배포 도메인을 환경변수로 등록해서 쓴다. Railway Variables 에 하나 더 등록해야 한다.
- **og:url 생략**: og:url 없이 type/title/description/image 만 넣는다.

**결정**

요청 호스트에서 계산 (`trust proxy` 활성화)

**근거**

> - BASE_URL 같은 환경변수를 Railway Variables 에 따로 등록·관리하지 않아도 된다.
> - 로컬·배포·도메인 변경 시에도 코드나 설정을 바꾸지 않고 요청에 맞는 URL 이 만들어진다.
> - `trust proxy` 로 `X-Forwarded-Proto` 를 읽어서 프록시 뒤에서도 https URL 이 나온다.
> - og:url 이 있어야 공유할 때 페이지의 정규 URL 을 지정할 수 있어서 생략하지 않는다.

**트레이드오프 / 영향**

- `trust proxy: true` 는 모든 프록시 헤더를 신뢰하므로 프록시 없이 직접 노출되면 `X-Forwarded-*` 헤더로 값이 바뀔 수 있다.

---

## D10. 포트

**배경**

현재 `server.ts` 는 8080 이 하드코딩되어 있다. Railway 는 `PORT` 환경변수를 주입한다.

**선택지**

- **`process.env.PORT ?? 8080`**: Railway 가 주입하는 PORT 를 우선 쓰고, 로컬에서는 8080 을 쓴다.
- **8080 유지**: 코드는 그대로 두고 Railway 에서 Generate Domain 할 때 target port 를 8080 으로 지정한다.

**결정**

`process.env.PORT ?? 8080`

**근거**

> `PORT` 환경변수는 Railway 뿐 아니라 Render·Heroku 등 다른 PaaS 에서도 쓰는 관례라, 배포 플랫폼을 바꿔도 코드 수정 없이 동작한다.

**트레이드오프 / 영향**

- 없음 (로컬 동작은 기존과 같다).

---

## D11. 모달 '내 별점' 영역

**배경**

react-csr 은 sessionStorage 에 저장한 별점을 보여준다. 서버는 브라우저의 sessionStorage 값을 알 수 없다.

**선택지**

- **초기 상태로 렌더**: react-csr 의 초기값과 같게 빈 별 5개 + "0 별점을 남겨주세요" 를 보여준다.
- **별점 영역 제거**: 동작하지 않는 UI 는 빼고 장르·평점·줄거리만 보여준다.

**결정**

초기 상태로 렌더

**근거**

> 처음 방문한 사용자가 react-csr 에서 보는 화면도 빈 별점이므로, 초기 상태로 렌더하면 react-csr 첫 화면과 같다.

**트레이드오프 / 영향**

- 클릭해도 반응하지 않는 별점 UI 가 노출된다.

---

## D12. 로컬 실행 준비

**배경**

node-ssr 에는 `node_modules` 와 `.env` 가 없다. react-csr 에는 `.env`(`VITE_TMDB_ACCESS_TOKEN`)가 있다.

**선택지**

- **토큰 복사 + npm install**: react-csr/.env 의 토큰 값을 node-ssr/.env 에 `TMDB_ACCESS_TOKEN` 으로 만들고(`.gitignore` 됨), `npm install` 후 서버를 띄워 curl 로 응답을 확인한다.
- **npm install 만**: `.env` 는 직접 만들고, `npm install` 과 빌드(tsc) 검증까지만 한다.
- **아무것도 하지 않기**: 코드만 작성한다.

**결정**

토큰 복사 + npm install

**근거**

> - 빌드(tsc)만으로는 런타임 에러나 마크업 문제를 알 수 없어서, 서버를 띄워 응답 HTML·og 태그를 직접 확인한다.
> - react-csr 의 토큰을 그대로 쓰면 새로 발급받지 않아도 되고, `.env` 는 `.gitignore` 대상이라 git 에 올라가지 않는다.

**트레이드오프 / 영향**

- 같은 토큰이 두 `.env` 파일에 중복된다 (둘 다 git 에 올라가지 않음).

---

## D13. '자세히 보기' 이동 방식

**배경**

`public/styles/main.css`, `media.css` 의 버튼 스타일이 `button.primary` 셀렉터에만 걸려 있어서, `<a class="primary">` 로 바꾸면 스타일이 빠진다.

**선택지**

- **GET form + button**: `<form action="/detail/:id" method="get"><button class="primary detail">`. public CSS 를 건드리지 않고 유효한 HTML 로 이동한다.
- **`<a>` + CSS 수정**: `<a class="primary detail">` 로 바꾸고 `button.primary` 셀렉터를 `.primary` 로 수정한다.
- **`<a>` 로 button 감싸기**: `<a href="/detail/:id"><button class="primary detail">`. CSS 수정이 없다.

**결정**

`<a>` 로 `<button>` 감싸기

**근거**

> - public CSS 의 `button.primary` 셀렉터를 건드리지 않고 버튼 스타일을 그대로 쓴다.
> - form 보다 마크업이 짧고 '이 주소로 이동한다'는 의도가 바로 보인다.
> - 스펙상 유효하지 않지만 대부분의 브라우저에서 클릭 이동은 동작하고, 인터랙션은 필수가 아니다(D6).
> - GET form 은 데이터 제출용이라 단순 이동에는 의미가 맞지 않고, 브라우저에 따라 URL 끝에 `?` 가 붙는다.

**트레이드오프 / 영향**

- HTML 스펙상 `<a>` 안에 interactive content(`<button>`)를 넣는 것은 유효하지 않다. 대부분의 브라우저에서 클릭은 동작하지만, HTML 검사기에서 오류가 나고 키보드 탭 포커스가 `<a>` 와 `<button>` 에 각각 잡힐 수 있다.

---

## D14. og:image

**배경**

og:image 는 링크 미리보기에 쓰이는 이미지다. 크롤러는 너무 큰 이미지를 가져오지 못하기도 한다.

**선택지**

- **포스터 w500**: 상세 페이지는 해당 영화 포스터(세로형), 홈은 1위 영화 포스터. react-csr 화면과 일관되고 `original` 보다 용량이 작다.
- **백드롭 w1280**: 가로형 `backdrop_path` 를 쓰고 없으면 포스터로 대체한다. 링크 미리보기(1.91:1) 비율에 더 잘 맞는다.

**결정**

포스터 w500

**근거**

> - 목록 썸네일과 같은 이미지라 미리보기와 실제 화면이 이어진다.
> - `original` 보다 용량이 작아 크롤러가 이미지를 가져오지 못할 위험이 적다.
> - 썸네일 URL 헬퍼(`getThumbnailUrl`)를 그대로 재사용한다.
> - 백드롭은 없는 영화도 있어서 '백드롭 → 포스터 → no_image' 처럼 대체 분기가 늘어난다.

**트레이드오프 / 영향**

- 세로형 이미지라 가로형 미리보기 카드에서는 잘리거나 작은 썸네일로 표시될 수 있다.
- 포스터가 없는 영화의 og:image 는 `/images/no_image.png` 를 절대 URL 로 만들어 넣는다.

---

## D15. 공통 유틸 위치

**배경**

escapeHtml(D8), 이미지 URL 헬퍼(D5) 같은 공통 함수를 둘 위치가 필요하다.

**선택지**

- **`src/utils/`**: `src/utils/escapeHtml.ts`, `src/utils/imageUrl.ts` 처럼 뷰와 독립된 폴더에 둔다.
- **`src/views/` 안에**: 뷰에서만 쓰이므로 `src/views/utils.ts` 하나에 모은다.

**결정**

`src/utils/`

**근거**

> - `imageUrl` 은 라우트(og:image)에서도 쓰므로 `views/` 에 두면 라우트가 뷰 폴더에 의존하게 된다.
> - `views/` 에는 HTML 조각을 만드는 파일만 둔다.
> - `escapeHtml.ts`, `imageUrl.ts` 처럼 기능별로 파일을 나눠 이름만 봐도 역할을 알 수 있다.
> - `utils/` 는 흔한 관례라 위치를 찾기 쉽다.

**트레이드오프 / 영향**

- 없음.

---

## D16. 헤더 평점 표기

**배경**

react-csr 의 `FeaturedMovie` 는 `vote_average` 를 그대로(예: `7.123`) 보여주고, `MovieItem`·`MovieDetailModal` 은 `toFixed(1)` 을 쓴다.

**선택지**

- **react-csr 그대로**: 헤더는 원본값, 목록·모달은 `toFixed(1)`. react-csr 화면과 글자 단위까지 같다.
- **모두 `toFixed(1)`**: 헤더도 소수점 한 자리로 맞춘다. react-csr 과 헤더 평점 표기만 달라진다.

**결정**

react-csr 그대로

**근거**

> 요구사항이 react-csr 과 동일한 화면이므로 평점 표기도 글자 단위까지 react-csr 과 맞춘다.

**트레이드오프 / 영향**

- 헤더와 목록에서 같은 영화의 평점 자릿수가 다르게 보일 수 있다.

---

## D17. 홈 화면 조합 위치

**배경**

`/` 와 `/detail/:id`(D3) 모두 Header + MovieList + Footer 조합을 쓴다. react-csr 에서는 `pages/MovieHomePage` 가 이 역할을 한다.

**선택지**

- **`views/MovieHomePage.ts`**: 뷰 파일로 두고 두 라우트가 재사용한다.
- **`routes/movies.ts` 내부 함수**: 라우트 파일 안에 로컬 함수로 둔다. D2 의 파일 목록을 그대로 유지한다.

**결정**

`views/MovieHomePage.ts`

**근거**

> - `/` 와 `/detail/:id` 가 같은 조합을 쓰므로 한 곳에 두고 재사용한다.
> - 화면 조합도 마크업이므로 뷰의 책임이고, 라우트는 데이터 fetch·상태코드에 집중한다(D2).

**트레이드오프 / 영향**

- D2 의 `views/` 에 파일이 하나 늘어난다.

---

## D18. 에러 페이지 뷰 위치

**배경**

D7 에서 404/500 에 에러 HTML 을 반환하기로 했다.

**선택지**

- **`views/ErrorPage.ts`**: 메시지를 받아 layout 으로 감싼 에러 HTML 을 만드는 별도 뷰 파일.
- **`routes/movies.ts` 내부**: 라우트에서 layout 을 직접 호출해 에러 문구를 넣는다.

**결정**

`views/ErrorPage.ts`

**근거**

> - 에러 화면도 마크업이므로 D2 의 라우트/뷰 분리를 그대로 따른다.
> - 404·500·빈 목록(D22) 등 여러 분기에서 같은 함수를 쓰고, 라우트에서는 `renderErrorPage(문구)` 한 줄만 호출한다.

**트레이드오프 / 영향**

- 없음.

---

## D19. 정의되지 않은 경로

**배경**

`/`, `/detail/:id`, `public/` 정적 파일 외의 경로(`/foo` 등)로 요청이 들어올 수 있다.

**선택지**

- **express 기본 404**: 아무것도 추가하지 않는다. `Cannot GET /foo` 텍스트가 404 로 나간다.
- **커스텀 404 페이지**: static 뒤에 catch-all 미들웨어를 두고 에러 페이지(D18)로 "페이지를 찾을 수 없습니다." 를 반환한다.

**결정**

express 기본 404

**근거**

> 요구사항은 `/` 와 `/detail/:id` 만 다루므로 정의되지 않은 경로 처리는 범위 밖이다.

**트레이드오프 / 영향**

- 정의되지 않은 경로에서는 스타일 없는 express 기본 응답이 보인다.
- `public/index.html`, `public/modal.html` 은 static 으로 `/index.html`, `/modal.html` 에서 그대로 노출된다 (`/` 는 라우트가 static 보다 먼저 등록되어 SSR 응답이 나간다).

---

## D20. 뷰 함수 네이밍

**배경**

뷰 파일명은 react-csr 컴포넌트처럼 PascalCase(D2)지만, 함수는 JSX 컴포넌트가 아니라 HTML 문자열을 반환한다.

**선택지**

- **`render` 접두사**: `renderHeader()`, `renderMovieList()` 처럼 HTML 문자열을 만드는 함수임을 드러낸다.
- **PascalCase**: `Header()`, `MovieList()` 처럼 react-csr 컴포넌트 이름과 똑같이 쓴다.

**결정**

`render` 접두사

**근거**

> - React 컴포넌트가 아니라 HTML 문자열을 반환하는 함수라는 것을 이름에서 드러낸다. PascalCase 로 쓰면 컴포넌트로 오해할 수 있다.
> - 일반 함수는 camelCase 동사로 시작하는 JS 관례를 따르고, `get*Url` 같은 유틸 함수와도 역할이 구분된다.

**트레이드오프 / 영향**

- 파일명(`Header.ts`)과 함수명(`renderHeader`)이 다르다.

---

## D21. og:type 과 `<title>` 포맷

**배경**

D4 에서 페이지별로 og 태그와 `<title>` 을 넣기로 했다. 홈은 `영화 리뷰`(기존 `<title>`)를 쓴다.

**선택지**

- og:type
  - **홈 `website` / 상세 `video.movie`**: Open Graph 표준 타입을 페이지 성격에 맞게 나눈다.
  - **모두 `website`**: 대부분의 미리보기는 타입과 무관하게 title/description/image 로 만들어진다.
- 상세 페이지 `<title>` / og:title
  - **`{영화 제목} | 영화 리뷰`**: 탭과 공유 미리보기에 사이트 이름도 함께 보인다.
  - **`{영화 제목}` 만**: 미리보기 제목이 짧고 영화 정보에 집중된다.
  - **`<title>` 은 포맷, og:title 은 제목만**: og:site_name 을 따로 추가한다.

**결정**

- og:type: 홈 `website` / 상세 `video.movie`
- 상세 `<title>`·og:title: `{영화 제목} | 영화 리뷰`

**근거**

> - 페이지 성격(사이트 홈 / 영화 한 편)에 맞는 Open Graph 표준 타입을 쓴다.
> - 탭과 공유 미리보기에 사이트 이름이 함께 보여서 어느 서비스의 페이지인지 알 수 있다.
> - 기존 홈 `<title>` 인 `영화 리뷰` 를 접미사로 재사용해 홈과 상세의 제목이 일관된다.
> - 영화 제목이 앞에 오므로 여러 탭을 열어도 구분하기 쉽다.

**트레이드오프 / 영향**

- 공유 미리보기 제목이 길어져 일부 플랫폼에서 뒷부분(`| 영화 리뷰`)이 잘릴 수 있다.

---

## D22. 인기 목록이 비어 있을 때

**배경**

react-csr 의 `MovieHomePage` 는 목록이 0개면 "영화 정보를 불러오는데 실패했습니다." 를 보여준다. 헤더(D16)는 `movies[0]` 을 쓰므로 빈 목록을 그대로 렌더할 수 없다.

**선택지**

- **500 에러 페이지**: TMDB 실패와 같게 취급해 500 + 실패 문구를 반환한다 (D7 과 일관).
- **200 + 문구 (react-csr)**: react-csr 처럼 상태코드 200 으로 실패 문구만 렌더한다.
- **헤더 없이 빈 목록**: 헤더를 생략하고 섹션 제목 + 빈 리스트 + 푸터를 200 으로 보여준다.

**결정**

200 + 문구 (react-csr). `/` 와 `/detail/:id` 모두 같은 방식으로 처리한다.

**근거**

> - react-csr 도 목록이 0개면 같은 문구를 보여준다.
> - TMDB 가 정상 응답한 경우라 서버 오류(500)로 보기 어렵다.
> - `/` 와 `/detail/:id` 를 같은 방식으로 처리한다.
> - 헤더 생략 같은 별도 화면 분기를 만들지 않아도 된다.

**트레이드오프 / 영향**

- TMDB 요청 실패(500, D7)와 목록이 비어 있는 경우(200)가 같은 문구지만 상태코드가 다르다.

---

## D23. 홈 description 문구

**배경**

D4 에서 홈(`/`)에도 og 태그와 meta description 을 넣기로 했다.

**선택지**

- **지금 인기 있는 영화**: 화면의 섹션 제목(h2)을 그대로 쓴다.
- **설명형 문장**: "지금 인기 있는 영화를 확인하고 별점을 남겨보세요." 처럼 서비스 설명 문장을 쓴다.
- **1위 영화 줄거리**: 헤더에 나오는 1위 영화의 overview 를 쓴다.

**결정**

지금 인기 있는 영화

**근거**

> 화면의 섹션 제목(h2)을 그대로 써서 description 이 실제 화면 내용과 일치한다.

**트레이드오프 / 영향**

- 문구가 짧아 검색 결과·미리보기에서 전달하는 정보가 적다.

---

## D24. public HTML 과 react-csr 마크업이 다른 곳

**배경**

로고와 모달 별점이 public HTML 은 `<img>`, react-csr 은 `IconButton`(`<button><img></button>`) 이다. 화면은 둘 다 동일하게 보인다.

**선택지**

- **public HTML (`<img>`)**: 로고는 `<a href="/"><img class="logo"></a>`, 별점은 `<img>` 5개. 동작하지 않는 버튼을 두지 않는다.
- **react-csr (`<button><img>`)**: IconButton 과 같은 구조. 로고는 `<a href="/">` 로 감싼 button, 별점은 클릭해도 반응 없는 button 5개.

**결정**

public HTML (`<img>`)

**근거**

> - 요구사항이 `/public` 의 html, css 리소스를 기반으로 구현하는 것이다.
> - 키보드 포커스·스크린리더가 동작하지 않는 요소를 '버튼'으로 안내하지 않는다.
> - 눌러도 반응하지 않는 버튼을 두지 않아 사용자가 헷갈리지 않는다.
> - DOM 구조만 다르고 보이는 화면은 react-csr 과 같다.

**트레이드오프 / 영향**

- react-csr 과 DOM 구조가 일부 다르다 (화면은 같다).
- 모달 닫기 버튼도 public HTML 처럼 `<img class="modal-close-btn">` 을 `<a href="/">` 로 감싼다.

---

## D25. 에러 페이지 구성

**배경**

D7·D18 의 에러 페이지(404/500, D22 의 빈 목록 포함)에 보여줄 내용을 정해야 한다.

**선택지**

- **문구 + 홈 링크**: 에러 문구와 '홈으로' 링크(`<a href="/">`)를 보여준다.
- **문구만**: react-csr 처럼 에러 문구만 보여준다.

**결정**

문구만

**근거**

> react-csr 도 실패 시 문구만 보여주므로 같게 맞춘다.

**트레이드오프 / 영향**

- 에러 페이지에서 홈으로 돌아가려면 주소를 직접 바꾸거나 뒤로 가기를 해야 한다.

---

## 구현 세부 사항 (확인 필요)

구현하면서 질문 없이 정한 작은 세부 사항이다. 바꾸고 싶은 것은 결정 형식으로 옮긴다.

- **영화 id 검증**: `/detail/:id` 의 id 가 `/^[1-9]\d*$/` 가 아니면 TMDB 를 호출하지 않고 바로 404 를 반환한다 (`abc`, `0`, `1e3` 등).
- **og:url 에 쿼리스트링 제외**: `${origin}${req.path}` 로 만들어 `?utm=...` 같은 쿼리는 넣지 않는다.
- **에러 로그는 메시지만**: axios 에러 객체에는 요청 설정(`Authorization: Bearer <토큰>`)이 들어 있어서, 전체 객체 대신 `error.message` 만 `console.error` 로 남긴다.
- **에러 페이지에는 meta/og 없음**: 에러 페이지(D18)는 `<title>영화 리뷰</title>` 만 두고 description·og 태그는 넣지 않는다.
- **og:description 빈 줄거리 대체 문구**: 줄거리가 없으면 모달과 같은 "줄거리 정보가 없습니다." 를 쓴다.
- **기존 코드 유지**: `server.ts` 의 `express.json()`, 라우트 → static 등록 순서는 그대로 둔다.

---

## 미결정

구현하면서 새로 생기는 결정은 여기에 먼저 적고, 정해지면 위 형식으로 옮긴다.

- (없음)

