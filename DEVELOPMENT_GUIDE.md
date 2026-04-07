# LMS 통합 크롤러 - 개발자 가이드 (Developer Guide)

이 문서는 추후 다른 대학의 LMS를 추가 연동하거나, 기존 코드를 유지보수할 때 참조하기 위해 작성되었습니다.

---

## 1. 프로젝트 아키텍처 개요

이 프로젝트는 **Electron** 기반으로, 크게 두 부분으로 나뉘어져 통신합니다.

1. **상태 및 크롤링 제어 (Main Process - Node.js)**
   * `main.js`: 앱의 생명주기를 관리하고 크롤링을 동작시킬 백그라운드 브라우저(`crawlWin`)를 생성/제어합니다.
   * `src/adapters/`: 실제 크롤링 또는 API Fetch 로직을 담은 클래스들이 위치합니다.
2. **UI 렌더링 (Renderer Process - Browser)**
   * `renderer/index.html` 및 `renderer/renderer.js`: 사용자가 보는 로그인 화면, 과목 대시보드 구조입니다.
   * Node.js 권한에서 분리되어 있어, 데이터 요청은 `window.api.send()` 및 `window.api.receive()` (IPC 통신, `preload.js`에 정의됨)를 통해 Main Process로 전달됩니다.

---

## 2. 어댑터(Adapter) 구조 설명

개발의 편의를 위해 **어댑터 패턴**을 차용했습니다. 메인 로직(`main.js`)은 바뀌지 않고, 각 대학의 세부 구현 클래스만 다르도록 설계되었습니다.

* `LmsAdapter` (Base): 모든 대학 Adapter가 상속받는 기본 뼈대입니다.
  * `login(credentials)`: 초기 로그인(인증)을 수행. `true`/`false` 반환.
  * `crawlMainPage()`: 로그인 된 후 유저 정보와 수강 과목 목록을 추출.
  * `crawlCourseDetail(course, index, total)`: 추출된 과목 목록 중 하나의 상세 내용(공지, 과제 등)을 추출.

---

## 3. Hello LMS 기반 학교 (ex. 대구대학교)

국내 웹케시(WebCash) 자회사가 만든 "스마트 출결/LMS" 시스템으로, 국내 대다수의 대학이 사용 중입니다. 이 시스템은 API가 폐쇄적이기 때문에 **HTML DOM 파싱(스크래핑)** 방식으로 접근합니다.

### 3-1. 동작 방식 (`HelloLmsAdapter.js`)
백그라운드에서 열린 `crawlWin` 브라우저 창에서, `fetch` 요청을 보내고 응답받은 HTML 텍스트를 DOM 객체로 만든 후 파싱하는 코드를 삽입하여 실행합니다.

### 3-2. HTML DOM 구조
* **메인 과정 추출 (`/ilos/main/main_form.acl`)**
  * 유저 정보: `document.getElementById('user')`
  * 과목 정보: `document.querySelectorAll('em.sub_open[kj]')` 안에 들어있으며 `kj`(교과목 고유 키)를 가집니다.
* **과목 상세 진입 (`/ilos/st/course/eclass_room2.acl`)**
  * `KJKEY`를 x-www-form-urlencoded 데이터로 POST 전송하면 해당 과목방으로 세션이 변경됨.
* **세부 메뉴 추출 (`/ilos/st/course/plan_form.acl` 등)**
  * 과목방 입장 후, `table tbody tr`을 돌면서 데이터 셀(`td`)들을 뽑아옵니다.

---

## 4. Canvas LMS 기반 학교 (ex. 경북대학교)

Instructure에서 만든 글로벌 오픈소스 LMS인 **Canvas**를 사용합니다. 완전한 **REST API**를 지원하므로 HTML 파싱이 사실상 필요 없고 매우 안정적입니다.

### 4-1. 동작 방식 (`KnuLms.js`)
KNU 통합 로그인(SSO) 창에서 로그인을 뚫어내면, 브라우저(`crawlWin`)에 Canvas 세션이 만들어집니다. 이 세션을 이용해서 곧바로 JSON API 엔드포인트로 `fetch` 합니다.

### 4-2. 필수 Canvas API 구조
Canvas는 `/api/v1/...` 형태로 거의 모든 데이터를 가져올 수 있습니다.
* **유저 정보**: `GET /api/v1/users/self/profile`
* **수강중 과목**: `GET /api/v1/courses?per_page=100&enrollment_state=active&include[]=term`
  * 반환된 객체의 `id`가 강좌 Key가 됩니다.
* **공지사항**: `GET /api/v1/announcements?context_codes[]=course_{id}`
* **과제 및 시험**: `GET /api/v1/courses/{id}/assignments`
* **강의 자료**: `GET /api/v1/courses/{id}/modules?include[]=items`
* **토론**: `GET /api/v1/courses/{id}/discussion_topics`

데이터가 JSON으로 예쁘게 떨어지기 때문에 UI 규칙(배열 속에 `title`, `link`, `cells` 항목) 규칙에 맞추어 담아서 리턴해주기만 하면 뷰에 예쁘게 그려집니다.

---

## 5. 새로운 학교 추가 튜토리얼

1. `src/adapters/universities/` 폺더에 새로운 파일 `MyUnivLms.js` 를 생성합니다.
2. 해당 학교가 Hello LMS 라면 `HelloLmsAdapter`를 상속 받고, Canvas 베이스라면 `KnuLms`를 복사해서 쓰거나, 자체 LMS라면 `LmsAdapter`를 상속받습니다.
3. `login()`, `crawlMainPage()`, `crawlCourseDetail()` 동작을 각 사이트에 맞게 짭니다.
4. `renderer/index.html` 파일을 열고 로그인 Select 박스(소속 학교)에 `option`을 추가해주어 사용자 화면에 띄웁니다.
```html
<option value="myuniv">새로운대학교</option>
```
5. `main.js`의 `ipcMain.on('login')` 에 분기(`if (univId === 'myuniv') currentAdapter = new MyUnivLms();`)를 추가합니다.
