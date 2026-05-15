# 训练组录入重构实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构弹窗内训练组录入区域：五列表格 + 组类型切换 + 自定义全宽数字键盘 + 历史数据查询

**Architecture:** 所有改动集中在 `fitness.html`。数据结构扩展 type/done 字段。组明细从 flex row 改为 CSS Grid 五列。数字键盘用 `<div>` + `tabindex` 模拟输入框阻止系统键盘弹出。全宽 Grid 布局按键。

**Tech Stack:** 纯 HTML/CSS/JS，localStorage

---

### Task 1: 数据迁移 — 扩展 type/done 字段

**Files:**
- Modify: `fitness.html`（数据迁移逻辑区域）

- [ ] **Step 1: 更新数据迁移逻辑**

找到现有的数据迁移代码（在 `let migrated = false;` 附近），扩展迁移逻辑以处理 `type` 和 `done` 字段：

```js
// 迁移旧数据：扁平结构 → 每组独立，并补充 type/done 字段
let migrated = false;
Object.keys(data).forEach(date => {
  data[date] = data[date].map(e => {
    // 旧格式：{sets,reps,weight} → 新格式
    if (Array.isArray(e.sets)) {
      // 已是新格式，检查每组是否有 type/done
      var setsMigrated = false;
      e.sets = e.sets.map(function(set) {
        if (set.type === undefined) { set.type = 'normal'; setsMigrated = true; }
        if (set.done === undefined) { set.done = false; setsMigrated = true; }
        return set;
      });
      if (setsMigrated) { migrated = true; return e; }
      return e;
    }
    migrated = true;
    return {
      id: e.id || genId(),
      name: e.name,
      sets: Array.from({ length: e.sets || 1 }, function(_, i) {
        return { reps: e.reps || 0, weight: e.weight || 0, type: 'normal', done: false };
      }),
      volume: e.volume || 0
    };
  });
});
if (migrated) save();
```

关键点：即使 sets 已为数组格式，也要补全 `type`/`done` 字段（向前兼容上一版本数据）。

- [ ] **Step 2: JS 语法检查**

```bash
node -e "const fs = require('fs'); const html = fs.readFileSync('E:/Claude Code File/Fitness File/fitness.html', 'utf8'); const match = html.match(/<script>([\s\S]*?)<\/script>/); new Function(match[1]); console.log('PASS')"
```
预期: PASS

- [ ] **Step 3: Commit**

```bash
git add fitness.html
git commit -m "feat: add type/done field migration for set entries"
```

---

### Task 2: CSS — 五列表格 + 组类型标签 + 自定义键盘

**Files:**
- Modify: `fitness.html`（`<style>` 块）

- [ ] **Step 1: 替换 .set-detail-list 及相关样式**

将现有的：
```css
.set-detail-list { margin-bottom: 12px; min-height: 180px; }
.set-detail-row { ... }
.set-detail-row .set-label { ... }
.set-detail-row input { ... }
.set-detail-row input:focus { ... }
.set-detail-row .set-row-vol { ... }
```

全部替换为以下新样式：

