# HTML을 PDF로 만들어 주는 Claude Code 스킬 만들기 (깃허브 공개)

> 태그: Claude Code, Claude Skills, Puppeteer, PDF, 이력서, print CSS

이력서를 HTML로 관리하고 있습니다. 회사마다 지원동기를 바꿔 쓰다 보니 PDF를 하루에도 몇 번씩 다시 뽑게 됐고, 그때마다 손으로 "인쇄 → PDF로 저장"을 했습니다. 이 일을 Claude Code에게 "PDF 만들어줘" 한마디로 맡길 수 있게 **스킬(Skill)** 로 만들고, 다른 사람도 쓸 수 있게 깃허브에 올렸습니다.

- 저장소: https://github.com/HeeJeShin/html-pdf-skill

---

## 1. 손으로 PDF를 뽑을 때 생기던 문제

| 문제 | 원인 |
|---|---|
| 배경색·테두리 박스가 사라짐 | 인쇄 창에서 "배경 그래픽" 체크를 빼먹음 |
| 같은 HTML인데 쪽수가 달라짐 | 웹 폰트가 다 내려오기 전에 저장돼 기본 폰트로 줄바꿈됨 |
| 넣으면 안 되는 문구가 들어감 | 회사별 버전을 고치다가 연봉 같은 문구가 공개용 PDF에 남음 |
| 한국어 단어가 줄 끝에서 쪼개짐 | 브라우저 기본 줄바꿈 규칙 |

정리하면 **"매번 같은 방식으로"** 하지 못해서 생기는 문제였습니다. 그래서 사람이 아니라 스크립트가 하게 바꿨습니다.

---

## 2. 스킬이란

Claude Code의 스킬은 **"이런 요청이 오면 이 순서대로 일해라"를 적어 둔 폴더**입니다. 폴더 안에 `SKILL.md`가 있고, 맨 위 `description`을 보고 Claude가 언제 이 스킬을 쓸지 정합니다.

```
~/.claude/skills/html-pdf/
├── SKILL.md              ← Claude가 읽는 작업 순서와 규칙
├── package.json
└── scripts/
    ├── generate-pdf.mjs  ← HTML → PDF
    └── check-pdf.py      ← 쪽수·링크·금지 문구 점검
```

```markdown
---
name: html-pdf
description: HTML 파일을 인쇄용 PDF로 만들고 쪽수·링크·금지 문구를 점검한다.
  "PDF 만들어줘", "HTML을 PDF로", "/html-pdf <파일>" 등에 사용.
---
```

스킬에서 중요한 건 코드보다 **규칙**이었습니다. 제가 넣은 규칙은 이렇습니다.

- 문서 **내용**은 바꾸지 않는다. 쪽수를 맞추려고 내용을 빼야 하면 먼저 묻는다. 모양(CSS)만 고친다.
- 경고가 나오면 고치기 전에 먼저 보여 준다.
- 커밋·푸시는 하지 않는다.

이력서는 한 글자도 함부로 바뀌면 안 되는 문서라서, AI가 쪽수를 맞추겠다고 문장을 지우는 일을 막는 게 먼저였습니다.

---

## 3. 생성: Puppeteer

Puppeteer는 화면 없이 크롬을 띄워 코드로 조작하는 라이브러리입니다. HTML을 열고 `page.pdf()`를 부르면 PDF가 나옵니다.

```js
const browser = await puppeteer.launch({ headless: true });
const page = await browser.newPage();
await page.goto(url, { waitUntil: 'networkidle0' }); // 네트워크 요청이 다 끝날 때까지
await page.evaluate(() => document.fonts.ready);     // 웹 폰트 적용까지 기다림
await page.pdf({
  path: output,
  format: 'A4',
  margin: { top: '14mm', right: '14mm', bottom: '14mm', left: '14mm' },
  printBackground: true, // "배경 그래픽" 체크와 같음
});
```

손으로 할 때 생기던 문제 두 가지가 여기서 해결됩니다. `printBackground: true`로 배경이 항상 나오고, `document.fonts.ready`를 기다려서 폰트 때문에 쪽수가 바뀌지 않습니다.

---

## 4. 점검: 쪽수·링크·금지 문구

```bash
python3 check-pdf.py resume.pdf --html resume.html --max-pages 2 --forbid "직전 연봉"
```

```
== resume.pdf
pages: 2
links (4):
  https://github.com/...
WARN: 금지 문구 '직전 연봉'가 HTML에 있음
```

- **쪽수**: 기준을 넘으면 경고합니다.
- **링크**: PDF 안에 실제로 눌리는 링크 주소를 모두 보여 줍니다.
- **금지 문구**: PDF 본문은 압축돼 있어서 글자를 바로 못 찾을 때가 있습니다. 그래서 원본 HTML도 같이 검사합니다.

---

## 5. 오늘 실제로 겪은 print CSS 문제

이력서에 프로젝트 상세 표를 다시 넣었더니 **5쪽**이 나왔는데, 5쪽에는 기술 스택과 학력만 덩그러니 있었습니다. 글자 크기를 줄여도 그대로였습니다.

원인은 표에 걸어 둔 `break-inside: avoid`였습니다. "표를 쪽 경계에서 자르지 마라"는 규칙이라, 큰 표가 쪽 끝에 걸리면 **표 전체가 다음 쪽으로 넘어가고 앞쪽은 비게** 됩니다.

```css
/* 전: 표 통째로 넘어감 → 빈 공간이 생김 */
@media print { .frame { break-inside: avoid; } }

/* 후: 표는 나뉘어도 되고, 한 줄(행)만 안 잘리게 */
@media print {
  .frame { break-inside: auto; }
  .frame tr { break-inside: avoid; }
}
```

이렇게 바꾸자 바로 **4쪽**이 됐습니다. 그 뒤로 정리한 표는 스킬에도 넣어 두었습니다.

| 증상 | 고치는 법 |
|---|---|
| 한글 단어가 줄 끝에서 쪼개짐 | `word-break: keep-all` |
| 카드가 쪽 경계에서 반으로 잘림 | `break-inside: avoid` |
| 큰 표가 통째로 넘어가 앞쪽이 빔 | `table { break-inside: auto } tr { break-inside: avoid }` |
| 제목만 쪽 끝에 홀로 남음 | `h2, h3 { break-after: avoid }` |
| 마지막 쪽에 한두 줄만 넘어감 | 글자 크기·줄 간격·제목 여백을 조금씩 줄임 |

---

## 6. 설치

```bash
git clone https://github.com/HeeJeShin/html-pdf-skill.git
cd html-pdf-skill/skills/html-pdf
npm install
ln -s "$PWD" ~/.claude/skills/html-pdf
```

그다음 Claude Code에서 이렇게 말하면 됩니다.

```
resume.html PDF로 만들어줘. 2쪽 넘으면 알려주고, "직전 연봉"은 들어가면 안 돼
```

---

## 정리

- 반복되는 일은 **사람이 기억해서 하는 방식**보다 **스크립트가 같은 방식으로 하는 방식**이 실수가 적습니다.
- AI에게 일을 맡길 때는 "무엇을 할지"만큼 **"무엇은 하지 말지"** 를 적어 두는 게 중요했습니다. 이 스킬에서는 "내용은 바꾸지 않는다"가 그 규칙이었습니다.
- print CSS의 `break-inside: avoid`는 편하지만, 큰 덩어리에 걸면 오히려 빈 공간을 만듭니다.
