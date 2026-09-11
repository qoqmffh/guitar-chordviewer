# Major Edu 코드초견(Chord Sight-Reading) 생성기 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** board-labs 사이트의 Major Edu 메뉴 안에 "코드초견" 출제지 생성기를 만든다. 사용자가 루트음/코드 난이도/리듬 난이도를 고르면, 마디 박스+코드기호+리듬 슬래시로 이루어진 8마디짜리 출제지를 그 자리에서 생성해 SVG로 보여주고, PNG로 저장할 수 있게 한다.

**Architecture:** 이 프로젝트는 빌드 과정이 없는 단일 HTML 파일(`index.html`)이다. 새 기능도 그 파일 안에 순수 함수(코드 진행 생성기, SVG 렌더러)로 먼저 만들고, 그 다음 기존 `state`/`renderApp()`/`WIP_VIEWS` 패턴에 맞춰 화면에 연결한다. 외부 라이브러리(악보 렌더링 라이브러리 포함) 없이 지금 있는 `buildFretboardSVG()`류와 같은 방식(문자열 조립형 SVG)으로 직접 그린다.

**Tech Stack:** 바닐라 JavaScript(ES5 스타일, `var`/함수선언 위주 — 기존 코드와 통일), 인라인 `<style>`/`<script>`, 빌드 없음. 테스트는 Node.js를 "브라우저 없는 DOM 스텁"으로 써서 인라인 스크립트를 그대로 로드해 함수 단위로 검증(이 세션에서 계속 써온 방식). 최종 검증은 Playwright(Chromium)로 실제 렌더링을 스크린샷으로 확인.

**Spec:** `docs/superpowers/specs/2026-09-11-major-edu-chord-sight-reading-design.md` — 이 플랜은 그 스펙을 그대로 구현한다. 실행하는 사람은 스펙 문서도 같이 참고할 것.

## Global Constraints

- 대상 파일은 오직 하나: `C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer\index.html`. 새 파일을 만들지 않는다(폰트/이미지 자산 제외).
- 8마디 고정, 마디를 넘어가는 타이(리듬이 다음 마디로 이어지는 것)는 만들지 않는다.
- 외부 악보 렌더링 라이브러리(VexFlow 등)를 추가하지 않는다.
- 생성 결과는 Firebase/localStorage 어디에도 저장하지 않는다 — 매번 새로 생성.
- 커밋은 `git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit`으로 하고, 각 태스크가 끝날 때마다 `git push origin main`까지 한다(이 저장소는 GitHub Pages로 바로 배포되는 `main` 브랜치).
- 커밋 메시지는 한글로, 이 세션에서 계속 써온 스타일(요약 줄 + 상세 불릿)을 따른다.

---

##공용 테스트 하네스 (모든 태스크에서 재사용할 코드 조각)

각 태스크의 테스트 스텝은 아래 하네스를 Node 스크립트 파일에 그대로 복붙해서, 맨 아래 `return { ... }`에 그 태스크에서 검증할 함수/상태만 나열하면 된다. 이 하네스는 `index.html`의 `<script>` 안에서, 맨 처음 `try {\n(function () {`와 맨 끝 `})();\n} catch (e) {` 사이의 코드만 뽑아내서(즉 앱을 실제로 구동하는 맨 아래 `renderApp(); setupFirebaseSync(); setupPresence();` 호출까지 전부 포함해서) 실행한다. Firebase SDK가 없는 환경이라 `fbEnabled`는 항상 `false`로 안전하게 폴백된다.

```js
// scratch_test.js — 매 태스크마다 이 파일을 새로 만들어서 씀 (경로는 자유, 커밋 대상 아님)
const fs = require('fs');

const filePath = 'C:\\Users\\user\\Desktop\\Dev\\태준개인개발파일\\CodeViewer\\index.html';
const html = fs.readFileSync(filePath, 'utf8');
const m = html.match(/<script>([\s\S]*?)<\/script>/);
let js = m[1];
const startMarker = 'try {\n(function () {';
const endMarker = '})();\n} catch (e) {';
const idxStart = js.indexOf(startMarker) + startMarker.length;
const idxEnd = js.lastIndexOf(endMarker);
const innerBody = js.slice(idxStart, idxEnd);

function makeStyle() { return new Proxy({}, { get(){return '';}, set(){return true;} }); }
function makeClassList(el) {
  el.__classes = new Set();
  return {
    add: (...cs) => cs.forEach(c => el.__classes.add(c)),
    remove: (...cs) => cs.forEach(c => el.__classes.delete(c)),
    contains: (c) => el.__classes.has(c),
    toggle: (c, force) => {
      if (force === undefined) {
        if (el.__classes.has(c)) { el.__classes.delete(c); return false; }
        el.__classes.add(c); return true;
      }
      if (force) { el.__classes.add(c); } else { el.__classes.delete(c); }
      return force;
    }
  };
}
function makeElement(tag) {
  const el = {
    tagName: (tag || 'div').toUpperCase(), children: [], attrs: {},
    style: makeStyle(), hidden: false, value: '', _text: '', _html: '',
  };
  el.classList = makeClassList(el);
  el.appendChild = (child) => { el.children.push(child); return child; };
  el.setAttribute = (k, v) => { el.attrs[k] = v; };
  el.getAttribute = (k) => el.attrs[k];
  el.addEventListener = () => {};
  el.querySelector = () => null;
  el.querySelectorAll = () => [];
  Object.defineProperty(el, 'textContent', { get() { return el._text; }, set(v) { el._text = v; el.children = []; } });
  Object.defineProperty(el, 'innerHTML', { get() { return el._html; }, set(v) { el._html = v; el.children = []; } });
  Object.defineProperty(el, 'className', {
    get() { return Array.from(el.__classes).join(' '); },
    set(v) { el.__classes = new Set(String(v).split(/\s+/).filter(Boolean)); }
  });
  return el;
}
const elementsById = new Map();
function getOrMake(id) { if (!elementsById.has(id)) elementsById.set(id, makeElement('div')); return elementsById.get(id); }

global.window = global;
global.scrollY = 0;
global.addEventListener = function () {};
global.matchMedia = function () { return { matches: false }; };
global.cancelAnimationFrame = function () {};
global.requestAnimationFrame = function (fn) { return setTimeout(fn, 0); };
global.document = {
  getElementById: (id) => getOrMake(id),
  createElement: (tag) => makeElement(tag),
  addEventListener: () => {},
  documentElement: makeElement('html'),
  body: makeElement('body'),
  querySelector: () => null,
};
global.localStorage = { getItem: () => null, setItem: () => {}, removeItem: () => {} };
global.sessionStorage = { getItem: () => null, setItem: () => {}, removeItem: () => {} };
global.navigator = { userAgent: 'node' };
global.location = { pathname: '/', search: '', hostname: 'localhost' };
global.history = { replaceState: () => {} };
global.console = console;

const wrapped = innerBody + '\nreturn { /* TODO: 이 태스크에서 검증할 이름들을 여기 나열 */ };';
let api;
try {
  api = new Function(wrapped)();
} catch (e) {
  console.log('EXPORT ERROR:', e.message);
  console.log(e.stack);
  process.exit(1);
}

// ... 여기에 태스크별 assert 코드 ...
```

이 파일을 `node scratch_test.js`로 실행해서 결과를 눈으로 확인한다(이 프로젝트엔 별도 테스트 러너가 없으므로 `console.log`로 기대값과 실제값을 나란히 찍고 직접 대조).

---

## Task 1: 데이터 기초 — 다이아토닉 트라이어드 표, 리듬 패턴 뱅크, state 필드

