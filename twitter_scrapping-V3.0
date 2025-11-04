(function () {
  // ===================== CONFIG =====================
  let maxTweets   = 1000;
  let batchSize   = 5;
  let maxAttempts = 200;
  let maxTimeMs   = 10 * 60 * 1000; // 10 min
  let scrollDelay = 1000;

  // Auto-stop por inatividade
  const IDLE_GRACE_MS = 30_000;   // encerra após 30s sem novos posts
  const IDLE_WARN_MS  = 20_000;   // começa aviso aos 20s (mostra contagem 10s)

  // =============== STATE / RUNTIME ==================
  let tweetsData = [];
  let tweetsLoaded = 0;
  let attempts = 0;
  let startTime = Date.now();
  let isPaused = false;
  let finished = false;

  let idleStart = null;
  let idleExtra = 0;

  // Controles manuais
  window.PauseScraping = () => { isPaused = true; updateUI(); };
  window.ResumeScraping = () => { isPaused = false; updateUI(); };
  window.ResumeScrapping = window.ResumeScraping;

  // ================== UI FLOAT PANEL =================
  const ui = (() => {
    const box = document.createElement('div');
    box.id = 'giba-scraper-ui';
    box.style.cssText = `
      position:fixed; right:16px; bottom:16px; z-index:999999;
      width:320px; background:#0f1419; color:#e6ecf0; font:13px/1.4 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu;
      border:1px solid #263340; border-radius:12px; box-shadow:0 6px 24px rgba(0,0,0,.35); overflow:hidden;
    `;
    box.innerHTML = `
      <div style="padding:10px 12px; display:flex; align-items:center; justify-content:space-between; background:#15202b;">
        <strong>Twitter Scraper • GOVRJ</strong>
        <button id="giba-close" style="all:unset; cursor:pointer; padding:4px 6px; border-radius:6px; border:1px solid #2f3b44;">×</button>
      </div>
      <div style="padding:12px;">
        <div style="margin-bottom:8px; display:flex; justify-content:space-between;">
          <span><b>Coletados:</b> <span id="giba-count">0</span>/<span id="giba-max">${maxTweets}</span></span>
          <span><b>Tentativas:</b> <span id="giba-attempts">0</span></span>
        </div>
        <div style="height:8px; background:#22303c; border-radius:999px; overflow:hidden; margin-bottom:8px;">
          <div id="giba-bar" style="height:8px; width:0%; background:#1d9bf0;"></div>
        </div>
        <div style="display:flex; justify-content:space-between; margin-bottom:12px;">
          <span><b>Tempo:</b> <span id="giba-time">00:00</span></span>
          <span id="giba-state" style="opacity:.85">rodando…</span>
        </div>

        <div id="giba-idle-wrap" style="display:none; border:1px solid #3a4a56; border-radius:10px; padding:10px; margin-bottom:10px; background:#13202b;">
          <div style="display:flex; justify-content:space-between; align-items:center; gap:8px; margin-bottom:6px;">
            <span>Sem novos posts. Encerrando em <b id="giba-idle-count">10</b>s…</span>
            <button id="giba-extend" style="all:unset; cursor:pointer; padding:6px 10px; border-radius:8px; border:1px solid #2f3b44; background:#192734;">+1 min</button>
          </div>
          <div style="height:6px; background:#22303c; border-radius:999px; overflow:hidden;">
            <div id="giba-idle-bar" style="height:6px; width:0%; background:#ffad1f;"></div>
          </div>
        </div>

        <div style="display:flex; gap:8px;">
          <button id="giba-toggle" style="flex:1; padding:8px 10px; border-radius:8px; border:1px solid #2f3b44; background:#192734; color:#e6ecf0; cursor:pointer;">Pausar</button>
          <button id="giba-download" style="flex:1; padding:8px 10px; border-radius:8px; border:1px solid #2f3b44; background:#192734; color:#e6ecf0; cursor:pointer;" disabled>Baixar CSV</button>
        </div>
      </div>
    `;
    document.body.appendChild(box);

    box.querySelector('#giba-close').onclick = () => box.remove();
    box.querySelector('#giba-toggle').onclick = () => { isPaused ? window.ResumeScraping() : window.PauseScraping(); };
    box.querySelector('#giba-download').onclick = () => downloadCSV(tweetsData);
    box.querySelector('#giba-extend').onclick = () => { idleExtra += 60_000; updateUI(true); };

    const $ = (sel) => box.querySelector(sel);
    return {
      setCounts: (loaded, max, att) => {
        $('#giba-count').textContent = String(loaded);
        $('#giba-max').textContent = String(max);
        $('#giba-attempts').textContent = String(att);
        const pct = Math.min(100, Math.round((loaded / max) * 100));
        $('#giba-bar').style.width = pct + '%';
        $('#giba-download').disabled = loaded <= 0;
      },
      setTime: (ms) => {
        const s = Math.floor(ms / 1000);
        const mm = String(Math.floor(s / 60)).padStart(2, '0');
        const ss = String(s % 60).padStart(2, '0');
        $('#giba-time').textContent = `${mm}:${ss}`;
      },
      setState: (txt) => {
        $('#giba-state').textContent = txt;
        $('#giba-toggle').textContent = isPaused ? 'Retomar' : 'Pausar';
      },
      showIdleWarning: (show, restMs = 0, totalMs = 10_000) => {
        const el = $('#giba-idle-wrap');
        el.style.display = show ? 'block' : 'none';
        if (!show) return;
        const sLeft = Math.max(0, Math.ceil(restMs / 1000));
        $('#giba-idle-count').textContent = String(sLeft);
        const pct = Math.max(0, Math.min(100, Math.round(((totalMs - restMs) / totalMs) * 100)));
        $('#giba-idle-bar').style.width = pct + '%';
      }
    };
  })();

  function updateUI(forceIdle = false) {
    ui.setCounts(tweetsLoaded, maxTweets, attempts);
    ui.setTime(Date.now() - startTime);
    ui.setState(finished ? 'concluído' : (isPaused ? 'pausado' : 'rodando…'));

    const now = Date.now();
    if (idleStart != null) {
      const elapsed = now - idleStart;
      const warnAt   = IDLE_WARN_MS;
      const deadline = IDLE_GRACE_MS + idleExtra;
      if (elapsed >= warnAt) {
        const remaining = Math.max(0, deadline - elapsed);
        const warnWindow = deadline - warnAt;
        ui.showIdleWarning(true, remaining, warnWindow);
      } else if (forceIdle) {
        ui.showIdleWarning(false);
      }
    } else {
      ui.showIdleWarning(false);
    }
  }
  const uiTimer = setInterval(updateUI, 300);

  // ================== HELPERS =========================
  function normalizeNumber(str) {
    if (!str) return '0';
    // remove espaços, tratar "mil", "k", "K", etc.
    let s = String(str).trim();
    // casos como "1 mil", "1,2 mil"
    if (/mil/i.test(s)) {
      const n = parseFloat(s.replace(/mil/i,'').replace(/\./g,'').replace(',','.')) || 0;
      return String(Math.round(n * 1000));
    }
    // casos K/M (backup)
    const km = s.match(/^([\d.,]+)\s*([kKmM])$/);
    if (km) {
      const n = parseFloat(km[1].replace(/\./g,'').replace(',','.')) || 0;
      const mul = km[2].toLowerCase() === 'k' ? 1e3 : 1e6;
      return String(Math.round(n * mul));
    }
    // número "cheio" possivelmente com . e ,
    s = s.replace(/[^\d]/g, '');
    return s || '0';
  }

  function numberBefore(aria, keywords) {
    if (!aria) return null;
    const kw = keywords.map(k => k.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')).join('|');
    const re = new RegExp(`(\\d[\\d\\.,\\s]*)\\s+(?:${kw})`, 'i');
    const m = aria.match(re);
    return m ? normalizeNumber(m[1]) : null;
  }

  // ================== EXTRACTORS =====================
  function extractTweetData(tweetElement) {
    try {
      const usernameElement = tweetElement.querySelector('div[data-testid="User-Name"] a');
      const username = usernameElement?.textContent?.trim() ?? '';
      const userId = usernameElement?.getAttribute('href')?.split('/').pop() ?? '';
      const tweetText = tweetElement.querySelector('div[data-testid="tweetText"]')?.textContent?.trim() ?? '';
      const tweetUrl = tweetElement.querySelector('a[href*="/status/"]')?.href ?? '';

      const timestampElement = tweetElement.querySelector('time');
      const tweetDateTime = timestampElement?.getAttribute('datetime') ?? '';
      let tweetDate = '', tweetTimeFormatted = '';
      if (tweetDateTime.includes('T')) {
        const [d, t] = tweetDateTime.split('T');
        tweetDate = d;
        tweetTimeFormatted = t.replace('Z','');
      }

      // 1) Tenta via aria-label (mais preciso)
      let replies = '0', retweets = '0', likes = '0', views = '0';
      const metricsGroup = tweetElement.querySelector('div[role="group"][aria-label]');
      const aria = metricsGroup?.getAttribute('aria-label') || '';

      if (aria) {
        replies  = numberBefore(aria, ['respostas','replies'])       ?? '0';
        retweets = numberBefore(aria, ['reposts','retweets'])        ?? '0';
        likes    = numberBefore(aria, ['curtidas','likes'])          ?? '0';
        views    = numberBefore(aria, ['visualizações','views'])     ?? '0';
      }

      // 2) Backup: data-testid (caso aria não exista/bug)
      const repliesBtn  = tweetElement.querySelector('button[data-testid="reply"]');
      const retweetsBtn = tweetElement.querySelector('button[data-testid="retweet"]');
      const likesBtn    = tweetElement.querySelector('button[data-testid="like"]');
      const viewsA      = tweetElement.querySelector('a[href*="/analytics"]');

      if ((!replies || replies === '0') && repliesBtn) {
        replies = normalizeNumber(repliesBtn.querySelector('span')?.textContent ?? '0');
      }
      if ((!retweets || retweets === '0') && retweetsBtn) {
        retweets = normalizeNumber(retweetsBtn.querySelector('span')?.textContent ?? '0');
      }
      if ((!likes || likes === '0') && likesBtn) {
        likes = normalizeNumber(likesBtn.querySelector('span')?.textContent ?? '0');
      }
      if ((!views || views === '0') && viewsA) {
        const ariaViews = viewsA.getAttribute('aria-label') || '';
        const m = ariaViews.match(/([\d.,\s]+)(?=\s+(visualizações|views))/i);
        views = m ? normalizeNumber(m[1]) : '0';
      }

      return { username, userId, tweetText, replies, retweets, likes, views, tweetUrl, tweetDate, tweetTime: tweetTimeFormatted };
    } catch (e) {
      console.error('Erro ao extrair um tweet:', e);
      return null;
    }
  }

  function extractAllTweets() {
    const tweetElements = document.querySelectorAll('article[data-testid="tweet"]');
    let newTweets = 0;
    tweetElements.forEach(el => {
      const data = extractTweetData(el);
      if (data && !tweetsData.some(t => t.tweetUrl === data.tweetUrl)) {
        tweetsData.push(data);
        newTweets++;
      }
    });
    tweetsLoaded += newTweets;
    return newTweets;
  }

  // ===================== CSV HELPER ==================
  function toCSV(rows, headers) {
    const esc = (v) => {
      const s = v == null ? '' : String(v);
      const needsQuote = /[",\n;]/.test(s);
      const cleaned = s.replace(/"/g, '""');
      return needsQuote ? `"${cleaned}"` : cleaned;
    };
    const head = headers.map(esc).join(';');
    const body = rows.map(r => headers.map(h => esc(r[h])).join(';')).join('\n');
    return head + '\n' + body;
  }

  function downloadCSV(data) {
    if (!data || !data.length) return;
    const rows = data.map((d, i) => ({ N: i + 1, ...d }));
    const headers = ['N','userId','tweetText','replies','retweets','likes','views','tweetUrl','tweetDate','tweetTime','username'];
    const csv = toCSV(rows, headers);
    const blob = new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8;' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    const ts = new Date().toISOString().replace(/[:.]/g,'-');
    a.download = `twitter_scrape_${ts}.csv`;
    document.body.appendChild(a);
    a.click();
    a.remove();
  }

  // ===================== LOOP ========================
  function shouldStop() {
    const timeUp = (Date.now() - startTime) >= maxTimeMs;

    let idleExpired = false;
    if (idleStart != null) {
      const elapsed = Date.now() - idleStart;
      const deadline = IDLE_GRACE_MS + idleExtra;
      idleExpired = elapsed >= deadline;
    }

    return finished || tweetsLoaded >= maxTweets || attempts >= maxAttempts || timeUp || idleExpired;
  }

  function finalize() {
    finished = true;
    updateUI();
    clearInterval(uiTimer);
    console.log('Coleta finalizada. Tweets coletados:', tweetsData.length);
    console.table(tweetsData.slice(0, 10));
    try { downloadCSV(tweetsData); } catch(e){ console.warn('Falha no download automático do CSV:', e); }
  }

  function tick() {
    if (shouldStop()) { finalize(); return; }
    if (isPaused) { updateUI(); return setTimeout(tick, scrollDelay); }

    const newTweets = extractAllTweets();

    if (newTweets > 0) {
      idleStart = null;
      idleExtra = 0;
      ui.showIdleWarning(false);
    } else if (idleStart == null) {
      idleStart = Date.now();
    }

    if (newTweets < batchSize) {
      window.scrollBy(0, Math.floor(window.innerHeight * 0.5));
    }

    attempts++;
    updateUI();
    setTimeout(tick, scrollDelay);
  }

  console.log('%cIniciando scraping…', 'color:#1d9bf0');
  tick();
})();
