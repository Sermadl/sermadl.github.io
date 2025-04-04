---
title: "Vue 시작하기(1) - 초기 구성 (API 연결)"
categories: [Vue, MSA]
tags: [MSA, Vue, SKALA, SKALA1기, SK]
---

> Skala 과정에서 마이크로 서비스 아키텍처 구조에 대해 새롭게 알게 되었습니다.<br>
> 배운 것을 실제로 구현해보기 위해 이 여정을 시작하기로 했습니다.<br>[처음 글부터 보러가기](<https://sermadl.github.io/posts/MSA(1)/>)

백엔드의 깊이를 넓혀가는 것도 좋지만, Skala 과정이 풀스택 과정인 만큼 프론트 공부도 시작해보기로 했습니다!<br>
화면을 구성하다보니 구현되지 않은 기능들도 조금 있어서 백엔드와 프론트를 오가며 구현해볼 것 같습니다.<br>

추후에 사용자가 원하는 아이템이 있는 카테고리를 추천해준다던가, 상품의 가격 및 정보를 비교해주는 챗봇도 구현해보고자 합니다(될까요?)<br>

이번 글에서는 Vue 초기 설정과 로그인, 홈 화면까지 간단하게 구성해봤습니다.<br>

화면을 구성하다보니 프로젝트에 이름이 필요해서 나만의 쿠팡이라는 뜻으로 `마이팡`이라고 지었습니다 ㅎㅎ<br>

<hr>

## 1. 프로젝트 생성 및 디렉토리 구조 파악하기

### 프로젝트 생성하기

Vue 프로젝트를 시작하고자 하는 폴더에 들어가서

```shell
npm create vue@latest

vue create [생성하고자 하는 프로젝트의 이름]
```

를 터미널에 입력하여 프로젝트를 시작합니다!<br>

처음에는 Vue가 자동으로 생성해주는 컴포넌트 및 페이지들이 있는데, 저는 이것들을 전부 삭제해주었습니다.<br>

<hr>

### 디렉토리 구조 파악하기

Vue의 디렉토리 구조는 다음과 같습니다.<br>

```
my-vue-app(프로젝트명)/
│
├── public/                     # 정적 파일 (favicon, index.html 등)
│   └── index.html
│
├── src/                        # 주요 소스 코드
│   ├── assets/                 # 이미지, 폰트 등 정적 리소스
│   ├── components/             # 재사용 가능한 Vue 컴포넌트들
│   │   └── MyButton.vue
│   ├── views/                  # 라우트 단위 페이지 컴포넌트
│   │   └── HomeView.vue
│   ├── router/                 # Vue Router 설정
│   │   └── index.js
│   ├── store/                  # Response/Request 등 상태 관리
│   │   └── index.js
│   ├── App.vue                 # 루트 컴포넌트
│   └── main.js                 # 앱 진입점
│
├── .env                        # 환경 변수 파일
├── package.json                # 의존성 및 스크립트
└── vite.config.js / vue.config.js # 빌드 도구 설정
```

앞으로 작성하게 될 파일들은 경로를 자세히 작성하지 않을 예정이니<br>
혹시 작성할 파일의 디렉토리가 헷갈린다면 위 구조를 참고하면 됩니다.<br>

명명 규칙은 vue 파일을 제외한 모든 파일은 kebab-case, vue 파일은 CamelCase입니다.<br>

Spring Boot 처럼 파일들을 잘 구분해서 작성하는 것이 좋겠습니다!<br>

<hr>

## 2. 초기 화면 구성하기

### 라우터 설정

`localhost:5173` 뒤에 올 url에 따라 보여질 페이지를 변경하기 위해 라우터를 설정합니다.<br>

![vue-pages](/assets/img/vue-pages.png)

저는 위와 같이 초기 페이지를 구성해보았습니다.<br>

[ router.js ]

```js
import { createRouter, createWebHistory } from "vue-router";
import Start from "./pages/Start.vue";
import Unknown from "./pages/Unknown.vue";
import Home from "./pages/Home.vue";
import Login from "./pages/Login.vue";
import Register from "./pages/Register.vue";

const routes = [
  { path: "/", component: Home },
  { path: "/start", component: Start },
  { path: "/login", component: Login },
  { path: "/register", component: Register },
  { path: "/:pathMatch(.*)*", component: Unknown }
];

const router = createRouter({
  history: createWebHistory(),
  routes
});

export default router;
```

`router.js` 를 통해 각 페이지 별로 url을 설정해줍니다.<br>
<br>

[ App.vue ]

```js
<template>
  <RouterView></RouterView>
</template>

<script setup>
import { RouterView } from 'vue-router'
</script>
```

라우터 설정을 완료하기 위해 `App.vue` 에 라우터를 등록합니다.<br>

<hr>

### 부트스트랩(스타일) 적용하기

우선 부트스트랩을 설치해줍니다<br>

```shell
npm install bootstrap
npm install bootstrap-icons # 부트스트랩 아이콘
```

[ main.js ]

```js
import "./assets/main.css";

import { createApp } from "vue";
import { createPinia } from "pinia"; // API 연결을 위해 필요한 코드입니다
import App from "./App.vue";
import router from "./router";

import "bootstrap/dist/css/bootstrap.min.css";
import "bootstrap/dist/js/bootstrap.bundle";
import "bootstrap-icons/font/bootstrap-icons.css";

const app = createApp(App);
app.use(router);
app.use(createPinia()); // API 연결을 위해 필요한 코드입니다
app.mount("#app");
```

`main.js` 에 import 해서 부트스트랩을 적용합니다!<br>

<hr>

### 시작 화면 구성하기

[ Start.vue ]

```js
<template>
  <div class="h-screen flex flex-col justify-center items-center bg-gray-100">
    <h1 class="text-center">마이팡</h1>
    <MainLoginButton></MainLoginButton>
  </div>
</template>

<script setup>
import { useRouter, useRoute } from "vue-router";
import MainLoginButton from '@/components/MainLoginButton.vue'

</script>
```

<br>

[ MainLoginButton.vue ]

```js
<template>
    <div class="text-center">
        <button class="btn btn-primary" @click="$router.push('/register')">시작하기</button>
        <br>
        <button class="btn btn-link" @click="$router.push('/login')">바로 로그인하러 가기</button>
    </div>
</template>

<script setup>

</script>
```

컴포넌트인 `MainLoginButton.vue` 파일을 `Start.vue` 에 등록합니다.<br>

시작 화면은 아주아주 간단하게 만들어봤습니다..!<br>

![vue-start](/assets/img/vue-start.png)

<hr>

### 로그인 화면 구성하기

로그인 화면을 구성하는데에도 꽤 시간이 걸렸습니닷.<br>

[ Login.vue ]

```js
<template>
    <LoginForm></LoginForm>
</template>

<script setup>
import { useRouter } from 'vue-router'
import LoginForm from '@/components/LoginForm.vue';
</script>
```

<br>

[ LoginForm.vue ]

```js
<script setup>
import { ref, computed } from "vue";
import { useRouter } from "vue-router";
import { useAuthStore } from "@/scripts/store-auth";
import apiCall from "@/scripts/api-call";

const email = ref("");
const password = ref("");
const router = useRouter();

const login = async () => {
  if (!email.value || !password.value) {
    alert("이메일과 비밀번호를 입력해주세요.");
    return;
  }

  const requestBody = {
    email: email.value,
    password: password.value,
  };

  const authStore = useAuthStore();

  const url = "/api/user/login";
  const result = await apiCall.post(url, null, requestBody);

  if (result.result === apiCall.Response.SUCCESS) {
    alert("로그인 성공");

    authStore.setAuth(result.data);

    router.push("/");
  } else {
    console.log(result.status);

    if (result.status < 500) {
      alert("이메일 또는 비밀번호가 잘못되었습니다.");
    } else {
      alert("로그인에 실패했습니다.");
    }
  }
};

const emailError = computed(() => {
  if (!email.value) return '';
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email.value) ? '' : '올바른 이메일 형식을 입력해주세요.';
});
</script>

<template>
  <div class="d-flex justify-content-center align-items-center">
    <div class="card p-4" style="min-width: 300px">
      <h3 class="text-center mb-3">로그인</h3>
      <div class="mb-3">
        <input
          v-model="email"
          type="email"
          class="form-control"
          placeholder="이메일"
        />
        <small class="text-danger" v-if="emailError">{{ emailError }}</small>
      </div>
      <div class="mb-3">
        <input
          v-model="password"
          type="password"
          class="form-control"
          placeholder="비밀번호"
        />
      </div>
      <button class="btn btn-primary w-100" @click="login">로그인</button>
    </div>
  </div>
</template>
```

<br>

[ api-call.ts ]

```js
export const Response = {
  SUCCESS: 0,
  FAIL: 1,
};

const apiHeaders = {
  "Content-Type": "application/json",
};

const handleResponse = async (response: Response) => {
  const status = response.status;
  const data = await response.json();

  if (response.ok) {
    return {
      result: Response.SUCCESS,
      code: 200,
      message: "",
      data: data,
    };
  } else {
    const errorMsg = data.message || "오류가 발생했습니다.";
    return {
      result: Response.FAIL,
      code: data.code,
      status: status,
      message: errorMsg,
      data: null,
    };
  }
};

const handleError = (error: any) => {
  const errorMsg = error.message || error || "Unknown error";
  return {
    result: Response.FAIL,
    code: "NETWORK_ERROR",
    message: errorMsg,
    data: null,
  };
};

const sendRequest = async (
  method: string,
  url: string,
  headers: { [key: string]: any } | null = {},
  body: any = null
) => {
  try {
    const options: RequestInit = {
      method,
      headers: { ...apiHeaders, ...headers },
    };

    if (method !== "GET" && method !== "HEAD") {
      options.body = JSON.stringify(body);
    }

    const response = await fetch(url, options);
    return await handleResponse(response);
  } catch (error: any) {
    return handleError(error);
  }
};

const buildUrlWithParams = (
  url: string,
  params: { [key: string]: any } | null
) => {
  if (!params) return url;
  const searchParams = new URLSearchParams();
  Object.entries(params).forEach(([key, value]) => {
    if (Array.isArray(value)) {
      value.forEach((val) => searchParams.append(key, val));
    } else {
      searchParams.append(key, value);
    }
  });
  return `${url}?${searchParams.toString()}`;
};

const apiCall = {
  Response,

  get: async (url: string, headers: any = null, queryParams: any = null) =>
    await sendRequest("GET", buildUrlWithParams(url, queryParams), headers),

  post: async (url: string, headers: any = null, body: any = null) =>
    await sendRequest("POST", url, headers, body),

  put: async (url: string, headers: any = null, body: any = null) =>
    await sendRequest("PUT", url, headers, body),

  delete: async (url: string, headers: any = null, body: any = null) =>
    await sendRequest("DELETE", url, headers, body),
};

export default apiCall;
```

<br>

[ store-auth.js ]

```js
import { defineStore } from "pinia";
import { ref, watch } from "vue";

export const useAuthStore = defineStore("auth", () => {
  const accessToken = ref(localStorage.getItem("accessToken") || "");
  const refreshToken = ref(localStorage.getItem("refreshToken") || "");
  const role = ref(localStorage.getItem("role") || "");

  const setAuth = (data) => {
    accessToken.value = data.accessToken;
    refreshToken.value = data.refreshToken;
    role.value = data.role;

    localStorage.setItem("accessToken", accessToken.value);
    // localStorage.setItem('refreshToken', val)
    localStorage.setItem("role", role.value);
  };

  const clearAuth = () => {
    accessToken.value = "";
    refreshToken.value = "";
    role.value = "";

    localStorage.removeItem("accessToken");
    // localStorage.removeItem('refreshToken');
    localStorage.removeItem("role");
  };

  const getAuth = () => {
    return {
      accessToken: accessToken.value,
      refreshToken: refreshToken.value,
      role: role.value
    };
  };

  // Todo: refreshToken은 https로 통신할 때 저장
  // watch(refreshToken, (val) => localStorage.setItem('refreshToken', val));

  return {
    accessToken,
    refreshToken,
    role,
    setAuth,
    clearAuth,
    getAuth
  };
});
```

Skala의 프론트 수업 때 받았던 소스코드의 일부를 리팩토링하여 적용했습니다<br>

`api-call.ts` 코드에서 비동기적으로 API를 호출하는 방식을 템플릿화(?) 합니다!<br>
`store-auth.js` 코드를 통해 API 호출 결과 및 요청 데이터를 템플릿화(..?) 합니다.<br>

토큰을 어디다가 저장할까 고민을 하다가 리프레시 토큰만 추후에 htts를 적용한다는 가정하에 쿠키에 저장하기로 했습니다.<br>
액세스 토큰과 유저 역할은 로컬 스토리지에 저장합니다<br>

백엔드에서 에러 메시지를 커스텀 해두었기 때문에 응답 코드 별로 성공 혹은 실패를 구별할 수 있고<br>
어떤 에러가 발생했는지 알 수 있어서 바로 에러 메시지를 `alert()` 로 출력되도록 할 수도 있지만 아이디를 잘못 입력하였을 때<br>
`사용자를 찾을 수 없습니다` 라는 메시지 보다는 `이메일 또는 비밀번호를 확인해주세요` 라는 메시지가 자연스러울 것 같아 그렇게 적용해두었습니다.<br>

이메일이 잘못된 경우나 비밀번호가 잘못된 경우에 각각 에러 메시지가 다른 경우가 잘 없었던 것 같아서 에러 메시지를 통일시켰습니다.(아무래도 보안 문제인 것 같습니다)<br>

이메일 형식을 입력하도록 유도하기 위해 검증 기능도 구현했습니다.<br>

<hr>

### 홈 화면 구성하기

홈 화면을 구성하는데 아주 많은 시간이...들 것 같습니다... ~~오늘도 많이 들었어요~~<br>

일단 홈 화면에 있어야 할 구성요소들을 추려봤는데<br>

1. 로그인/로그아웃, 회원가입 등이 있는 네비게이션 바
2. 카테고리, 플랫폼(?)명, 장바구니, 마이페이지가 있는 네비게이션 바
3. 검색 창
4. 둘러보기(?) 상품 목록들

꽤...많더라구요...<br>

아직 검색 기능이나 이미지 업로드 기능을 추가하진 않아서 일단 화면만 먼저 구성해봤습니다.<br>

![vue-home-init](/assets/img/vue-home-init.png)

[쿠팡 홈페이지](https://www.coupang.com/)를 보고 영감을 조금 받았습니다. ㅎㅎ..<br>

[ Home.vue ]

```js
<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import apiCall from "@/scripts/api-call";
import { useAuthStore } from "@/scripts/store-auth";

const authStore = useAuthStore();
const categories = ref([]);
const isDropdownOpen = ref(false);

const fetchCategories = async () => {
  const result = await apiCall.get("/api/item/category", null);
  if (result.result === apiCall.Response.SUCCESS) {
    categories.value = result.data;
  }
};

const closeOnClickOutside = (event) => {
  if (!event.target.closest(".dropdown")) {
    isDropdownOpen.value = false;
  }
};

onMounted(() => {
  fetchCategories();
  document.addEventListener("click", closeOnClickOutside);
});
onBeforeUnmount(() => {
  document.removeEventListener("click", closeOnClickOutside);
});
</script>

<template>
  <div>
    <!-- 상단 메뉴 부분 시작! -->
    <header>
      <!-- 첫번째 줄 -->
      <nav class="navbar bg-body-tertiary border-bottom p-0">
        <div class="container-fluid">
          <a></a>
          <div>
            <button
              class="btn btn-sm"
              @click="
                $router.push(authStore.accessToken ? '/logout' : '/login')
              "
            >
              {% raw %}{{ authStore.accessToken ? "로그아웃" : "로그인" }}{% endraw %}
            </button>
            <button class="btn btn-sm" @click="$router.push('/register')">
              회원가입
            </button>
            <button class="btn btn-sm" @click="$router.push('/register')">
              판매자 가입
            </button>
          </div>
        </div>
      </nav>

      <!-- 두번째 줄 -->
      <nav
        class="d-flex justify-content-between align-items-center px-3 py-2 border-bottom"
        style="width: 100%"
      >
        <!-- 왼쪽: 드롭다운 메뉴 + 로고 -->
        <div class="d-flex align-items-center">
          <div class="dropdown me-1">
            <button
              class="btn dropdown-toggle"
              type="button"
              data-bs-toggle="dropdown"
              aria-expanded="false"
              @click.stop="isDropdownOpen = !isDropdownOpen"
            >
              <i v-if="!isDropdownOpen" class="bi bi-list fs-4"></i>
              <i v-else class="bi bi-x-lg fs-4"></i>
            </button>
            <ul class="dropdown-menu">
              <li
                v-for="(category, index) in categories"
                :key="index"
                class="dropdown-submenu"
              >
                <a class="dropdown-item" href="#">
                  {{ category.largeCategory }}
                </a>
                <div
                  v-if="
                    category.smallCategories && category.smallCategories.length
                  "
                  class="submenu bg-white border p-2"
                >
                  <button
                    v-for="(small, idx) in category.smallCategories"
                    :key="idx"
                    class="btn btn-sm w-100 mb-1 text-start"
                  >
                    {{ small }}
                  </button>
                </div>
              </li>
            </ul>
          </div>
          <h3 class="mb-0">마이팡</h3>
        </div>

        <!-- 오른쪽: 장바구니 / 마이페이지 -->
        <div>
          <button class="btn position-relative">
            <i class="bi bi-cart2 fs-4"></i>
            <span
              class="badge rounded-pill bg-primary top-50 start-100 position-absolute translate-middle"
            >
              0
            </span>
          </button>
          <button class="btn">
            <i class="bi bi-person fs-4"></i>
          </button>
        </div>
      </nav>
    </header>
    <!-- 상단 메뉴 부분 끝! -->

    <!-- 전체 내용 래퍼 -->
    <div>
      <!-- 안내 문구 -->
      <h3 class="text-center mt-5">상품을 검색해보세요</h3>

      <!-- 검색 입력 -->
      <div class="d-flex justify-content-center mt-4">
        <input
          class="form-control form-control-lg w-50"
          type="text"
          placeholder="검색어를 입력해주세요"
        />
        <button class="btn btn-primary ms-2">검색</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
button.dropdown-toggle::after {
  display: none !important;
}

dropdown-submenu {
  position: relative;
}

.dropdown-submenu .submenu {
  position: absolute;
  top: 0;
  left: 100%;
  min-width: 160px;
  display: none;
  z-index: 1000;
}

.dropdown-submenu:hover .submenu {
  display: block;
}
</style>
```

일단 `Home.vue` 코드는 위와 같습니다.<br>

하나하나씩 천천히 뜯어볼게용
<br>

<hr>

[ 최상단 네비게이션 바 ]

```js
<nav class="navbar bg-body-tertiary border-bottom p-0">
    <div class="container-fluid">
        <a></a>
        <div>
        <button
            class="btn btn-sm"
            @click="
            $router.push(authStore.accessToken ? '/logout' : '/login')
            "
        >
            {% raw %}{{ authStore.accessToken ? "로그아웃" : "로그인" }}{% endraw %}
        </button>
        <button class="btn btn-sm" @click="$router.push('/register')">
            회원가입
        </button>
        <button class="btn btn-sm" @click="$router.push('/register')">
            판매자 가입
        </button>
        </div>
    </div>
</nav>
```

로그인/로그아웃, 회원가입, 판매자 가입이 있는 최상단 네비게이션 바 구성 코드입니다.<br>
오른쪽으로 붙이고 싶어서 `<a>` 태그를 이용했습니다.<br>

부트스트랩 스타일 중 하나인 navbar를 이용해서 구성했는데, 이렇게 구성하는 방법 외에 [`<ui>` 와 `<li>` 를 이용하는 방법](https://getbootstrap.com/docs/5.3/components/navbar/)도 있더라구용...?<br>

사실 큰 차이는 없을 것 같아서 (그리고 지금 화면이 충분히 마음에 들어서..~~귀찮아서..~~) 이대로 구성하기로 했습니다<br>

카테고리 연결하다가 이 부분 연결을 못해서,,,다음 글에서는 홈화면을 완벽하게 구성해서 돌아올게용,,,<br>

<hr>

[ 검색창 ]

```js
<div>
  <h3 class="text-center mt-5">상품을 검색해보세요</h3>

  <div class="d-flex justify-content-center mt-4">
    <input
      class="form-control form-control-lg w-50"
      type="text"
      placeholder="검색어를 입력해주세요"
    />
    <button class="btn btn-primary ms-2">검색</button>
  </div>
</div>
```

두번째 상단 네비게이션 바를 구성하기 위해서 거의 홈 구성 코드의 70%를 작성해서 검색창을 먼저 살펴보겠습니다<br>

실제로 동작하는 창이 아니라서 껍데기만 만들어뒀습니다!<br>

이 구성은 [당근](https://www.daangn.com/kr)에서 영감을 받았습니다 ㅎ.ㅎ<br>

<hr>

[ 두번째 상단 네비게이션 바 ]

- 스크립트 내부 코드

```js
import { ref, onMounted, onBeforeUnmount } from "vue";
import apiCall from "@/scripts/api-call";
import { useAuthStore } from "@/scripts/store-auth";

const authStore = useAuthStore();
const categories = ref([]);
const isDropdownOpen = ref(false);

const fetchCategories = async () => {
  const result = await apiCall.get("/api/item/category", null);
  if (result.result === apiCall.Response.SUCCESS) {
    categories.value = result.data;
  }
};

const closeOnClickOutside = (event) => {
  if (!event.target.closest(".dropdown")) {
    isDropdownOpen.value = false;
  }
};

onMounted(() => {
  fetchCategories();
  document.addEventListener("click", closeOnClickOutside);
});
onBeforeUnmount(() => {
  document.removeEventListener("click", closeOnClickOutside);
});
```

- 템플릿 내부 코드

```js
<nav
    class="d-flex justify-content-between align-items-center px-3 py-2 border-bottom"
    style="width: 100%"
    >
    <!-- 왼쪽: 드롭다운 메뉴 + 로고 -->
    <div class="d-flex align-items-center">
        <div class="dropdown me-1">
        <button
            class="btn dropdown-toggle"
            type="button"
            data-bs-toggle="dropdown"
            aria-expanded="false"
            @click.stop="isDropdownOpen = !isDropdownOpen"
        >
            <i v-if="!isDropdownOpen" class="bi bi-list fs-4"></i>
            <i v-else class="bi bi-x-lg fs-4"></i>
        </button>
        <ul class="dropdown-menu">
            <li
            v-for="(category, index) in categories"
            :key="index"
            class="dropdown-submenu"
            >
            <a class="dropdown-item" href="#">
                {{ category.largeCategory }}
            </a>
            <div
                v-if="
                category.smallCategories && category.smallCategories.length
                "
                class="submenu bg-white border p-2"
            >
                <button
                v-for="(small, idx) in category.smallCategories"
                :key="idx"
                class="btn btn-sm w-100 mb-1 text-start"
                >
                {{ small }}
                </button>
            </div>
            </li>
        </ul>
        </div>
        <h3 class="mb-0">마이팡</h3>
    </div>

    <!-- 오른쪽: 장바구니 / 마이페이지 -->
    <div>
        <button class="btn position-relative">
        <i class="bi bi-cart2 fs-4"></i>
        <span
            class="badge rounded-pill bg-primary top-50 start-100 position-absolute translate-middle"
        >
            0
        </span>
        </button>
        <button class="btn">
        <i class="bi bi-person fs-4"></i>
        </button>
    </div>
    </nav>
</header>
```

`justify-content-between` 가 해당 태그 안에 들어있는 요소 사이의 간격을 만들어주는 스타일인데 구성요소들을 양쪽으로 붙이고 싶어서 사용했습니다.<br>

장바구니 기능은 아마 로컬스토리지에 아이템을 저장하는 형식으로 구현하게 되지 않을까..싶습니다..ㅎㅎ...<br>

마이페이지 기능은 이미 구현되어 있긴하지만 빈약하게 구현이 되어있는터라 완벽(?)하게 구현되어있는 카테고리 먼저 연결을 해봤습니다!<br>
이전에 구현해두었던 `api-call.ts` 를 사용해 API를 호출합니다.<br>

부트스트랩에서 지원하는 스타일을 최대한 많이 적용해보려고 노력했습니다...!!<br>
제 눈에 계속 못생겨 보이니까 자꾸 욕심이..나네용...<br>

<hr>

## 마치며

처음으로 제대로 Vue 프로젝트를 시작해봤는데<br>
화면 구성..어려워요........귀찮아요............🥲<br>

`api-call.ts` 리팩토링 과정도 정말 쉽지 않았습니다 ㅜ^ㅜ..<br>
자바스크립트와 뷰 문법에 대한 지식이 한층 더 늘어난 시간이었습니다 ㅎㅎ<br>
프론트 잘 하는 조원과 GPT와 함께 더욱 성장해보겠습니닷..우하하<br>

다음 글부터는 Vue 관련 트러블 슈팅 위주로 작성해보겠습니다!<br>

🐛

<hr>
<br>

> 참고 자료

[하나](https://getbootstrap.com/), [둘](https://uibowl.io/website), [셋](https://chatgpt.com/)