```css
/* 五列训练组表格 */
.set-table { width: 100%; border-collapse: collapse; margin-bottom: 12px; min-height: 160px; }
.set-table th {
  font-size: 11px; color: #9ca3af; font-weight: 500;
  padding: 6px 2px; text-align: center;
}
.set-table th:first-child { text-align: left; padding-left: 4px; }
.set-table td { padding: 5px 2px; text-align: center; vertical-align: middle; }
.set-row { transition: opacity 0.15s; }
.set-row.done-row { opacity: 0.45; }

/* 组类型标签 */
.set-type-tag {
  display: inline-block; padding: 5px 8px; border-radius: 6px;
  font-size: 12px; font-weight: 600; cursor: pointer;
  background: #eef2ff; color: #4f46e5; min-width: 34px;
  text-align: center; position: relative; user-select: none;
  white-space: nowrap;
}
.set-type-tag.warmup { background: #fef3c7; color: #d97706; }
.set-type-tag.dropset { background: #fce7f3; color: #db2777; }

.type-popup {
  display: none; position: absolute; top: 100%; left: 0; z-index: 20;
  background: #fff; border-radius: 10px; box-shadow: 0 4px 16px rgba(0,0,0,0.15);
  overflow: hidden; min-width: 80px; margin-top: 4px;
}
.type-popup.show { display: block; }
.type-popup div {
  padding: 9px 14px; font-size: 12px; font-weight: 500;
  cursor: pointer; white-space: nowrap;
}
.type-popup div:hover { background: #f3f4f6; }

/* 历史数据 */
.last-data { font-size: 12px; color: #9ca3af; white-space: nowrap; }

/* 数值单元格 */
.num-cell {
  display: inline-block; padding: 9px 4px; border-radius: 8px;
  background: #f3f4f6; font-size: 15px; font-weight: 600;
  color: #374151; cursor: pointer; min-width: 40px;
  border: 2px solid transparent; text-align: center;
  transition: all 0.15s; outline: none;
}
.num-cell.active { border-color: #4f46e5; background: #fff; }
.num-cell:focus { border-color: #4f46e5; background: #fff; }

/* 完成 checkbox */
.cb-cell {
  width: 26px; height: 26px; border-radius: 6px;
  border: 2px solid #d1d5db; cursor: pointer; display: inline-flex;
  align-items: center; justify-content: center;
  font-size: 13px; color: transparent; transition: all 0.15s;
  user-select: none;
}
.cb-cell.done { background: #4f46e5; border-color: #4f46e5; color: #fff; }

/* 自定义键盘 */
.kb-wrap { display: none; margin-top: 12px; }
.kb-wrap.show { display: block; }
.kb-focus-hint {
  text-align: center; font-size: 12px; color: #4f46e5;
  margin-bottom: 8px; font-weight: 600;
}
.kb-grid {
  display: grid; grid-template-columns: repeat(4, 1fr);
  gap: 6px; background: #f8f9fa; padding: 10px;
  border-radius: 14px;
}
.kb-key {
  padding: 14px 0; border-radius: 10px; border: none;
  font-size: 20px; font-weight: 600; cursor: pointer;
  background: #fff; color: #1a1a1a;
  box-shadow: 0 1px 3px rgba(0,0,0,0.06);
  text-align: center; transition: all 0.1s;
  font-family: inherit;
}
.kb-key:active { background: #e5e7eb; transform: scale(0.95); }
.kb-key.fn { font-size: 13px; color: #6b7280; background: #f3f4f6; }
.kb-key.accent { background: #4f46e5; color: #fff; font-size: 14px; }
.kb-key.adj { background: #eef2ff; color: #4f46e5; font-size: 22px; }
.kb-key.span2 { grid-column: span 2; }
```

删除 `.copy-all` 相关样式（不再需要"复制第一组到所有"功能）。

- [ ] **Step 2: JS 语法检查 + Commit**

```bash
git add fitness.html
git commit -m "style: add 5-column table, type tags, and custom keyboard CSS"
```

---

### Task 3: HTML — 自定义键盘 + 更新组明细区域

**Files:**
- Modify: `fitness.html`（sheet 内 HTML）

- [ ] **Step 1: 更新组明细区域**

找到 sheet 内的 set-detail-list 区域，替换现有的：

```html
<div class="set-detail-list" id="setDetailList"></div>
<button class="copy-all" id="copyAll">⬇ 第一组数据应用到所有组</button>
```

替换为：

