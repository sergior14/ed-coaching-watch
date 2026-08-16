import { buildPushPayload } from '@block65/webcrypto-web-push';

const SOLD_OUT = [
  'alle plätze sind ausgebucht',
  'leider sind aktuell alle coaching-plätze ausgebucht'
];

const json = (x, status = 200) => new Response(JSON.stringify(x), {
  status,
  headers: {
    'content-type': 'application/json;charset=UTF-8',
    'cache-control': 'no-store'
  }
});

async function getState(env) {
  return (await env.ED_WATCH.get('state', 'json')) || {
    available: false,
    status: 'unknown',
    checks: 0,
    alerts: 0,
    last_check: null,
    last_change: null,
    evidence: 'Noch nicht geprüft.'
  };
}

async function getSubscriptions(env) {
  return (await env.ED_WATCH.get('subscriptions', 'json')) || [];
}

async function saveSubscriptions(env, subs) {
  await env.ED_WATCH.put('subscriptions', JSON.stringify(subs));
}

async function sendPush(env, title, body) {
  if (!env.VAPID_PRIVATE_KEY) return 0;
  const subs = await getSubscriptions(env);
  let sent = 0;
  const keep = [];

  for (const sub of subs) {
    try {
      const payload = await buildPushPayload(
        {
          data: JSON.stringify({ title, body, url: env.TARGET_URL, tag: 'ed-coaching-slot' }),
          options: { ttl: 300 }
        },
        sub,
        {
          subject: env.VAPID_SUBJECT,
          publicKey: env.VAPID_PUBLIC_KEY,
          privateKey: env.VAPID_PRIVATE_KEY
        }
      );
      const r = await fetch(sub.endpoint, payload);
      if (r.ok) {
        sent++;
        keep.push(sub);
      } else if (![404, 410].includes(r.status)) {
        keep.push(sub);
      }
    } catch {
      keep.push(sub);
    }
  }

  if (keep.length !== subs.length) await saveSubscriptions(env, keep);
  return sent;
}

async function check(env, forceWrite = false) {
  const prev = await getState(env);
  const checkedAt = new Date().toISOString();

  try {
    const r = await fetch(env.TARGET_URL, {
      headers: { 'user-agent': 'Mozilla/5.0 ED-Coaching-Watch/1.0' },
      cf: { cacheTtl: 0 }
    });
    if (!r.ok) throw new Error(`HTTP ${r.status}`);

    const html = (await r.text()).toLowerCase();
    const soldOut = SOLD_OUT.some(p => html.includes(p));
    const available = !soldOut;
    const changed = available !== Boolean(prev.available);

    const next = {
      ...prev,
      available,
      status: available ? 'available' : 'sold_out',
      checks: (prev.checks || 0) + 1,
      last_check: checkedAt,
      last_error: null,
      evidence: soldOut
        ? 'Der Ausgebucht-Hinweis ist weiterhin sichtbar.'
        : 'Der Ausgebucht-Hinweis ist verschwunden – sofort prüfen und buchen!'
    };

    if (changed) next.last_change = checkedAt;
    if (available && !prev.available) {
      next.alerts = (prev.alerts || 0) + 1;
      await sendPush(
        env,
        '🚨 ED Coaching: PLATZ FREI!',
        'Der Ausgebucht-Hinweis ist verschwunden. Jetzt sofort prüfen und buchen.'
      );
    }

    if (changed || forceWrite || (next.checks % 15 === 0)) {
      await env.ED_WATCH.put('state', JSON.stringify(next));
    }

    return next;
  } catch (e) {
    const next = {
      ...prev,
      status: 'error',
      last_check: checkedAt,
      last_error: String(e).slice(0, 300)
    };
    if (forceWrite) await env.ED_WATCH.put('state', JSON.stringify(next));
    return next;
  }
}

const manifest = {
  name: 'ED Coaching Watch',
  short_name: 'ED Watch',
  start_url: '/',
  display: 'standalone',
  background_color: '#ffffff',
  theme_color: '#111111'
};

const serviceWorker = `
self.addEventListener('push', event => {
  let data = {};
  try { data = event.data ? event.data.json() : {}; } catch {}
  event.waitUntil(self.registration.showNotification(data.title || 'ED Coaching Watch', {
    body: data.body || 'Status geändert.',
    tag: data.tag || 'ed-coaching-slot',
    data: { url: data.url || 'https://www.coachingbyed.de/jetzt-buchen/' }
  }));
});
self.addEventListener('notificationclick', event => {
  event.notification.close();
  event.waitUntil(clients.openWindow(event.notification.data?.url || '/'));
});
`;

