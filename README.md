# social-ops-warmup

社媒养号与内容运营自动化实验（CDP 驱动，非隐藏式自动化）。

用 Chrome DevTools Protocol 驱动一个**真实、已登录**的 Chrome，代替手工点击：改资料、发内容、建仓库、回读校验。
浏览器就是桌面上那个带界面的 Chrome（独立 user-data-dir，调试端口只监听 `127.0.0.1`），没有任何注入 hook、不伪造 UA、不改指纹、不模拟人类轨迹。所有动作都发生在可见的页面里，人随时可以接管。

## 目录结构

```text
social-ops-warmup/
├── config/       # 账号、平台清单、文案模板
├── state/        # 每一步的状态快照：是否已发布、上次检查时间、回读结果
├── queue/        # 待发布内容队列（一条内容一个文件，便于人工复核）
├── logs/         # 运行日志与踩坑记录（现象 / 原因 / 解法）
├── scripts/      # 入口脚本：cdp.mjs、cm-replace.mjs、启动脚本
├── js/           # 注入页面执行的 JS 片段，供 evalf 使用
└── artifacts/    # 截图与渲染结果
```

## 核心脚本

### scripts/cdp.mjs（完整源码）

零依赖的 CDP 客户端（Node ≥ 22，用内置 `fetch` + `WebSocket`）。子命令覆盖了本实验需要的全部动作：
`version / targets / new / goto / eval / evalf / ax / clickax / typeax / clicknode / buttons / pierce / insertfield / shot / key / a11y / close / cookiecount / send`。

```js
#!/usr/bin/env node
// Minimal Chrome DevTools Protocol client (zero dependencies, Node >= 22).
// Talks to a Chrome started with --remote-debugging-port (loopback only).
//
// usage:
//   node cdp.mjs version
//   node cdp.mjs targets
//   node cdp.mjs new <url>                       -> prints targetId
//   node cdp.mjs eval <targetId|url-substr> <js> -> evaluates, prints JSON
//   node cdp.mjs a11y <targetId|url-substr> [maxChars]
//   node cdp.mjs shot <targetId|url-substr> <out.png>
//   node cdp.mjs goto <targetId|url-substr> <url>
//   node cdp.mjs close <targetId>
//
// env: CDP_PORT (default 9222), CDP_HOST (default 127.0.0.1)

import { writeFileSync, readFileSync } from 'node:fs';

const PORT = process.env.CDP_PORT || '9222';
const HOST = process.env.CDP_HOST || '127.0.0.1';
const BASE = `http://${HOST}:${PORT}`;

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

async function httpJson(path, method = 'GET') {
  const res = await fetch(BASE + path, { method });
  if (!res.ok) throw new Error(`HTTP ${res.status} ${method} ${path}`);
  const text = await res.text();
  try {
    return JSON.parse(text);
  } catch {
    return { raw: text };
  }
}

async function listPages() {
  return (await httpJson('/json/list')).filter((t) => t.type === 'page');
}

function resolveTarget(targets, key) {
  const byId = targets.find((t) => t.id === key);
  if (byId) return byId;
  const hits = targets.filter((t) => (t.url || '').includes(key));
  if (hits.length === 1) return hits[0];
  if (hits.length > 1) {
    throw new Error(
      `ambiguous target "${key}":\n` +
        hits.map((t) => `  ${t.id}  ${t.url}`).join('\n')
    );
  }
  throw new Error(
    `no target matching "${key}". open pages:\n` +
      targets.map((t) => `  ${t.id}  ${t.url}`).join('\n')
  );
}

class Session {
  constructor(wsUrl) {
    this.ws = null;
    this.wsUrl = wsUrl;
    this.id = 0;
    this.pending = new Map();
    this.events = [];
  }

  async connect() {
    this.ws = new WebSocket(this.wsUrl);
    await new Promise((resolve, reject) => {
      const t = setTimeout(() => reject(new Error('ws connect timeout')), 10000);
      this.ws.addEventListener('open', () => {
        clearTimeout(t);
        resolve();
      });
      this.ws.addEventListener('error', (e) => {
        clearTimeout(t);
        reject(new Error('ws error: ' + (e.message || 'unknown')));
      });
    });
    this.ws.addEventListener('message', (ev) => {
      const msg = JSON.parse(ev.data);
      if (msg.id && this.pending.has(msg.id)) {
        const { resolve, reject } = this.pending.get(msg.id);
        this.pending.delete(msg.id);
        if (msg.error) reject(new Error(`${msg.error.message} (${msg.error.code})`));
        else resolve(msg.result);
      } else if (msg.method) {
        this.events.push(msg);
      }
    });
  }

