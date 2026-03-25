# KIRO TASK: Chess Game — 5 Feature Upgrades
# File: chess.html (single-file, no external dependencies)
# Execute all 5 features in sequence. Do not skip any.

---

## CONTEXT

You are upgrading an existing single-file chess game (chess.html).
The file uses:
- Vanilla JS (ES6+), no frameworks
- Global state variables: gBoard, gTurn, gCR, gEP, gOver, gSel, gLeg, gLast, gSnaps, gHist, gCapW, gCapB, gAI
- Piece constants: WP=1,WN=2,WB=3,WR=4,WQ=5,WK=6 / BP=7,BN=8,BB=9,BR=10,BQ=11,BK=12
- Core functions: legalOf(), attacked(), applyMove(), minimax(), bestAI(), doMove(), render(), updateUI()
- All CSS in <style> tag, all JS in <script> tag before </body>

Read the existing file fully before making any changes.
Preserve all existing functionality — do not break anything.

---

## FEATURE 1: ANIMATED PIECE MOVEMENT

### Goal
Pieces glide smoothly from source to target square (200ms).
Captured pieces fly to the captured panel.

### Implementation

1. Add a floating <div id="anim-piece"> to the DOM (position:fixed, z-index:50, pointer-events:none)

2. Create function `animateMove(fromSq, toSq, pieceGlyph, callback)`:
   - Get bounding rect of both squares using document.querySelector(`[data-sq="${fromSq}"]`).getBoundingClientRect()
   - Set anim-piece innerHTML = pieceGlyph, position it at the fromSq rect coordinates
   - Use CSS transition: `transform 200ms cubic-bezier(0.25, 0.46, 0.45, 0.94)`
   - Apply translateX/Y to move to toSq coordinates
   - On transitionend: hide anim-piece, call callback()

3. Wrap both human doMove() and AI doMove() calls:
   - Instead of calling doMove() then render(), call:
     animateMove(m.fr, m.to, GLYPH[gBoard[m.fr]], () => { doMove(m); render(); updateUI(); checkEnd(); })
   - The board must NOT re-render until the animation completes (callback fires)

4. For captures: after animation ends, add a 150ms "pop" animation on the captured pieces panel
   - Add CSS: @keyframes capPop { 0%{transform:scale(0)} 60%{transform:scale(1.3)} 100%{transform:scale(1)} }
   - Apply to the last child of #cap-b or #cap-w after it is appended

5. CSS for #anim-piece:
   position: fixed;
   font-size: 44px;
   line-height: 1;
   pointer-events: none;
   z-index: 50;
   display: none;
   will-change: transform;
   text-shadow: 2px 2px 8px rgba(0,0,0,0.5);

### Acceptance Criteria
- Piece visually slides across the board, does not teleport
- Captured piece "pops" into captured panel
- Clicking another piece during animation is ignored (guard with isAnimating flag)
- Animation works for castling (animate king only, rook snaps instantly)

---

## FEATURE 2: AI PERSONALITY SYSTEM

### Goal
Three named AI personalities with distinct playing styles, each with an opening book.
User selects personality via a dropdown in the Controls panel.

### Implementation

1. Add a <select id="ai-personality"> to the Controls pbox, above the depth slider:
   ```html
   <div style="display:flex;flex-direction:column;gap:5px">
     <div style="font-size:.67rem;color:var(--muted);letter-spacing:.1em">AI Personality</div>
     <select id="ai-personality" style="background:var(--panel);color:var(--text);border:1px solid var(--border);border-radius:4px;padding:6px 8px;font-family:'Source Code Pro',monospace;font-size:.7rem;cursor:pointer">
       <option value="aggressor">⚔ The Aggressor</option>
       <option value="defender">🛡 The Defender</option>
       <option value="gambler">🎲 The Gambler</option>
     </select>
   </div>
   ```

