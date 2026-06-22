# TIL: Playwright Web Crawling Basics

> 날짜: 2026-06-22  
> 태그: `#Python` `#Playwright` `#Crawling` `#AI` `#Backend`

---

## 📌 Playwright란?

**Playwright** 는 Microsoft가 만든 브라우저 자동화 라이브러리다.
JavaScript로 동적 렌더링되는 페이지도 크롤링할 수 있다는 게 핵심이다.

```
requests + BeautifulSoup:
  → HTML을 그냥 가져옴
  → JavaScript로 렌더링되는 콘텐츠는 못 가져옴 ❌

Playwright:
  → 실제 브라우저를 실행해서 JS 렌더링까지 기다림
  → 동적 페이지 크롤링 가능 ✅
  → 로그인, 버튼 클릭, 스크롤 등 사용자 동작 자동화 가능 ✅
```

> 채용공고 사이트(원티드, 사람인 등)는 대부분 JS 렌더링 → Playwright 필수!

---

## 📌 설치 & 기본 설정

```bash
pip install playwright
playwright install chromium  # 브라우저 설치 (chromium 권장)
```

```python
# 동기 방식 (기본)
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)  # headless=False면 브라우저 창 보임
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()
```

---

## 📌 핵심 기능

### 페이지 이동 & 대기

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()

    # 페이지 이동
    page.goto("https://www.wanted.co.kr/jobsfeed")

    # 로딩 대기 방법 3가지
    page.wait_for_load_state("networkidle")       # 네트워크 요청이 끝날 때까지
    page.wait_for_selector(".job-card")           # 특정 요소가 나타날 때까지
    page.wait_for_timeout(2000)                   # 2초 고정 대기 (비권장)

    browser.close()
```

---

### 요소 선택 & 텍스트 추출

```python
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page()
    page.goto("https://example.com/job/123")

    # CSS 선택자로 요소 찾기
    title = page.locator("h1.job-title").inner_text()
    company = page.locator(".company-name").inner_text()
    description = page.locator("#job-description").inner_text()

    # 여러 요소 한 번에 가져오기
    tags = page.locator(".tech-tag").all_inner_texts()
    # → ["Python", "FastAPI", "PostgreSQL"]

    # 속성 가져오기
    link = page.locator("a.apply-btn").get_attribute("href")

    print(f"제목: {title}")
    print(f"회사: {company}")
    print(f"기술스택: {tags}")

    browser.close()
```

---

### 클릭 & 입력

```python
with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    page = browser.new_page()
    page.goto("https://example.com/login")

    # 입력
    page.fill("input[name='email']", "test@test.com")
    page.fill("input[name='password']", "password123")

    # 클릭
    page.click("button[type='submit']")

    # 로그인 후 페이지 로딩 대기
    page.wait_for_url("**/dashboard")
    print("로그인 성공!")

    browser.close()
```

---

### 스크롤 (무한 스크롤 처리)

```python
def scroll_to_bottom(page, max_scrolls: int = 10):
    """무한 스크롤 페이지에서 모든 콘텐츠 로드"""
    prev_height = 0
    scroll_count = 0

    while scroll_count < max_scrolls:
        # 페이지 맨 아래로 스크롤
        page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
        page.wait_for_timeout(1500)  # 새 콘텐츠 로딩 대기

        # 스크롤 후 높이 변화 확인
        curr_height = page.evaluate("document.body.scrollHeight")
        if curr_height == prev_height:
            break  # 더 이상 새 콘텐츠 없음

        prev_height = curr_height
        scroll_count += 1
```

---

## 📌 비동기 방식 (FastAPI + Celery 연동 시)

```python
# async 방식 — FastAPI의 비동기 환경에 적합
from playwright.async_api import async_playwright
import asyncio

async def crawl_job_posting(url: str) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()

        await page.goto(url)
        await page.wait_for_load_state("networkidle")

        title = await page.locator("h1").inner_text()
        description = await page.locator(".job-description").inner_text()

        await browser.close()

        return {"title": title, "description": description}

# 실행
result = asyncio.run(crawl_job_posting("https://example.com/job/123"))
```

---

## 📌 실전 — 채용공고 크롤러

```python
from playwright.sync_api import sync_playwright
from dataclasses import dataclass
from typing import Optional

@dataclass
class JobPosting:
    title: str
    company: str
    location: str
    description: str
    tech_stack: list[str]
    url: str

def crawl_wanted_job(job_url: str) -> Optional[JobPosting]:
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)

        # User-Agent 설정 (봇 차단 우회)
        context = browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        )
        page = context.new_page()

        try:
            page.goto(job_url, timeout=30000)  # 30초 타임아웃
            page.wait_for_load_state("networkidle")

            title = page.locator("h2.job-title").inner_text()
            company = page.locator(".company-name").inner_text()
            location = page.locator(".location").inner_text()
            description = page.locator(".job-detail-content").inner_text()
            tags = page.locator(".tag").all_inner_texts()

            return JobPosting(
                title=title.strip(),
                company=company.strip(),
                location=location.strip(),
                description=description.strip(),
                tech_stack=tags,
                url=job_url
            )

        except Exception as e:
            print(f"크롤링 실패: {e}")
            return None

        finally:
            browser.close()
```

---

## 📌 크롤링 시 주의사항

```
✅ robots.txt 확인 — 크롤링 허용 여부 확인 (https://사이트/robots.txt)
✅ 요청 간 딜레이 — 서버 부하 방지, 차단 예방
   page.wait_for_timeout(random.randint(1000, 3000))  # 1~3초 랜덤 대기
✅ User-Agent 설정 — 봇 차단 우회
✅ 타임아웃 설정 — 무한 대기 방지
✅ 예외 처리 — 페이지 구조 변경에 대비

❌ 과도한 요청 금지 — 짧은 시간에 대량 요청은 IP 차단 위험
❌ 개인정보 크롤링 금지 — 법적 문제 발생 가능
```

---

## 📌 Celery Task로 통합

```python
# tasks.py
from celery_app import celery_app
from crawler import crawl_wanted_job

@celery_app.task(bind=True, max_retries=3)
def crawl_job_task(self, job_url: str):
    """Celery Worker에서 실행되는 크롤링 태스크"""
    try:
        self.update_state(state="PROGRESS", meta={"step": "크롤링 중"})
        job = crawl_wanted_job(job_url)

        if not job:
            raise ValueError("크롤링 결과 없음")

        return {"status": "success", "data": job.__dict__}

    except Exception as e:
        raise self.retry(exc=e, countdown=10)
```

---

## 💭 오늘 느낀 점

- Playwright가 단순한 HTML 파싱이 아니라 실제 브라우저를 제어한다는 점에서 requests보다 훨씬 강력하다는 걸 실감했다
- 채용공고 사이트처럼 JS 렌더링 페이지를 다룰 때 `wait_for_load_state("networkidle")`이 핵심이라는 걸 배웠다
- Celery Task와 연결하면 크롤링이 백그라운드에서 자동으로 돌아가는 구조가 완성된다 — 채용공고 에이전트의 마지막 퍼즐 조각이 맞춰지는 느낌이다

---

## 📚 참고

- [Playwright Python 공식 문서](https://playwright.dev/python/docs/intro)
- [Playwright Locators 가이드](https://playwright.dev/python/docs/locators)
- [Playwright 비동기 API](https://playwright.dev/python/docs/api/class-playwright)
