\<\!doctype html\>  
\<html lang\="en"\>  
\<head\>  
  \<meta charset\="UTF-8" /\>  
  \<meta name\="viewport" content\="width=device-width, initial-scale=1.0" /\>  
  \<title\>Chorus Lapilli — Counter-System Simulator\</title\>  
  \<script src\="https://cdn.tailwindcss.com"\>\</script\>  
  \<script\>  
    tailwind.config \= {  
      theme: {  
        extend: {  
          colors: {  
            deep:    '\#050b1a',  
            panel:   '\#0a1228',  
            neon:    '\#00f5ff',  
            neonSoft:'\#63f7ff',  
          },  
          animation: {  
            pulse2: 'pulse2 1.1s ease-in-out infinite',  
            fadeIn: 'fadeIn 0.3s ease-out',  
          },  
          keyframes: {  
            pulse2:  { '0%,100%': { opacity: 1 }, '50%': { opacity: 0.45 } },  
            fadeIn:  { from: { opacity: 0, transform: 'translateY(6px)' }, to: { opacity: 1, transform: 'none' } },  
          },  
        },  
      },  
    };  
  \</script\>  
  \<style\>  
    /\* ── glow helpers ─────────────────────── \*/  
    .cell-glow   { box-shadow: 0 0 .7rem rgba(0,245,255,.5), inset 0 0 .5rem rgba(0,245,255,.15); }  
    .ring-suggest{ box-shadow: 0 0 0 3px \#22c55e, 0 0 14px 2px \#16a34a88; }  
    .ring-warn   { box-shadow: 0 0 0 3px \#ef4444, 0 0 14px 2px \#dc262688; }  
    .ring-trap   { box-shadow: 0 0 0 3px \#f97316, 0 0 14px 2px \#ea580c88; }  
    .ring-vanish { box-shadow: 0 0 0 2px \#fbbf24; animation: pulse2 1.1s ease-in-out infinite; }  
    .ring-oold   { box-shadow: 0 0 0 2px \#f87171; animation: pulse2 1.1s ease-in-out infinite; }  
    .ring-win    { box-shadow: 0 0 0 3px \#84cc16, 0 0 18px 4px \#65a30d99; }  
    .thinking-dot{ animation: pulse2 .8s ease-in-out infinite; }  
    /\* ── grid lines ───────────────────────── \*/  
    .board-cell  { border: 1px solid rgba(0,245,255,.18); }  
  \</style\>  
\</head\>  
\<body class\="min-h-screen bg-deep text-cyan-100 font-mono"\>  
  \<div id\="root"\>\</div\>

  \<script crossorigin src\="https://unpkg.com/react@18/umd/react.development.js"\>\</script\>  
  \<script crossorigin src\="https://unpkg.com/react-dom@18/umd/react-dom.development.js"\>\</script\>  
  \<script src\="https://unpkg.com/@babel/standalone/babel.min.js"\>\</script\>

  \<script type\="text/babel"\>  
  /\* ═══════════════════════════════════════════════════════════  
     CONSTANTS  
  ═══════════════════════════════════════════════════════════ \*/  
  const { useState, useEffect, useRef, useCallback } \= React;

  const O \= 'O'; // YOU — casino first-mover you are learning to beat  
  const X \= 'X'; // AI coach — optimal counter-system

  const WIN\_LINES \= \[  
    \[0,1,2\],\[3,4,5\],\[6,7,8\],  
    \[0,3,6\],\[1,4,7\],\[2,5,8\],  
    \[0,4,8\],\[2,4,6\],  
  \];

  // Square labels: row-major A1..C3  
  const SQ \= \['A1','A2','A3','B1','B2','B3','C1','C2','C3'\];

  /\* ═══════════════════════════════════════════════════════════  
     PURE GAME ENGINE  
  ═══════════════════════════════════════════════════════════ \*/  
  const getWinLine \= (board) \=\> {  
    for (const \[a,b,c\] of WIN\_LINES)  
      if (board\[a\] && board\[a\] \=== board\[b\] && board\[b\] \=== board\[c\])  
        return \[a,b,c\];  
    return null;  
  };

  const checkWinner \= (board) \=\> {  
    const ln \= getWinLine(board);  
    return ln ? board\[ln\[0\]\] : null;  
  };

  // Applies a move with the vanishing-oldest-piece mechanic.  
  // Returns a new immutable state object.  
  const applyMove \= (state, player, move) \=\> {  
    const board  \= \[...state.board\];  
    const queues \= { X: \[...state.queues.X\], O: \[...state.queues.O\] };  
    let removed  \= null;

    if (queues\[player\].length \=== 3) {  
      removed \= queues\[player\].shift();  
      board\[removed\] \= null;  
    }

    board\[move\] \= player;  
    queues\[player\].push(move);

    const winLine \= getWinLine(board);  
    return { board, queues, removed, winner: winLine ? board\[winLine\[0\]\] : null, winLine: winLine || \[\] };  
  };

  // Queue-aware legal moves: treats oldest cell as vacant when queue is full.  
  const legalMoves \= (state, player) \=\> {  
    const board \= \[...state.board\];  
    if (state.queues\[player\].length \=== 3) board\[state.queues\[player\]\[0\]\] \= null;  
    return board.map((v,i) \=\> v ? null : i).filter(i \=\> i \!== null);  
  };

  /\* ═══════════════════════════════════════════════════════════  
     EVALUATION — X-biased heuristic (X \= AI coach)  
  ═══════════════════════════════════════════════════════════ \*/  
  const evaluate \= (state) \=\> {  
    const { board } \= state;  
    let score \= 0;  
    for (const \[a,b,c\] of WIN\_LINES) {  
      const line \= \[board\[a\], board\[b\], board\[c\]\];  
      const xn   \= line.filter(v \=\> v \=== X).length;  
      const on   \= line.filter(v \=\> v \=== O).length;  
      // X threats  
      if (xn \=== 2 && on \=== 0) score \+= 8;  
      if (xn \=== 1 && on \=== 0) score \+= 1;  
      // O threats (penalised — X must avoid giving O winning lines)  
      if (on \=== 2 && xn \=== 0) score \-= 14;  
      if (on \=== 1 && xn \=== 0) score \-= 1.5;  
    }  
    // Center bonus  
    if (board\[4\] \=== X) score \+= 5;  
    if (board\[4\] \=== O) score \-= 4;  
    // Corner bonus  
    for (const c of \[0,2,6,8\]) {  
      if (board\[c\] \=== X) score \+= 1.5;  
      if (board\[c\] \=== O) score \-= 2;  
    }  
    // Queue expiry pressure: reward positions where O's oldest piece is about to vanish  
    if (state.queues\[O\].length \=== 3) {  
      const dying \= state.queues\[O\]\[0\];  
      const linesWithDying \= WIN\_LINES.filter(ln \=\> ln.includes(dying));  
      for (const \[a,b,c\] of linesWithDying) {  
        const on \= \[board\[a\],board\[b\],board\[c\]\].filter(v \=\> v \=== O).length;  
        if (on \>= 2) score \+= 4; // O is about to lose an anchor piece  
      }  
    }  
    return score;  
  };

  /\* ═══════════════════════════════════════════════════════════  
     TRANSPOSITION TABLE (module-level — persists across turns)  
  ═══════════════════════════════════════════════════════════ \*/  
  const TTABLE \= new Map();  
  const TTABLE\_LIMIT \= 8000;

  const boardKey \= (state, current) \=\> {  
    return state.board.map(v \=\> v ?? '-').join('') \+ '|' \+  
           state.queues.X.join(',') \+ '|' \+ state.queues.O.join(',') \+ '|' \+ current;  
  };

  /\* ═══════════════════════════════════════════════════════════  
     MINIMAX — X is maximiser, O is minimiser.  
     pathStates prevents infinite loops from the vanishing mechanic.  
  ═══════════════════════════════════════════════════════════ \*/  
  function minimax(state, current, depth, maxDepth, alpha, beta, pathStates) {  
    // Terminal checks  
    if (state.winner \=== X) return 120 \- depth;  
    if (state.winner \=== O) return depth \- 120;  
    if (depth \>= maxDepth) return evaluate(state);

    const cycleKey \= boardKey(state, current) \+ '|' \+ depth;  
    if (pathStates.has(cycleKey)) return 0;

    const ttKey \= boardKey(state, current) \+ '|' \+ depth \+ '|' \+ maxDepth;  
    if (TTABLE.has(ttKey)) return TTABLE.get(ttKey);

    const moves \= legalMoves(state, current);  
    if (\!moves.length) return 0;

    pathStates.add(cycleKey);  
    let best \= current \=== X ? \-Infinity : Infinity;

    for (const mv of moves) {  
      const next  \= applyMove(state, current, mv);  
      const score \= minimax(next, current \=== X ? O : X, depth \+ 1, maxDepth, alpha, beta, pathStates);

      if (current \=== X) {  
        if (score \> best) best \= score;  
        if (best \> alpha)  alpha \= best;  
      } else {  
        if (score \< best) best \= score;  
        if (best \< beta)   beta \= best;  
      }  
      if (beta \<= alpha) break; // prune  
    }

    pathStates.delete(cycleKey);

    TTABLE.set(ttKey, best);  
    if (TTABLE.size \> TTABLE\_LIMIT) TTABLE.delete(TTABLE.keys().next().value);

    return best;  
  }

  /\* ═══════════════════════════════════════════════════════════  
     PICK BEST MOVE FOR X (coach / counter-system)  
     difficulty: 'Easy'=depth 3, 'Medium'=depth 5, 'Hard'=depth 7  
  ═══════════════════════════════════════════════════════════ \*/  
  function pickBestX(state, difficulty) {  
  const depths \= { Easy: 3, Medium: 5, Hard: 7 };  
  const maxDepth \= depths\[difficulty\] ?? 7;

  const moves \= legalMoves(state, X);  
  if (\!moves.length) return null;

  // 1\) Immediate win — always take it  
  for (const mv of moves) {  
    const next \= applyMove(state, X, mv);  
    if (next.winner \=== X) {  
      return { move: mv, score: 120, isForcedWin: true };  
    }  
  }

  // 2\) Single sweep: score all moves ONCE  
  const scored \= moves.map(mv \=\> {  
    const next  \= applyMove(state, X, mv);  
    const score \= minimax(next, O, 0, maxDepth, \-Infinity, Infinity, new Set());  
    return { mv, sc: score };  
  });

  // Sort descending by score  
  scored.sort((a, b) \=\> b.sc \- a.sc);

  // 3\) Easy mode: occasionally pick 2nd-best (no extra minimax)  
  if (difficulty \=== 'Easy' && scored.length \> 1 && Math.random() \< 0.35) {  
    return { move: scored\[1\].mv, score: scored\[1\].sc, isSubOptimal: true };  
  }

  // 4\) Default: best move  
  return { move: scored\[0\].mv, score: scored\[0\].sc };  
}

  function pickAggressiveMove(state) {  
  const moves \= legalMoves(state, 'X');  
  if (\!moves.length) return null;

  // 1\. Immediate win  
  for (const mv of moves) {  
    const next \= applyMove(state, 'X', mv);  
    if (next.winner \=== 'X') {  
      return { move: mv, score: 120, type: 'win' };  
    }  
  }

  // 2\. Create fork (best exploit)  
  const forks \= findForkMoves(state);  
  if (forks.length) {  
    return { move: forks\[0\], score: 90, type: 'fork' };  
  }

  // 3\. Force pressure (create 2-in-row)  
  for (const mv of moves) {  
    const next \= applyMove(state, 'X', mv);

    for (const \[a,b,c\] of WIN\_LINES) {  
      const line \= \[next.board\[a\], next.board\[b\], next.board\[c\]\];  
      const xCount \= line.filter(v \=\> v \=== 'X').length;  
      const empty \= line.filter(v \=\> v \=== null).length;

      if (xCount \=== 2 && empty \=== 1) {  
        return { move: mv, score: 70, type: 'pressure' };  
      }  
    }  
  }

  // 4\. fallback to normal  
  return pickBestX(state, 'Hard');  
}

function pickSafeMove(state, difficulty) {  
  const moves \= legalMoves(state, 'X');  
  if (\!moves.length) return null;

  let best \= null;  
  let bestScore \= \-Infinity;

  for (const mv of moves) {  
    const next \= applyMove(state, 'X', mv);

    // If move immediately wins → always take it  
    if (next.winner \=== 'X') {  
      return { move: mv, score: 120, type: 'win' };  
    }

    const baseScore \= minimax(next, 'O', 0, 5, \-Infinity, Infinity, new Set());  
    const riskPenalty \= evaluateRiskyState(next);

    const finalScore \= baseScore \+ riskPenalty;

    if (finalScore \> bestScore) {  
      bestScore \= finalScore;  
      best \= mv;  
    }  
  }

  return { move: best, score: bestScore, type: 'safe' };  
}

function decideAdaptiveMove(state, difficulty, exploitMode, personality, rtpScore) {  
  if (\!personality) {  
  return pickSafeMove(state, difficulty);  
}  
  // EASY → always try to attack  
  if (personality && personality.includes('EASY')) {  
    return pickAggressiveMove(state);  
  }

  // MEDIUM → attack only if exploit window  
  if (personality && personality.includes('MEDIUM')) {  
  if (exploitMode) return pickAggressiveMove(state);

  if (rtpScore \> 0.18) return pickAggressiveMove(state);

  return pickSafeMove(state, difficulty);  
}

  // HARD → only attack on confirmed mistake  
  if (personality && personality.includes('HARD')) {  
  if (exploitMode) return pickAggressiveMove(state);

  // If no immediate O threat → prioritize safety  
  const threats \= findImmediateOThreats(state);

  if (threats.length \=== 0) {  
    return pickSafeMove(state, difficulty);  
  }

  return pickBestX(state, 'Hard');  
}

  // UNKNOWN → fallback  
  return exploitMode ? pickAggressiveMove(state) : pickBestX(state, difficulty);  
}

  function detectSubOptimalMove(prevState, currentState) {  
  // Find what move O actually made  
  let actualMove \= null;

  for (let i \= 0; i \< 9; i++) {  
    if (prevState.board\[i\] \!== currentState.board\[i\] && currentState.board\[i\] \=== 'O') {  
      actualMove \= i;  
      break;  
    }  
  }

  if (actualMove \=== null) return false;

  // Get all possible legal O moves before it played  
  const possibleMoves \= legalMoves(prevState, 'O');

  // Find "good" moves (moves that don’t lead to immediate loss)  
  const safeMoves \= possibleMoves.filter(mv \=\> {  
    const next \= applyMove(prevState, 'O', mv);

    // If X can win immediately after this move, it's bad  
    const xMoves \= legalMoves(next, 'X');  
    for (const xm of xMoves) {  
      const afterX \= applyMove(next, 'X', xm);  
      if (afterX.winner \=== 'X') return false;  
    }

    return true;  
  });

  // If actual move is NOT in safe moves → it's a mistake  
  return \!safeMoves.includes(actualMove);  
}

  /\* ═══════════════════════════════════════════════════════════  
     ANALYSIS SUITE — run after every O move so UI can annotate  
  ═══════════════════════════════════════════════════════════ \*/

  // Returns all O cells whose vanishing on O's NEXT turn breaks an O threat  
  function findDangerousOPieces(state) {  
    if (state.queues\[O\].length \< 3) return \[\];  
    const dyingIdx \= state.queues\[O\]\[0\];  
    const atRisk \= \[\];  
    for (const ln of WIN\_LINES) {  
      if (\!ln.includes(dyingIdx)) continue;  
      const lineVals \= ln.map(i \=\> state.board\[i\]);  
      const oCount   \= lineVals.filter(v \=\> v \=== O).length;  
      if (oCount \>= 2) atRisk.push(dyingIdx); // O about to lose a 2-in-row anchor  
    }  
    return \[...new Set(atRisk)\];  
  }

  // Find X cells that, if played, create a fork (two simultaneous winning threats)  
  function findForkMoves(state) {  
    return legalMoves(state, X).filter(mv \=\> {  
      const after \= applyMove(state, X, mv);  
      if (after.winner \=== X) return false; // already a win, not a fork  
      let threats \= 0;  
      for (const \[a,b,c\] of WIN\_LINES) {  
        const line \= \[after.board\[a\], after.board\[b\], after.board\[c\]\];  
        const xn   \= line.filter(v \=\> v \=== X).length;  
        const en   \= line.filter(v \=\> v \=== null).length;  
        if (xn \=== 2 && en \=== 1) threats++;  
      }  
      return threats \>= 2;  
    });  
  }

  // Squares where O playing next turn would win — X must block these  
  function findImmediateOThreats(state) {  
    return legalMoves(state, O).filter(mv \=\> applyMove(state, O, mv).winner \=== O);  
  }

  // Score every cell from X's perspective for the heatmap (0–1 normalised)  
  function computeHeatmap(state, difficulty) {  
    const depths \= { Easy: 2, Medium: 4, Hard: 6 };  
    const d \= depths\[difficulty\] ?? 6;  
    const moves \= legalMoves(state, X);  
    if (\!moves.length) return Array(9).fill(0);  
    const scored \= moves.map(mv \=\> {  
      const next  \= applyMove(state, X, mv);  
      const score \= next.winner \=== X ? 120 : minimax(next, O, 0, d, \-Infinity, Infinity, new Set());  
      return { mv, score };  
    });  
    const min \= Math.min(...scored.map(s \=\> s.score));  
    const max \= Math.max(...scored.map(s \=\> s.score));  
    const heat \= Array(9).fill(0);  
    for (const { mv, score } of scored)  
      heat\[mv\] \= max \=== min ? 0.5 : (score \- min) / (max \- min);  
    return heat;  
  }

  // Compute a "threat score" for each empty O cell (how dangerous is that square for O)  
  function computeOThreatMap(state) {  
    const danger \= Array(9).fill(0);  
    const moves  \= legalMoves(state, O);  
    for (const mv of moves) {  
      let score \= 0;  
      const after \= applyMove(state, O, mv);  
      if (after.winner \=== O) { danger\[mv\] \= 1; continue; }  
      for (const \[a,b,c\] of WIN\_LINES) {  
        const line \= \[after.board\[a\], after.board\[b\], after.board\[c\]\];  
        const on   \= line.filter(v \=\> v \=== O).length;  
        const en   \= line.filter(v \=\> \!v).length;  
        if (on \=== 2 && en \=== 1) score \+= 0.45;  
        if (on \=== 1 && en \=== 2) score \+= 0.08;  
      }  
      danger\[mv\] \= Math.min(1, score);  
    }  
    return danger;  
  }

  function evaluateRiskyState(state) {  
  // Count how many immediate winning moves O would have  
  const oWinningMoves \= legalMoves(state, 'O').filter(mv \=\> {  
    return applyMove(state, 'O', mv).winner \=== 'O';  
  });

  // If O has 2+ threats → very dangerous (fork-like)  
  if (oWinningMoves.length \>= 2) return \-50;

  // If O has 1 threat → risky  
  if (oWinningMoves.length \=== 1) return \-20;

  return 0;  
}

  function evaluateOMoveRisk(state, move) {  
  const next \= applyMove(state, 'O', move);

  // 1\. Immediate X win after this move → VERY BAD  
  const xMoves \= legalMoves(next, 'X');  
  for (const xm of xMoves) {  
    const afterX \= applyMove(next, 'X', xm);  
    if (afterX.winner \=== 'X') {  
      return 'BLUNDER';  
    }  
  }

  // 2\. Check if X can create fork after this  
  const forks \= findForkMoves(next);  
  if (forks.length) {  
    return 'TRAP';  
  }

  // 3\. Otherwise safe  
  return 'SAFE';  
}

  /\* ═══════════════════════════════════════════════════════════  
     PATTERN RECOGNISER — label O's opening patterns  
  ═══════════════════════════════════════════════════════════ \*/  
  function recognisePattern(oHistory) {  
    if (\!oHistory.length) return null;  
    const \[f, s, t\] \= oHistory;  
    if (f \=== 4) return 'Center opener — most flexible; X must contest corners.';  
    if (\[0,2,6,8\].includes(f)) {  
      if (s \!== undefined && \[0,2,6,8\].includes(s)) {  
        const pairs \= \[\[0,8\],\[2,6\],\[0,2\],\[6,8\],\[0,6\],\[2,8\]\];  
        for (const \[a,b\] of pairs) if ((f===a&\&s===b)||(f===b&\&s===a))  
          return a===0&\&b===8||a===2&\&b===6  
            ? 'Diagonal corners — dangerous fork potential. X must occupy center \+ edge.'  
            : 'Adjacent corners — X center \+ opposite edge neutralises.';  
      }  
      return 'Corner opener — X center reply is strongest counter.';  
    }  
    if (\[1,3,5,7\].includes(f)) return 'Edge opener — X corners \+ center dominate.';  
    return null;  
  }

  function detectPersonality(oHistory, board) {  
  if (oHistory.length \< 2) return null;

  const \[first, second, third\] \= oHistory;

  // Default confidence  
  let confidence \= 0.5;

  // \--- HARD patterns \---  
  if (first \=== 4) {  
    return { label: 'HARD (Center control)', confidence: 0.6 };  
  }

  if (\[0,2,6,8\].includes(first) && second \=== 4) {  
    return { label: 'HARD (Corner → Center)', confidence: 0.7 };  
  }

  // \--- 3rd move recovery (higher confidence) \---  
  if (oHistory.length \>= 3) {  
    if (\[1,3,5,7\].includes(first) && third \=== 4) {  
      return { label: 'HARD (Recovered to center)', confidence: 0.9 };  
    }  
  }

  // \--- MEDIUM \---  
  if (\[1,3,5,7\].includes(second)) {  
    return { label: 'MEDIUM (Defensive / reactive)', confidence: 0.6 };  
  }

  // \--- EASY \---  
  if (\!\[0,2,4,6,8\].includes(second)) {  
    return { label: 'EASY (Random / weak)', confidence: 0.5 };  
  }

  return { label: 'UNKNOWN', confidence: 0.4 };  
}

function estimateRTPProbability(oHistory, personality, exploitWindow) {  
  let base \= 0.03; // default 3% (your RTP assumption)

  if (\!personality) return base;

  if (personality.includes('EASY')) base \= 0.25;  
  else if (personality.includes('MEDIUM')) base \= 0.12;  
  else if (personality.includes('HARD')) base \= 0.04;

  // If mistake already happened recently → higher chance of chain errors  
  if (exploitWindow \> 0) {  
  base \+= 0.08 \* exploitWindow; // stronger when fresh  
}

  // Early game \= more randomness  
  if (oHistory.length \<= 2) base \+= 0.05;

  return Math.min(base, 0.5);  
}

  /\* ═══════════════════════════════════════════════════════════  
     INSIGHT ENGINE — plain-language coaching per turn  
  ═══════════════════════════════════════════════════════════ \*/  
  function generateInsight(state, xMove, oMove, analysis) {  
    const { forks, threats, dangerO } \= analysis;  
    const lines \= \[\];

    if (threats.length \> 0)  
      lines.push(\`🔴 O threatened to win at ${threats.map(i\=\>SQ\[i\]).join('/')} — X blocked.\`);  
    if (forks.length \> 0)  
      lines.push(\`⚡ X has fork options at ${forks.map(i\=\>SQ\[i\]).join('/')} — playing any creates two simultaneous threats.\`);  
    if (dangerO.length \> 0)  
      lines.push(\`⏳ O's oldest piece at ${dangerO.map(i\=\>SQ\[i\]).join('/')} vanishes next O turn — exploit this window.\`);  
    if (xMove?.isForcedWin)  
      lines.push(\`✅ X found a forced win sequence.\`);

      const baseText \= lines.length  
  ? lines.join(' ')  
  : \`X placed at ${SQ\[xMove?.move ?? 0\]} — continuing counter-structure.\`;

// Confidence estimation  
let confidence \= 'Balanced position';  
if (xMove?.score \>= 100) confidence \= 'High win probability';  
else if (xMove?.score \>= 40) confidence \= 'Favorable position';  
else if (xMove?.score \>= 0) confidence \= 'Likely draw';  
else confidence \= 'Risky position';

// If exploit mode triggered  
if (xMove?.type \=== 'fork' || xMove?.type \=== 'win') {  
  confidence \= 'Exploit window — high chance to win';  
}

// Convert minimax score → win probability  
let winProb \= null;  
if (typeof xMove?.score \=== 'number') {  
  winProb \= Math.max(0, Math.min(1, (xMove.score \+ 120) / 240));  
}

const probText \= winProb \!== null  
  ? \` | 📈 Win probability: ${(winProb \* 100).toFixed(0)}%\`  
  : '';

return baseText \+ \` | 📊 ${confidence}\` \+ probText;  
  }

  /\* ═══════════════════════════════════════════════════════════  
     APP COMPONENT  
  ═══════════════════════════════════════════════════════════ \*/  
  function App() {  
    // ── state ────────────────────────────────────────────────  
    const \[board,      setBoard\]      \= useState(Array(9).fill(null));  
   
    const \[exploitWindow, setExploitWindow\] \= useState(0);  
    const \[queues,     setQueues\]     \= useState({ X:\[\], O:\[\] });  
    const \[winner,     setWinner\]     \= useState(null);  
    const \[winLine,    setWinLine\]    \= useState(\[\]);  
    const \[phase,      setPhase\]      \= useState('setup'); // 'setup' | 'game'  
    const \[difficulty, setDifficulty\] \= useState('Hard');  
    const \[thinking,   setThinking\]   \= useState(false);

    // Coach signals  
    const \[heatmap,    setHeatmap\]    \= useState(Array(9).fill(0));  
    const \[oThreat,    setOThreat\]    \= useState(Array(9).fill(0));  
    const \[forkMoves,  setForkMoves\]  \= useState(\[\]);  
    const \[blockMoves, setBlockMoves\] \= useState(\[\]);  
    const \[dyingO,     setDyingO\]     \= useState(\[\]);  
    const \[insight,    setInsight\]    \= useState('');  
    const \[pattern,    setPattern\]    \= useState(null);  
    const \[personality, setPersonality\] \= useState(null);  
// now will store: { label: string, confidence: number }  
    const \[rtpScore, setRtpScore\] \= useState(0); // 0 to 1 (probability of mistake)

    // History & stats  
    const \[oHistory,   setOHistory\]   \= useState(\[\]);  
    const \[sessionHistory, setSessionHistory\] \= useState(\[\]);  
    const \[log,        setLog\]        \= useState(\[\]);  
    const \[moveHistory, setMoveHistory\] \= useState(\[\]);  
    const \[winningPatterns, setWinningPatterns\] \= useState(\[\]);  
    const \[stats,      setStats\]      \= useState({ games:0, xWins:0, oWins:0, draws:0 });  
    const \[showHints,  setShowHints\]  \= useState(true);  
    const \[hoverIdx,   setHoverIdx\]   \= useState(null);  
    const \[replayInput, setReplayInput\] \= useState('');  
const \[replayResult, setReplayResult\] \= useState(null);  
    const \[hoverWarning, setHoverWarning\] \= useState(null);

    const logRef \= useRef(null);  
    const hasLoadedRef \= useRef(false);

    useEffect(() \=\> {  
  if (\!hasLoadedRef.current) return;

  if (sessionHistory.length \> 0) {  
    localStorage.setItem('ttt\_session\_history', JSON.stringify(sessionHistory));  
  }  
}, \[sessionHistory\]);

useEffect(() \=\> {  
  const saved \= localStorage.getItem('ttt\_session\_history');  
  if (saved) {  
    setSessionHistory(JSON.parse(saved));  
  }  
  hasLoadedRef.current \= true;  
}, \[\]);

    // ── derived display values ────────────────────────────────  
    const oldestO \= queues.O.length \=== 3 ? queues.O\[0\] : null;  
    const oldestX \= queues.X.length \=== 3 ? queues.X\[0\] : null;

    // When hovering an empty cell, show if that O move would lose its oldest  
    const hoverVanishPreview \= hoverIdx \!== null && \!board\[hoverIdx\] && queues.O.length \=== 3  
      ? queues.O\[0\] : null;

    // ── analysis effect — runs whenever board changes ─────────  
    useEffect(() \=\> {  
      if (phase \!== 'game' || winner) {  
        setHeatmap(Array(9).fill(0));  
        setOThreat(Array(9).fill(0));  
        setForkMoves(\[\]);  
        setBlockMoves(\[\]);  
        setDyingO(\[\]);  
        return;  
      }  
      const st \= { board, queues };  
      setHeatmap(computeHeatmap(st, difficulty));  
      setOThreat(computeOThreatMap(st));  
      setForkMoves(findForkMoves(st));  
      setBlockMoves(findImmediateOThreats(st));  
      setDyingO(findDangerousOPieces(st));  
    }, \[board, queues, phase, winner, difficulty\]);

    // Scroll log  
    useEffect(() \=\> {  
      if (logRef.current) logRef.current.scrollTop \= logRef.current.scrollHeight;  
    }, \[log\]);

    // ── helpers ──────────────────────────────────────────────  
    const pushLog \= (entry) \=\> setLog(prev \=\> \[...prev.slice(-60), entry\]);

    const runAnalysis \= (st) \=\> ({  
      forks:   findForkMoves(st),  
      threats: findImmediateOThreats(st),  
      dangerO: findDangerousOPieces(st),  
    });

    function runReplaySimulation(sequence, difficulty) {  
  let state \= { board: Array(9).fill(null), queues: { X: \[\], O: \[\] } };  
  const transcript \= \[\];

  for (let i \= 0; i \< sequence.length; i++) {  
    const oMove \= sequence\[i\];

    // Apply O move  
    const afterO \= applyMove(state, 'O', oMove);  
    state \= { board: afterO.board, queues: afterO.queues };

    transcript.push({ player: 'O', sq: SQ\[oMove\] });

    if (afterO.winner \=== 'O') {  
      return { result: 'O\_WIN', step: i \+ 1, transcript };  
    }

    // X response (use adaptive logic)  
    const xMv \= decideAdaptiveMove(state, difficulty, false, null, 0.03);  
    if (\!xMv) return { result: 'DRAW', transcript };

    const afterX \= applyMove(state, 'X', xMv.move);  
    state \= { board: afterX.board, queues: afterX.queues };

    // Compute win probability  
    let winProb \= null;  
    if (typeof xMv?.score \=== 'number') {  
      winProb \= Math.max(0, Math.min(1, (xMv.score \+ 120) / 240));  
    }

    transcript.push({  
      player: 'X',  
      sq: SQ\[xMv.move\],  
      winProb: winProb \!== null ? \`${(winProb \* 100).toFixed(0)}%\` : null  
    });

    if (afterX.winner \=== 'X') {  
      return { result: 'X\_WIN', step: i \+ 1, transcript };  
    }  
  }

  return { result: 'INCOMPLETE', transcript };  
}

    // ── main click handler: YOU place O, AI responds with X ──  
    const handleCellClick \= useCallback((idx) \=\> {  
      if (winner || thinking || phase \!== 'game') return;

      // Validate: cell must be in legal O moves  
      const legal \= new Set(legalMoves({ board, queues }, O));  
      if (\!legal.has(idx)) return;

      // Save previous state BEFORE O move  
      const snapshot \= { board: \[...board\], queues: { X: \[...queues.X\], O: \[...queues.O\] } };

      // 1\. Apply O move  
      const afterO \= applyMove({ board, queues }, O, idx);  
      // Detect if casino made a mistake  
      const isMistake \= detectSubOptimalMove(snapshot, afterO);

// Compute next window locally (IMPORTANT)  
const nextWindow \= isMistake ? 2 : Math.max(0, exploitWindow \- 1);

if (isMistake) {  
  pushLog('⚠ RTP window detected: Casino made sub-optimal move');  
}

// Update state AFTER computing nextWindow  
setExploitWindow(nextWindow);

      setBoard(afterO.board);  
      setQueues(afterO.queues);  
      const newOHist \= \[...oHistory, idx\];  
      setOHistory(newOHist);  
      setPattern(recognisePattern(newOHist));

const detected \= detectPersonality(newOHist, afterO.board);  
      if (detected && (\!personality || detected.label \!== personality.label)) {  
   setPersonality(detected);  
  pushLog(\`🧠 Profile updated: ${detected.label} (${Math.round(detected.confidence \* 100)}%)\`);  
}

const newRtp \= estimateRTPProbability(  
  newOHist,  
  (detected || personality)?.label || null,  
  nextWindow  
);  
setRtpScore(newRtp);  
pushLog(\`→ O placed at ${SQ\[idx\]}${afterO.removed \!== null ? \` (removed ${SQ\[afterO.removed\]})\` : ''}\`);  
      setMoveHistory(prev \=\> \[...prev, { player: 'O', pos: idx }\]);  
      if (afterO.winner \=== O) {  
        setWinner(O);  
        setWinLine(afterO.winLine);  
        setStats(p \=\> ({...p, games: p.games+1, oWins: p.oWins+1}));  
        setSessionHistory(prev \=\> \[...prev, \[...moveHistory, { player: 'O', pos: idx }\]\]);  
        pushLog('✗ O wins this round. Study what allowed this — use "Reset" to try again.');  
        return;  
      }

      // 2\. X responds  
      setThinking(true);  
      setTimeout(() \=\> {  
        const st  \= { board: afterO.board, queues: afterO.queues };  
        const ana \= runAnalysis(st);  
        const xMv \= decideAdaptiveMove(  
  st,  
  difficulty,  
  nextWindow \> 0,  
  personality?.label || null,  
  newRtp  
);

        if (\!xMv) { setThinking(false); return; }

        const afterX \= applyMove(st, X, xMv.move);  
        setBoard(afterX.board);  
        setQueues(afterX.queues);

        const ins \= generateInsight(st, xMv, idx, ana);  
        setInsight(ins);  
        let winProb \= null;  
if (typeof xMv?.score \=== 'number') {  
  winProb \= Math.max(0, Math.min(1, (xMv.score \+ 120) / 240));  
}  
setMoveHistory(prev \=\> \[...prev, { player: 'X', pos: xMv.move }\]);  
pushLog(  
  \`← X replied at ${SQ\[xMv.move\]}${afterX.removed \!== null ? \` (removed ${SQ\[afterX.removed\]})\` : ''}\` \+  
  (winProb \!== null ? \` | WinProb ${(winProb \* 100).toFixed(0)}%\` : '') \+  
  \` | ${ins}\`  
);

        // Build readable sequence  
        if (afterX.winner \=== X) {  
          setWinner(X);  
          setWinLine(afterX.winLine);  
          setStats(p \=\> ({...p, games: p.games+1, xWins: p.xWins+1}));  
          setSessionHistory(prev \=\> \[...prev, \[...moveHistory, { player: 'X', pos: xMv.move }\]\]);  
          pushLog('✓ X wins\! This counter-pattern works. Note the sequence.');

// Build winning sequence ONLY here  
const finalHistory \= \[...moveHistory, { player: 'X', pos: xMv.move }\];

const winSequence \= finalHistory  
  .map(m \=\> \`${m.player}→${SQ\[m.pos\]}\`)  
  .join(', ');

  const winPatternLabel \= pattern || 'Unknown pattern';

// Save only real winning sequences  
setWinningPatterns(prev \=\> \[  
  { sequence: winSequence, pattern: winPatternLabel },  
  ...prev.slice(0, 4)  
\]);  
        } else {  
          // Check draw  
          const noMovesO \= \!legalMoves(afterX, O).length;  
          const noMovesX \= \!legalMoves(afterX, X).length;  
          if (noMovesO || noMovesX) {  
            setWinner('draw');  
            setStats(p \=\> ({...p, games: p.games+1, draws: p.draws+1}));  
            setSessionHistory(prev \=\> \[...prev, \[...oHistory\]\]);  
            pushLog('— Draw (no legal moves). Analyse the queue state that caused this.');  
          }  
        }  
        setThinking(false);  
      }, 380);  
    }, \[board, queues, winner, thinking, phase, difficulty, oHistory\]);

    // ── start / reset ─────────────────────────────────────────  
    const startGame \= () \=\> {  
      setBoard(Array(9).fill(null));  
      setQueues({ X:\[\], O:\[\] });  
      setExploitWindow(0);  
      setWinner(null);  
      setWinLine(\[\]);  
      setOHistory(\[\]);  
      setMoveHistory(\[\]);  
      setInsight('');  
      setPattern(null);  
      setPhase('game');  
      setThinking(false);  
      pushLog('── New round started. You play O. AI coaches X. ──');  
    };

    const handleReplay \= () \=\> {  
  if (\!replayInput.trim()) return;

  const sequence \= replayInput  
    .split(',')  
    .map(s \=\> parseInt(s.trim()))  
    .filter(n \=\> \!isNaN(n) && n \>= 0 && n \<= 8);

  if (\!sequence.length) return;

  const result \= runReplaySimulation(sequence, difficulty);  
  setReplayResult(result);  
};

const copyGameLog \= () \=\> {  
  if (\!sessionHistory.length) return;

  const text \= sessionHistory  
    .map((game, i) \=\> \`Game ${i \+ 1}: ${game.map(m \=\> \`${m.player}:${SQ\[m.pos\]}\`).join(',')}\`)  
    .join('\\n');

  navigator.clipboard.writeText(text);  
  pushLog('📋 Game log copied to clipboard');  
};

    const resetToSetup \= () \=\> {  
      setPhase('setup');  
      setBoard(Array(9).fill(null));  
      setQueues({ X:\[\], O:\[\] });  
      setWinner(null);  
      setWinLine(\[\]);  
      setOHistory(\[\]);  
      setInsight('');  
      setPattern(null);  
      setThinking(false);  
    };

        // ── cell appearance calculator ────────────────────────────  
    const cellStyle \= (idx) \=\> {  
      const cell    \= board\[idx\];  
      const isWin   \= winLine.includes(idx);  
      const isOOld  \= idx \=== oldestO;  
      const isXOld  \= idx \=== oldestX;  
      const isFork  \= \!cell && forkMoves.includes(idx) && showHints;  
      const isBlock \= \!cell && blockMoves.includes(idx) && showHints;  
      const isDyO   \= \!cell && dyingO.includes(idx) && showHints;  
      const isHover \= hoverIdx \=== idx;  
      const isVPrev \= hoverVanishPreview \=== idx;  
      const heat    \= heatmap\[idx\];  
      const othr    \= oThreat\[idx\];  
      const isLegal \= phase \=== 'game' && \!winner && \!thinking &&  
                      legalMoves({ board, queues }, O).includes(idx);

      let extra \= 'board-cell cell-glow ';  
      if (isWin)   extra \+= 'ring-win ';  
      else if (isBlock) extra \+= 'ring-warn ';  
      else if (isFork)  extra \+= 'ring-trap ';  
      else if (isVPrev) extra \+= 'ring-vanish ';  
      else if (isOOld || isXOld) extra \+= 'ring-oold ';

      const cursor \= isLegal ? 'cursor-pointer hover:brightness-125' : 'cursor-not-allowed opacity-60';

      // Background: heat tint for empty cells when showHints  
      let bg \= 'bg-deep';  
      if (\!cell && showHints && phase \=== 'game') {  
        // Green tint \= good for X (coach hint), Red tint \= danger (O threat)  
        const g \= Math.round(heat \* 60);  
        const r \= Math.round(othr \* 80);  
        bg \= '';  
        extra \+= \`bg-\[rgba(${r},${g},60,0.55)\] \`;  
      }  
      if (cell \=== X) bg \= 'bg-cyan-900/40';  
      if (cell \=== O) bg \= 'bg-fuchsia-900/40';  
      if (isWin)      bg \= 'bg-green-900/40';

      return \`${extra}${bg} ${cursor} h-20 w-20 rounded-lg text-3xl font-black flex items-center justify-center relative transition-all duration-150\`;  
    };

    /\* ── RENDER ──────────────────────────────────────────────── \*/  
    const xWR \= stats.games ? ((stats.xWins/stats.games)\*100).toFixed(0) : '--';  
    const oWR \= stats.games ? ((stats.oWins/stats.games)\*100).toFixed(0) : '--';

    return (  
      \<div className\="min-h-screen bg-gradient-to-br from-deep via-\[\#050e20\] to-deep flex flex-col items-center justify-start py-8 px-4"\>

        {/\* ── Header ────────────────────────────────────────── \*/}  
        \<div className\="mb-6 text-center"\>  
          \<h1 className\="text-3xl font-black text-neon tracking-widest mb-1"\>  
            CHORUS LAPILLI  
          \</h1\>  
          \<p className\="text-xs text-cyan-400/70 tracking-wider uppercase"\>  
            Counter-System Training Simulator  
          \</p\>  
          \<p className\="text-xs text-cyan-600 mt-1"\>  
            You input \<span className\="text-fuchsia-300 font-bold"\>O\</span\> (casino first-mover) ·  
            AI coaches \<span className\="text-neon font-bold"\>X\</span\> (optimal response)  
          \</p\>  
        \</div\>

        {/\* ── Setup screen ──────────────────────────────────── \*/}  
        {phase \=== 'setup' && (  
          \<div className\="bg-panel border border-cyan-700/40 rounded-2xl p-8 w-full max-w-md text-center space-y-6 animate-\[fadeIn\_.3s\_ease-out\]"\>  
            \<div\>  
              \<h2 className\="text-neonSoft text-lg font-bold mb-2"\>Mission Briefing\</h2\>  
              \<p className\="text-sm text-cyan-300/80 leading-relaxed"\>  
                You will play \<strong className\="text-fuchsia-300"\>O\</strong\> — copying moves from a casino system you're studying.\<br/\>  
                The AI plays \<strong className\="text-neon"\>X\</strong\> as your counter-coach, always searching for the optimal response.\<br/\>  
                Learn which O patterns the AI can crack, and how.  
              \</p\>  
            \</div\>  
            \<div\>  
              \<p className\="text-xs text-cyan-500 uppercase tracking-widest mb-3"\>AI Coach Depth\</p\>  
              \<div className\="flex gap-3 justify-center"\>  
                {\['Easy','Medium','Hard'\].map(d \=\> (  
                  \<button key\={d} onClick\={() \=\> setDifficulty(d)}  
                    className\={\`px-4 py-2 rounded-lg border text-sm font-bold transition-all ${  
                      difficulty \=== d  
                        ? 'border-neon bg-neon/10 text-neon'  
                        : 'border-cyan-700 text-cyan-500 hover:border-cyan-400'  
                    }\`}\>  
                    {d}  
                  \</button\>  
                ))}  
              \</div\>  
              \<p className\="text-xs text-cyan-600 mt-2"\>  
                {difficulty \=== 'Easy'   && 'Depth 3 — occasionally plays suboptimal X (for learning gaps)'}  
                {difficulty \=== 'Medium' && 'Depth 5 — strong coaching, allows some misses'}  
                {difficulty \=== 'Hard'   && 'Depth 7 — near-perfect X counter every turn'}  
              \</p\>  
            \</div\>  
            \<button onClick\={startGame}  
              className\="w-full py-3 rounded-xl bg-neon text-deep font-black text-lg tracking-widest hover:bg-neonSoft transition"\>  
              START TRAINING  
            \</button\>  
            {stats.games \> 0 && (  
              \<p className\="text-xs text-cyan-600"\>  
                Session: {stats.games} rounds · X wins {xWR}% · O wins {oWR}%  
              \</p\>  
            )}  
          \</div\>  
        )}

        {/\* ── Game screen ───────────────────────────────────── \*/}  
        {phase \=== 'game' && (  
          \<div className\="w-full max-w-4xl grid grid-cols-1 lg:grid-cols-\[1fr\_340px\] gap-6"\>

            {/\* LEFT: board \+ controls \*/}  
            \<div className\="flex flex-col items-center gap-4"\>

              {/\* Status bar \*/}  
              \<div className\="w-full bg-panel border border-cyan-700/30 rounded-xl px-4 py-2 flex flex-wrap gap-3 justify-between items-center text-xs"\>  
                \<span className\="text-cyan-500"\>Difficulty: \<span className\="text-neon font-bold"\>{difficulty}\</span\>\</span\>  
                {thinking ? (  
                  \<span className\="text-cyan-400"\>  
                    X thinking  
                    \<span className\="thinking-dot inline-block ml-1"\>.\</span\>  
                    \<span className\="thinking-dot inline-block" style\={{animationDelay:'.25s'}}\>.\</span\>  
                    \<span className\="thinking-dot inline-block" style\={{animationDelay:'.5s'}}\>.\</span\>  
                  \</span\>  
                ) : winner ? (  
                  \<span className\={\`font-bold ${winner===X?'text-neon':winner===O?'text-fuchsia-300':'text-yellow-400'}\`}\>  
                    {winner===X ? '✓ X WINS' : winner===O ? '✗ O WINS' : '— DRAW'}  
                  \</span\>  
                ) : (  
                  \<span className\="text-cyan-300"\>Click an empty cell to place \<span className\="text-fuchsia-300 font-bold"\>O\</span\>\</span\>  
                )}  
                \<span className\="text-cyan-500"\>  
                  O queue: {queues.O.length}/3 · X queue: {queues.X.length}/3  
                \</span\>  
              \</div\>

              {/\* Pattern recognition \*/}  
              {pattern && (  
                \<div className\="w-full bg-\[\#0a1a10\] border border-green-700/40 rounded-lg px-4 py-2 text-xs text-green-300"\>  
                  📐 Pattern: {pattern}  
                \</div\>  
              )}

              {personality && (  
  \<div className\="w-full bg-\[\#1a0a0a\] border border-red-700/40 rounded-lg px-4 py-2 text-xs text-red-300"\>  
    🧠 Casino Profile: {personality.label} — {Math.round(personality.confidence \* 100)}% confidence  
  \</div\>  
)}

              {/\* BOARD \*/}  
              \<div className\="grid grid-cols-3 gap-2"\>  
                {board.map((cell, idx) \=\> (  
                  \<button  
                    key\={idx}  
                    onClick\={() \=\> handleCellClick(idx)}  
                    onMouseEnter\={() \=\> {  
  setHoverIdx(idx);

  if (\!board\[idx\] && phase \=== 'game' && \!winner) {  
    const risk \= evaluateOMoveRisk({ board, queues }, idx);  
    setHoverWarning({ idx, risk });  
  }  
}}  
onMouseLeave\={() \=\> {  
  setHoverIdx(null);  
  setHoverWarning(null);  
}}  
                    className\={cellStyle(idx)}  
                    title\={SQ\[idx\]}  
                    disabled\={\!\!winner || thinking}  
                  \>  
                    {/\* Piece \*/}  
                    \<span className\={cell \=== X ? 'text-neon' : cell \=== O ? 'text-fuchsia-300' : 'text-cyan-800'}\>  
                      {cell || '·'}  
                    \</span\>

                    {/\* Oldest badge \*/}  
                    {(idx \=== oldestO || idx \=== oldestX) && (  
                      \<span className\={\`absolute top-0.5 right-1 text-\[9px\] font-bold ${idx===oldestO?'text-fuchsia-400':'text-cyan-400'}\`}\>  
                        {idx \=== oldestO ? 'O↓' : 'X↓'}  
                      \</span\>  
                    )}

                    {/\* Heat value for empty cells (coach overlay) \*/}  
                    {\!cell && showHints && phase \=== 'game' && \!winner && (  
                      \<span className\="absolute bottom-0.5 left-1 text-\[8px\] text-cyan-700"\>  
                        {SQ\[idx\]}  
                      \</span\>  
                    )}

                    {hoverWarning && hoverWarning.idx \=== idx && (  
  \<span className\={\`absolute bottom-0.5 right-1 text-\[8px\] font-bold ${  
    hoverWarning.risk \=== 'BLUNDER'  
      ? 'text-red-500'  
      : hoverWarning.risk \=== 'TRAP'  
      ? 'text-orange-400'  
      : 'text-green-400'  
  }\`}\>  
    {hoverWarning.risk}  
  \</span\>  
)}

                    {/\* Block / fork / fork-move labels \*/}  
                    {\!cell && showHints && blockMoves.includes(idx) && (  
                      \<span className\="absolute top-0.5 left-1 text-\[8px\] text-red-400 font-bold"\>BLK\</span\>  
                    )}  
                    {\!cell && showHints && forkMoves.includes(idx) && \!blockMoves.includes(idx) && (  
                      \<span className\="absolute top-0.5 left-1 text-\[8px\] text-orange-400 font-bold"\>FORK\</span\>  
                    )}  
                  \</button\>  
                ))}  
              \</div\>

              {/\* Legend \*/}  
              {showHints && (  
                \<div className\="w-full bg-panel border border-cyan-800/30 rounded-lg px-4 py-2"\>  
                  \<p className\="text-xs text-cyan-500 mb-1 font-bold uppercase tracking-wide"\>Coach Overlay\</p\>  
                  \<div className\="flex flex-wrap gap-3 text-xs text-cyan-400"\>  
                    \<span\>\<span className\="text-green-400"\>■\</span\> Green \= weak O positions (X controls these)\</span\>  
                    \<span\>\<span className\="text-red-400"\>■\</span\> Red \= strong O positions (dangerous for X)\</span\>  
                    \<span\>\<span className\="text-red-400 font-bold"\>BLK\</span\> \= X will block here\</span\>  
                    \<span\>\<span className\="text-orange-400 font-bold"\>FORK\</span\> \= X fork opportunity\</span\>  
                    \<span\>\<span className\="text-yellow-400"\>pulsing border\</span\> \= oldest piece (vanishes next turn)\</span\>  
                    \<p className\="text-\[10px\] text-cyan-600 mt-1"\>You play O — avoid green cells, prefer red when safe.\</p\>  
                  \</div\>  
                \</div\>  
              )}

              {/\* Controls \*/}

              \<button  
  onClick\={copyGameLog}  
  className\="px-4 py-1.5 rounded-lg border border-green-500 text-green-300 text-xs font-bold hover:bg-green-900/30 transition"  
\>  
  Copy Game Log  
\</button\>

              \<div className\="flex flex-wrap gap-2 justify-center"\>  
                \<button onClick\={() \=\> setShowHints(h \=\> \!h)}  
                  className\={\`px-4 py-1.5 rounded-lg border text-xs font-bold transition ${showHints ? 'border-neon text-neon' : 'border-cyan-700 text-cyan-500'}\`}\>  
                  Coach Hints: {showHints ? 'ON' : 'OFF'}  
                \</button\>  
                \<button onClick\={startGame}  
                  className\="px-4 py-1.5 rounded-lg border border-cyan-500 text-cyan-200 text-xs font-bold hover:bg-cyan-900/30 transition"\>  
                  New Round  
                \</button\>  
                \<button onClick\={resetToSetup}  
                  className\="px-4 py-1.5 rounded-lg border border-cyan-700 text-cyan-500 text-xs hover:bg-cyan-900/20 transition"\>  
                  ← Setup  
                \</button\>  
              \</div\>

              \<div className\="w-full bg-panel border border-cyan-700/30 rounded-lg px-4 py-3 mt-2 space-y-2"\>  
  \<p className\="text-xs text-cyan-500 font-bold uppercase tracking-wide"\>  
    Mirror Test (Replay)  
  \</p\>

  \<input  
    type\="text"  
    value\={replayInput}  
    onChange\={(e) \=\> setReplayInput(e.target.value)}  
    placeholder\="Enter O moves e.g. 4,0,8"  
    className\="w-full bg-deep border border-cyan-700 rounded px-2 py-1 text-xs text-cyan-200"  
  /\>

  \<button  
    onClick\={handleReplay}  
    className\="w-full py-1.5 rounded bg-neon text-deep text-xs font-bold"  
  \>  
    Run Simulation  
  \</button\>

  {replayResult && (  
  \<div className\="text-xs text-cyan-300 space-y-1"\>  
    \<p\>  
      Result: {replayResult.result}  
      {replayResult.step && \` at step ${replayResult.step}\`}  
    \</p\>

    {replayResult.transcript && (  
      \<div className\="bg-deep border border-cyan-800/30 rounded p-2 max-h-32 overflow-y-auto"\>  
        {replayResult.transcript.map((step, i) \=\> (  
          \<p key\={i}\>  
            {step.player} → {step.sq}  
            {step.player \=== 'X' && step.winProb ? \` (${step.winProb})\` : ''}  
          \</p\>  
        ))}  
      \</div\>  
    )}  
  \</div\>  
)}  
\</div\>

              {/\* Queue tracker \*/}  
              \<div className\="w-full bg-panel border border-cyan-800/30 rounded-lg px-4 py-2 text-xs"\>  
                \<p className\="text-cyan-500 font-bold uppercase tracking-wide mb-1"\>Queue State Tracker\</p\>  
                \<p className\="text-cyan-400"\>  
                  O queue (oldest→newest):\&nbsp;  
                  {queues.O.length ? queues.O.map(i \=\> SQ\[i\]).join(' → ') : '—'}\&nbsp;  
                  {oldestO \!== null && \<span className\="text-fuchsia-400"\>(next to vanish: {SQ\[oldestO\]})\</span\>}  
                \</p\>  
                \<p className\="text-cyan-400 mt-0.5"\>  
                  X queue (oldest→newest):\&nbsp;  
                  {queues.X.length ? queues.X.map(i \=\> SQ\[i\]).join(' → ') : '—'}\&nbsp;  
                  {oldestX \!== null && \<span className\="text-neon"\>(next to vanish: {SQ\[oldestX\]})\</span\>}  
                \</p\>  
              \</div\>  
            \</div\>

            {/\* RIGHT: coaching panel \*/}  
            \<div className\="flex flex-col gap-4"\>

              {/\* Current coaching insight \*/}  
              \<div className\="bg-\[\#040e14\] border border-neon/30 rounded-xl p-4"\>  
                \<p className\="text-xs text-neon uppercase tracking-widest font-bold mb-2"\>Coach Analysis\</p\>  
                \<p className\="text-sm text-cyan-200 leading-relaxed min-h-\[3rem\]"\>  
                  {insight || 'Waiting for your first O move…'}  
                \</p\>  
              \</div\>

              {/\* Current signal summary \*/}  
              \<div className\="bg-panel border border-cyan-700/30 rounded-xl p-4 space-y-2 text-xs"\>  
                \<p className\="text-cyan-500 uppercase tracking-widest font-bold"\>Live Signals\</p\>

                \<div className\={\`flex items-start gap-2 ${blockMoves.length ? 'text-red-300' : 'text-cyan-700'}\`}\>  
                  \<span className\="mt-0.5"\>🔴\</span\>  
                  \<span\>  
                    \<strong\>Immediate O threats:\</strong\>\&nbsp;  
                    {blockMoves.length ? blockMoves.map(i\=\>SQ\[i\]).join(', ') \+ ' — X must block' : 'None'}  
                  \</span\>  
                \</div\>

                \<div className\={\`flex items-start gap-2 ${forkMoves.length ? 'text-orange-300' : 'text-cyan-700'}\`}\>  
                  \<span className\="mt-0.5"\>⚡\</span\>  
                  \<span\>  
                    \<strong\>Fork opportunities for X:\</strong\>\&nbsp;  
                    {forkMoves.length ? forkMoves.map(i\=\>SQ\[i\]).join(', ') : 'None yet'}  
                  \</span\>  
                \</div\>

                \<div className\={\`flex items-start gap-2 ${dyingO.length ? 'text-yellow-300' : 'text-cyan-700'}\`}\>  
                  \<span className\="mt-0.5"\>⏳\</span\>  
                  \<span\>  
                    \<strong\>O expiry pressure:\</strong\>\&nbsp;  
                    {dyingO.length ? \`O loses anchor at ${dyingO.map(i\=\>SQ\[i\]).join(',')} next O turn\` : 'Not yet'}  
                  \</span\>  
                \</div\>

                \<div className\="flex items-start gap-2 text-cyan-600"\>  
                  \<span className\="mt-0.5"\>📊\</span\>  
                  \<span\>  
                    \<strong\>O threat map:\</strong\>\&nbsp;  
                    {Array.from({length:9},(\_,i)\=\>i)  
                      .filter(i \=\> \!board\[i\] && oThreat\[i\] \> 0.2)  
                      .sort((a,b)\=\>oThreat\[b\]-oThreat\[a\])  
                      .slice(0,3)  
                      .map(i\=\>\`${SQ\[i\]}(${(oThreat\[i\]\*100).toFixed(0)}%)\`)  
                      .join(' · ') || 'Minimal O pressure'}  
                  \</span\>  
                \</div\>  
              \</div\>

              {/\* Move log \*/}  
              \<div className\="bg-panel border border-cyan-700/30 rounded-xl p-4 flex flex-col flex-1"\>  
                \<p className\="text-xs text-cyan-500 uppercase tracking-widest font-bold mb-2"\>Move Log\</p\>  
                \<div className\="text-cyan-400 text-xs"\>  
  🎲 Mistake Probability: {(rtpScore \* 100).toFixed(0)}%  
\</div\>  
                \<div ref\={logRef}  
                  className\="flex-1 overflow-y-auto max-h-48 space-y-1 text-xs text-cyan-400 pr-1"\>  
                  {log.map((l, i) \=\> (  
                    \<p key\={i} className\={\`leading-snug ${  
                      l.startsWith('✓') ? 'text-green-400' :  
                      l.startsWith('✗') ? 'text-red-400'   :  
                      l.startsWith('←') ? 'text-neon'      :  
                      l.startsWith('→') ? 'text-fuchsia-300': 'text-cyan-600'  
                    }\`}\>{l}\</p\>  
                  ))}  
                  {\!log.length && \<p className\="text-cyan-800"\>No moves yet.\</p\>}  
                \</div\>  
              \</div\>

              \<div className\="bg-panel border border-green-700/30 rounded-xl p-4 text-xs"\>  
  \<p className\="text-green-400 uppercase tracking-widest font-bold mb-2"\>  
    Winning Patterns (X)  
  \</p\>

  {\!winningPatterns.length && (  
    \<p className\="text-cyan-700"\>No wins recorded yet.\</p\>  
  )}

  {winningPatterns.map((wp, i) \=\> (  
    \<div key\={i} className\="mb-2 border-b border-cyan-800/30 pb-1"\>  
      \<p className\="text-green-300"\>{wp.sequence}\</p\>  
      \<p className\="text-cyan-600 text-\[10px\]"\>Pattern: {wp.pattern}\</p\>  
    \</div\>  
  ))}  
\</div\>

              {/\* Session stats \*/}  
              \<div className\="bg-panel border border-cyan-700/30 rounded-xl p-4 text-xs"\>  
                \<p className\="text-cyan-500 uppercase tracking-widest font-bold mb-2"\>Session Stats\</p\>  
                \<div className\="grid grid-cols-2 gap-2"\>  
                  \<div className\="bg-deep rounded-lg p-2"\>  
                    \<p className\="text-cyan-600"\>Rounds\</p\>  
                    \<p className\="text-neon font-bold text-lg"\>{stats.games}\</p\>  
                  \</div\>  
                  \<div className\="bg-deep rounded-lg p-2"\>  
                    \<p className\="text-cyan-600"\>X win rate\</p\>  
                    \<p className\="text-neon font-bold text-lg"\>{xWR}%\</p\>  
                  \</div\>  
                  \<div className\="bg-deep rounded-lg p-2"\>  
                    \<p className\="text-cyan-600"\>O wins (you)\</p\>  
                    \<p className\="text-fuchsia-300 font-bold text-lg"\>{stats.oWins}\</p\>  
                  \</div\>  
                  \<div className\="bg-deep rounded-lg p-2"\>  
                    \<p className\="text-cyan-600"\>Draws\</p\>  
                    \<p className\="text-yellow-400 font-bold text-lg"\>{stats.draws}\</p\>  
                  \</div\>  
                \</div\>  
                \<p className\="text-cyan-700 mt-2 leading-snug"\>  
                  When X win rate drops below 60% on Hard, O's pattern has structure X can't easily crack — note and study that O sequence.  
                \</p\>  
              \</div\>

            \</div\>  
          \</div\>  
        )}

      \</div\>  
    );  
  }

  ReactDOM.createRoot(document.getElementById('root')).render(\<App /\>);  
  \</script\>  
\</body\>  
\</html\>