```html
<div id="setTableWrap">
  <table class="set-table" id="setTable">
    <thead>
      <tr>
        <th style="width:17%;text-align:left;">组</th>
        <th style="width:20%;">上一次</th>
        <th style="width:24%;">重量 (kg)</th>
        <th style="width:22%;">次数</th>
        <th style="width:10%;">✓</th>
      </tr>
    </thead>
    <tbody id="setTableBody"></tbody>
  </table>
</div>

<div class="kb-wrap" id="kbWrap">
  <div class="kb-focus-hint" id="kbHint">正在输入：重量 (±0.25 kg)</div>
  <div class="kb-grid" id="kbGrid">
    <button class="kb-key" data-key="1">1</button>
    <button class="kb-key" data-key="2">2</button>
    <button class="kb-key" data-key="3">3</button>
    <button class="kb-key fn" data-key="back">⌫</button>
    <button class="kb-key" data-key="4">4</button>
    <button class="kb-key" data-key="5">5</button>
    <button class="kb-key" data-key="6">6</button>
    <button class="kb-key adj" data-key="plus">+</button>
    <button class="kb-key" data-key="7">7</button>
    <button class="kb-key" data-key="8">8</button>
    <button class="kb-key" data-key="9">9</button>
    <button class="kb-key adj" data-key="minus">−</button>
    <button class="kb-key fn" data-key="dot">.</button>
    <button class="kb-key" data-key="0">0</button>
    <button class="kb-key accent span2" data-key="confirm">确认</button>
  </div>
</div>
```

注意：删除 `<button class="copy-all" id="copyAll">` 及其 JS 事件监听器。

- [ ] **Step 2: 更新 JS 中 setDetailList 引用**

`setDetailList` 现在指向旧元素。后续 Task 5 会重建渲染逻辑，此处先改引用：

查找 `const setDetailList = document.getElementById('setDetailList');`，改为：
```js
const setTableBody = document.getElementById('setTableBody');
```

删除 `const setDetailList` 行。

- [ ] **Step 3: Commit**

```bash
git add fitness.html
git commit -m "feat: add custom keyboard markup and 5-column table structure"
```

---

### Task 4: JS — getLastExerciseData 历史查询

**Files:**
- Modify: `fitness.html`（在 `genId()` 之后）

- [ ] **Step 1: 添加函数**

```js
function getLastExerciseData(exerciseName, currentDate) {
  var dates = Object.keys(data).filter(function(d) { return d !== currentDate; });
  dates.sort(function(a, b) { return b.localeCompare(a); }); // 降序
  for (var i = 0; i < dates.length; i++) {
    var entries = data[dates[i]] || [];
    for (var j = 0; j < entries.length; j++) {
      if (entries[j].name === exerciseName) {
        return entries[j].sets; // 返回最近一次该动作的所有组
      }
    }
  }
  return null;
}
```

- [ ] **Step 2: Commit**

```bash
git add fitness.html
git commit -m "feat: add getLastExerciseData for history lookup"
```

---

### Task 5: JS — 重写组渲染逻辑（核心）

**Files:**
- Modify: `fitness.html`（替换 addSetRow / setSetCount / 相关逻辑）

- [ ] **Step 1: 删除旧函数并添加新渲染函数**

删除 `addSetRow` 函数和 `setSetCount` 函数。删除 copyAll 事件监听器（`document.getElementById('copyAll').addEventListener...`）。

添加以下新函数：