  send(method, params = {}) {
    const id = ++this.id;
    return new Promise((resolve, reject) => {
      this.pending.set(id, { resolve, reject });
      this.ws.send(JSON.stringify({ id, method, params }));
      setTimeout(() => {
        if (this.pending.has(id)) {
          this.pending.delete(id);
          reject(new Error(`timeout: ${method}`));
        }
      }, 45000);
    });
  }

  close() {
    try {
      this.ws.close();
    } catch {}
  }
}

async function withPage(key, fn) {
  const targets = await listPages();
  const t = key ? resolveTarget(targets, key) : targets[0];
  if (!t) throw new Error('no page targets open');
  const s = new Session(t.webSocketDebuggerUrl);
  await s.connect();
  try {
    return await fn(s, t);
  } finally {
    s.close();
  }
}

function flattenAX(nodes) {
  const lines = [];
  for (const n of nodes) {
    if (n.ignored) continue;
    const role = n.role?.value || '';
    if (!role || role === 'none' || role === 'generic' || role === 'InlineTextBox') continue;
    const name = (n.name?.value || '').replace(/\s+/g, ' ').trim();
    const value = (n.value?.value ?? '').toString().replace(/\s+/g, ' ').trim();
    if (!name && !value) continue;
    let line = role;
    if (name) line += ` "${name.slice(0, 220)}"`;
    if (value && value !== name) line += ` value="${value.slice(0, 120)}"`;
    if (n.checked?.value) line += ` checked=${n.checked.value}`;
    if (n.disabled?.value) line += ' disabled';
    lines.push(line);
  }
  return lines;
}

const [cmd, ...rest] = process.argv.slice(2);

// locate an element through the accessibility tree and click its centre with a
// real mouse event (no hard-coded coordinates, per handbook §5)
async function axClick(s, name, nth = 0) {
  await s.send('DOM.enable');
  const tree = await s.send('Accessibility.getFullAXTree', {});
  const norm = (v) => (v || '').replace(/\s+/g, ' ').trim();
  const usable = tree.nodes.filter((n) => !n.ignored && n.backendDOMNodeId && norm(n.name?.value));
  const clickableRoles = new Set([
    'button', 'link', 'textbox', 'tab', 'menuitem', 'menuitemcheckbox', 'checkbox', 'radio', 'switch', 'combobox',
  ]);
  const exact = usable.filter((n) => norm(n.name?.value) === name);
  const pool = exact.length ? exact : usable.filter((n) => norm(n.name?.value).includes(name));
  const preferred = pool.filter((n) => clickableRoles.has(n.role?.value));
  const cand = (preferred.length ? preferred : pool)[nth];
  if (!cand) return 'AX_NOT_FOUND: ' + name;
  try {
    await s.send('DOM.scrollIntoViewIfNeeded', { backendNodeId: cand.backendDOMNodeId });
  } catch {}
  const box = await s.send('DOM.getBoxModel', { backendNodeId: cand.backendDOMNodeId });
  const q = box.model.content;
  const x = (q[0] + q[2] + q[4] + q[6]) / 4;
  const y = (q[1] + q[3] + q[5] + q[7]) / 4;
  await s.send('Input.dispatchMouseEvent', { type: 'mouseMoved', x, y });
  await s.send('Input.dispatchMouseEvent', { type: 'mousePressed', x, y, button: 'left', buttons: 1, clickCount: 1 });
  await s.send('Input.dispatchMouseEvent', { type: 'mouseReleased', x, y, button: 'left', buttons: 0, clickCount: 1 });
  return `CLICKED ${cand.role?.value} "${norm(cand.name?.value).slice(0, 60)}" @ ${Math.round(x)},${Math.round(y)}`;
}


// watchdog: never hang the caller (renderers can stall on evaluate)
const WATCHDOG_MS = Number(process.env.CDP_TIMEOUT_MS || 25000);
const watchdog = setTimeout(() => {
  console.error(`TIMEOUT after ${WATCHDOG_MS}ms (cmd=${cmd})`);
  process.exit(3);
}, WATCHDOG_MS);
watchdog.unref();

