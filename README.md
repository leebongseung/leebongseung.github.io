# Bong's Dev Blog

[![Live Site](https://img.shields.io/badge/Live-leebongseung.github.io-blue)](https://leebongseung.github.io)
[![Theme](https://img.shields.io/badge/theme-Chirpy-green)](https://github.com/cotes2020/jekyll-theme-chirpy)

개발 학습과 실무 경험을 기록하는 기술 블로그입니다.

## 관심 분야

- 인증/인가 (Keycloak, Spring Security)
- 분산 시스템 (Redis, Kafka)
- 백엔드 아키텍처 (Spring Boot, DDD)

## 기술 스택

- **SSG**: Jekyll 4.x
- **Theme**: [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 7.5
- **호스팅**: GitHub Pages (GitHub Actions 빌드)
- **댓글**: giscus (GitHub Discussions 기반)

## 로컬 실행

```bash
bundle install
bundle exec jekyll s
# http://localhost:4000
```

## 글 작성

```bash
# _posts/YYYY-MM-DD-slug.md
---
title: "제목"
date: 2026-04-07
categories: [대분류, 소분류]
tags: [tag1, tag2]
pin: false          # 상단 고정 여부
math: false         # 수식 사용
mermaid: false      # 다이어그램 사용
---
```

## License

Posts are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