```js
// 当前焦点状态
var kbTarget = null;   // { setIndex: 0, field: 'weight'|'reps' }
var currentSets = [];  // 当前编辑中的 sets 数组

function getFormalCount(sets, upTo) {
  var count = 0;
  for (var i = 0; i < upTo; i++) {
    if (sets[i] && sets[i].type === 'normal') count++;
  }
  return count;
}

function getSetLabel(sets, i) {
  if (!sets[i]) return '第' + (i + 1) + '组';
  if (sets[i].type === 'warmup') return '热';
  if (sets[i].type === 'dropset') return '减';
  var num = getFormalCount(sets, i) + 1;
  return '第' + num + '组';
}

function getSetTypeClass(sets, i) {
  if (!sets[i]) return '';
  if (sets[i].type === 'warmup') return ' warmup';
  if (sets[i].type === 'dropset') return ' dropset';
  return '';
}

function renderSetTable() {
  var tbody = document.getElementById('setTableBody');
  var lastSets = null;
  // 尝试获取历史数据
  var exName = document.getElementById('exCustom').value.trim() || selectedExercise;
  if (exName) {
    lastSets = getLastExerciseData(exName, selectedDate);
  }

  var html = '';
  for (var i = 0; i < currentSetCount; i++) {
    var set = currentSets[i] || { reps: 0, weight: 0, type: 'normal', done: false };
    var typeClass = getSetTypeClass(currentSets, i);
    var label = getSetLabel(currentSets, i);
    var doneClass = set.done ? ' done-row' : '';
    var cbClass = set.done ? ' done' : '';

    // 上一次数据
    var lastText = '—';
    if (lastSets && lastSets[i]) {
      var ls = lastSets[i];
      lastText = ls.weight + '×' + ls.reps;
    }

    // 当前值
    var weightVal = set.weight || '';
    var repsVal = set.reps || '';

    html += '<tr class="set-row' + doneClass + '">' +
      '<td><span class="set-type-tag' + typeClass + '" data-set="' + i + '">' + label + '</span></td>' +
      '<td><span class="last-data">' + lastText + '</span></td>' +
      '<td><span class="num-cell" data-set="' + i + '" data-field="weight" tabindex="0">' + weightVal + '</span></td>' +
      '<td><span class="num-cell" data-set="' + i + '" data-field="reps" tabindex="0">' + repsVal + '</span></td>' +
      '<td><span class="cb-cell' + cbClass + '" data-set="' + i + '">✓</span></td>' +
      '</tr>';
  }
  tbody.innerHTML = html;

  // 绑定事件
  bindSetTableEvents();
  updateTotalVol();
}

function bindSetTableEvents() {
  // 类型标签点击
  document.querySelectorAll('.set-type-tag').forEach(function(tag) {
    tag.addEventListener('click', function(e) {
      e.stopPropagation();
      showTypePopup(tag);
    });
  });

  // 数值单元格点击
  document.querySelectorAll('.num-cell').forEach(function(cell) {
    cell.addEventListener('click', function() {
      openKeyboard(cell);
    });
    cell.addEventListener('focus', function() {
      openKeyboard(cell);
    });
  });

  // 完成 checkbox
  document.querySelectorAll('.cb-cell').forEach(function(cb) {
    cb.addEventListener('click', function() {
      var si = parseInt(cb.dataset.set);
      if (currentSets[si]) {
        currentSets[si].done = !currentSets[si].done;
        renderSetTable();
      }
    });
  });
}

function showTypePopup(tag) {
  // 关闭其他弹窗
  document.querySelectorAll('.type-popup.show').forEach(function(p) {
    p.classList.remove('show');
  });

  var popup = tag.querySelector('.type-popup');
  if (!popup) {
    var si = parseInt(tag.dataset.set);
    popup = document.createElement('div');
    popup.className = 'type-popup show';
    popup.innerHTML = '<div data-type="normal">正式组</div><div data-type="warmup">热身组</div><div data-type="dropset">递减组</div>';
    tag.appendChild(popup);

    popup.querySelectorAll('div').forEach(function(div) {
      div.addEventListener('click', function(e) {
        e.stopPropagation();
        var newType = div.dataset.type;
        if (currentSets[si]) {
          currentSets[si].type = newType;
        }
        popup.remove();
        renderSetTable();
      });
    });
  } else {
    popup.classList.toggle('show');
  }

  // 点击外部关闭
  setTimeout(function() {
    document.addEventListener('click', function closePopup() {
      var p = tag.querySelector('.type-popup');
      if (p) p.classList.remove('show');
      document.removeEventListener('click', closePopup);
    }, { once: true });
  }, 0);
}
```

- [ ] **Step 2: 重写 setSetCount 函数**

```js
function setSetCount(n) {
  n = Math.max(1, Math.min(20, n));
  // 保持旧数据
  var oldSets = currentSets.slice();
  currentSets = [];
  for (var i = 0; i < n; i++) {
    currentSets[i] = oldSets[i] || { reps: 0, weight: 0, type: 'normal', done: false };
  }
  currentSetCount = n;
  setCountDisplay.textContent = n;
  renderSetTable();
}
```

- [ ] **Step 3: 更新 updateTotalVol 和 updateRowVol 逻辑**

删除 `updateRowVol` 函数（不再需要单行更新）。重写 `updateTotalVol`：

```js
function updateTotalVol() {
  var total = 0;
  currentSets.forEach(function(set) {
    total += (set.reps || 0) * (set.weight || 0);
  });
  volPreview.textContent = total.toLocaleString();
}
```

