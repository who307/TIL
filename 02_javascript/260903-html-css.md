## html css

## TIL(Today I Learned)
- HTML은 웹 페이지의 구조를 만드는 마크업 언어
- 기본 태그와 시맨틱 태그의 개념 및 사용 방법 학습
- CSS 선택자와 글꼴 스타일 속성을 학습

## 핵심 내용(코드)
/* 태그 선택자 */
p {
  color: #333;
}

/* 클래스 선택자 */
.title {
  color: royalblue;
}

/* 아이디 선택자 */
#description {
  font-size: 18px;
}

/* 자손 선택자 */
main p {
  line-height: 1.6;
}

/* 여러 선택자에 동일한 스타일 적용 */
h1,
h2 {
  font-family: Arial, sans-serif;
}

반영 우선순위
전체 선택자(*) < 태그 선택자 < 클래스 선택자 < 아이디 선택자 < 인라인 스타일 < !important

## 느낀 점
- HTML 기본 태그와 시맨틱 태그의 개념을 배우고 CSS 선택자를 사용해 원하는 요소에 스타일을 적용하는 방법을 익힘