2. Define opening books (array of move strings in UCI format "e2e4"):
   ```js
   const OPENING_BOOKS = {
     aggressor: [
       // Sicilian-attacking lines (King's Indian Attack style)
       ['e2e4','e7e5','g1f3','b8c6','f1c4'],  // Italian Game
       ['e2e4','c7c5','g1f3','d7d6','d2d4'],  // Open Sicilian
       ['e2e4','e7e6','d2d4','d7d5','e4e5'],  // French Advance
     ],
     defender: [
       // Solid positional lines
       ['d2d4','d7d5','c2c4','e7e6','g1f3'],  // Queen's Gambit
       ['d2d4','g8f6','c2c4','e7e6','g2g3'],  // Catalan
       ['c2c4','e7e5','g2g3','g8f6','f1g2'],  // English Opening
     ],
     gambler: [
       // Sharp tactical lines
       ['e2e4','e7e5','f2f4'],                 // King's Gambit
       ['e2e4','e7e5','g1f3','g8f6'],          // Petroff (gambler plays actively)
       ['d2d4','d7d5','c2c4','d5c4','e2e4'],   // Queen's Gambit Accepted
     ]
   };
   ```

3. Add `let currentOpeningLine = null; let openingMoveIndex = 0;` to state.

4. On newGame(): pick a random opening line from the selected personality's book.
   `currentOpeningLine = randomChoice(OPENING_BOOKS[personality]); openingMoveIndex = 0;`

5. In bestAI() (or the triggerAI wrapper): 
   - If openingMoveIndex < currentOpeningLine.length, parse the next UCI move string into a move object, validate it exists in legalOf(), and play it. Increment openingMoveIndex.
   - If the book move is not legal (position deviated), set currentOpeningLine = null and fall through to minimax.

6. Define personality evaluation BIAS applied on top of the base evaluate() function:
   ```js
   const PERSONALITY_BIAS = {
     aggressor: (b) => {
       // Bonus for pieces near enemy king, penalty for undeveloped pieces
       let score = 0;
       const enemyKing = kingOf(b, WHITE);
       const ekr = R(enemyKing), ekf = F(enemyKing);
       for(let i=0;i<64;i++) {
         if(isB(b[i]) && b[i]!==BK) {
           const dist = Math.max(Math.abs(R(i)-ekr), Math.abs(F(i)-ekf));
           score += Math.max(0, (4-dist)) * 8;
         }
       }
       return score;
     },
     defender: (b) => {
       // Bonus for pawn structure (connected pawns, protected king)
       let score = 0;
       for(let i=0;i<64;i++) {
         if(b[i]===BP) {
           if(F(i)>0 && b[i-1]===BP) score+=15;  // connected pawns
           if(F(i)<7 && b[i+1]===BP) score+=15;
         }
       }
       return score;
     },
     gambler: (b) => {
       // Bonus for pawn advances, bonus for piece activity (legal moves count)
       let score = 0;
       for(let i=0;i<64;i++) {
         if(b[i]===BP) score += (R(i)-1) * 12;  // reward advanced pawns
       }
       return score + (Math.random() * 20 - 10);  // slight randomness for unpredictability
     }
   };
   ```

7. Modify evaluate(b, color) to add the personality bias:
   ```js
   const personality = document.getElementById('ai-personality').value;
   const bias = PERSONALITY_BIAS[personality] ? PERSONALITY_BIAS[personality](b) : 0;
   // bias is from Black's perspective, so subtract for WHITE evaluation
   return color===WHITE ? score - bias : -score + bias;
   ```

8. Show personality name in the status panel when AI is thinking:
   `"⚔ Aggressor thinking…"` / `"🛡 Defender thinking…"` / `"🎲 Gambler thinking…"`

### Acceptance Criteria
- Dropdown visible in Controls panel
- Opening book plays correctly for first N moves
- Personality visibly affects AI playstyle
- Changing personality only takes effect on New Game

---

## FEATURE 3: THREAT VISUALISATION HEATMAP

