<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Reset your StudyGate password</title>
  <style>
    :root{
      --bg:#0F0F0F;
      --panel:#141414;
      --accent:#C9A85C;
      --muted:#9a9a9a;
      --text:#fff;
      --radius:10px;
      font-family: "Segoe UI", Roboto, Arial, sans-serif;
    }
    html,body{height:100%;margin:0;background:var(--bg);color:var(--text)}
    .wrap{max-width:880px;margin:40px auto;padding:28px;}
    .header{display:flex;align-items:center;gap:14px}
    .logo{width:56px;height:56px;border-radius:8px;background:linear-gradient(180deg,#0b0b0b,#1b1b1b);display:flex;align-items:center;justify-content:center;border:2px solid rgba(255,255,255,0.04)}
    h1{margin:0;font-size:20px;color:var(--accent)}
    .panel{background:var(--panel);padding:22px;border-radius:var(--radius);border:1px solid rgba(201,168,92,0.12);margin-top:18px}
    label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
    input[type=password]{width:100%;padding:12px 14px;border-radius:8px;border:1px solid #222;background:#0c0c0c;color:var(--text);font-size:15px}
    .btn{background:#000;color:var(--accent);border:1px solid var(--accent);padding:10px 18px;border-radius:20px;cursor:pointer;font-weight:600}
    .btn:active{transform:translateY(1px)}
    .btn-primary{background:var(--accent);color:#000;border:0}
    .muted{color:var(--muted);font-size:13px}
    .debug{margin-top:14px;padding:10px;background:#080808;border-radius:8px;color:#ccc;font-size:12px;white-space:pre-wrap;max-height:220px;overflow:auto}
    .notice{padding:12px;border-radius:8px;background:#0b0b0b;border:1px solid rgba(201,168,92,0.08);margin-bottom:14px}
    .center{display:flex;justify-content:flex-end;gap:10px;margin-top:14px}
    .success{background: #093; color:#001; padding:12px;border-radius:8px}
    .error{background:#600;color:#fff;padding:12px;border-radius:8px}
    .hidden{display:none}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="header">
      <div class="logo">SG</div>
      <div>
        <h1>Reset your StudyGate password</h1>
        <div class="muted">Follow the instructions below to choose a new password.</div>
      </div>
    </div>

    <div id="tokenNotice" class="notice hidden"></div>

    <div id="panel" class="panel">
      <div id="intro" class="muted">Detecting token...</div>

      <div style="margin-top:12px">
        <label for="pw">New password</label>
        <input id="pw" type="password" autocomplete="new-password" placeholder="Enter a secure new password" />
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
    </div>
  </div>

<script>
/*
  IMPORTANT: set SUPABASE_URL to your project's URL (example from your app):
  const SUPABASE_URL = 'https://rbyjwdvnfbdtavxwkqyx.supabase.co';
  No API key is needed for the PATCH /auth/v1/user call when using the access_token.
*/
const SUPABASE_URL = 'https://rbyjwdvnfbdtavxwkqyx.supabase.co'; // <-- edit before deploying

// Utility: append debug lines
function dbg(...args){
  const el=document.getElementById('debug');
  el.textContent += args.map(a=>typeof a==='object'?JSON.stringify(a,null,2):String(a)).join(' ') + '\n';
  console.log(...args);
}

// Try to extract an access token from the fragment (#...) or query (?...) or from full url
function extractToken(){
  try {
    // Fragment style: #access_token=...&expires_in=...
    const hash = window.location.hash || '';
    dbg('location.hash ->', hash);
    if (hash){
      const cleaned = hash.startsWith('#') ? hash.slice(1) : hash;
      const params = new URLSearchParams(cleaned);
      if (params.get('access_token')) return params.get('access_token');
      if (params.get('token')) return params.get('token');
      if (params.get('refresh_token')) return params.get('refresh_token'); // not ideal, but try
    }

    // Query style: ?access_token=...
    const q = window.location.search || '';
    dbg('location.search ->', q);
    if (q){
      const params = new URLSearchParams(q);
      if (params.get('access_token')) return params.get('access_token');
    }

    // sometimes token is provided as #id_token= or #recovery_token; check generically:
    if (hash){
      const match = hash.match(/access_token=([^&]+)/) || hash.match(/id_token=([^&]+)/) || hash.match(/recovery=([^&]+)/i);
      if (match) return decodeURIComponent(match[1]);
    }
  } catch(e){
    dbg('extractToken error', e)
  }
  return null;
}

function setStatus(message, kind){
  const r = document.getElementById('result');
  r.innerHTML = message;
  r.className = kind ? kind : '';
}

async function doReset(token, newPassword){
  const url = SUPABASE_URL.replace(/\/$/,'') + '/auth/v1/user';
  dbg('PATCH ->', url);
  const body = { password: newPassword };
  try{
    const resp = await fetch(url, {
      method: 'PATCH',
      headers: {
        'Content-Type':'application/json',
        'Authorization':'Bearer ' + token
      },
      body: JSON.stringify(body)
    });
    dbg('response status', resp.status, resp.statusText);
    const text = await resp.text();
    dbg('response body:', text);

    if (resp.status >= 200 && resp.status < 300){
      setStatus('<div class="success">Password updated — you can now sign in with your new password.</div>');
      document.getElementById('pw').value = '';
      document.getElementById('pw2').value = '';
      return true;
    } else {
      let errMsg = `Failed to update password (status ${resp.status}).`;
      try { const json = JSON.parse(text); if (json?.msg) errMsg = json.msg; } catch {}
      setStatus(`<div class="error">${escapeHtml(errMsg)}</div>`);
      return false;
    }
  }catch(err){
    dbg('network error', err);
    setStatus(`<div class="error">Network error: ${escapeHtml(err.message || err)}</div>`);
    return false;
  }
}

function escapeHtml(s){ return String(s).replace(/[&<>"']/g, c=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' })[c]); }

document.addEventListener('DOMContentLoaded', () => {
  const token = extractToken();
  const intro = document.getElementById('intro');
  const tn = document.getElementById('tokenNotice');
  const submit = document.getElementById('submitBtn');
  const cancel = document.getElementById('cancelBtn');

  if (!SUPABASE_URL || SUPABASE_URL.includes('REPLACE_WITH_YOUR_SUPABASE_URL')){
    setStatus('<div class="error">You must edit this file and set SUPABASE_URL to your Supabase project URL before deploying.</div>');
    dbg('SUPABASE_URL not set');
    return;
  }

  if (!token){
    intro.textContent = 'No access token found in the URL — open the link from your email. (The token should appear in the URL fragment, e.g. #access_token=...)';
    tn.classList.remove('hidden');
    tn.textContent = 'No reset token found in this page. Make sure you opened the original link from the email, and that the page is loaded as a top-level page (not embedded).';
    dbg('token not found. location.href:', location.href);
    return;
  }

  dbg('Token detected (truncated):', token && token.slice(0,30) + '...');

  intro.textContent = 'Token detected. Enter a new password and press "Set new password".';
  tn.classList.add('hidden');

  submit.addEventListener('click', async () => {
    const pw = document.getElementById('pw').value || '';
    const pw2 = document.getElementById('pw2').value || '';
    setStatus('');
    if (pw.length < 8){
      setStatus('<div class="error">Password must be at least 8 characters.</div>');
      return;
    }
    if (pw !== pw2){
      setStatus('<div class="error">Passwords do not match.</div>');
      return;
    }
    submit.disabled = true;
    cancel.disabled = true;
    setStatus('<div class="muted">Submitting — please wait…</div>');
    const ok = await doReset(token, pw);
    submit.disabled = false;
    cancel.disabled = false;
    if (ok) {
      // show link to login
      const r = document.getElementById('result');
      r.innerHTML += '<div style="margin-top:10px" class="muted">You may now <a href="/" style="color:var(--accent)">sign in</a>.</div>';
    }
  });

  cancel.addEventListener('click', () => { window.location.href = '/'; });
});
</script>
</body>
</html>