**Files:**
- Modify: `index.html` (아래 3곳)
  1. `var SCALE_CHORD_EXTRA = { minMaj7: ... };` 바로 뒤 (현재 1853~1855번째 줄 부근, `function scaleChordName` 정의 바로 앞)
  2. `var DIATONIC_DEGREES = [...]` 선언 바로 뒤 (1823~1831번째 줄 부근)
  3. `var state = { ... freeboardPosts: [], mobileAdClosed: false };` 안 (2466~2467번째 줄 부근)

**Interfaces:**
- Consumes: 없음(이 태스크가 가장 먼저 실행됨)
- Produces:
  - `DIATONIC_TRIADS` (배열, `DIATONIC_DEGREES`와 같은 모양: `{label, offset, qual}[]`)
  - `SCALE_CHORD_EXTRA.dimTriad` (`{label:'dim', intervals:[[0,'R'],[3,'b3'],[6,'b5']]}`)
  - `RHYTHM_PATTERNS` (객체: `{low: [...], mid: [...], high: [...]}`, 각 원소는 `{id, units}` — `units`는 1~8 사이 정수 오름차순 배열, 한 마디를 8분음표 단위 8칸으로 나눈 것 중 어디를 짚는지)
  - `state.majorEduRoot`(0~11, 기본 0), `state.majorEduChordTier`('low'|'mid'|'high', 기본 'mid'), `state.majorEduRhythmTier`(기본 'mid'), `state.majorEduSheet`(생성된 출제지 객체 또는 null, 기본 null)

- [ ] **Step 1: `SCALE_CHORD_EXTRA`에 `dimTriad` 추가**

`index.html`에서 다음 코드를 찾는다:

```js
  var SCALE_CHORD_EXTRA = {
    minMaj7: { label: 'm(maj7)', intervals: [[0, 'R'], [3, 'b3'], [7, '5'], [11, '7']] }
  };
```

다음으로 바꾼다:

```js
  var SCALE_CHORD_EXTRA = {
    minMaj7: { label: 'm(maj7)', intervals: [[0, 'R'], [3, 'b3'], [7, '5'], [11, '7']] },
    // Major Edu 코드초견의 "하" 난이도(다이아토닉 트라이어드)에서 vii°를
    // 표기하려고 추가 — QUALITIES에는 순수 디미니쉬 트라이어드가 없음(dim7만 있음).
    dimTriad: { label: 'dim', intervals: [[0, 'R'], [3, 'b3'], [6, 'b5']] }
  };
```

- [ ] **Step 2: `DIATONIC_TRIADS` 추가**

`var DIATONIC_DEGREES = [...]` 선언(7개 항목, `vii°`로 끝나고 `];`로 닫힘) 바로 다음 줄에 아래를 추가한다:

```js

  // Major Edu 코드초견 "하" 난이도 전용 — DIATONIC_DEGREES(7th 코드 하모나이제이션)의
  // 트라이어드 버전. label/offset은 DIATONIC_DEGREES와 완전히 동일하게 맞춘다.
  var DIATONIC_TRIADS = [
    { label: 'I',    offset: 0,  qual: 'major' },
    { label: 'ii',   offset: 2,  qual: 'minor' },
    { label: 'iii',  offset: 4,  qual: 'minor' },
    { label: 'IV',   offset: 5,  qual: 'major' },
    { label: 'V',    offset: 7,  qual: 'major' },
    { label: 'vi',   offset: 9,  qual: 'minor' },
    { label: 'vii°', offset: 11, qual: 'dimTriad' }
  ];
```

- [ ] **Step 3: `RHYTHM_PATTERNS` 추가**

`DIATONIC_TRIADS` 바로 다음에 이어서 추가한다:

```js

  // Major Edu 코드초견 리듬 패턴 뱅크. 한 마디(4/4)를 8분음표 단위로 8칸
  // (1~8)으로 나눈다: 1=1박, 2="&1", 3=2박, 4="&2", 5=3박, 6="&3", 7=4박,
  // 8="&4". units는 코드가 바뀌는(=리듬이 attack하는) 위치들의 오름차순
  // 배열이다. 각 히트의 실제 길이(몇 8분음표만큼 유지되는지)는 렌더링
  // 시점에 "다음 히트까지의 간격"으로 계산한다(마지막 히트는 9-unit).
  // high 티어의 4개 패턴은 전부 한 마디 안에서 완결되고(마디를 넘어가는
  // 타이 없음), 연속된 8분음표 두 개가 붙어 나오는 경우가 없도록 골라서
  // (비트/플래그만으로 그릴 수 있고 빔(beam) 연결선을 그릴 필요가 없다.
  var RHYTHM_PATTERNS = {
    low: [
      { id: 'low-1', units: [1] } // 온음표 한 방
    ],
    mid: [
      { id: 'mid-1', units: [1, 5] } // 1박, 3박에 2분음표씩
    ],
    high: [
      { id: 'high-1', units: [1, 4, 5] },    // 1박(점4분) + "&2"(8분) + 3박(2분)
      { id: 'high-2', units: [1, 2, 5] },    // 1박(8분) + "&1"(점4분) + 3박(2분) — "찰스턴" 리듬
      { id: 'high-3', units: [1, 4, 7] },    // 1박(점4분) + "&2"(점4분) + 4박(4분) — 3+3+2 분할
      { id: 'high-4', units: [1, 3, 4, 7] }  // 1박(4분)+2박(8분)+"&2"(점4분)+4박(4분)
    ]
  };
```

- [ ] **Step 4: `state`에 Major Edu 필드 추가**

다음 코드를 찾는다:

```js
    freeboardPosts: [],     // 자유게시판 글 목록, 최신순 (Firebase 연결 시에만 실제 값)
    mobileAdClosed: false   // 좁은 화면 하단 광고 띠를 사용자가 직접 닫았는지 (세션 한정)
```

다음으로 바꾼다(마지막 필드이므로 콤마 위치에 주의):

```js
    freeboardPosts: [],     // 자유게시판 글 목록, 최신순 (Firebase 연결 시에만 실제 값)
    mobileAdClosed: false,  // 좁은 화면 하단 광고 띠를 사용자가 직접 닫았는지 (세션 한정)
    majorEduRoot: 0,        // Major Edu 코드초견: 조성 루트음 (0~11, C=0)
    majorEduChordTier: 'mid',   // 'low' | 'mid' | 'high'
    majorEduRhythmTier: 'mid',  // 'low' | 'mid' | 'high'
    majorEduSheet: null     // 마지막으로 생성한 출제지 객체(Task 2 참고), 생성 전엔 null
```

- [ ] **Step 5: 문법 검사**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:\\\\Users\\\\user\\\\Desktop\\\\Dev\\\\태준개인개발파일\\\\CodeViewer\\\\index.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
scripts.forEach((js,i)=>{ try { new Function(js); console.log('block',i,'OK'); } catch(e){ console.log('block',i,'SYNTAX ERROR:', e.message); } });
"
```

Expected: `block 0 OK`

- [ ] **Step 6: 데이터 구조 테스트**

위의 "공용 테스트 하네스"를 `scratch_task1.js`로 저장하되, `return { ... }` 줄을 아래로 바꾼다:

```js
const wrapped = innerBody + '\nreturn { DIATONIC_TRIADS, RHYTHM_PATTERNS, SCALE_CHORD_EXTRA, scaleChordName, NOTES, state };';
```

그리고 파일 맨 끝(`let api; try { ... }` 블록 다음)에 아래를 추가한다:

```js
console.log('DIATONIC_TRIADS length (expect 7):', api.DIATONIC_TRIADS.length);
console.log('DIATONIC_TRIADS vii deg (expect offset 11, qual dimTriad):', JSON.stringify(api.DIATONIC_TRIADS[6]));

