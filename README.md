<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Aello — Reset password</title>
<style>
  :root {
    --bg: #0F0F0F;
    --panel: #141414;
    --accent: #C9A85C;
    --muted: #9a9a9a;
    --text: #fff;
    --radius: 10px;
    font-family: "Segoe UI", Roboto, Arial, sans-serif;
  }
  html,body{height:100%;margin:0;background:var(--bg);color:var(--text)}
  .wrap{max-width:900px;margin:28px auto;padding:20px}
  .header{display:flex;align-items:center;gap:12px}
  .logo{width:56px;height:56px;border-radius:8px;background:linear-gradient(180deg,#0b0b0b,#1b1b1b);display:flex;align-items:center;justify-content:center;border:2px solid rgba(201,168,92,0.08);font-weight:700;color:var(--muted)}
  h1{margin:0;font-size:28px;color:var(--accent)}
  .subtitle{color:var(--muted);margin-top:6px}
  .panel{background:var(--panel);padding:20px;border-radius:var(--radius);border:1px solid rgba(201,168,92,0.08);margin-top:16px}
  label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
  input[type=password]{width:100%;padding:12px 14px;border-radius:8px;border:1px solid #222;background:#0c0c0c;color:var(--text);font-size:15px}
  .btn{background:#000;color:var(--accent);border:1px solid var(--accent);padding:10px 18px;border-radius:24px;cursor:pointer;font-weight:600}
  .btn:active{transform:translateY(1px)}
  .btn-primary{background:var(--accent);color:#000;border:0}
  .muted{color:var(--muted);font-size:13px}
  .debug{margin-top:14px;padding:10px;background:#080808;border-radius:8px;color:#ccc;font-size:12px;white-space:pre-wrap;max-height:260px;overflow:auto}
  .notice{padding:12px;border-radius:8px;background:#0b0b0b;border:1px solid rgba(201,168,92,0.06);margin-bottom:12px}
  .center{display:flex;justify-content:flex-end;gap:12px;margin-top:12px}
  .success{background:#093;color:#001;padding:12px;border-radius:8px}
  .error{background:#600;color:#fff;padding:12px;border-radius:8px}
  .hidden{display:none}
  .smallmuted{font-size:12px;color:#9a9a9a;margin-top:8px}
</style>
</head>
<body>
  <div class="wrap">
    <div class="header">
      <div class="logo">Ae</div>
      <div>
        <h1>Aello — Reset your password</h1>
        <div class="subtitle">Enter a new password. The token from your email must be present in the URL.</div>
      </div>
    </div>

    <div id="tokenNotice" class="notice hidden"></div>

    <div id="panel" class="panel">
      <div id="intro" class="muted">Looking for reset token in URL...</div>

      <div style="margin-top:14px">
        <label for="pw">New password</label>
        <input id="pw" type="password" autocomplete="new-password" placeholder="At least 8 characters" />
      </div>

      <div style="margin-top:12px">
        <label for="pw2">Confirm</label>
        <input id="pw2" type="password" autocomplete="new-password" placeholder="Repeat new password" />
      </div>

      <div class="center">
        <button id="cancelBtn" class="btn">Cancel</button>
        <button id="submitBtn" class="btn btn-primary">Set new password</button>
      </div>

      <div id="result" style="margin-top:12px"></div>
      <div id="debug" class="debug"></div>
      <div class="smallmuted">If you open this page in an iframe (Google Sites embed), the token may not be visible. Open the reset link in a top-level tab.</div>
    </div>
  </div>

<script>
/* -------------------------
   CONFIG: fill your project
   ------------------------- */
const SUPABASE_URL = 'https://rbyjwdvnfbdtavxwkqyx.supabase.co'; // <<--- set this
const SUPABASE_ANON_KEY = ''; // optional: your anon key for a retry (leave blank to not send it)

/* -------------------------
   Helper debug output
   ------------------------- */
function dbg(...args){
  const el = document.getElementById('debug');
  const text = args.map(a => typeof a === 'object' ? JSON.stringify(a, null, 2) : String(a)).join(' ');
  el.textContent += text + '\\n';
  console.log(...args);
}

/* -------------------------
   Token extraction (fragment & query)
   ------------------------- */
function extractToken() {
  try {
    const hash = window.location.hash || '';
    dbg('location.hash ->', hash);
    if (hash) {
      const cleaned = hash.startsWith('#') ? hash.slice(1) : hash;
      const params = new URLSearchParams(cleaned);
      // common names
      const names = ['access_token','id_token','token','recovery_token','refresh_token'];
      for (const n of names) {
        if (params.get(n)) return { token: params.get(n), source: 'hash', name: n };
      }
      // also try generic 'access_token=...' pattern
      const m = cleaned.match(/access_token=([^&]+)/) || cleaned.match(/id_token=([^&]+)/);
      if (m) return { token: decodeURIComponent(m[1]), source: 'hash', name: 'regex' };
    }

    const search = window.location.search || '';
    dbg('location.search ->', search);
    if (search) {
      const params = new URLSearchParams(search);
      if (params.get('access_token')) return { token: params.get('access_token'), source: 'search', name: 'access_token' };
      if (params.get('token')) return { token: params.get('token'), source: 'search', name: 'token' };
    }

    // Last-ditch: try pulling from full URL after '#'
    const full = window.location.href || '';
    const frag = full.split('#')[1] || '';
    if (frag) {
      const m2 = frag.match(/access_token=([^&]+)/) || frag.match(/id_token=([^&]+)/);
      if (m2) return { token: decodeURIComponent(m2[1]), source: 'href', name: 'fallback' };
    }
  } catch(e) {
    dbg('extractToken error', e);
  }
  return null;
}

/* -------------------------
   JWT decode (to inspect exp etc)
   ------------------------- */
function decodeJwtPayload(tok) {
  try {
    const parts = tok.split('.');
    if (parts.length < 2) return null;
    const payload = parts[1].replace(/-/g,'+').replace(/_/g,'/');
    const json = decodeURIComponent(atob(payload).split('').map(function(c){
      return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
    }).join(''));
    return JSON.parse(json);
  } catch(e) { return null; }
}

function setStatus(html, kind) {
  const el = document.getElementById('result');
  el.innerHTML = html || '';
  el.className = kind || '';
}

/* -------------------------
   Core: attempt to update password (PATCH /auth/v1/user)
   Will try once with Authorization Bearer; on 401 will retry adding apikey header if configured.
   ------------------------- */
async function patchPassword(token, newPassword, attemptWithApiKey=false){
  const url = SUPABASE_URL.replace(/\/$/,'') + '/auth/v1/user';
  dbg('PATCH url =>', url);
  let headers = { 'Content-Type': 'application/json', 'Authorization': 'Bearer ' + token };
  if (attemptWithApiKey && SUPABASE_ANON_KEY) {
    headers['apikey'] = SUPABASE_ANON_KEY;
  }
  try {
    const resp = await fetch(url, {
      method: 'PATCH',
      headers,
      body: JSON.stringify({ password: newPassword })
    });
    const body = await resp.text();
    dbg('PATCH status:', resp.status, resp.statusText);
    dbg('PATCH body:', body);
    return { status: resp.status, body };
  } catch (err) {
    dbg('PATCH exception:', err);
    throw err;
  }
}

/* -------------------------
   UI wiring
   ------------------------- */
document.addEventListener('DOMContentLoaded', () => {
  if (!SUPABASE_URL || SUPABASE_URL.includes('REPLACE')) {
    setStatus('<div class="error">SUPABASE_URL not configured in the page. Edit the file and set your project URL.</div>', 'error');
    dbg('SUPABASE_URL not set');
    return;
  }

  const tokenInfo = extractToken();
  dbg('tokenInfo:', tokenInfo);
  const intro = document.getElementById('intro');
  const tn = document.getElementById('tokenNotice');

  if (!tokenInfo || !tokenInfo.token) {
    intro.textContent = 'No reset token found in URL. Open the link from your email in a top-level tab.';
    tn.classList.remove('hidden');
    tn.textContent = 'No reset token found in this page. Do not embed this page in an iframe; open it directly.';
    dbg('No token present. location.href:', location.href);
    return;
  }

  const tok = tokenInfo.token;
  const claims = decodeJwtPayload(tok);
  dbg('decoded claims:', claims);

  intro.textContent = 'Token detected. Enter a new password and click "Set new password".';
  tn.classList.add('hidden');

  // show some claim details for debugging
  if (claims) {
    const notice = document.getElementById('tokenNotice');
    notice.textContent = 'Token detected (exp: ' + (claims.exp ? new Date(claims.exp * 1000).toLocaleString() : 'unknown') + ').';
    notice.classList.remove('hidden');
  }

  const submit = document.getElementById('submitBtn');
  const cancel = document.getElementById('cancelBtn');

  cancel.addEventListener('click', () => { window.location.href = '/'; });

  submit.addEventListener('click', async () => {
    const pw = document.getElementById('pw').value || '';
    const pw2 = document.getElementById('pw2').value || '';
    setStatus('', '');
    if (pw.length < 8) { setStatus('<div class="error">Password must be at least 8 characters.</div>','error'); return; }
    if (pw !== pw2) { setStatus('<div class="error">Passwords do not match.</div>','error'); return; }

    submit.disabled = true; cancel.disabled = true;
    setStatus('<div class="muted">Submitting…</div>');

    try {
      // 1) first try without apikey (preferred)
      let res = await patchPassword(tok, pw, false);
      dbg('first attempt result:', res);

      if (res.status >= 200 && res.status < 300) {
        setStatus('<div class="success">Password updated — you can now sign in with your new password.</div>');
        document.getElementById('pw').value = '';
        document.getElementById('pw2').value = '';
        submit.disabled = false; cancel.disabled = false;
        return;
      }

      // 2) if 401 and anon key present, retry adding apikey header (helps some setups)
      if ((res.status === 401 || res.status === 403) && SUPABASE_ANON_KEY) {
        dbg('Received 401/403 — retrying with apikey header');
        res = await patchPassword(tok, pw, true);
        if (res.status >= 200 && res.status < 300) {
          setStatus('<div class="success">Password updated — you can now sign in with your new password.</div>');
          submit.disabled = false; cancel.disabled = false;
          return;
        }
      }

      // default error handling: show returned body or status
      setStatus('<div class="error">Failed to update password (status ' + res.status + ').</div>','error');
      dbg('Final response:', res);
    } catch (err) {
      dbg('exception during reset:', err);
      setStatus('<div class="error">Network error: ' + (err && err.message ? err.message : err) + '</div>','error');
    } finally {
      submit.disabled = false;
      cancel.disabled = false;
    }
  });
});
</script>
</body>
</html>