- [ ] **Step 4: JS 语法检查 + Commit**

```bash
node -e "..." && git add fitness.html && git commit -m "feat: rewrite set rendering with 5-column table layout"
```

---

### Task 6: JS — 自定义数字键盘逻辑

**Files:**
- Modify: `fitness.html`

- [ ] **Step 1: 添加键盘逻辑函数**

```js
function openKeyboard(cell) {
  var si = parseInt(cell.dataset.set);
  var field = cell.dataset.field;
  kbTarget = { setIndex: si, field: field };

  // 高亮当前 cell
  document.querySelectorAll('.num-cell.active').forEach(function(c) { c.classList.remove('active'); });
  cell.classList.add('active');

  // 更新键盘提示
  var hint = document.getElementById('kbHint');
  if (field === 'weight') {
    hint.textContent = '正在输入：重量 (步长 ±0.25 kg)';
  } else {
    hint.textContent = '正在输入：次数 (步长 ±1 次)';
  }

  // 显示键盘
  document.getElementById('kbWrap').classList.add('show');
}

function closeKeyboard() {
  document.getElementById('kbWrap').classList.remove('show');
  document.querySelectorAll('.num-cell.active').forEach(function(c) { c.classList.remove('active'); });
  kbTarget = null;
}

function handleKBInput(value) {
  if (!kbTarget || !currentSets[kbTarget.setIndex]) return;
  var set = currentSets[kbTarget.setIndex];

  if (value === 'back') {
    var str = String(set[kbTarget.field] || '');
    set[kbTarget.field] = str.length > 1 ? parseFloat(str.slice(0, -1)) || 0 : 0;
  } else if (value === 'dot') {
    if (kbTarget.field === 'weight') {
      var str = String(set[kbTarget.field] || '0');
      if (str.indexOf('.') < 0) {
        set[kbTarget.field] = str + '.';
      }
    }
  } else if (value === 'plus') {
    var step = kbTarget.field === 'weight' ? 0.25 : 1;
    set[kbTarget.field] = roundVal((set[kbTarget.field] || 0) + step, kbTarget.field);
  } else if (value === 'minus') {
    var step = kbTarget.field === 'weight' ? 0.25 : 1;
    set[kbTarget.field] = roundVal(Math.max(0, (set[kbTarget.field] || 0) - step), kbTarget.field);
  } else if (value === 'confirm') {
    // 确认：移动到下一个 cell（或关闭键盘）
    closeKeyboard();
  } else {
    // 数字
    var cur = String(set[kbTarget.field] || '');
    if (cur === '0' && value !== '0') cur = '';
    if (cur === '' && value === '0') cur = '0';
    else cur = cur + value;
    set[kbTarget.field] = parseFloat(cur) || 0;
  }

  renderSetTable();
  // 重新高亮当前 cell
  var cell = document.querySelector('.num-cell[data-set="' + kbTarget.setIndex + '"][data-field="' + kbTarget.field + '"]');
  if (cell) { cell.classList.add('active'); cell.focus(); }
}

function roundVal(val, field) {
  if (field === 'weight') {
    return Math.round(val * 100) / 100; // 保留2位小数
  }
  return Math.round(val);
}
```

- [ ] **Step 2: 绑定键盘按键事件**

```js
document.getElementById('kbGrid').addEventListener('click', function(e) {
  var key = e.target.closest('.kb-key');
  if (!key) return;
  handleKBInput(key.dataset.key);
});
```

- [ ] **Step 3: 点击键盘外区域关闭**

在 overlay 点击事件中添加（不影响弹窗关闭逻辑）：

```js
document.getElementById('kbWrap').addEventListener('click', function(e) {
  e.stopPropagation(); // 阻止冒泡到 overlay
});
```

修改确认按钮逻辑：确认后焦点移到下一格或关闭键盘。

- [ ] **Step 4: Commit**

```bash
git add fitness.html && git commit -m "feat: add custom numeric keyboard with +/- step adjustment"
```

---

### Task 7: JS — 集成：更新 openAddForm/openEditForm/提交逻辑

**Files:**
- Modify: `fitness.html`