const commands = {
  async version() {
    console.log(JSON.stringify(await httpJson('/json/version'), null, 2));
  },

  async targets() {
    const targets = await listPages();
    if (!targets.length) return console.log('(no page targets)');
    for (const t of targets) {
      console.log(`${t.id}  ${t.url}`);
      if (t.title) console.log(`    title: ${t.title}`);
    }
  },

  async new() {
    const url = rest[0] || 'about:blank';
    // Chrome >= 111 requires PUT for /json/new
    const target = await httpJson('/json/new?' + encodeURIComponent(url), 'PUT');
    console.log(target.id);
  },

  async eval() {
    const [key, expr] = rest;
    const out = await withPage(key, (s) =>
      s.send('Runtime.evaluate', {
        expression: expr,
        awaitPromise: true,
        returnByValue: true,
        userGesture: true,
      })
    );
    if (out.exceptionDetails) {
      console.error('EXCEPTION: ' + JSON.stringify(out.exceptionDetails, null, 2));
      process.exitCode = 2;
      return;
    }
    const v = out.result?.value;
    console.log(typeof v === 'string' ? v : JSON.stringify(v, null, 2));
  },

  // same as eval but reads the expression from a file (avoids shell quoting problems)
  async evalf() {
    const [key, file] = rest;
    const expr = readFileSync(file, 'utf8');
    const out = await withPage(key, (s) =>
      s.send('Runtime.evaluate', {
        expression: expr,
        awaitPromise: true,
        returnByValue: true,
        userGesture: true,
      })
    );
    if (out.exceptionDetails) {
      console.error('EXCEPTION: ' + JSON.stringify(out.exceptionDetails, null, 2));
      process.exitCode = 2;
      return;
    }
    const v = out.result?.value;
    console.log(typeof v === 'string' ? v : JSON.stringify(v, null, 2));
  },

  async a11y() {
    const [key, maxChars = '20000'] = rest;
    const out = await withPage(key, (s) => s.send('Accessibility.getFullAXTree', {}));
    const text = flattenAX(out.nodes).join('\n');
    console.log(text.slice(0, Number(maxChars)));
  },

  // find accessibility nodes by name substring (role + name only, no coords)
  async ax() {
    const [key, name = '', limit = '40'] = rest;
    const out = await withPage(key, (s) => s.send('Accessibility.getFullAXTree', {}));
    const hits = out.nodes
      .filter((n) => !n.ignored && n.backendDOMNodeId && (n.name?.value || '').includes(name))
      .slice(0, Number(limit))
      .map((n) => ({
        role: n.role?.value,
        name: (n.name?.value || '').replace(/\s+/g, ' ').slice(0, 90),
        backendNodeId: n.backendDOMNodeId,
      }));
    console.log(JSON.stringify(hits, null, 1));
  },

  // real mouse click on an element located through the accessibility tree
  async clickax() {
    const [key, name, nth = '0'] = rest;
    console.log(await withPage(key, (s) => axClick(s, name, Number(nth))));
  },

  // press a single key (Escape / Tab / Enter ...) through the input domain
  async key() {
    const [key, name] = rest;
    const table = {
      Escape: ['Escape', 'Escape', 27],
      Tab: ['Tab', 'Tab', 9],
      Enter: ['Enter', 'Enter', 13],
      Backspace: ['Backspace', 'Backspace', 8],
      ArrowDown: ['ArrowDown', 'ArrowDown', 40],
    };
    const [k, code, vk] = table[name] || [name, name, 0];
    await withPage(key, async (s) => {
      await s.send('Input.dispatchKeyEvent', { type: 'keyDown', key: k, code, windowsVirtualKeyCode: vk });
      await s.send('Input.dispatchKeyEvent', { type: 'keyUp', key: k, code, windowsVirtualKeyCode: vk });
    });
    console.log('key ' + name);
  },

  // click a field via AX, then insert text as if typed
  async typeax() {
    const [key, name, text, nthArg = '0'] = rest;
    const res = await withPage(key, async (s) => {
      const clicked = await axClick(s, name, Number(nthArg));
      if (clicked.startsWith('AX_NOT_FOUND')) return clicked;
      await sleep(350);
      await s.send('Input.insertText', { text });
      return clicked + ` ; typed ${text.length} chars`;
    });
    console.log(res);
  },

  // evaluate a JS file that returns {x,y}; click there with real mouse events
  async clickpt() {
    const [key, file] = rest;
    const expr = readFileSync(file, 'utf8');
    const res = await withPage(key, async (s) => {
      const out = await s.send('Runtime.evaluate', { expression: expr, returnByValue: true, awaitPromise: true });
      if (out.exceptionDetails) throw new Error(JSON.stringify(out.exceptionDetails.exception?.description || out.exceptionDetails.text));
      const v = out.result?.value;
      if (!v || typeof v.x !== 'number') return 'NO_POINT: ' + JSON.stringify(v);
      await s.send('Input.dispatchMouseEvent', { type: 'mouseMoved', x: v.x, y: v.y });
      await s.send('Input.dispatchMouseEvent', { type: 'mousePressed', x: v.x, y: v.y, button: 'left', buttons: 1, clickCount: 1 });
      await s.send('Input.dispatchMouseEvent', { type: 'mouseReleased', x: v.x, y: v.y, button: 'left', buttons: 0, clickCount: 1 });
      return `CLICKED @ ${Math.round(v.x)},${Math.round(v.y)}` + (v.note ? ' (' + v.note + ')' : '');
    });
    console.log(res);
  },

  // list editable nodes, piercing open shadow roots
  async pierce() {
    const [key, filter = 'TEXTAREA'] = rest;
    const res = await withPage(key, async (s) => {
      await s.send('DOM.enable');
      const doc = await s.send('DOM.getDocument', { depth: -1, pierce: true });
      const out = [];
      const walk = (n) => {
        if (n.nodeName === filter || filter === '*') {
          const attrs = {};
          for (let i = 0; i < (n.attributes || []).length; i += 2) attrs[n.attributes[i]] = n.attributes[i + 1];
          out.push({ backendNodeId: n.backendNodeId, tag: n.nodeName, aria: attrs['aria-label'] || '', cls: (attrs.class || '').slice(0, 40), val: (n.nodeValue || '').slice(0, 30) });
        }
        for (const c of n.children || []) walk(c);
        for (const sr of n.shadowRoots || []) walk(sr);
        if (n.contentDocument) walk(n.contentDocument);
      };
      walk(doc.root);
      return out;
    });
    console.log(JSON.stringify(res, null, 1));
  },

  // focus an element by backendNodeId and insert text from a file
  async insertfield() {
    const [key, backendNodeId, file] = rest;
    const text = readFileSync(file, 'utf8').trim();
    const res = await withPage(key, async (s) => {
      await s.send('DOM.enable');
      await s.send('DOM.focus', { backendNodeId: Number(backendNodeId) });
      await sleep(200);
      await s.send('Input.insertText', { text });
      const out = await s.send('Runtime.evaluate', {
        expression: 'document.activeElement ? document.activeElement.tagName : "none"',
        returnByValue: true,
      });
      return `focused=${out.result?.value} inserted=${text.length}`;
    });
    console.log(res);
  },

  // list button-ish nodes (piercing shadow roots) with their text
  async buttons() {
    const [key, maxN = '30'] = rest;
    const res = await withPage(key, async (s) => {
      await s.send('DOM.enable');
      const doc = await s.send('DOM.getDocument', { depth: -1, pierce: true });
      const out = [];
      const textOf = (n) => {
        let t = n.nodeName === '#text' ? n.nodeValue || '' : '';
        for (const c of n.children || []) t += ' ' + textOf(c);
        return t.replace(/\s+/g, ' ').trim();
      };
      const walk = (n) => {
        const attrs = {};
        for (let i = 0; i < (n.attributes || []).length; i += 2) attrs[n.attributes[i]] = n.attributes[i + 1];
        if (n.nodeName === 'BUTTON' || attrs.role === 'button') {
          out.push({ backendNodeId: n.backendNodeId, tag: n.nodeName, text: textOf(n).slice(0, 20), aria: attrs['aria-label'] || '' });
        }
        for (const c of n.children || []) walk(c);
        for (const sr of n.shadowRoots || []) walk(sr);
        if (n.contentDocument) walk(n.contentDocument.root ? n.contentDocument : n.contentDocument);
      };
      walk(doc.root);
      return out.slice(0, Number(maxN));
    });
    console.log(JSON.stringify(res, null, 1));
  },

  // click a node by backendNodeId, works inside shadow roots (resolves to a JS object)
  async clicknode() {
    const [key, backendNodeId] = rest;
    const res = await withPage(key, async (s) => {
      await s.send('DOM.enable');
      const { object } = await s.send('DOM.resolveNode', { backendNodeId: Number(backendNodeId) });
      const out = await s.send('Runtime.callFunctionOn', {
        objectId: object.objectId,
        functionDeclaration: 'function(){ this.click(); return (this.innerText||this.tagName||"?").slice(0,30); }',
        returnByValue: true,
      });
      return 'clicked node => ' + (out.result?.value ?? '?');
    });
    console.log(res);
  },

  async shot() {
    const [key, outPath] = rest;
    const out = await withPage(key, async (s) => {
      await s.send('Page.enable');
      return s.send('Page.captureScreenshot', { format: 'png', fromSurface: true, captureBeyondViewport: false });
    });
    const buf = Buffer.from(out.data, 'base64');
    writeFileSync(outPath, buf);
    console.log(`saved ${outPath} (${buf.length} bytes)`);
  },

  async goto() {
    const [key, url] = rest;
    await withPage(key, async (s) => {
      await s.send('Page.enable');
      await s.send('Page.navigate', { url });
      await sleep(1500);
    });
    console.log('navigated');
  },

  async close() {
    const [key] = rest;
    const targets = await listPages();
    const t = resolveTarget(targets, key);
    await fetch(`${BASE}/json/close/${t.id}`);
    console.log('closed ' + t.id);
  },

  // cookie presence audit: domains + counts only, never values
  async cookiecount() {
    const [key] = rest;
    const out = await withPage(key, async (s) => {
      await s.send('Network.enable');
      return s.send('Network.getAllCookies');
    });
    const byDomain = new Map();
    for (const c of out.cookies) {
      const k = '.' + String(c.domain).replace(/^\./, '');
      byDomain.set(k, (byDomain.get(k) || 0) + 1);
    }
    console.log(`total cookies: ${out.cookies.length}`);
    for (const [d, n] of [...byDomain].sort((a, b) => b[1] - a[1])) {
      console.log(`  ${String(n).padStart(4)}  ${d}`);
    }
  },

  // raw protocol escape hatch: node cdp.mjs send <method> <json>
  async send() {
    const [method, json = '{}', key] = rest;
    const out = await withPage(key, (s) => s.send(method, JSON.parse(json)));
    console.log(JSON.stringify(out, null, 2));
  },
};