const bDim = api.scaleChordName(api.NOTES.indexOf('C'), 11, 'dimTriad');
console.log('scaleChordName(C, 11, dimTriad) (expect "Bdim"):', bDim);

['low', 'mid', 'high'].forEach(tier => {
  const bank = api.RHYTHM_PATTERNS[tier];
  console.log(tier, 'bank size:', bank.length);
  bank.forEach(p => {
    const ok = p.units.every((u, i) => i === 0 || u > p.units[i - 1]);
    console.log(' ', p.id, JSON.stringify(p.units), ok ? 'ascending OK' : 'FAIL: not strictly ascending');
  });
});

console.log('initial state.majorEduRoot (expect 0):', api.state.majorEduRoot);
console.log('initial state.majorEduChordTier (expect mid):', api.state.majorEduChordTier);
console.log('initial state.majorEduRhythmTier (expect mid):', api.state.majorEduRhythmTier);
console.log('initial state.majorEduSheet (expect null):', api.state.majorEduSheet);
```

Run: `node scratch_task1.js`

Expected output에서 확인할 것:
- `DIATONIC_TRIADS length (expect 7): 7`
- `vii deg`가 `{"label":"vii°","offset":11,"qual":"dimTriad"}`
- `scaleChordName(...)`가 정확히 `Bdim`
- 세 티어 전부 `ascending OK`만 나오고 `FAIL`이 하나도 없음
- state 초기값 4개가 위에 적은 기대값과 정확히 일치

기대와 다르면 Step 1~4로 돌아가서 고친다.

- [ ] **Step 7: 커밋 + 푸시**

```bash
cd "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer"
git add index.html
git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit -m "Major Edu 코드초견: 데이터 기초(다이아토닉 트라이어드, 리듬 패턴 뱅크, state 필드) 추가

아직 화면에는 연결 안 됨 — 다음 태스크에서 진행 생성기, 그 다음 태스크에서
렌더러와 UI를 붙인다."
git push origin main
```

---

## Task 2: 코드 진행 생성기 (순수 함수)

**Files:**
- Modify: `index.html` — Task 1에서 추가한 `RHYTHM_PATTERNS` 블록 바로 다음에 새 함수들을 추가

**Interfaces:**
- Consumes: `DIATONIC_DEGREES`, `DIATONIC_TRIADS`, `RHYTHM_PATTERNS`, `scaleChordName(rootPC, offset, qualKey)` (Task 1 및 기존 코드)
- Produces:
  - `shuffleArray(arr)` — 배열을 제자리에서 섞어서 그대로 반환(Fisher-Yates)
  - `pickDiatonicSequence(slotCount)` — roman 문자열 배열 반환, 길이 `slotCount`
  - `secondaryDominantOffset(targetRoman)` — 정수(0~11) 또는 `null` 반환
  - `applySecondaryDominants(seq, maxSubs)` — roman 문자열 배열 반환(길이는 `seq`와 동일)
  - `romanToChordLabel(rootPC, roman, chordTier)` — 코드 기호 문자열(예: `"Cmaj7"`) 또는 `null` 반환
  - `generateMajorEduProgression(rootPC, chordTier, rhythmTier)` — 아래 모양의 객체 반환:
    ```js
    {
      root: rootPC, chordTier: chordTier, rhythmTier: rhythmTier,
      measures: [
        { rhythmPatternId: 'mid-1', hits: [ { unit: 1, roman: 'I', chordLabel: 'Cmaj7' }, { unit: 5, roman: 'V', chordLabel: 'G7' } ] },
        // ... 총 8개
      ]
    }
    ```

- [ ] **Step 1: `shuffleArray`, `pickDiatonicSequence`, `secondaryDominantOffset`, `applySecondaryDominants` 추가**

`RHYTHM_PATTERNS` 정의 바로 다음에 추가:

```js

  // Fisher-Yates — 배열을 제자리에서 섞어서 그대로 반환.
  function shuffleArray(arr) {
    for (var i = arr.length - 1; i > 0; i--) {
      var j = Math.floor(Math.random() * (i + 1));
      var tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
    }
    return arr;
  }

  // DIATONIC_DEGREES/DIATONIC_TRIADS의 도수 라벨 순서(둘 다 이 순서로 정의돼 있음).
  var DIATONIC_ROMANS = ['I', 'ii', 'iii', 'IV', 'V', 'vi', 'vii°'];

  // slotCount개의 다이아토닉 도수를 무작위로 뽑는다. 규칙: 1) 바로 연속으로
  // 같은 도수가 두 번 나오지 않음 2) 마지막 슬롯은 항상 I(토닉으로 종지)
  // 3) 최소 한 번은 V->I 진행이 들어가게(없으면 뒤에서 두 번째 슬롯을 V로
  // 강제 교체 — 단, 그렇게 했을 때 뒤에서 세 번째와 연속으로 겹치면 포기하고
  // 그대로 둔다. 8마디짜리 연습용 진행이라 이 정도 소프트 규칙이면 충분).
  function pickDiatonicSequence(slotCount) {
    var seq = [];
    for (var i = 0; i < slotCount; i++) {
      if (i === slotCount - 1) {
        seq.push('I');
        continue;
      }
      var choices = DIATONIC_ROMANS.filter(function (r) { return r !== seq[seq.length - 1]; });
      seq.push(choices[Math.floor(Math.random() * choices.length)]);
    }
    var hasVtoI = false;
    for (var j = 0; j < seq.length - 1; j++) {
      if (seq[j] === 'V' && seq[j + 1] === 'I') { hasVtoI = true; break; }
    }
    if (!hasVtoI && seq.length >= 2) {
      var pos = seq.length - 2;
      if (seq[pos - 1] !== 'V') { seq[pos] = 'V'; }
    }
    return seq;
  }

  // targetRoman(예: 'ii')의 5도 위 음정(반음 offset, 0~11)을 계산 — 그 위에
  // dom7을 쌓으면 targetRoman으로 가는 이차도미넌트(V7/ii 등)가 된다.
  function secondaryDominantOffset(targetRoman) {
    for (var i = 0; i < DIATONIC_DEGREES.length; i++) {
      if (DIATONIC_DEGREES[i].label === targetRoman) {
        return (DIATONIC_DEGREES[i].offset + 7) % 12;
      }
    }
    return null;
  }

  // seq 안에서 다음 슬롯이 'ii'/'V'/'vi'인 자리를 찾아 최대 maxSubs개를
  // 'V7/<다음도수>'로 치환한다(마지막 슬롯 앞까지만 대상 — 마지막은 항상 I).
  // 적절한 자리가 없으면 그냥 원래 seq를 그대로 돌려준다(강제하지 않음).
  function applySecondaryDominants(seq, maxSubs) {
    var eligible = ['ii', 'V', 'vi'];
    var candidates = [];
    for (var i = 0; i < seq.length - 1; i++) {
      if (eligible.indexOf(seq[i + 1]) !== -1 && seq[i] !== 'V') { candidates.push(i); }
    }
    shuffleArray(candidates);
    var subs = candidates.slice(0, maxSubs);
    return seq.map(function (roman, i) {
      return subs.indexOf(i) !== -1 ? ('V7/' + seq[i + 1]) : roman;
    });
  }

  // roman(예: 'ii°'는 없음, 'vii°'만 있음)에 해당하는 코드 기호를 계산.
  // chordTier==='low'면 DIATONIC_TRIADS(트라이어드), 아니면 DIATONIC_DEGREES
  // (7th 코드)에서 찾는다. 'V7/...' 형태(이차도미넌트)는 이 함수가 아니라
  // generateMajorEduProgression 안에서 별도로 처리한다.
  function romanToChordLabel(rootPC, roman, chordTier) {
    var table = chordTier === 'low' ? DIATONIC_TRIADS : DIATONIC_DEGREES;
    for (var i = 0; i < table.length; i++) {
      if (table[i].label === roman) { return scaleChordName(rootPC, table[i].offset, table[i].qual); }
    }
    return null;
  }