const html = `<!doctype html>
<html lang="de"><head>
<meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<meta name="theme-color" content="#111111"><link rel="manifest" href="/manifest.json">
<title>ED Coaching Watch</title>
<style>
body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;margin:0;background:#f5f5f7;color:#111}.wrap{max-width:680px;margin:auto;padding:22px}.card{background:#fff;border-radius:22px;padding:22px;margin:14px 0;box-shadow:0 8px 30px rgba(0,0,0,.06)}h1{font-size:30px;margin:8px 0}.badge{display:inline-block;padding:8px 12px;border-radius:999px;background:#eee;font-weight:700}.ok{background:#e8f7ed}.warn{background:#fff0e8}.btn{width:100%;border:0;border-radius:16px;padding:15px 16px;font-size:17px;font-weight:700;margin-top:10px;background:#111;color:#fff}.secondary{background:#e9e9ee;color:#111}.small{color:#666;font-size:14px;line-height:1.45}.row{display:flex;justify-content:space-between;gap:14px;padding:9px 0;border-bottom:1px solid #eee}.row:last-child{border:0}
</style></head><body><div class="wrap">
<div class="card"><div class="small">24/7 Platzwächter</div><h1>ED Coaching Watch 🔔</h1><p class="small">Überwacht automatisch die Coaching-Buchungsseite und meldet sich, sobald der Ausgebucht-Hinweis verschwindet.</p><div id="badge" class="badge">Lade Status…</div></div>
<div class="card"><div class="row"><span>Status</span><strong id="status">–</strong></div><div class="row"><span>Letzter Check</span><strong id="last">–</strong></div><div class="row"><span>Checks</span><strong id="checks">0</strong></div><div class="row"><span>Push-Geräte</span><strong id="subs">0</strong></div><button class="btn" onclick="enablePush()">Push auf diesem iPhone aktivieren</button><button class="btn secondary" onclick="testPush()">Test-Push senden</button><button class="btn secondary" onclick="manualCheck()">Jetzt prüfen</button></div>
<div class="card"><button class="btn" onclick="location.href='https://www.coachingbyed.de/jetzt-buchen/'">Zur Buchungsseite</button><p class="small">Für Push auf dem iPhone: in Safari öffnen → Teilen → Zum Home-Bildschirm → App vom Home-Bildschirm öffnen → Push aktivieren.</p></div>
</div><script>
let vapid='';
const b64ToU8=s=>{const p='='.repeat((4-s.length%4)%4),b=(s+p).replace(/-/g,'+').replace(/_/g,'/'),r=atob(b);return Uint8Array.from([...r].map(c=>c.charCodeAt(0)))};
async function refresh(){const r=await fetch('/api/status',{cache:'no-store'});const s=await r.json();vapid=s.vapid_public_key;document.getElementById('status').textContent=s.status||'–';document.getElementById('last').textContent=s.last_check?new Date(s.last_check).toLocaleString():'Noch nie';document.getElementById('checks').textContent=s.checks||0;document.getElementById('subs').textContent=s.push_subscribers||0;const badge=document.getElementById('badge');badge.textContent=s.available?'🚨 Möglicher Platz frei!':s.status==='sold_out'?'✅ Weiterhin ausgebucht':'⚠️ Status prüfen';badge.className='badge '+(s.available?'warn':'ok')}
async function enablePush(){if(!('serviceWorker'in navigator)||!('PushManager'in window)){alert('Push wird hier nicht unterstützt. Bitte die Seite in Safari zum Home-Bildschirm hinzufügen und von dort öffnen.');return}const reg=await navigator.serviceWorker.register('/sw.js');const perm=await Notification.requestPermission();if(perm!=='granted'){alert('Mitteilungen wurden nicht erlaubt.');return}let sub=await reg.pushManager.getSubscription();if(!sub)sub=await reg.pushManager.subscribe({userVisibleOnly:true,applicationServerKey:b64ToU8(vapid)});const r=await fetch('/api/push/subscribe',{method:'POST',headers:{'content-type':'application/json'},body:JSON.stringify(sub)});const j=await r.json();alert(j.ok?'Push ist aktiviert ✅':'Push konnte nicht aktiviert werden.');refresh()}
async function testPush(){const r=await fetch('/api/test-alert',{method:'POST'});const j=await r.json();alert(j.ok?'Test-Push gesendet ✅':'Noch kein Push-Gerät aktiv oder Push-Schlüssel fehlt.')}
async function manualCheck(){await fetch('/api/check',{method:'POST'});await refresh()}
refresh();setInterval(refresh,30000);
</script></body></html>`;

export default {
  async scheduled(controller, env, ctx) {
    ctx.waitUntil(check(env, false));
  },

  async fetch(request, env) {
    const u = new URL(request.url);

    if (u.pathname === '/api/status') {
      const s = await getState(env);
      const subs = await getSubscriptions(env);
      return json({
        ...s,
        push_subscribers: subs.length,
        vapid_public_key: env.VAPID_PUBLIC_KEY,
        target_url: env.TARGET_URL
      });
    }

    if (u.pathname === '/api/check' && request.method === 'POST') {
      return json(await check(env, true));
    }

    if (u.pathname === '/api/push/subscribe' && request.method === 'POST') {
      const sub = await request.json();
      if (!sub?.endpoint || !sub?.keys?.p256dh || !sub?.keys?.auth) return json({ ok: false }, 400);
      const subs = await getSubscriptions(env);
      const filtered = subs.filter(x => x.endpoint !== sub.endpoint);
      filtered.push(sub);
      await saveSubscriptions(env, filtered);
      return json({ ok: true });
    }

    if (u.pathname === '/api/test-alert' && request.method === 'POST') {
      const sent = await sendPush(env, '✅ ED Coaching Watch funktioniert', 'Du bekommst eine Push-Mitteilung, sobald ein Platz frei wird.');
      return json({ ok: sent > 0, sent });
    }

    if (u.pathname === '/sw.js') {
      return new Response(serviceWorker, { headers: { 'content-type': 'application/javascript; charset=UTF-8', 'cache-control': 'no-cache' } });
    }

    if (u.pathname === '/manifest.json') {
      return new Response(JSON.stringify(manifest), { headers: { 'content-type': 'application/manifest+json' } });
    }

    return new Response(html, { headers: { 'content-type': 'text/html;charset=UTF-8', 'cache-control': 'no-store' } });
  }
};