if (!commands[cmd]) {
  console.error('unknown command: ' + cmd);
  console.error('commands: ' + Object.keys(commands).join(', '));
  process.exit(1);
}
await commands[cmd]();
clearTimeout(watchdog);
// explicit exit: a lingering socket must not keep the CLI alive
process.exit(process.exitCode || 0);
```

### scripts/start-ops-chrome.cmd（完整内容）

调试端口只监听回环地址，独立用户数据目录，避免和日常浏览器的 cookie 打架。

```bat
@echo off
REM 启动「梵高 fangao019@gmail.com」专用自动化 Chrome 实例
REM 方案 B：独立 user-data-dir + 本目录内手动登录一次（Chrome 136+ 要求独立目录；
REM          Chrome 127+ 的 App-Bound Encryption 使克隆来的 cookie 无法跨目录解密）
REM 调试端口仅监听 127.0.0.1
setlocal
set "CHROME=C:\Program Files\Google\Chrome\Application\chrome.exe"
set "UDD=D:\DSH\ops\chrome-automation\User Data"

if not exist "%UDD%" mkdir "%UDD%"

start "" "%CHROME%" ^
 --user-data-dir="%UDD%" ^
 --profile-directory=Default ^
 --remote-debugging-port=9222 ^
 --remote-allow-origins=* ^
 --no-first-run ^
 --no-default-browser-check