```

- [ ] **Step 2: `generateMajorEduProgression` 추가**

바로 이어서 추가:

```js

  // Major Edu 코드초견 출제지 하나(8마디)를 생성. rootPC: 0~11.
  // chordTier/rhythmTier: 'low'|'mid'|'high'.
  function generateMajorEduProgression(rootPC, chordTier, rhythmTier) {
    var measureCount = 8;
    var bank = RHYTHM_PATTERNS[rhythmTier];
    var measureRhythms = [];
    for (var m = 0; m < measureCount; m++) {
      measureRhythms.push(bank[Math.floor(Math.random() * bank.length)]);
    }

    var slotCount = 0;
    for (var mi = 0; mi < measureRhythms.length; mi++) { slotCount += measureRhythms[mi].units.length; }

    var romanSeq = pickDiatonicSequence(slotCount);
    if (chordTier === 'high') { romanSeq = applySecondaryDominants(romanSeq, 2); }

    var measures = [];
    var slotIdx = 0;
    for (var i = 0; i < measureCount; i++) {
      var pattern = measureRhythms[i];
      var hits = [];
      for (var u = 0; u < pattern.units.length; u++) {
        var roman = romanSeq[slotIdx];
        var chordLabel;
        if (roman.indexOf('V7/') === 0) {
          var targetRoman = roman.slice(3);
          chordLabel = scaleChordName(rootPC, secondaryDominantOffset(targetRoman), 'dom7');
        } else {
          chordLabel = romanToChordLabel(rootPC, roman, chordTier);
        }
        hits.push({ unit: pattern.units[u], roman: roman, chordLabel: chordLabel });
        slotIdx++;
      }
      measures.push({ rhythmPatternId: pattern.id, hits: hits });
    }

    return { root: rootPC, chordTier: chordTier, rhythmTier: rhythmTier, measures: measures };
  }
```

- [ ] **Step 3: 문법 검사**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:\\\\Users\\\\user\\\\Desktop\\\\Dev\\\\태준개인개발파일\\\\CodeViewer\\\\index.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
scripts.forEach((js,i)=>{ try { new Function(js); console.log('block',i,'OK'); } catch(e){ console.log('block',i,'SYNTAX ERROR:', e.message); } });
"
```

Expected: `block 0 OK`

- [ ] **Step 4: 생성기 테스트**

"공용 테스트 하네스"를 `scratch_task2.js`로 저장, `return` 줄을 다음으로:

```js
const wrapped = innerBody + '\nreturn { generateMajorEduProgression, pickDiatonicSequence, secondaryDominantOffset, applySecondaryDominants, NOTES };';
```

파일 끝에 추가:

```js
// 1) secondaryDominantOffset 정확성
console.log('secondaryDominantOffset(ii) (expect 9):', api.secondaryDominantOffset('ii'));
console.log('secondaryDominantOffset(V) (expect 2):', api.secondaryDominantOffset('V'));
console.log('secondaryDominantOffset(vi) (expect 4):', api.secondaryDominantOffset('vi'));

// 2) pickDiatonicSequence 불변식 — 200번 반복해서 전부 통과해야 함
let seqFailures = 0;
for (let i = 0; i < 200; i++) {
  const seq = api.pickDiatonicSequence(8 + (i % 20)); // 길이도 다양하게
  if (seq[seq.length - 1] !== 'I') { seqFailures++; console.log('FAIL last!=I', seq); }
  for (let j = 1; j < seq.length; j++) {
    if (seq[j] === seq[j - 1]) { seqFailures++; console.log('FAIL consecutive repeat', seq); break; }
  }
}
console.log('pickDiatonicSequence invariant failures (expect 0):', seqFailures);

// 3) applySecondaryDominants — 마지막 원소는 절대 안 바뀜, 길이 불변
let subFailures = 0;
for (let i = 0; i < 100; i++) {
  const base = api.pickDiatonicSequence(16);
  const result = api.applySecondaryDominants(base.slice(), 2);
  if (result.length !== base.length) { subFailures++; console.log('FAIL length changed'); }
  if (result[result.length - 1] !== base[base.length - 1]) { subFailures++; console.log('FAIL last element changed'); }
}
console.log('applySecondaryDominants invariant failures (expect 0):', subFailures);

// 4) generateMajorEduProgression — 12루트 x 3코드난이도 x 3리듬난이도 = 108 조합
let genFailures = 0;
const tiers = ['low', 'mid', 'high'];
for (let root = 0; root < 12; root++) {
  tiers.forEach(chordTier => {
    tiers.forEach(rhythmTier => {
      try {
        const sheet = api.generateMajorEduProgression(root, chordTier, rhythmTier);
        if (sheet.measures.length !== 8) { genFailures++; console.log('FAIL measure count', root, chordTier, rhythmTier); }
        sheet.measures.forEach(measure => {
          measure.hits.forEach(hit => {
            if (!hit.chordLabel) { genFailures++; console.log('FAIL empty chordLabel', root, chordTier, rhythmTier, JSON.stringify(hit)); }
          });
        });
        const lastMeasure = sheet.measures[sheet.measures.length - 1];
        const lastHit = lastMeasure.hits[lastMeasure.hits.length - 1];
        if (lastHit.roman !== 'I') { genFailures++; console.log('FAIL last hit not I', root, chordTier, rhythmTier, lastHit); }
      } catch (e) {
        genFailures++;
        console.log('EXCEPTION', root, chordTier, rhythmTier, e.message);
      }
    });
  });
}
console.log('generateMajorEduProgression failures across 108 combos (expect 0):', genFailures);
```

Run: `node scratch_task2.js`

Expected: 4개의 "failures (expect 0)" 줄이 전부 정확히 `0`이어야 하고, `secondaryDominantOffset` 3개 값이 각각 9/2/4여야 한다. 하나라도 다르면 Step 1~2로 돌아가서 고친다.

- [ ] **Step 5: 커밋 + 푸시**

```bash
cd "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer"
git add index.html
git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit -m "Major Edu 코드초견: 코드 진행 생성기(generateMajorEduProgression) 추가

리듬 패턴이 먼저 마디당 히트 수를 정하고, 거기 맞춰 다이아토닉 진행을
채우는 순서. 상 난이도는 이차도미넌트 최대 2곳 치환. 108개 조합(루트12
x코드난이도3x리듬난이도3) 전부 예외 없이 생성되는 것 확인."
git push origin main
```

---

## Task 3: SVG 렌더러 (순수 함수)

**Files:**
- Modify: `index.html` — Task 2의 `generateMajorEduProgression` 바로 다음에 렌더러 함수 추가
- Modify: `index.html` — `I18N.ko`/`I18N.en`에 SVG 접근성 텍스트 키 추가 (기존 `svgScaleTitle`/`svgScaleDesc` 키 바로 다음)

**Interfaces:**
- Consumes: `generateMajorEduProgression()`의 반환 객체 모양(Task 2), `t(key)` (기존)
- Produces:
  - `hitDurationUnits(measure, hitIndex)` — 정수 반환(1/2/3/4/8 중 하나)
  - `buildMajorEduSVG(sheet)` — `<svg ...>...</svg>` 문자열 반환

- [ ] **Step 1: i18n 키 추가**

`index.html`에서 `svgScaleTitle: '스케일 다이어그램',` 다음 줄(`svgScaleDesc: '...',`) 뒤에 추가:

```js
      svgMajorEduTitle: '코드초견 출제지',
      svgMajorEduDesc: '선택한 조성·난이도로 생성한 8마디 코드초견 연습용 리듬 차트',
```

`I18N.en` 블록에서도 같은 위치(`svgScaleTitle: 'Scale diagram',` / `svgScaleDesc: '...',` 다음)에 추가:

```js
      svgMajorEduTitle: 'Chord sight-reading sheet',
      svgMajorEduDesc: 'An 8-bar chord sight-reading rhythm chart generated for the selected key and difficulty',
```

- [ ] **Step 2: `hitDurationUnits`, `buildMajorEduSVG` 추가**

`generateMajorEduProgression` 함수 바로 다음에 추가:

```js

  // measure.hits[hitIndex]가 몇 개의 8분음표 단위만큼 유지되는지(다음
  // 히트 전까지, 마지막 히트면 마디 끝(9)까지) 계산. 1=8분, 2=4분, 3=점4분,
  // 4=2분, 8=온음표.
  function hitDurationUnits(measure, hitIndex) {
    var thisUnit = measure.hits[hitIndex].unit;
    var nextUnit = hitIndex + 1 < measure.hits.length ? measure.hits[hitIndex + 1].unit : 9;
    return nextUnit - thisUnit;
  }

  // 코드초견 출제지 SVG — 8마디를 한 줄에 4마디씩 2줄로 그린다. 정식
  // 5선보 대신 "마디 박스 + 코드기호 + 리듬 슬래시" 형태(실기시험 출제지
  // 형식). 색은 사이트 테마 변수(var(--...))를 쓰지 않고 고정 색만 쓴다 —
  // 이 SVG는 PNG로 내보낼 때 페이지 스타일시트에서 분리된 독립 문서가
  // 되기 때문에 CSS 커스텀 프로퍼티가 해석되지 않는다.
  function buildMajorEduSVG(sheet) {
    var measuresPerRow = 4;
    var measureW = 210, measureGap = 14, rowH = 170, startX = 16, startY = 50;
    var totalW = startX * 2 + measuresPerRow * measureW + (measuresPerRow - 1) * measureGap;
    var totalH = startY + 2 * rowH + 20;
    var INK = '#1a1a1a';
    var GUIDE = '#c9c5b8';

    var svg = '<svg width="100%" viewBox="0 0 ' + totalW + ' ' + totalH + '" role="img">' +
      '<title>' + t('svgMajorEduTitle') + '</title>' +
      '<desc>' + t('svgMajorEduDesc') + '</desc>';

    for (var i = 0; i < sheet.measures.length; i++) {
      var row = Math.floor(i / measuresPerRow);
      var col = i % measuresPerRow;
      var x0 = startX + col * (measureW + measureGap);
      var y0 = startY + row * rowH;
      var lineY = y0 + 46;
      var stemTopY = lineY - 34;
      var xEnd = x0 + measureW;
      var isLastMeasure = (i === sheet.measures.length - 1);

      svg += '<line x1="' + x0 + '" y1="' + y0 + '" x2="' + x0 + '" y2="' + (lineY + 10) + '" stroke="' + INK + '" stroke-width="' + (col === 0 ? 2.5 : 1.25) + '"/>';
      svg += '<line x1="' + xEnd + '" y1="' + y0 + '" x2="' + xEnd + '" y2="' + (lineY + 10) + '" stroke="' + INK + '" stroke-width="' + (isLastMeasure ? 2.5 : 1.25) + '"/>';
      if (isLastMeasure) {
        svg += '<line x1="' + (xEnd - 5) + '" y1="' + y0 + '" x2="' + (xEnd - 5) + '" y2="' + (lineY + 10) + '" stroke="' + INK + '" stroke-width="1.25"/>';
      }
      svg += '<line x1="' + x0 + '" y1="' + lineY + '" x2="' + xEnd + '" y2="' + lineY + '" stroke="' + GUIDE + '" stroke-width="1"/>';

      var hits = sheet.measures[i].hits;
      for (var h = 0; h < hits.length; h++) {
        var hit = hits[h];
        var cx = x0 + ((hit.unit - 0.5) / 8) * measureW;
        var dur = hitDurationUnits(sheet.measures[i], h);

        svg += '<text x="' + cx + '" y="' + (y0 + 14) + '" text-anchor="middle" font-size="15" font-weight="700" fill="' + INK + '" style="font-family:var(--font-sans)">' + hit.chordLabel + '</text>';
        svg += '<line x1="' + (cx - 6) + '" y1="' + (lineY + 6) + '" x2="' + (cx + 6) + '" y2="' + (lineY - 6) + '" stroke="' + INK + '" stroke-width="3"/>';

        if (dur !== 8) {
          svg += '<line x1="' + (cx + 5) + '" y1="' + (lineY - 5) + '" x2="' + (cx + 5) + '" y2="' + stemTopY + '" stroke="' + INK + '" stroke-width="2"/>';
          if (dur === 1) {
            svg += '<path d="M' + (cx + 5) + ' ' + stemTopY + ' Q' + (cx + 16) + ' ' + (stemTopY + 6) + ' ' + (cx + 14) + ' ' + (stemTopY + 16) + '" stroke="' + INK + '" stroke-width="2" fill="none"/>';
          }
          if (dur === 3) {
            svg += '<circle cx="' + (cx + 12) + '" cy="' + lineY + '" r="2" fill="' + INK + '"/>';
          }
        }
      }

      var measureNum = document; // no-op guard removed below; measure numbers are optional and out of scope for v1.
    }

    svg += '</svg>';
    return svg;
  }
```

주의: 위 코드 블록 안의 `var measureNum = document;` 줄은 실수로 넣은 죽은 코드처럼 보일 수 있는데, **넣지 않는다** — 그 줄은 삭제하고 `for` 루프를 바로 닫는다. (마디 번호 표시는 v1 범위 밖.)

- [ ] **Step 3: 문법 검사**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:\\\\Users\\\\user\\\\Desktop\\\\Dev\\\\태준개인개발파일\\\\CodeViewer\\\\index.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
scripts.forEach((js,i)=>{ try { new Function(js); console.log('block',i,'OK'); } catch(e){ console.log('block',i,'SYNTAX ERROR:', e.message); } });
"
```

Expected: `block 0 OK`

- [ ] **Step 4: 렌더러 테스트**

"공용 테스트 하네스"를 `scratch_task3.js`로 저장, `return` 줄을 다음으로:

```js
const wrapped = innerBody + '\nreturn { hitDurationUnits, buildMajorEduSVG, generateMajorEduProgression };';
```

파일 끝에 추가:

```js
// 1) hitDurationUnits
const fakeMeasure1 = { hits: [{ unit: 1 }] };
console.log('duration [1] single (expect 8):', api.hitDurationUnits(fakeMeasure1, 0));
const fakeMeasure2 = { hits: [{ unit: 1 }, { unit: 4 }, { unit: 5 }] };
console.log('durations [1,4,5] (expect 3,1,4):', [0, 1, 2].map(i => api.hitDurationUnits(fakeMeasure2, i)).join(','));

