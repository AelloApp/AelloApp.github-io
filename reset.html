<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Reset password</title>
  <style>
    :root{
      --bg:#0E0E0E; --panel:#141414; --accent:#C9A85C; --text:#FFF;
    }
    body{margin:0;background:var(--bg);color:var(--text);font-family:Segoe UI,Roboto,Arial;}
    .wrap{max-width:980px;margin:34px auto;padding:18px;}
    .panel{background:var(--panel);padding:20px;border-radius:10px;border:1px solid rgba(201,168,92,0.85)}
    h1{margin:0 0 8px 0;font-size:20px}
    .small{font-size:13px;color:#d7d0b3}
    input{width:100%;padding:12px;border-radius:8px;border:1px solid #222;background:transparent;color:var(--text);box-sizing:border-box}
    .controls{display:flex;justify-content:flex-end;gap:10px;margin-top:12px}
    button{background:#000;border:1px solid var(--accent);color:var(--accent);padding:10px 16px;border-radius:20px;cursor:pointer}
    button:disabled{opacity:.6}
    .status{margin-top:12px;font-size:14px}
    .dbg{margin-top:14px;padding:10px;background:#0b0b0b;border-radius:6px;border:1px dashed #222;white-space:pre-wrap;font-family:monospace;font-size:12px}
    .success{color:var(--accent)}
    .error{color:salmon}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="panel">
      <h1>Reset your password</h1>
      <p class="small">Open the link sent to your email and use this page to set a new password.</p>

      <div id="noToken" style="display:none">
        <p class="small">No token detected in URL. If you clicked the email link and still see this message, check the debug box below and confirm the address bar includes <code>#access_token=...</code>.</p>
      </div>

      <div id="haveToken" style="display:none">
        <label class="small">New password</label>
        <input id="pw" type="password" placeholder="Min 6 characters"/>
        <label class="small" style="margin-top:8px">Confirm password</label>
        <input id="pw2" type="password" placeholder="Repeat password"/>
        <div class="controls">
          <button id="cancelBtn" style="background:transparent;border:1px solid #333;color:#fff">Cancel</button>
          <button id="submitBtn">Set new password</button>
        </div>
        <div id="result" class="status"></div>
      </div>

      <div style="margin-top:12px">
        <div class="small">Debug (visible intentionally — shows what this page sees):</div>
        <div id="dbg" class="dbg"></div>
      </div>
    </div>
  </div>

<script>
/*
  Hardened reset.html
  - Update PROJECT_URL if needed (your Supabase project URL)
  - This script attempts many extraction strategies and prints debug info
*/

(async function(){
  // === CONFIG ===
  const PROJECT_URL = 'https://rbyjwdvnfbdtavxwkqyx.supabase.co'; // <-- change here if needed

  // === helpers ===
  function logDbg(obj){
    const el = document.getElementById('dbg');
    try{
      el.textContent = typeof obj === 'string' ? obj : JSON.stringify(obj, null, 2);
    }catch(e){
      el.textContent = String(obj);
    }
    console.log('[reset.html debug]', obj);
  }

  function parseKVFromString(s, keys){
    if(!s) return null;
    try{
      const dec = decodeURIComponent(s);
      if(dec !== s) console.log('decoded fragment', dec);
    }catch(e){}
    // try URLSearchParams on substring that might contain ? or &
    try{
      const q = new URLSearchParams(s.replace(/^.*[?#]/,''));
      for(const k of keys) if(q.has(k)) return {key:k, val:q.get(k), source:'URLSearchParams'};
    }catch(e){}
    // regex fallback
    for(const k of keys){
      const re = new RegExp(k + '=([^&\\s#/]+)', 'i');
      const m = s.match(re);
      if(m && m[1]) return {key:k, val:decodeURIComponent(m[1]), source:'regex'};
    }
    return null;
  }

  function extractTokenAllWays(){
    const out = { href:window.location.href, hash:window.location.hash, search:window.location.search, pathname:window.location.pathname, attempts:[] };
    // direct hash / search
    let keys = ['access_token','token','refresh_token'];
    out.attempts.push({method:'raw hash', value:window.location.hash});
    out.attempts.push({method:'raw search', value:window.location.search});
    // 1) try location.hash
    let found = parseKVFromString(window.location.hash.replace(/^#/,''), keys);
    if(found){ out.attempts.push({step:1, found}); return { token:found.val, debug:out }; }
    // 2) try entire href (after first '#')
    const afterHash = window.location.href.split('#').slice(1).join('#');
    out.attempts.push({method:'afterHash', value:afterHash});
    found = parseKVFromString(afterHash, keys);
    if(found){ out.attempts.push({step:2, found}); return { token:found.val, debug:out }; }
    // 3) cleaned variants (remove leading / or !/)
    const cleaned = afterHash.replace(/^\/+|^!\/+/,'');
    out.attempts.push({method:'cleaned', value:cleaned});
    found = parseKVFromString(cleaned, keys);
    if(found){ out.attempts.push({step:3, found}); return { token:found.val, debug:out }; }
    // 4) fallback: search anywhere in href for access_token=
    const m = (window.location.href.match(/access_token=([^&]+)/i) || window.location.hash.match(/access_token=([^&]+)/i) || (afterHash.match(/access_token=([^&]+)/i) || null));
    if(m && m[1]){ out.attempts.push({step:4, value:m[1].slice(0,60)+'...' }); try{ return { token:decodeURIComponent(m[1]), debug:out }; }catch(e){ return {token:m[1], debug:out}; } }
    // nothing
    out.attempts.push({step:'not found'});
    return { token:null, debug:out };
  }

  // run extraction
  const res = extractTokenAllWays();
  logDbg(res.debug);

  // If token found: store and remove fragment for cleanliness
  if(res.token){
    try{
      sessionStorage.setItem('supabase_access_token', res.token);
      // remove fragment so refreshing the page won't lose things
      try{ history.replaceState({}, document.title, window.location.pathname + window.location.search); }catch(e){}
    }catch(e){ console.warn('failed to store token', e); }
  }

  // UI: show appropriate panels
  if(res.token){
    document.getElementById('haveToken').style.display='';
    document.getElementById('noToken').style.display='none';
    const pw = document.getElementById('pw');
    const pw2 = document.getElementById('pw2');
    const submit = document.getElementById('submitBtn');
    const cancel = document.getElementById('cancelBtn');
    const result = document.getElementById('result');

    cancel.addEventListener('click', ()=> window.location.href = '/');

    submit.addEventListener('click', async ()=>{
      result.textContent = ''; result.className='status';
      const a = pw.value||''; const b = pw2.value||'';
      if(a.length < 6){ result.textContent='Password must be at least 6 characters'; result.className='status error'; return; }
      if(a !== b){ result.textContent='Passwords do not match'; result.className='status error'; return; }
      submit.disabled = true; submit.textContent = 'Saving...';
      try{
        const token = sessionStorage.getItem('supabase_access_token') || res.token;
        // call Supabase auth user PATCH
        const url = PROJECT_URL + '/auth/v1/user';
        console.log('PATCH', url);
        const r = await fetch(url, {
          method:'PATCH',
          headers:{
            'Content-Type':'application/json',
            'Authorization': 'Bearer ' + token
          },
          body: JSON.stringify({ password: a })
        });
        const txt = await r.text();
        console.log('reset response', r.status, txt);
        if(r.ok){
          result.className = 'status success';
          result.textContent = 'Password updated — you may now sign in.';
          setTimeout(()=> window.location.href = '/', 1400);
        } else {
          result.className = 'status error';
          result.textContent = `Reset failed: ${r.status} ${txt}`;
        }
      }catch(err){
        console.error('network/CORS error', err);
        result.className = 'status error';
        result.textContent = 'Network or CORS error — see console for details: ' + String(err);
      } finally {
        submit.disabled = false; submit.textContent = 'Set new password';
      }
    });

  } else {
    document.getElementById('haveToken').style.display='none';
    document.getElementById('noToken').style.display='';
  }

})();
</script>
</body>
</html>