### Goal
A toggle button overlays a colour heatmap on the board:
- Squares attacked by White glow blue
- Squares attacked by Black glow red
- Squares attacked by both glow purple
- Intensity scales with number of attackers

### Implementation

1. Add toggle button to Controls pbox:
   ```html
   <button class="btn dim" id="heatmap-btn" onclick="toggleHeatmap()">Show Threats</button>
   ```

2. Add state: `let heatmapOn = false;`

3. Define toggleHeatmap():
   ```js
   function toggleHeatmap() {
     heatmapOn = !heatmapOn;
     const btn = document.getElementById('heatmap-btn');
     btn.textContent = heatmapOn ? 'Hide Threats' : 'Show Threats';
     btn.classList.toggle('btn', true);
     btn.classList.toggle('dim', !heatmapOn);
     render();
   }
   ```

4. Define computeHeatmap(b):
   Returns an object `{ white: Float32Array(64), black: Float32Array(64) }`
   ```js
   function computeHeatmap(b) {
     const wHeat = new Float32Array(64);
     const bHeat = new Float32Array(64);
     for(let s=0;s<64;s++) {
       // Count how many white pieces attack square s
       let wCount = 0, bCount = 0;
       for(let p=0;p<64;p++) {
         if(isW(b[p])) {
           // Check if piece at p attacks s using existing attacked logic per piece
           if(squareAttackedByPiece(b, s, p)) wCount++;
         }
         if(isB(b[p])) {
           if(squareAttackedByPiece(b, s, p)) bCount++;
         }
       }
       wHeat[s] = Math.min(wCount / 3, 1);  // normalise 0-1
       bHeat[s] = Math.min(bCount / 3, 1);
     }
     return { white: wHeat, black: bHeat };
   }
   ```

5. Implement squareAttackedByPiece(b, targetSq, pieceSq):
   Returns true if the piece at pieceSq can attack targetSq.
   Use the same ray-casting logic as the existing `attacked()` function but for individual pieces.
   Tip: generate pseudo-legal moves for only the piece at pieceSq, check if any move.to === targetSq.
   ```js
   function squareAttackedByPiece(b, targetSq, pieceSq) {
     const p = b[pieceSq];
     if(!p) return false;
     const color = pcol(p);
     // Generate moves for this single piece (no ep/castling needed for attack detection)
     const singleBoard = new Array(64).fill(EMPTY);
     singleBoard[pieceSq] = p;
     // Place a dummy enemy piece at target to allow captures
     singleBoard[targetSq] = color===WHITE ? BP : WP;
     return pseudoLegal(singleBoard, color, -1, {wK:false,wQ:false,bK:false,bQ:false})
       .some(m => m.fr===pieceSq && m.to===targetSq);
   }
   ```

6. In render(), after building each square div, if heatmapOn:
   ```js
   if(heatmapOn && heatData) {
     const w = heatData.white[s], bk = heatData.black[s];
     if(w > 0 || bk > 0) {
       let r=0,g=0,b=0,a=0;
       if(w > 0 && bk > 0) {
         // Both attack: purple
         r = Math.round(150 * Math.max(w,bk));
         b = Math.round(200 * Math.max(w,bk));
         a = Math.max(w,bk) * 0.55;
       } else if(w > 0) {
         // White attacks: blue
         b = 220; r = 60;
         a = w * 0.5;
       } else {
         // Black attacks: red
         r = 220; b = 40;
         a = bk * 0.5;
       }
       d.style.boxShadow = `inset 0 0 0 9999px rgba(${r},${g},${b},${a})`;
     }
   }
   ```

7. Compute heatData once per render call at the top of render():
   `const heatData = heatmapOn ? computeHeatmap(gBoard) : null;`