// 2) buildMajorEduSVG — 108개 조합 전부 예외 없이 문자열을 만들고, 코드기호
// 개수가 실제 히트 개수와 일치하는지 확인
let renderFailures = 0;
const tiers = ['low', 'mid', 'high'];
for (let root = 0; root < 12; root++) {
  tiers.forEach(chordTier => {
    tiers.forEach(rhythmTier => {
      try {
        const sheet = api.generateMajorEduProgression(root, chordTier, rhythmTier);
        const svg = api.buildMajorEduSVG(sheet);
        if (typeof svg !== 'string' || svg.indexOf('<svg') !== 0) { renderFailures++; console.log('FAIL not svg string', root, chordTier, rhythmTier); }
        const totalHits = sheet.measures.reduce((sum, m) => sum + m.hits.length, 0);
        const textCount = (svg.match(/<text /g) || []).length; // 코드기호 텍스트 개수
        if (textCount !== totalHits) { renderFailures++; console.log('FAIL text count mismatch', root, chordTier, rhythmTier, 'expect', totalHits, 'got', textCount); }
      } catch (e) {
        renderFailures++;
        console.log('EXCEPTION', root, chordTier, rhythmTier, e.message);
      }
    });
  });
}
console.log('buildMajorEduSVG failures across 108 combos (expect 0):', renderFailures);
```

Run: `node scratch_task3.js`

Expected: `duration [1] single (expect 8): 8`, `durations [1,4,5] (expect 3,1,4): 3,1,4`, 마지막 줄이 정확히 `0`.

- [ ] **Step 5: 커밋 + 푸시**

```bash
cd "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer"
git add index.html
git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit -m "Major Edu 코드초견: SVG 렌더러(buildMajorEduSVG) 추가

마디 박스+코드기호+리듬 슬래시(스템/플래그/점) 형태로 8마디를 2줄(4마디씩)
로 그린다. 외부 악보 라이브러리 없이 기존 buildFretboardSVG류와 같은
문자열 조립 방식. 색은 CSS 변수 대신 고정 hex(PNG 내보내기 시 스타일시트가
분리되기 때문). 108개 조합 전부 예외 없이 렌더 확인."
git push origin main
```

---

## Task 4: 화면 연결 (HTML + state 배선 + 내비게이션)

**Files:**
- Modify: `index.html`
  1. `#app-wip` 바로 앞에 `#app-major-edu` div 추가
  2. `#app, #app-progression, #app-admin, #app-wip { max-width:1200px; margin:0 auto; }` 규칙에 `#app-major-edu` 추가
  3. `WIP_VIEWS` 배열에서 `'majorEdu'` 제거
  4. `renderApp()`에 `else if (state.activeView === 'majorEdu')` 분기 추가
  5. `renderMajorEdu()`, `renderMajorEduStatic()`, `renderMajorEduRoots()`, `renderMajorEduTier(kind)` 함수 추가(Task 3의 `generateMajorEduProgression`/`buildMajorEduSVG` 바로 다음)
  6. 새 이벤트 리스너 등록(기존 `document.getElementById('ad-sticky-close').onclick = closeMobileAdBar;` 등이 모여 있는 구역)
  7. `I18N.ko`/`I18N.en`에 라벨 키 추가

**Interfaces:**
- Consumes: `generateMajorEduProgression`, `buildMajorEduSVG`, `NOTES`, `t()`, `state`, `goToView()`, `WIP_VIEWS`, `NAV_PARENT_VIEWS`(이미 `tools: [..., 'majorEdu']` 포함돼 있어 수정 불필요 — 확인만 할 것)
- Produces: `renderMajorEdu()` (다른 태스크가 호출할 일 없음, `renderApp()` 내부에서만 호출)

- [ ] **Step 1: i18n 키 추가 (ko)**

`svgMajorEduDesc: '...',` (Task 3에서 추가한 줄) 다음에 이어서 추가:

```js
      majorEduTitle: 'Major Edu — 코드초견',
      labelMajorEduRoot: '루트음',
      labelMajorEduChordTier: '코드 난이도',
      labelMajorEduRhythmTier: '리듬 난이도',
      majorEduTierLow: '하', majorEduTierMid: '중', majorEduTierHigh: '상',
      majorEduRandomRoot: '🎲 랜덤',
      majorEduGenerateBtn: '출제지 생성',
      majorEduSaveBtn: 'PNG로 저장',
```

- [ ] **Step 2: i18n 키 추가 (en)**

같은 위치의 `I18N.en` 블록에 추가:

```js
      majorEduTitle: 'Major Edu — Chord Sight-Reading',
      labelMajorEduRoot: 'Root',
      labelMajorEduChordTier: 'Chord Difficulty',
      labelMajorEduRhythmTier: 'Rhythm Difficulty',
      majorEduTierLow: 'Easy', majorEduTierMid: 'Medium', majorEduTierHigh: 'Hard',
      majorEduRandomRoot: '🎲 Random',
      majorEduGenerateBtn: 'Generate Sheet',
      majorEduSaveBtn: 'Save as PNG',
```

- [ ] **Step 3: HTML — `#app-major-edu` 추가**

`index.html`에서 다음을 찾는다:

```html
<!-- 아직 안 만든 메뉴(TOOLS/MAJOR EDU/HARMONY/MY) 전부가 공유하는 자리표시자
     페이지 — 어느 메뉴를 눌렀든 똑같이 "개발중입니다" 안내만 보여준다. -->
<div id="app-wip" style="display:none;">
```

바로 앞에 삽입:

```html
<div id="app-major-edu" style="display:none;">
  <h1 id="major-edu-h1">Major Edu — 코드초견</h1>

  <div style="margin-bottom:16px;">
    <div class="gt-label" id="label-major-edu-root">루트음</div>
    <div id="major-edu-roots" class="gt-flex gt-grid-fixed"></div>
  </div>

  <div style="margin-bottom:16px;">
    <div class="gt-label" id="label-major-edu-chord-tier">코드 난이도</div>
    <div id="major-edu-chord-tiers" class="gt-flex gt-segmented"></div>
  </div>

  <div style="margin-bottom:20px;">
    <div class="gt-label" id="label-major-edu-rhythm-tier">리듬 난이도</div>
    <div id="major-edu-rhythm-tiers" class="gt-flex gt-segmented"></div>
  </div>

  <button id="major-edu-generate-btn" class="home-cta home-cta-primary" style="margin-bottom:20px;">출제지 생성</button>

  <div id="major-edu-sheet"></div>

  <button id="major-edu-save-btn" class="gt-btn" style="margin-top:16px; display:none;">PNG로 저장</button>
</div>

```

(`#app-wip` div는 그대로 둔다 — 삭제하지 않는다. 다른 5개 메뉴가 아직 그걸 쓰고 있다.)

- [ ] **Step 4: CSS — `#app-major-edu`를 카드-없음 규칙에 추가**

다음을 찾는다:

```css
  #app, #app-progression, #app-admin, #app-wip {
    max-width: 1200px;
    margin: 0 auto;
  }
```

다음으로 바꾼다:

```css
  #app, #app-progression, #app-admin, #app-wip, #app-major-edu {
    max-width: 1200px;
    margin: 0 auto;
  }
```

- [ ] **Step 5: `WIP_VIEWS`에서 `'majorEdu'` 제거**

다음을 찾는다:

```js
  var WIP_VIEWS = ['majorEdu', 'harmony', 'my', 'userCommunity', 'guitarShop', 'effecter'];
```

다음으로 바꾼다:

```js
  var WIP_VIEWS = ['harmony', 'my', 'userCommunity', 'guitarShop', 'effecter'];
```

`NAV_PARENT_VIEWS.tools`는 이미 `['code', 'scale', 'harmony', 'majorEdu']`로 `'majorEdu'`를 포함하고 있으므로 그대로 둔다(수정하지 않는다) — TOOLS 카테고리 active 표시가 코드초견 화면에서도 계속 켜져야 하기 때문.

- [ ] **Step 6: 렌더 함수 추가**

Task 3의 `buildMajorEduSVG` 함수 바로 다음에 추가:

```js

  function renderMajorEduStatic() {
    document.getElementById('major-edu-h1').textContent = t('majorEduTitle');
    document.getElementById('label-major-edu-root').textContent = t('labelMajorEduRoot');
    document.getElementById('label-major-edu-chord-tier').textContent = t('labelMajorEduChordTier');
    document.getElementById('label-major-edu-rhythm-tier').textContent = t('labelMajorEduRhythmTier');
    document.getElementById('major-edu-generate-btn').textContent = t('majorEduGenerateBtn');
    document.getElementById('major-edu-save-btn').textContent = t('majorEduSaveBtn');
  }

  function renderMajorEduRoots() {
    var el = document.getElementById('major-edu-roots');
    el.innerHTML = '';
    for (var i = 0; i < NOTES.length; i++) {
      (function (idx) {
        var b = document.createElement('button');
        b.className = 'gt-btn' + (state.majorEduRoot === idx ? ' active' : '');
        b.textContent = NOTES[idx];
        b.onclick = function () { state.majorEduRoot = idx; renderMajorEdu(); };
        el.appendChild(b);
      })(i);
    }
    var randomBtn = document.createElement('button');
    randomBtn.className = 'gt-btn';
    randomBtn.textContent = t('majorEduRandomRoot');
    randomBtn.onclick = function () { state.majorEduRoot = Math.floor(Math.random() * 12); renderMajorEdu(); };
    el.appendChild(randomBtn);
  }

  // kind: 'chord' | 'rhythm' — 두 세그먼티드 컨트롤(코드 난이도/리듬 난이도)이
  // 구조가 완전히 같아서 하나의 함수로 처리.
  function renderMajorEduTier(kind) {
    var containerId = kind === 'chord' ? 'major-edu-chord-tiers' : 'major-edu-rhythm-tiers';
    var stateKey = kind === 'chord' ? 'majorEduChordTier' : 'majorEduRhythmTier';
    var el = document.getElementById(containerId);
    el.innerHTML = '';
    var tiers = [['low', t('majorEduTierLow')], ['mid', t('majorEduTierMid')], ['high', t('majorEduTierHigh')]];
    for (var i = 0; i < tiers.length; i++) {
      (function (tier) {
        var b = document.createElement('button');
        b.className = 'gt-btn' + (state[stateKey] === tier[0] ? ' active' : '');
        b.textContent = tier[1];
        b.onclick = function () { state[stateKey] = tier[0]; renderMajorEdu(); };
        el.appendChild(b);
      })(tiers[i]);
    }
  }

  function renderMajorEduSheet() {
    var sheetEl = document.getElementById('major-edu-sheet');
    var saveBtn = document.getElementById('major-edu-save-btn');
    if (!state.majorEduSheet) {
      sheetEl.innerHTML = '';
      saveBtn.style.display = 'none';
      return;
    }
    sheetEl.innerHTML = buildMajorEduSVG(state.majorEduSheet);
    saveBtn.style.display = 'inline-block';
  }

  function renderMajorEdu() {
    renderMajorEduStatic();
    renderMajorEduRoots();
    renderMajorEduTier('chord');
    renderMajorEduTier('rhythm');
    renderMajorEduSheet();
  }
```

- [ ] **Step 7: `renderApp()`에 분기 추가**

다음을 찾는다:

```js
    } else if (state.activeView === 'admin') {
      renderAdminView();
    } else if (isWip) {
      renderWipView();
    }
  }
```

다음으로 바꾼다:

```js
    } else if (state.activeView === 'admin') {
      renderAdminView();
    } else if (state.activeView === 'majorEdu') {
      renderMajorEdu();
    } else if (isWip) {
      renderWipView();
    }
  }
```

그리고 같은 함수 안 위쪽의 `display` 토글 줄들 중 아래 줄:

```js
    document.getElementById('app-wip').style.display = isWip ? 'block' : 'none';
```

바로 다음에 추가:

```js
    document.getElementById('app-major-edu').style.display = state.activeView === 'majorEdu' ? 'block' : 'none';
```

- [ ] **Step 8: 이벤트 리스너 등록**

`document.getElementById('ad-sticky-close').onclick = closeMobileAdBar;` 줄 다음에 추가:

```js
  document.getElementById('major-edu-generate-btn').onclick = function () {
    state.majorEduSheet = generateMajorEduProgression(state.majorEduRoot, state.majorEduChordTier, state.majorEduRhythmTier);
    renderMajorEduSheet();
  };
```

(`major-edu-save-btn`의 클릭 핸들러는 Task 5에서 추가한다 — 지금은 버튼만 있고 동작은 아직 없다.)

- [ ] **Step 9: 문법 검사**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:\\\\Users\\\\user\\\\Desktop\\\\Dev\\\\태준개인개발파일\\\\CodeViewer\\\\index.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
scripts.forEach((js,i)=>{ try { new Function(js); console.log('block',i,'OK'); } catch(e){ console.log('block',i,'SYNTAX ERROR:', e.message); } });
"
```

Expected: `block 0 OK`

- [ ] **Step 10: 통합 렌더 테스트**

"공용 테스트 하네스"를 `scratch_task4.js`로 저장, `return` 줄을 다음으로:

```js
const wrapped = innerBody + '\nreturn { state, renderApp, WIP_VIEWS };';
```

파일 끝에 추가:

```js
console.log('WIP_VIEWS still contains majorEdu? (expect false):', api.WIP_VIEWS.indexOf('majorEdu') !== -1);

['home', 'code', 'scale', 'progression', 'admin', 'majorEdu', 'harmony', 'my', 'userCommunity', 'guitarShop', 'effecter'].forEach(view => {
  ['ko', 'en'].forEach(lang => {
    api.state.lang = lang;
    api.state.activeView = view;
    try {
      api.renderApp();
    } catch (e) {
      console.log('FAIL renderApp()', view, lang, e.message);
    }
  });
});
console.log('renderApp() sweep done (no FAIL lines above = all OK)');

// majorEdu 화면이 실제로 보이는지 확인
api.state.activeView = 'majorEdu';
api.renderApp();
console.log('app-major-edu display (expect block):', global.document.getElementById('app-major-edu').style.display);
console.log('app-wip display (expect none):', global.document.getElementById('app-wip').style.display);

// 생성 버튼 클릭 시뮬레이션 — onclick이 등록돼 있어야 함
const genBtn = global.document.getElementById('major-edu-generate-btn');
console.log('generate button onclick registered (expect true):', typeof genBtn.onclick === 'function');
genBtn.onclick();
console.log('after generate click, majorEduSheet exists (expect true):', !!api.state.majorEduSheet);
console.log('after generate click, sheet has 8 measures (expect true):', api.state.majorEduSheet.measures.length === 8);
console.log('sheet svg non-empty (expect true):', global.document.getElementById('major-edu-sheet').innerHTML.indexOf('<svg') === 0);
console.log('save button visible after generate (expect inline-block):', global.document.getElementById('major-edu-save-btn').style.display);
```

Run: `node scratch_task4.js`

Expected:
- `WIP_VIEWS still contains majorEdu? (expect false): false`
- `renderApp() sweep done` 위에 `FAIL` 줄이 하나도 없음
- `app-major-edu display (expect block): block`
- `app-wip display (expect none): none`
- `generate button onclick registered (expect true): true`
- `after generate click, majorEduSheet exists (expect true): true`
- `after generate click, sheet has 8 measures (expect true): true`
- `sheet svg non-empty (expect true): true`
- `save button visible after generate (expect inline-block): inline-block`

- [ ] **Step 11: 브라우저 육안 확인**

```powershell
Start-Process "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer\index.html"
```

브라우저에서: 상단 메뉴 TOOLS 호버(또는 모바일이면 햄버거→TOOLS 탭) → MAJOR EDU 클릭 → 루트음/코드난이도/리듬난이도 버튼이 보이는지, "출제지 생성" 누르면 8마디 차트가 나오는지, 마디마다 코드기호와 슬래시가 그려지는지 확인. "PNG로 저장" 버튼은 이 시점엔 눌러도 아직 아무 동작 안 함(Task 5에서 연결).

- [ ] **Step 12: 커밋 + 푸시**

```bash
cd "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer"
git add index.html
git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit -m "Major Edu 코드초견: 화면 연결(WIP 자리표시자 → 실제 기능)