echo started. verify: http://127.0.0.1:9222/json/version
endlocal
```

## 用法示例

```powershell
# 1. 确认调试端口可用
node cdp.mjs version

# 2. 列出当前所有标签页（找到要操作的 target）
node cdp.mjs targets

# 3. 执行一段页面脚本（脚本写在文件里，避免 shell 改写引号）
node cdp.mjs evalf instagram js/profile-text.js

# 4. 按可访问性名字点击控件
node cdp.mjs clickax threads "编辑主页"

# 5. 列出页面上的输入框，拿到 backendNodeId
node cdp.mjs pierce github TEXTAREA

# 6. 按 backendNodeId 聚焦并整段写入（比坐标点击稳）
node cdp.mjs insertfield github 914 js/text-bio.txt

# 7. 整段替换 CodeMirror 之类编辑器里的内容
node cm-replace.mjs github ".cm-content" js/gh-profile-readme.md

# 8. 截图留证
node cdp.mjs shot github artifacts/gh-profile.png
```

## 踩坑记录 / Lessons learned

1. **内联 JS 会被 PowerShell 改写。** `node cdp.mjs eval github 'document.title=="x"'` 里的中文、引号、`|`、`=>` 到不了浏览器，报 `SyntaxError` 或静默返回空。解法：脚本一律写成 `js/*.js`，用 `evalf` 执行。
2. **`cdp.mjs send` 的 JSON 参数会被吞掉双引号。** `'{"type":"keyDown"}'` 到 Node 里变成 `{type:keyDown}`，`JSON.parse` 报 `Expected property name or '}' in JSON at position 1`。解法：在 PowerShell 单引号字符串里把双引号写成 `\"`（`'{\"type\":\"keyDown\"}'`），或用专门的小脚本。
3. **`goto` 之后必须再等。** `goto` 内部只等 1.5s，设置页保存后有跳转/重渲染，紧接着读 DOM 拿到的是旧页面。解法：`Start-Sleep -Seconds 5` 再读，且保存后重新 `goto` 目标页回读，而不是相信表单里的值。
4. **GitHub 新版建仓页不是普通表单。** 仓库名是 `#repository-name-input`，"Add README" 是开关按钮 `button[aria-labelledby="add-readme"]`（状态看 `data-checked` 或旁边 `On/Off` 文本），点标题文字、点 `aria-label` 都无效。解法：读 `main` 的 `innerText` 判断开关状态，再点真正的 switch。
5. **React 按钮对合成鼠标事件不总买账。** `clickax "Create repository"` 报 `CLICKED` 但页面没提交；换成筛选 `innerText === 'Create repository'` 后 `.click()` 才成功。解法：先看 URL 是否真的变了，没变就换 `clicknode` / JS `.click()`。
6. **CodeMirror 6 没有 `.value`。** 直接 `el.value = ...` 或改 DOM 不会同步到编辑器状态，提交的还是旧内容。解法：`DOM.focus` → `Input.dispatchKeyEvent` 带 `commands:["selectAll"]` → `Input.insertText`（本仓库 `cm-replace.mjs`）。
7. **`typeax` 依赖坐标，页面一滚就点空。** 本次 `typeax "Repository name *" social-ops-warmup` 返回 `typed 17 chars`，但输入框仍是空——文字进了虚空。解法：用 `insertfield <backendNodeId> <file>`，再用 `document.querySelector('#repository-name-input').value` 回读。
8. **自定义元素 / Shadow DOM 里的按钮，`querySelector` 找不到。** 解法：`pierce` 列出 DOM（`DOM.getDocument {depth:-1, pierce:true}`）拿 `backendNodeId`，再 `clicknode` 直接 `.click()`，可穿透 shadow root。
9. **`/json/new` 必须用 PUT。** Chrome 111+ 起 `GET /json/new?url=` 返回 405，`cdp.mjs new` 里用的是 `fetch(url, {method:'PUT'})`。
10. **写入必须回读，回读要看渲染结果。** `insertfield` 返回 `inserted=N` 只代表事件发出去了；本次 profile README 的链接数是靠读 `.markdown-body` 里 `<a href>` 才确认的（页面上 `h3` 外面还包着 `.markdown-heading`，直接取 `h3.nextElementSibling` 会是 `null`）。

## 合规边界

- 只操作本实验自己注册的账号（`fangao019` 及其在各平台的同身份账号）。
- 不绕过任何平台的人机校验，不做隐藏式自动化，不改指纹与 UA。
- 每步写入后都回读校验；失败就记日志，不静默重试到风控触发。