8. Add small legend below the board (only visible when heatmapOn):
   ```html
   <div id="heatmap-legend" style="display:none;gap:12px;font-size:.62rem;color:var(--muted);padding:6px 0;justify-content:center;letter-spacing:.1em">
     <span>🔵 White attacks</span>
     <span>🔴 Black attacks</span>
     <span>🟣 Contested</span>
   </div>
   ```
   Toggle its display in toggleHeatmap().

### Acceptance Criteria
- Toggle button visible in Controls panel
- Heatmap overlays correctly coloured tint per square
- Tint intensity increases with number of attackers
- Heatmap updates after every move
- Legend visible when heatmap is on
- Does not interfere with legal move dots, check highlight, or selection highlight

---

## FEATURE 4: PAWN PROMOTION DIALOG

### Goal
When a pawn reaches the back rank, show a modal letting the player choose:
Queen, Rook, Bishop, or Knight.
AI always auto-promotes to Queen (no dialog).

### Implementation

1. Add promotion modal HTML (inside body, alongside existing #overlay):
   ```html
   <div id="promo-overlay" style="display:none;position:fixed;inset:0;background:rgba(0,0,0,.78);z-index:200;align-items:center;justify-content:center;">
     <div style="background:var(--panel);border:1px solid var(--accent);border-radius:10px;padding:28px 36px;text-align:center;box-shadow:0 16px 60px rgba(0,0,0,.85)">
       <div style="font-family:'Playfair Display',serif;font-size:1.3rem;color:var(--accent);margin-bottom:6px">Pawn Promotion</div>
       <div style="font-size:.72rem;color:var(--muted);margin-bottom:20px;letter-spacing:.08em">Choose your piece</div>
       <div id="promo-choices" style="display:flex;gap:12px;justify-content:center"></div>
     </div>
   </div>
   ```

2. Add state: `let pendingPromoMove = null; let pendingPromoCallback = null;`

3. Detect promotion moves in handleClick():
   When a legal move is selected where:
   - `gBoard[m.fr] === WP` AND `R(m.to) === 0` (white pawn reaching rank 8)
   - Currently `m.promo` is set to WQ automatically

   Instead: call `showPromoDialog(m, callback)` and pause execution.

4. Implement showPromoDialog(m, callback):
   ```js
   function showPromoDialog(m, callback) {
     pendingPromoMove = m;
     pendingPromoCallback = callback;
     const choices = document.getElementById('promo-choices');
     choices.innerHTML = '';
     const pieces = [
       {p: WQ, g: '♕', label: 'Queen'},
       {p: WR, g: '♖', label: 'Rook'},
       {p: WB, g: '♗', label: 'Bishop'},
       {p: WN, g: '♘', label: 'Knight'}
     ];
     pieces.forEach(({p, g, label}) => {
       const btn = document.createElement('div');
       btn.style.cssText = `
         width:64px;height:72px;display:flex;flex-direction:column;align-items:center;
         justify-content:center;gap:4px;cursor:pointer;border:1px solid var(--border);
         border-radius:8px;transition:border-color .15s,background .15s;
         font-size:36px;background:var(--panel);
       `;
       btn.innerHTML = `<span>${g}</span><span style="font-size:.6rem;color:var(--muted);letter-spacing:.08em">${label}</span>`;
       btn.onmouseenter = () => btn.style.borderColor = 'var(--accent)';
       btn.onmouseleave = () => btn.style.borderColor = 'var(--border)';
       btn.onclick = () => selectPromo(p);
       choices.appendChild(btn);
     });
     document.getElementById('promo-overlay').style.display = 'flex';
   }
   ```

5. Implement selectPromo(pieceType):
   ```js
   function selectPromo(pieceType) {
     document.getElementById('promo-overlay').style.display = 'none';
     if(pendingPromoMove && pendingPromoCallback) {
       pendingPromoMove.promo = pieceType;
       pendingPromoCallback(pendingPromoMove);
     }
     pendingPromoMove = null;
     pendingPromoCallback = null;
   }
   ```

6. Refactor handleClick() to pass a callback:
   ```js
   const executeMove = (m) => {
     gSnaps.push(snapShot());
     doMove(m);
     gSel=-1; gLeg=[];
     render(); updateUI(); checkEnd();
     if(!gOver) triggerAI();
   };

   if(gBoard[m.fr]===WP && R(m.to)===0) {
     showPromoDialog(m, executeMove);
   } else {
     executeMove(m);
   }
   ```

7. AI promotion: in doMove(), when AI moves a BP to rank 7 (row index 7), the promo is already set to BQ. Keep that behaviour — AI always picks Queen.

8. Handle Escape key to dismiss (re-select piece instead):
   ```js
   document.addEventListener('keydown', e => {
     if(e.key === 'Escape' && pendingPromoMove) {
       document.getElementById('promo-overlay').style.display = 'none';
       pendingPromoMove = null; pendingPromoCallback = null;
       gSel=-1; gLeg=[]; render();
     }
   });
   ```

### Acceptance Criteria
- Dialog appears ONLY for human player (White) pawn reaching rank 8
- All four pieces shown with Unicode glyphs and labels
- Clicking a piece promotes correctly, game continues
- AI still auto-promotes to Queen without showing dialog
- Escape dismisses dialog and deselects the pawn
- Undo works correctly after promotion

---

## FEATURE 5: GAME EXPORT — PGN + CLIPBOARD

### Goal
A "Copy PGN" button generates a complete, valid PGN string and copies it to clipboard.
A success message confirms the copy.
Players can paste the PGN into Lichess/Chess.com for analysis.

### Implementation

1. Add "Copy PGN" button at the bottom of the move history panel (inside .right-panel .pbox):
   ```html
   <div style="margin-top:10px;border-top:1px solid var(--border);padding-top:10px;display:flex;flex-direction:column;gap:6px">
     <button class="btn dim" id="pgn-btn" onclick="copyPGN()">Copy PGN</button>
     <div id="pgn-status" style="font-size:.62rem;color:var(--accent);text-align:center;min-height:14px;letter-spacing:.08em"></div>
   </div>
   ```

2. Implement generatePGN():
   ```js
   function generatePGN() {
     const date = new Date();
     const dateStr = `${date.getFullYear()}.${String(date.getMonth()+1).padStart(2,'0')}.${String(date.getDate()).padStart(2,'0')}`;
     
     // Determine result
     let result = '*';
     if(gOver) {
       const legals = legalOf(gBoard, gTurn, gEP, gCR);
       if(!legals.length) {
         const ks = kingOf(gBoard, gTurn);
         const inCheck = attacked(gBoard, ks, opp(gTurn));
         if(inCheck) result = gTurn===WHITE ? '0-1' : '1-0';
         else result = '1/2-1/2';
       }
     }

     // PGN headers
     const personality = document.getElementById('ai-personality').value;
     const personalityNames = { aggressor:'The Aggressor', defender:'The Defender', gambler:'The Gambler' };
     const headers = [
       `[Event "Chess vs AI"]`,
       `[Site "chess.html"]`,
       `[Date "${dateStr}"]`,
       `[Round "1"]`,
       `[White "Human"]`,
       `[Black "AI (${personalityNames[personality]||'AI'})"]`,
       `[Result "${result}"]`,
       `[Termination "Normal"]`
     ].join('\n');

     // Move text (wrap at 80 chars per PGN spec)
     let moveText = '';
     for(let i=0;i<gHist.length;i++) {
       if(i%2===0) moveText += `${Math.floor(i/2)+1}. `;
       moveText += gHist[i] + ' ';
     }
     moveText += result;

     // Word-wrap at 80 chars
     const lines = [];
     let line = '';
     for(const word of moveText.split(' ')) {
       if(!word) continue;
       if((line + ' ' + word).length > 80 && line) {
         lines.push(line.trim());
         line = word;
       } else {
         line += (line ? ' ' : '') + word;
       }
     }
     if(line) lines.push(line.trim());

     return headers + '\n\n' + lines.join('\n');
   }
   ```

3. Implement copyPGN():
   ```js
   function copyPGN() {
     if(gHist.length === 0) {
       showPGNStatus('No moves to export');
       return;
     }
     const pgn = generatePGN();
     navigator.clipboard.writeText(pgn).then(() => {
       showPGNStatus('✓ PGN copied!');
       setTimeout(() => showPGNStatus(''), 3000);
     }).catch(() => {
       // Fallback for browsers without clipboard API
       const ta = document.createElement('textarea');
       ta.value = pgn;
       ta.style.position = 'fixed';
       ta.style.opacity = '0';
       document.body.appendChild(ta);
       ta.select();
       document.execCommand('copy');
       document.body.removeChild(ta);
       showPGNStatus('✓ PGN copied!');
       setTimeout(() => showPGNStatus(''), 3000);
     });
   }

   function showPGNStatus(msg) {
     document.getElementById('pgn-status').textContent = msg;
   }
   ```

4. Also add a "View PGN" expandable section:
   - Small clickable "▶ View raw PGN" text below the Copy button
   - On click, reveals a <pre> or <textarea> containing the PGN text (readonly)
   - Allows users to manually copy if clipboard API fails
   ```js
   function togglePGNView() {
     const box = document.getElementById('pgn-raw-box');
     const btn = document.getElementById('pgn-view-btn');
     if(box.style.display==='none') {
       box.style.display='block';
       box.value = generatePGN();
       btn.textContent = '▼ Hide PGN';
     } else {
       box.style.display='none';
       btn.textContent = '▶ View PGN';
     }
   }
   ```
   ```html
   <button class="btn dim" id="pgn-view-btn" onclick="togglePGNView()" style="font-size:.62rem;padding:5px 8px">▶ View PGN</button>
   <textarea id="pgn-raw-box" readonly style="display:none;width:100%;height:120px;background:var(--bg);color:var(--muted);border:1px solid var(--border);border-radius:4px;padding:8px;font-family:'Source Code Pro',monospace;font-size:.6rem;line-height:1.6;resize:none;margin-top:4px"></textarea>
   ```

5. Update PGN view automatically whenever updateUI() is called (if the view is open):
   ```js
   const pgnBox = document.getElementById('pgn-raw-box');
   if(pgnBox && pgnBox.style.display!=='none') pgnBox.value = generatePGN();
   ```

### Acceptance Criteria
- Button visible at bottom of move history panel
- PGN includes all 7 required headers (Event, Site, Date, Round, White, Black, Result)
- Move notation matches the move history panel
- Result correctly shows 1-0, 0-1, 1/2-1/2, or * (ongoing)
- Clipboard copy confirmed with ✓ message that disappears after 3s
- Works in Chrome, Firefox, and Safari
- View PGN textarea shows correct PGN and updates after each move

---

## FINAL INTEGRATION CHECKLIST

After implementing all 5 features, verify:

[ ] All 5 features work independently and together
[ ] Existing game logic (castling, en passant, promotion, check, checkmate, stalemate) still works
[ ] New Game button resets ALL new state (opening book index, personality selection persists but book resets)
[ ] Undo Move correctly reverses animated moves
[ ] heatmapOn, ai-personality, and depth slider values persist across New Game
[ ] No console errors on startup
[ ] AI depth slider still works at all 5 levels
[ ] File remains a single HTML file — no external scripts, no CDN links

## CODE QUALITY RULES
- Add concise inline comments above each new function explaining its purpose
- Keep all new CSS inside the existing <style> tag
- Keep all new JS inside the existing <script> tag
- Do not introduce any global variable name conflicts
- Prefer const/let — no var
- All new DOM IDs must be kebab-case (e.g. promo-overlay, pgn-status)
- Test edge cases: promotion on capture, promotion with check, AI castling animation

## OUTPUT
Return the complete updated chess.html file.
No explanation needed — only the file.