- [ ] **Step 1: 更新 openAddForm 和 openEditForm**

在 `openAddForm` 中初始化 `currentSets`：
```js
function openAddForm() {
  editingId = null;
  sheetTitle.textContent = '添加训练';
  submitBtn.textContent = '确认添加';
  currentCategory = CATEGORY_KEYS[0];
  selectedExercise = '';
  document.getElementById('exCustom').value = '';
  currentSets = [];
  setSetCount(3);
  renderCatTabs();
  renderExGrid();
  overlay.classList.add('show');
  closeKeyboard();
}
```

在 `openEditForm` 中初始化 `currentSets`（从 entry 复制）：
```js
function openEditForm(entry) {
  editingId = entry.id;
  sheetTitle.textContent = '编辑训练';
  submitBtn.textContent = '确认修改';

  // ... 分类查找逻辑不变 ...

  // 初始化 currentSets（深拷贝）
  currentSets = entry.sets.map(function(set) {
    return { reps: set.reps, weight: set.weight, type: set.type || 'normal', done: set.done || false };
  });
  currentSetCount = currentSets.length;
  setCountDisplay.textContent = currentSetCount;
  updateTotalVol();
  renderCatTabs();
  renderExGrid();
  overlay.classList.add('show');
  closeKeyboard();
}
```

- [ ] **Step 2: 更新提交逻辑**

修改 submit 按钮的 click handler 中创建 entry 的部分：

```js
// 提交时从 currentSets 构建数据
var sets = [];
currentSets.forEach(function(set) {
  if ((set.reps > 0 || set.weight > 0) || set.done) {
    sets.push({
      reps: set.reps || 0,
      weight: set.weight || 0,
      type: set.type || 'normal',
      done: set.done || false
    });
  }
});

if (sets.length === 0 && currentSets.every(function(s) { return !s.done; })) {
  alert('请至少填写一组的次数或重量');
  return;
}

var volume = sets.reduce(function(s, set) { return s + set.reps * set.weight; }, 0);
var entry = { id: editingId || genId(), name: name, sets: sets, volume: volume };
```

- [ ] **Step 3: 删除 copyAll 的 JS 事件**

删除：
```js
document.getElementById('copyAll').addEventListener('click', ...);
```

- [ ] **Step 4: JS 语法检查 + Commit**

```bash
node -e "..." && git add fitness.html && git commit -m "feat: integrate new set entry with openAddForm/openEditForm/submit"
```

---

### Task 8: 最终验证

- [ ] **Step 1: JS 语法检查**

```bash
node -e "const fs = require('fs'); const html = fs.readFileSync('E:/Claude Code File/Fitness File/fitness.html', 'utf8'); const match = html.match(/<script>([\s\S]*?)<\/script>/); new Function(match[1]); console.log('PASS')"
```
预期: PASS

- [ ] **Step 2: 无残留引用检查**

```bash
grep -n "setDetailList\|addSetRow\|copyAll\|updateRowVol" "E:/Claude Code File/Fitness File/fitness.html"
```
预期: 无结果（或被注释掉）

- [ ] **Step 3: 关键函数存在检查**

```bash
grep -n "function renderSetTable\|function getLastExerciseData\|function openKeyboard\|function closeKeyboard\|function handleKBInput\|function showTypePopup\|function setSetCount" "E:/Claude Code File/Fitness File/fitness.html"
```
预期: 7 个函数全部存在

- [ ] **Step 4: 在浏览器中验证**

打开 fitness.html：
1. 点击"添加训练" → 选择动作 → 确认五列表格显示
2. 点击"第1组"标签 → 弹出类型菜单 → 切换到热身组 → 显示"热"
3. 点击重量/次数单元格 → 自定义键盘弹出 → 系统键盘不弹出
4. 键盘 +/- 功能：重量 ±0.25，次数 ±1
5. 点击 ✓ → 该行变灰
6. 编辑已有记录 → 自动填充历史值
7. 填写数据 → 提交 → 列表中正确显示

- [ ] **Step 5: Commit（如有修复）**

```bash
git add fitness.html && git commit -m "chore: final verification fixes"
```
