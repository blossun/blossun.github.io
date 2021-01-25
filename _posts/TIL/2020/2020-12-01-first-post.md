---
title: "TIL 기록 시작"
excerpt: "Project 진행 로그와 TIL을 기록하기 위해 블로그 개설"

categories:
  - TIL/2020
tags:
  - TIL
---

YFM에서 정의한 제목을 이중 괄호 구문으로 본문에 추가할 수 있다.
이 글의 제목은 {{ page.title }}이고
마지막으로 수정된 시간은 {{ page.last_modified_at }}이다.

**지킬이 해석하는 YFM(YAML Front Matter) 포맷**
* markdown 파일의 최상단에 위치하며 3개의 하이픈으로 시작과 끝을 표시한다. YAML은 오픈 소스 프로젝트에서 많이 사용하는 구조화된 데이터 형식
* YFM은 이 YAML을 사용해서 글의 제목, 날짜, 카테고리, 태그, 레이아웃 등을 정의할 수 있다.
* YFM에서 정의한 제목인 title을 이중 괄호 구문으로 본문에 추가할 수 있다.
  makrdown 파일 본문에는 이중 괄호 구문을 사용하여 표기하면 이를 지킬은 page.title를 YFM에서 정의한 문구로 교체하여 html로 변환한다.
