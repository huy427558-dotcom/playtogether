# playtogether
Lật thẻ
<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Memory Flip - Tùy chỉnh hàng/cột + đặt thẻ theo ý</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; margin: 0; padding: 16px; }
    h1 { margin: 0 0 8px; font-size: 18px; }
    .wrap { display: grid; grid-template-columns: 1fr; gap: 12px; max-width: 900px; margin: 0 auto; }
    .panel { border: 1px solid #ddd; border-radius: 12px; padding: 12px; }
    .controls { display: flex; flex-wrap: wrap; gap: 8px; align-items: center; }
    .controls input { width: 84px; padding: 6px 8px; border: 1px solid #ccc; border-radius: 8px; }
    .controls button { padding: 8px 10px; border: 1px solid #ccc; border-radius: 10px; background: #fff; cursor: pointer; }
    .controls button:hover { background: #f6f6f6; }
    #status { margin-top: 8px; }
    #board {
      display: grid;
      gap: 10px;
      justify-content: start;
      align-content: start;
      padding: 8px;
    }
    .card {
      width: 72px; height: 72px;
      border-radius: 14px;
      border: 1px solid #cfcfcf;
      background: #ffffff;
      font-size: 28px;
      cursor: pointer;
      display: grid;
      place-items: center;
      user-select: none;
    }
    .card.revealed { background: #f5faff; border-color: #9ac7ff; }
    .card.matched  { background: #f0fff4; border-color: #86efac; cursor: default; }
    textarea { width: 100%; min-height: 160px; padding: 10px; border-radius: 10px; border: 1px solid #ccc; font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }
    #log { white-space: pre-wrap; background: #0b1020; color: #e6edf3; border-radius: 12px; padding: 10px; min-height: 140px; overflow: auto; }
    .hint { color: #555; font-size: 13px; margin-top: 6px; }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="panel">
      <h1>Memory Flip (Web) — chọn hàng/cột + đặt thẻ theo ý</h1>

      <div class="controls">
        <label>Rows <input id="rows" type="number" min="1" value="4"></label>
        <label>Cols <input id="cols" type="number" min="1" value="4"></label>
        <button id="applySize">Apply size</button>
        <button id="newGame">New game</button>
        <button id="clearLog">Xóa nhật ký</button>
      </div>

      <div id="status">Lật 2 thẻ để tìm cặp. Nhật ký bên dưới sẽ giúp bạn nhớ vị trí.</div>
      <div id="board"></div>
    </div>

    <div class="panel">
      <h2 style="margin:0 0 8px; font-size:16px;">Layout (đặt thẻ theo vị trí bạn muốn)</h2>
      <div class="hint">
        - Mỗi ô là 1 thẻ. Thẻ giống nhau phải xuất hiện đúng 2 lần. <br/>
        - Bạn có thể dùng emoji hoặc chữ/số. <br/>
        - Để trống ô sẽ báo lỗi. Kích thước phải đúng Rows × Cols.
      </div>
      <textarea id="layout"></textarea>
      <div class="controls" style="margin-top:8px;">
        <button id="applyLayout">Apply layout</button>
        <button id="fillRandom">Random deck</button>
      </div>
    </div>

    <div class="panel">
      <h2 style="margin:0 0 8px; font-size:16px;">Nhật ký ghi nhớ (thẻ → vị trí)</h2>
      <div id="log">Nhật ký:\n</div>
    </div>
  </div>

<script>
(() => {
  const HIDE = "❓";

  const $ = (id) => document.getElementById(id);
  const boardEl = $("board");
  const statusEl = $("status");
  const logEl = $("log");

  let ROWS = 4, COLS = 4;
  let board = [];        // 2D array [r][c] = symbol
  let revealed = new Set(); // "r,c" for currently revealed (not matched)
  let matched  = new Set(); // "r,c" for matched
  let firstPick = null;  // {r,c}
  let lock = false;

  function key(r,c){ return `${r},${c}`; }

  function logLine(line) {
    logEl.textContent += line + "\n";
    logEl.scrollTop = logEl.scrollHeight;
  }

  function clearLog() {
    logEl.textContent = "Nhật ký:\n";
  }

  function setStatus(msg) {
    statusEl.textContent = msg;
  }

  function defaultSymbols(pairs) {
    const base = ["🍎","🍌","🍇","🍓","🍒","🥝","🍍","🥑","🍑","🍉","🥕","🌽","🍪","🍩","🍫","🧁","🍔","🍕","🌮","🍜","🍣","🍙","🥨","🧀"];
    if (pairs <= base.length) return base.slice(0, pairs);
    return Array.from({length:pairs}, (_,i)=> String(i+1));
  }

  function shuffle(arr) {
    for (let i = arr.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
    return arr;
  }

  function buildRandomBoard() {
    const total = ROWS * COLS;
    if (total % 2 !== 0) throw new Error("Rows*Cols phải là số chẵn để tạo cặp.");
    const pairs = total / 2;
    const syms = defaultSymbols(pairs);
    const deck = shuffle([...syms, ...syms]);
    board = [];
    let idx = 0;
    for (let r=0; r<ROWS; r++) {
      const row = [];
      for (let c=0; c<COLS; c++) row.push(deck[idx++]);
      board.push(row);
    }
  }

  function parseLayoutText(text) {
    // Layout format: each row is a line, cells separated by spaces.
    // Example:
    // 🍎 🍌 🍎 🍌
    // 🍇 🍓 🍇 🍓
    const lines = text.trim().split(/\n+/).map(s => s.trim()).filter(Boolean);
    if (lines.length !== ROWS) throw new Error(`Layout phải có đúng ${ROWS} dòng (rows).`);
    const parsed = lines.map(line => line.split(/\s+/));
    for (let r=0; r<ROWS; r++) {
      if (parsed[r].length !== COLS) {
        throw new Error(`Dòng ${r+1} phải có đúng ${COLS} ô (cols).`);
      }
      for (let c=0; c<COLS; c++) {
        if (!parsed[r][c]) throw new Error(`Ô (${r+1},${c+1}) bị trống.`);
      }
    }

    // Validate pairs: each symbol appears exactly 2 times
    const freq = new Map();
    for (let r=0; r<ROWS; r++) {
      for (let c=0; c<COLS; c++) {
        const s = parsed[r][c];
        freq.set(s, (freq.get(s)||0) + 1);
      }
    }
    for (const [sym, count] of freq.entries()) {
      if (count !== 2) throw new Error(`Thẻ "${sym}" xuất hiện ${count} lần (phải đúng 2).`);
    }
    return parsed;
  }

  function renderBoard() {
    boardEl.style.gridTemplateColumns = `repeat(${COLS}, 72px)`;
    boardEl.innerHTML = "";

    for (let r=0; r<ROWS; r++) {
      for (let c=0; c<COLS; c++) {
        const btn = document.createElement("button");
        btn.className = "card";
        btn.textContent = HIDE;
        btn.addEventListener("click", () => onCardClick(r,c,btn));
        btn.dataset.r = r;
        btn.dataset.c = c;
        boardEl.appendChild(btn);
      }
    }
  }

  function resetState() {
    revealed.clear();
    matched.clear();
    firstPick = null;
    lock = false;
  }

  function updateCardVisual(r,c) {
    const btn = [...boardEl.children].find(el => el.dataset.r == r && el.dataset.c == c);
    if (!btn) return;
    const k = key(r,c);
    if (matched.has(k)) {
      btn.classList.add("matched");
      btn.classList.remove("revealed");
      btn.textContent = board[r][c];
      btn.disabled = true;
    } else if (revealed.has(k)) {
      btn.classList.add("revealed");
      btn.textContent = board[r][c];
      btn.disabled = true;
    } else {
      btn.classList.remove("revealed");
      btn.textContent = HIDE;
      btn.disabled = false;
    }
  }

  function reveal(r,c) {
    revealed.add(key(r,c));
    updateCardVisual(r,c);
    logLine(`${board[r][c]}  -> (hàng ${r+1}, cột ${c+1})`);
  }

  function hide(r,c) {
    revealed.delete(key(r,c));
    updateCardVisual(r,c);
  }

  function markMatched(r,c) {
    const k = key(r,c);
    matched.add(k);
    revealed.delete(k);
    updateCardVisual(r,c);
  }

  function onCardClick(r,c) {
    if (lock) return;
    const k = key(r,c);
    if (matched.has(k)) return;
    if (revealed.has(k)) return;

    reveal(r,c);

    if (!firstPick) {
      firstPick = {r,c};
      setStatus(`Đã lật 1 thẻ ở (hàng ${r+1}, cột ${c+1}). Lật thẻ thứ 2!`);
      return;
    }

    // second pick
    lock = true;
    const a = firstPick;
    const b = {r,c};
    firstPick = null;

    setTimeout(() => {
      const symA = board[a.r][a.c];
      const symB = board[b.r][b.c];

      if (symA === symB) {
        markMatched(a.r,a.c);
        markMatched(b.r,b.c);
        setStatus("✅ Trùng cặp! Giữ mở.");
      } else {
        hide(a.r,a.c);
        hide(b.r,b.c);
        setStatus("❌ Không trùng! Úp lại và nhớ vị trí nhé.");
      }

      lock = false;

      if (matched.size === ROWS * COLS) {
        setStatus("🎉 Bạn đã hoàn thành! Bấm New game để chơi lại.");
      }
    }, 550);
  }

  function setLayoutTextFromBoard() {
    const lines = board.map(row => row.join(" "));
    $("layout").value = lines.join("\n");
  }

  function newGameFromCurrentLayoutOrRandom(useLayout) {
    resetState();
    renderBoard();
    clearLog();

    try {
      if (useLayout) {
        const parsed = parseLayoutText($("layout").value);
        board = parsed;
      } else {
        buildRandomBoard();
        setLayoutTextFromBoard();
      }
      setStatus("Game mới! Lật 2 thẻ để tìm cặp!");
    } catch (e) {
      setStatus("⚠️ Lỗi layout: " + e.message);
      // fallback: random so you still can play
      try {
        buildRandomBoard();
        setLayoutTextFromBoard();
        setStatus("⚠️ Layout lỗi nên mình tạo random để bạn chơi trước. Sửa layout rồi bấm Apply layout.");
      } catch (_) {}
    }
  }

  function applySize() {
    const r = parseInt($("rows").value, 10);
    const c = parseInt($("cols").value, 10);
    if (!Number.isFinite(r) || !Number.isFinite(c) || r < 1 || c < 1) {
      setStatus("⚠️ Rows/Cols không hợp lệ.");
      return;
    }
    if ((r*c) % 2 !== 0) {
      setStatus("⚠️ Rows*Cols phải là số chẵn để chơi lật cặp.");
      return;
    }
    ROWS = r; COLS = c;
    newGameFromCurrentLayoutOrRandom(false); // random after resizing
  }

  // Buttons
  $("applySize").addEventListener("click", applySize);
  $("newGame").addEventListener("click", () => newGameFromCurrentLayoutOrRandom(true));
  $("clearLog").addEventListener("click", clearLog);
  $("applyLayout").addEventListener("click", () => newGameFromCurrentLayoutOrRandom(true));
  $("fillRandom").addEventListener("click", () => newGameFromCurrentLayoutOrRandom(false)p);

  // init
  buildRandomBoard();
  renderBoard();
  setLayoutTextFromBoard();
  setStatus("Sẵn sàng! Bạn có thể sửa Layout rồi bấm Apply layout.");
})();
</script>
</body>
</html>