루트음/코드난이도/리듬난이도 선택 + 생성 버튼 + SVG 출제지 표시.
WIP_VIEWS에서 majorEdu 제거. PNG 저장 버튼은 아직 자리만 있고
동작은 다음 태스크에서 붙인다."
git push origin main
```

---

## Task 5: PNG 저장 + 최종 Playwright 검증

**Files:**
- Modify: `index.html` — `major-edu-save-btn` 클릭 핸들러 추가

**Interfaces:**
- Consumes: `#major-edu-sheet svg`(DOM), 브라우저 `Image`/`canvas`/`Blob`/`XMLSerializer` API
- Produces: 없음(최종 사용자 기능)

- [ ] **Step 1: PNG 내보내기 함수 추가**

Task 4에서 등록한 `major-edu-generate-btn`의 `onclick` 코드 바로 다음에 추가:

```js
  document.getElementById('major-edu-save-btn').onclick = function () {
    var svgEl = document.querySelector('#major-edu-sheet svg');
    if (!svgEl) { return; }
    var viewBoxParts = svgEl.getAttribute('viewBox').split(' ').map(Number);
    var vbW = viewBoxParts[2], vbH = viewBoxParts[3];
    var svgData = new XMLSerializer().serializeToString(svgEl);
    var svgBlob = new Blob([svgData], { type: 'image/svg+xml;charset=utf-8' });
    var url = URL.createObjectURL(svgBlob);
    var img = new Image();
    img.onload = function () {
      var scale = 2; // 고해상도로 저장
      var canvas = document.createElement('canvas');
      canvas.width = vbW * scale;
      canvas.height = vbH * scale;
      var ctx = canvas.getContext('2d');
      ctx.fillStyle = '#f6f4ee';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
      URL.revokeObjectURL(url);
      canvas.toBlob(function (blob) {
        var link = document.createElement('a');
        link.href = URL.createObjectURL(blob);
        link.download = 'board-labs-major-edu-sheet.png';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
      });
    };
    img.src = url;
  };
```

- [ ] **Step 2: 문법 검사**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('C:\\\\Users\\\\user\\\\Desktop\\\\Dev\\\\태준개인개발파일\\\\CodeViewer\\\\index.html','utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]);
scripts.forEach((js,i)=>{ try { new Function(js); console.log('block',i,'OK'); } catch(e){ console.log('block',i,'SYNTAX ERROR:', e.message); } });
"
```

Expected: `block 0 OK`

- [ ] **Step 3: Playwright로 실제 브라우저 검증 (스크린샷 + 다운로드 확인)**

이 프로젝트에는 아직 Playwright가 `package.json`으로 설치돼 있지 않다. 임시 폴더에 설치해서 쓴다(리포지토리에는 아무것도 추가하지 않음):

```bash
mkdir -p /tmp/major-edu-pw-check 2>/dev/null || true
cd /tmp/major-edu-pw-check
npm init -y
npm install playwright@1.63.0
npx playwright install chromium
```

같은 폴더에 `check.js`를 만든다:

```js
const { chromium } = require('playwright');
const path = require('path');
const fs = require('fs');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 1280, height: 900 }, acceptDownloads: true });
  const fileUrl = 'file:///' + path.resolve('C:\\Users\\user\\Desktop\\Dev\\태준개인개발파일\\CodeViewer\\index.html').replace(/\\/g, '/');
  await page.goto(fileUrl, { waitUntil: 'networkidle' });

  // TOOLS 메가메뉴 호버 후 MAJOR EDU 클릭 (데스크톱 폭이므로 호버로 열림)
  await page.hover('#nav-parent-tools');
  await page.click('#nav-major-edu');
  await page.waitForTimeout(300);

  await page.click('#major-edu-generate-btn');
  await page.waitForTimeout(300);
  await page.screenshot({ path: 'major_edu_generated.png', fullPage: true });
  console.log('saved major_edu_generated.png');

  const svgCount = await page.locator('#major-edu-sheet svg').count();
  console.log('svg rendered (expect 1):', svgCount);

  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.click('#major-edu-save-btn'),
  ]);
  const downloadPath = path.join(__dirname, 'downloaded.png');
  await download.saveAs(downloadPath);
  const stat = fs.statSync(downloadPath);
  console.log('downloaded PNG size in bytes (expect > 1000):', stat.size);

  await browser.close();
})();
```

Run: `node check.js`

Expected:
- `svg rendered (expect 1): 1`
- `downloaded PNG size in bytes (expect > 1000):` 1000보다 큰 숫자
- `major_edu_generated.png` 스크린샷을 열어서 직접 확인: 상단 메뉴/루트음 버튼/난이도 버튼/8마디 코드초견 차트가 제대로 보이는지, 코드기호가 마디마다 붙어있는지, 리듬 슬래시가 마디마다 최소 1개씩 있는지 눈으로 확인.

문제가 있으면(스크린샷이 깨져 보이거나, svgCount가 0이거나, 다운로드가 실패하면) Task 3~5로 돌아가서 원인을 찾는다.

- [ ] **Step 4: 전체 회귀 스윕 (기존 기능이 안 깨졌는지)**

"공용 테스트 하네스"를 `scratch_task5_regression.js`로 저장, `return` 줄을 다음으로:

```js
const wrapped = innerBody + '\nreturn { state, renderApp };';
```

파일 끝에 추가:

```js
const allViews = ['home', 'code', 'scale', 'progression', 'admin', 'majorEdu', 'harmony', 'my', 'userCommunity', 'guitarShop', 'effecter'];
let failures = 0;
['ko', 'en'].forEach(lang => {
  api.state.lang = lang;
  allViews.forEach(view => {
    api.state.activeView = view;
    try { api.renderApp(); } catch (e) { failures++; console.log('FAIL', lang, view, e.message); }
  });
});
console.log('Full renderApp() regression sweep failures (expect 0):', failures);
```

Run: `node scratch_task5_regression.js`

Expected: `Full renderApp() regression sweep failures (expect 0): 0`

- [ ] **Step 5: 임시 Playwright 설치 폴더 정리**

```bash
rm -rf /tmp/major-edu-pw-check
```

- [ ] **Step 6: 커밋 + 푸시**

```bash
cd "C:\Users\user\Desktop\Dev\태준개인개발파일\CodeViewer"
git add index.html
git -c user.name="qoqmffh" -c user.email="qoqmffh@gmail.com" commit -m "Major Edu 코드초견: PNG 저장 기능 추가 + 최종 검증

SVG를 캔버스에 그려서 PNG로 다운로드(2배 해상도). Playwright로 실제
브라우저에서 생성->스크린샷->PNG 다운로드까지 end-to-end 확인, 전체
뷰 렌더 회귀 스윕도 통과. 이제 Major Edu > 코드초견 v1 완료."
git push origin main
```

---

## 스펙에서 의도적으로 이번 플랜에 포함하지 않은 것 (그대로 유지)

- 마디를 넘어가는 타이/당김음
- "반주에 솔로 출제"(Major Edu의 다른 하위 기능)
- 리얼북 PDF 연동, Rehamony
- 생성 결과 저장/공유
