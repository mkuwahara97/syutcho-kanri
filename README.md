<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, maximum-scale=1, user-scalable=no">
  <meta name="theme-color" content="#0c0f1a">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
  <meta name="apple-mobile-web-app-title" content="出張管理">
  <title>出張報告書管理</title>
  <link rel="apple-touch-icon" id="apple-icon">
  <link rel="icon" id="fav-icon">
  <link rel="manifest" id="manifest-link">
  <style>
    *{-webkit-tap-highlight-color:transparent;-webkit-touch-callout:none}
    html,body{height:100%;margin:0;background:#0c0f1a;overscroll-behavior:none}
    #root{padding-top:env(safe-area-inset-top);padding-bottom:env(safe-area-inset-bottom)}
    .hdr{padding-top:max(13px,env(safe-area-inset-top))!important}
    input,select,textarea{font-size:16px!important}
  </style>
</head>
<body>
<div id="root"></div>

<!-- Firebase SDK -->
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore-compat.js"></script>

<!-- PWA setup -->
<script>
const iconSvg=`<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512"><rect width="512" height="512" rx="100" fill="#0c0f1a"/><text x="256" y="340" text-anchor="middle" font-size="300" font-family="serif">✈</text></svg>`;
const iconUrl='data:image/svg+xml,'+encodeURIComponent(iconSvg);
document.getElementById('apple-icon').href=iconUrl;
document.getElementById('fav-icon').href=iconUrl;
const mBlob=new Blob([JSON.stringify({name:"出張報告書管理",short_name:"出張管理",start_url:"./",display:"standalone",orientation:"portrait",background_color:"#0c0f1a",theme_color:"#0c0f1a",lang:"ja",icons:[{src:iconUrl,sizes:"512x512",type:"image/svg+xml",purpose:"maskable"}]})],{type:"application/json"});
document.getElementById('manifest-link').href=URL.createObjectURL(mBlob);

// localStorage for media only
window._lsMedia = {
  get: (k) => { try { const v=localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch { return null; } },
  set: (k,v) => { try { localStorage.setItem(k,JSON.stringify(v)); } catch {} },
};
</script>

<!-- React + Babel -->
<script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

<script type="text/babel" data-presets="react">
const { useState, useEffect, useRef, useCallback } = React;

// ════════════════════════════════════════════════════════
//  UTILITIES
// ════════════════════════════════════════════════════════
const uid = () => Math.random().toString(36).slice(2, 9);
const nowISO = () => new Date().toISOString();

const fmtTime = (iso) => {
  if (!iso) return "--:--";
  const d = new Date(iso);
  return `${String(d.getHours()).padStart(2,"0")}:${String(d.getMinutes()).padStart(2,"0")}`;
};
const fmtDate = (s) => {
  if (!s) return "";
  const d = new Date(s);
  return `${d.getFullYear()}/${String(d.getMonth()+1).padStart(2,"0")}/${String(d.getDate()).padStart(2,"0")}`;
};
const fmtDateTime = (iso) => {
  if (!iso) return "--";
  const d = new Date(iso);
  return `${String(d.getMonth()+1).padStart(2,"0")}/${String(d.getDate()).padStart(2,"0")} ${String(d.getHours()).padStart(2,"0")}:${String(d.getMinutes()).padStart(2,"0")}`;
};
const diffMin = (planned, actual) => {
  if (!planned || !actual) return null;
  const [h,m] = planned.split(":").map(Number);
  const a = new Date(actual);
  const p = new Date(a.getFullYear(), a.getMonth(), a.getDate(), h, m);
  return Math.round((a - p) / 60000);
};
const stayMin = (a, b) => (!a||!b) ? null : Math.round((new Date(b) - new Date(a)) / 60000);
const fmtMin = (m) => {
  if (m === null || m === undefined) return null;
  if (Math.abs(m) < 60) return `${m}分`;
  return `${Math.floor(Math.abs(m)/60)}時間${Math.abs(m)%60}分`;
};
const fmtRecTime = (s) => `${String(Math.floor(s/60)).padStart(2,"0")}:${String(s%60).padStart(2,"0")}`;

const compress = (file, max = 800) => new Promise(res => {
  const r = new FileReader();
  r.onload = e => {
    const img = new Image();
    img.onload = () => {
      let [w,h] = [img.width, img.height];
      if (w > max || h > max) {
        if (w > h) { h = Math.round(h/w*max); w = max; }
        else { w = Math.round(w/h*max); h = max; }
      }
      const c = document.createElement("canvas");
      [c.width, c.height] = [w, h];
      c.getContext("2d").drawImage(img, 0, 0, w, h);
      res(c.toDataURL("image/jpeg", 0.72));
    };
    img.src = e.target.result;
  };
  r.readAsDataURL(file);
});

const POSTAL_RE = /^\d{3}-?\d{4}$/;

const searchGSI = async (q) => {
  try {
    const r = await fetch(`https://msearch.gsi.go.jp/address-search/AddressSearch?q=${encodeURIComponent(q)}`, { signal: AbortSignal.timeout(5000) });
    if (!r.ok) return [];
    const d = await r.json();
    if (!Array.isArray(d)) return [];
    return d.map(x => ({
      address: x.properties.title,
      lat: x.geometry.coordinates[1],
      lng: x.geometry.coordinates[0],
      source: "国土地理院",
    }));
  } catch { return []; }
};

const searchZipcloud = async (zip) => {
  try {
    const r = await fetch(`https://zipcloud.ibsnet.co.jp/api/search?zipcode=${zip.replace("-","")}`, { signal: AbortSignal.timeout(5000) });
    if (!r.ok) return [];
    const d = await r.json();
    if (!d.results) return [];
    return d.results.map(x => ({
      address: `${x.address1}${x.address2}${x.address3}`,
      lat: null, lng: null,
      source: "〒",
    }));
  } catch { return []; }
};

const searchNominatim = async (q) => {
  try {
    const r = await fetch(
      `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(q)}&limit=8&addressdetails=0&countrycodes=jp`,
      { headers: { "Accept-Language": "ja,en" }, signal: AbortSignal.timeout(6000) }
    );
    if (!r.ok) return [];
    const d = await r.json();
    return d.map(x => ({
      address: x.display_name,
      lat: parseFloat(x.lat),
      lng: parseFloat(x.lon),
      source: "OSM",
    }));
  } catch { return []; }
};

const reverseGeocode = async (lat, lng) => {
  try {
    const r = await fetch(
      `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}&accept-language=ja`,
      { headers: { "Accept-Language": "ja,en" }, signal: AbortSignal.timeout(6000) }
    );
    if (!r.ok) return null;
    const d = await r.json();
    return d.display_name || null;
  } catch { return null; }
};

const multiSearch = async (q) => {
  if (POSTAL_RE.test(q.trim())) {
    const zip = await searchZipcloud(q.trim());
    if (zip.length > 0) return zip;
    // zipcloudがCORSでブロックされた場合はNominatimでフォールバック
    return searchNominatim(q + " 日本");
  }
  const [gsi, osm] = await Promise.allSettled([searchGSI(q), searchNominatim(q)]);
  const gsiR = gsi.status === "fulfilled" ? gsi.value : [];
  const osmR = osm.status === "fulfilled" ? osm.value : [];
  const seen = new Set();
  const merged = [];
  for (const r of [...gsiR, ...osmR]) {
    const key = r.address.slice(0, 18);
    if (!seen.has(key)) { seen.add(key); merged.push(r); }
  }
  return merged.slice(0, 12);
};

const openNav = ({ address, name, lat, lng } = {}) => {
  const dest = (lat && lng) ? `${lat},${lng}` : encodeURIComponent(address || name || "");
  const base = `https://www.google.com/maps/dir/?api=1&destination=${dest}&travelmode=driving`;
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      p => window.open(`https://www.google.com/maps/dir/?api=1&origin=${p.coords.latitude},${p.coords.longitude}&destination=${dest}&travelmode=driving`, "_blank"),
      () => window.open(base, "_blank"),
      { timeout: 4000, enableHighAccuracy: false, maximumAge: 60000 }
    );
  } else { window.open(base, "_blank"); }
};

const sg = async (k) => { try { const r = await window.storage.get(k); return r ? JSON.parse(r.value) : null; } catch { return null; } };
const ss = async (k,v) => { try { await window.storage.set(k, JSON.stringify(v)); } catch {} };
const sd = async (k) => { try { await window.storage.delete(k); } catch {} };

const loadAllTrips = async () => {
  const list = await sg("trips:list") || [];
  const arr = [];
  for (const id of list) { const t = await sg(`trip:${id}`); if (t) arr.push(t); }
  return arr;
};
const saveTrip = async (trip) => {
  await ss(`trip:${trip.id}`, trip);
  const list = await sg("trips:list") || [];
  if (!list.includes(trip.id)) await ss("trips:list", [...list, trip.id]);
};
const deleteTrip = async (id) => {
  const list = await sg("trips:list") || [];
  await ss("trips:list", list.filter(i => i !== id));
  await sd(`trip:${id}`);
};
const loadMedia = async (pid) => (await sg(`media:${pid}`)) || { photos: [], recordings: [] };
const saveMedia = async (pid, media) => await ss(`media:${pid}`, media);
const loadTravelers = async () => (await sg("travelers")) || [];
const saveTravelers = async (list) => await ss("travelers", list);



// Media: always localStorage (photos/recordings too large for Firestore)
const loadMedia  = async (pid) => window._lsMedia.get(`media:${pid}`) || { photos:[], recordings:[] };
const saveMedia  = async (pid, m) => window._lsMedia.set(`media:${pid}`, m);

// addr history: localStorage
const sg = async (k) => { try { const v=localStorage.getItem(k); return v ? JSON.parse(v) : null; } catch { return null; } };
const ss = async (k,v) => { try { localStorage.setItem(k,JSON.stringify(v)); } catch {} };

// ════════════════════════════════════════════════════════
//  FIREBASE HELPERS
// ════════════════════════════════════════════════════════
let _db = null;
let _teamId = null;

const initFirebase = (config, teamId) => {
  try {
    if (!firebase.apps.length) firebase.initializeApp(config);
    else firebase.app();
    _db = firebase.firestore();
    _teamId = teamId;
    return true;
  } catch(e) { console.error(e); return false; }
};

const teamRef  = () => _db.collection("teams").doc(_teamId);
const tripsCol = () => teamRef().collection("trips");
const tvDoc    = () => teamRef().collection("meta").doc("travelers");

const fsaveTrip    = async (trip) => { await tripsCol().doc(trip.id).set(trip); };
const fdeleteTrip  = async (id)   => { await tripsCol().doc(id).delete(); };
const fsaveTvs     = async (list) => { await tvDoc().set({ list }); };

// Compat shims so TravelerMasterModal works unchanged
const loadTravelers = async () => {
  try { const snap = await tvDoc().get(); return snap.data()?.list || []; } catch { return []; }
};
const saveTravelers = async (list) => { await fsaveTvs(list); };

// Export helpers (read all trips for zip export)
const floadTrips = async () => {
  const snap = await tripsCol().get();
  return snap.docs.map(d => ({ ...d.data(), id: d.id }));
};

// ════════════════════════════════════════════════════════
//  SETUP SCREEN
// ════════════════════════════════════════════════════════
const SETUP_CSS = `
.setup-wrap{min-height:100vh;background:#0c0f1a;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:24px;font-family:'Hiragino Kaku Gothic ProN','Meiryo',sans-serif;color:#e2e8f8}
.setup-card{background:#12151e;border:1px solid #252d42;border-radius:14px;padding:24px;width:100%;max-width:420px}
.setup-title{font-size:22px;font-weight:800;margin-bottom:4px;text-align:center}
.setup-sub{font-size:13px;color:#6b7594;text-align:center;margin-bottom:20px}
.setup-label{display:block;font-size:11px;color:#6b7594;margin-bottom:5px;font-weight:700;letter-spacing:.06em;text-transform:uppercase}
.setup-inp{width:100%;background:#0d1018;border:1px solid #2e3548;border-radius:8px;color:#e2e8f8;padding:9px 12px;font-size:14px;font-family:inherit;outline:none;box-sizing:border-box;margin-bottom:14px}
.setup-inp:focus{border-color:#3b82f6}
.setup-inp::placeholder{color:#4a5570}
.setup-btn{width:100%;padding:11px;background:#2563eb;color:#fff;border:none;border-radius:9px;font-size:15px;font-weight:700;cursor:pointer;font-family:inherit;margin-top:4px}
.setup-btn:hover{background:#1d4ed8}
.setup-btn:disabled{opacity:.4;cursor:not-allowed}
.setup-err{color:#f87171;font-size:13px;margin-top:8px;text-align:center}
.setup-link{color:#60a5fa;font-size:12px;text-align:center;margin-top:12px;cursor:pointer;text-decoration:underline}
.setup-step{display:flex;align-items:center;justify-content:center;gap:6px;margin-bottom:20px}
.setup-dot{width:8px;height:8px;border-radius:50%}
`;

const BAKED_CONFIG = {"apiKey": "AIzaSyDombC-mDxM6TC6Eo1Zqu33FkWdagt1xm4", "authDomain": "syuttyoukannri-app.firebaseapp.com", "projectId": "syuttyoukannri-app", "storageBucket": "syuttyoukannri-app.firebasestorage.app", "messagingSenderId": "118459868912", "appId": "1:118459868912:web:798472d020f181da440435", "measurementId": "G-F24M5VGJPK"};

function SetupScreen({ onComplete }) {
  const [teamCode, setTeamCode] = useState('');
  const [err, setErr] = useState('');

  const handleStart = () => {
    const t = teamCode.trim().replace(/\s+/g,'_');
    if (!t) { setErr('チームIDを入力してください'); return; }
    const ok = initFirebase(BAKED_CONFIG, t);
    if (!ok) { setErr('Firebase への接続に失敗しました'); return; }
    localStorage.setItem('fb_config', JSON.stringify(BAKED_CONFIG));
    localStorage.setItem('fb_team', t);
    onComplete(BAKED_CONFIG, t);
  };

  return (
    <>
      <style>{SETUP_CSS}</style>
      <div className="setup-wrap">
        <div className="setup-card">
          <div className="setup-title">✈ 出張報告書管理</div>
          <div className="setup-sub">リアルタイム共有版</div>

          <label className="setup-label">チームID を入力</label>
          <input className="setup-inp" value={teamCode}
            onChange={e=>{setTeamCode(e.target.value);setErr('');}}
            placeholder="例: yamada-sales-team"
            onKeyDown={e=>e.key==='Enter'&&handleStart()} autoFocus />
          <div style={{fontSize:11,color:'#6b7594',marginBottom:16,lineHeight:1.7}}>
            チームメンバー全員が<strong style={{color:'#e2e8f8'}}>同じチームID</strong>を入力するとデータがリアルタイムで共有されます。<br/>
            英数字・ハイフン・アンダースコアが使えます。
          </div>
          <button className="setup-btn" onClick={handleStart} disabled={!teamCode.trim()}>
            開始する
          </button>
          {err && <div className="setup-err">{err}</div>}
        </div>
      </div>
    </>
  );
}

// ════════════════════════════════════════════════════════
//  UI COMPONENTS (shared with standalone version)
// ════════════════════════════════════════════════════════
const PT = {
  departure: { label: "出発地", icon: "▶", color: "#60a5fa", bg: "rgba(96,165,250,0.12)" },
  waypoint:  { label: "経由地", icon: "◆", color: "#c084fc", bg: "rgba(192,132,252,0.12)" },
  visit:     { label: "訪問地", icon: "●", color: "#fb923c", bg: "rgba(251,146,60,0.12)" },
  arrival:   { label: "到着地", icon: "■", color: "#4ade80", bg: "rgba(74,222,128,0.12)" },
};

const STATUS = {
  planned:   { label: "予定",   color: "#60a5fa", bg: "rgba(96,165,250,0.12)" },
  active:    { label: "進行中", color: "#4ade80", bg: "rgba(74,222,128,0.12)" },
  completed: { label: "完了",   color: "#9ca3af", bg: "rgba(156,163,175,0.12)" },
};

const css = `
*{box-sizing:border-box;margin:0;padding:0}
:root{--bg:#0c0f1a;--surf:#121624;--surf2:#1a1f2e;--surf3:#212840;--border:#252d42;--border2:#2e3a52;--text:#e2e8f8;--text2:#8b95b0;--text3:#4a5570;--accent:#3b82f6;--green:#4ade80;--amber:#fbbf24;--red:#f87171;--radius:10px}
body,html{height:100%;background:var(--bg);overflow:hidden}
#root{height:100vh;display:flex;flex-direction:column;background:var(--bg);font-family:'Hiragino Kaku Gothic ProN','Noto Sans CJK JP','Meiryo',system-ui,sans-serif;font-size:14px;color:var(--text);overflow:hidden}
.scrl{flex:1;overflow-y:auto;-webkit-overflow-scrolling:touch}
.scrl::-webkit-scrollbar{width:4px}
.scrl::-webkit-scrollbar-track{background:transparent}
.scrl::-webkit-scrollbar-thumb{background:var(--border2);border-radius:2px}
.pad{padding:12px 14px}
.hdr{background:var(--surf);border-bottom:1px solid var(--border);padding:13px 14px;display:flex;align-items:center;gap:10px;flex-shrink:0;position:relative;z-index:10}
.hdr h1{font-size:15px;font-weight:700;letter-spacing:.03em;flex:1}
.card{background:var(--surf);border:1px solid var(--border);border-radius:var(--radius);overflow:hidden;margin-bottom:10px}
.card-hdr{padding:10px 14px;border-bottom:1px solid var(--border);display:flex;align-items:center;justify-content:space-between;gap:8px}
.card-body{padding:14px}
.sec-title{padding:10px 14px 8px;font-size:11px;font-weight:700;letter-spacing:.1em;color:var(--text3);text-transform:uppercase;border-bottom:1px solid var(--border)}
.btn{display:inline-flex;align-items:center;gap:5px;padding:7px 13px;border-radius:8px;border:none;font-size:13px;font-weight:600;cursor:pointer;transition:opacity .15s,transform .1s;font-family:inherit}
.btn:active{transform:scale(.97)}
.btn-blue{background:var(--accent);color:#fff}
.btn-blue:hover{opacity:.9}
.btn-green{background:#166534;color:#4ade80;border:1px solid #166534}
.btn-green:hover{background:#14532d}
.btn-amber{background:#78350f;color:#fbbf24;border:1px solid #78350f}
.btn-amber:hover{background:#6c2d0b}
.btn-red{background:#7f1d1d;color:#f87171;border:1px solid #7f1d1d}
.btn-red:hover{background:#6b1919}
.btn-ghost{background:var(--surf2);color:var(--text2);border:1px solid var(--border2)}
.btn-ghost:hover{background:var(--surf3)}
.btn-sm{padding:5px 10px;font-size:12px}
.btn-xs{padding:3px 8px;font-size:11px;border-radius:6px}
.inp{width:100%;background:var(--surf2);border:1px solid var(--border2);border-radius:8px;color:var(--text);padding:8px 11px;font-size:14px;font-family:inherit;outline:none;transition:border-color .15s}
.inp:focus{border-color:var(--accent)}
.inp::placeholder{color:var(--text3)}
select.inp{appearance:none;padding-right:28px;background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='6'%3E%3Cpath d='M5 6L0 0h10z' fill='%234a5570'/%3E%3C/svg%3E");background-repeat:no-repeat;background-position:right 10px center}
textarea.inp{resize:vertical;min-height:80px;line-height:1.5}
.lbl{display:block;font-size:11px;color:var(--text3);margin-bottom:5px;font-weight:600;letter-spacing:.03em}
.g2{display:grid;grid-template-columns:1fr 1fr;gap:10px}
.row{display:flex;align-items:center;gap:8px}
.flex1{flex:1}
.mt2{margin-top:8px}
.mt3{margin-top:12px}
.mb2{margin-bottom:8px}
.mono{font-family:'SF Mono','Fira Code','Courier New',monospace}
.badge{display:inline-flex;align-items:center;padding:2px 8px;border-radius:20px;font-size:11px;font-weight:700}
.time-box{background:var(--surf2);border:1px solid var(--border);border-radius:8px;padding:10px 12px;text-align:center}
.time-val{font-family:'SF Mono','Fira Code',monospace;font-size:22px;font-weight:700;letter-spacing:.05em;color:var(--text)}
.time-lbl{font-size:11px;color:var(--text3);margin-bottom:6px;font-weight:600;text-transform:uppercase;letter-spacing:.06em}
.time-diff{font-size:11px;margin-top:4px;font-weight:600;font-family:monospace}
.diff-late{color:#f87171}
.diff-early{color:#4ade80}
.diff-ok{color:var(--text3)}
.prow{padding:11px 14px;border-bottom:1px solid var(--border);cursor:pointer;display:flex;align-items:center;gap:10px;transition:background .12s}
.prow:last-child{border-bottom:none}
.prow:hover{background:var(--surf2)}
.picon{width:34px;height:34px;border-radius:9px;display:flex;align-items:center;justify-content:center;font-size:15px;flex-shrink:0}
.photo-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:6px}
.photo-wrap{position:relative;aspect-ratio:1}
.photo-img{width:100%;height:100%;object-fit:cover;border-radius:7px;cursor:pointer;border:1px solid var(--border);display:block}
.photo-del{position:absolute;top:3px;right:3px;background:rgba(0,0,0,.7);border:none;color:#fff;border-radius:50%;width:20px;height:20px;cursor:pointer;font-size:11px;display:flex;align-items:center;justify-content:center;line-height:1}
.rec-item{display:flex;align-items:center;gap:8px;padding:8px 10px;background:var(--surf2);border:1px solid var(--border);border-radius:8px;margin-bottom:6px}
.modal-bg{position:fixed;inset:0;background:rgba(0,0,0,.8);display:flex;align-items:flex-end;justify-content:center;z-index:100;padding-bottom:0}
.modal{background:var(--surf);border:1px solid var(--border2);border-radius:14px 14px 0 0;width:100%;max-width:520px;max-height:88vh;display:flex;flex-direction:column}
.modal-hdr{padding:14px 16px;border-bottom:1px solid var(--border);display:flex;align-items:center;justify-content:space-between;font-weight:700;font-size:15px;flex-shrink:0}
.modal-body{padding:16px;overflow-y:auto;flex:1}
.modal-ftr{padding:12px 16px;border-top:1px solid var(--border);display:flex;justify-content:flex-end;gap:8px;flex-shrink:0}
.sres{padding:9px 11px;cursor:pointer;font-size:13px;color:var(--text2);border-bottom:1px solid var(--border);transition:background .1s;line-height:1.4}
.sres:last-child{border-bottom:none}
.sres:hover{background:var(--surf2);color:var(--text)}
.empty{text-align:center;padding:40px 16px;color:var(--text3)}
.lightbox{position:fixed;inset:0;background:rgba(0,0,0,.95);display:flex;align-items:center;justify-content:center;z-index:200;cursor:pointer}
.lightbox img{max-width:95vw;max-height:95vh;object-fit:contain;border-radius:8px}
.pulse{animation:pulse 1s ease-in-out infinite}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.35}}
.stay-bar{margin-top:10px;padding:9px 14px;background:rgba(251,191,36,.08);border:1px solid rgba(251,191,36,.2);border-radius:8px;text-align:center}
.trip-item{background:var(--surf);border:1px solid var(--border);border-radius:var(--radius);margin-bottom:10px;overflow:hidden;cursor:pointer;transition:border-color .15s}
.trip-item:hover{border-color:var(--border2)}
.chip{display:inline-flex;align-items:center;padding:2px 8px;border-radius:12px;font-size:11px;font-weight:600;white-space:nowrap}
.divider{height:1px;background:var(--border);margin:12px 0}
.w100{width:100%}
audio{filter:invert(0.85) hue-rotate(180deg);width:100%;height:32px;border-radius:6px}
`;

function Btn({ cls="btn-ghost", sm, xs, onClick, disabled, children, style }) {
  const sz = xs ? " btn-xs" : sm ? " btn-sm" : "";
  return <button className={`btn ${cls}${sz}`} onClick={onClick} disabled={disabled} style={style}>{children}</button>;
}

function Modal({ title, onClose, children, footer }) {
  return (
    <div className="modal-bg" onClick={e => e.target === e.currentTarget && onClose()}>
      <div className="modal">
        <div className="modal-hdr">
          <span>{title}</span>
          <Btn sm onClick={onClose}>✕</Btn>
        </div>
        <div className="modal-body">{children}</div>
        {footer && <div className="modal-ftr">{footer}</div>}
      </div>
    </div>
  );
}

const SRC_COLOR = { "国土地理院":"#60a5fa", "〒":"#4ade80", "OSM":"#c084fc", "現在地":"#fb923c" };

function AddrSearchModal({ onSelect, onClose }) {
  const [q, setQ] = useState("");
  const [results, setResults] = useState([]);
  const [history, setHistory] = useState([]);
  const [loading, setLoading] = useState(false);
  const [locating, setLocating] = useState(false);
  const [error, setError] = useState("");
  const debRef = useRef(null);

  useEffect(() => { sg("addr:history").then(h => setHistory(h || [])); }, []);

  const saveHistory = (h) => { setHistory(h); ss("addr:history", h); };
  const addToHistory = (r) => saveHistory([r, ...history.filter(x => x.address !== r.address)].slice(0,10));

  const doSearch = async (query) => {
    if (!query.trim()) { setResults([]); return; }
    setLoading(true); setError("");
    const res = await multiSearch(query.trim());
    setResults(res);
    if (res.length === 0) setError("「" + query.trim() + "」の結果が見つかりませんでした。別のキーワードや郵便番号をお試しください。");
    setLoading(false);
  };

  const handleChange = (val) => {
    setQ(val); setError("");
    clearTimeout(debRef.current);
    if (val.length < 2) { setResults([]); return; }
    debRef.current = setTimeout(() => doSearch(val), 420);
  };

  const handleLocate = () => {
    if (!navigator.geolocation) {
      setError("このブラウザは位置情報に対応していません。住所を直接入力してください。");
      return;
    }
    setLocating(true); setError("");
    navigator.geolocation.getCurrentPosition(
      async pos => {
        const { latitude: lat, longitude: lng } = pos.coords;
        const addr = await reverseGeocode(lat, lng);
        setLocating(false);
        if (addr) { const r = { address: addr, lat, lng, source: "現在地" }; addToHistory(r); onSelect(r); }
        else setError("住所の変換に失敗しました。座標: " + lat.toFixed(5) + ", " + lng.toFixed(5));
      },
      (err) => {
        setLocating(false);
        if (err.code === 1) setError("位置情報へのアクセスが拒否されています。ブラウザの設定で許可してください。");
        else if (err.code === 2) setError("位置情報を取得できませんでした。電波状況を確認してください。");
        else setError("位置情報の取得がタイムアウトしました。もう一度お試しください。");
      },
      { timeout: 10000, enableHighAccuracy: true, maximumAge: 0 }
    );
  };

  const handlePick = (r) => { addToHistory(r); onSelect(r); };

  const isPostal = POSTAL_RE.test(q.trim());
  const showList = results.length > 0 ? results : (q.length < 2 ? history : []);
  const isHistory = results.length === 0 && q.length < 2 && history.length > 0;

  return (
    <Modal title="住所を検索" onClose={onClose}>
      <div style={{display:"flex",flexDirection:"column",gap:10}}>
        <div className="row">
          <div style={{position:"relative",flex:1}}>
            <input className="inp" style={{width:"100%",paddingRight: isPostal ? 52 : 11}} value={q}
              onChange={e=>handleChange(e.target.value)}
              placeholder="場所名・住所・郵便番号（例: 100-0001）"
              onKeyDown={e=>e.key==="Enter"&&doSearch(q)} autoFocus />
            {isPostal && (
              <span style={{position:"absolute",right:8,top:"50%",transform:"translateY(-50%)",
                background:"rgba(74,222,128,.15)",color:"#4ade80",fontSize:10,fontWeight:700,
                padding:"2px 6px",borderRadius:6,pointerEvents:"none"}}>〒</span>
            )}
          </div>
          {q && <Btn xs onClick={()=>{setQ("");setResults([]);setError("");}} style={{flexShrink:0}}>✕</Btn>}
        </div>

        <Btn cls="btn-ghost" style={{width:"100%",justifyContent:"center",gap:6}} onClick={handleLocate} disabled={locating||loading}>
          {locating
            ? <span className="pulse" style={{color:"var(--text2)"}}>📍 取得中…</span>
            : <><span>📍</span><span>現在地から住所を取得</span></>
          }
        </Btn>

        {loading && (
          <div style={{textAlign:"center",padding:"8px 0",color:"var(--text3)",fontSize:13,display:"flex",alignItems:"center",justifyContent:"center",gap:8}}>
            <span className="pulse">●</span>
            {isPostal ? "郵便番号を検索中…" : "国土地理院・OSM を検索中…"}
          </div>
        )}

        {error && !loading && (
          <div style={{padding:"8px 10px",background:"rgba(248,113,113,.08)",border:"1px solid rgba(248,113,113,.2)",borderRadius:8,color:"var(--red)",fontSize:13,textAlign:"center"}}>{error}</div>
        )}

        {isHistory && !loading && (
          <div style={{fontSize:11,color:"var(--text3)",display:"flex",alignItems:"center",justifyContent:"space-between"}}>
            <span>最近の検索</span>
            <Btn xs onClick={()=>saveHistory([])}>クリア</Btn>
          </div>
        )}

        {!loading && showList.length > 0 && (
          <div className="card" style={{marginBottom:0,maxHeight:320,overflowY:"auto"}}>
            {showList.map((r, i) => {
              const c = SRC_COLOR[r.source] || "#8b95b0";
              return (
                <div key={i} className="sres" style={{display:"flex",alignItems:"flex-start",gap:8}} onClick={()=>handlePick(r)}>
                  <div style={{flex:1,minWidth:0}}>
                    <div style={{marginBottom:4,lineHeight:1.45,color:"var(--text)"}}>{r.address}</div>
                    <div style={{display:"flex",alignItems:"center",gap:6}}>
                      <span className="badge" style={{background:`${c}22`,color:c,fontSize:10,fontWeight:700}}>{r.source}</span>
                      {r.lat != null && (
                        <span className="mono" style={{fontSize:10,color:"var(--text3)"}}>
                          {r.lat.toFixed(4)}, {r.lng.toFixed(4)}
                        </span>
                      )}
                    </div>
                  </div>
                  <span style={{color:"var(--text3)",fontSize:16,lineHeight:1,marginTop:2}}>›</span>
                </div>
              );
            })}
          </div>
        )}

        {!loading && !error && showList.length === 0 && q.length >= 2 && (
          <div style={{textAlign:"center",padding:"20px 0",color:"var(--text3)"}}>
            <div style={{fontSize:13}}>結果が見つかりませんでした</div>
            <div style={{fontSize:11,marginTop:5}}>別のキーワードや郵便番号で試してください</div>
          </div>
        )}

        <div style={{padding:"10px 0 0",borderTop:"1px solid var(--border)",display:"flex",gap:8,flexWrap:"wrap"}}>
          {[{c:"#60a5fa",s:"国土地理院"},{c:"#4ade80",s:"〒 郵便番号"},{c:"#c084fc",s:"OSM"}].map(({c,s})=>(
            <span key={s} className="badge" style={{background:`${c}18`,color:c,fontSize:10}}>{s}</span>
          ))}
          <span style={{fontSize:10,color:"var(--text3)",marginLeft:"auto",alignSelf:"center"}}>3ソース並行検索</span>
        </div>
      </div>
    </Modal>
  );
}

function TravelerMasterModal({ onClose }) {
  const [travelers, setTravelers] = useState([]);
  const [form, setForm] = useState({ name: "", dept: "" });
  const [editId, setEditId] = useState(null);
  const [confirmId, setConfirmId] = useState(null);

  useEffect(() => { loadTravelers().then(setTravelers); }, []);

  const save = async (list) => { setTravelers(list); await saveTravelers(list); };

  const add = () => {
    if (!form.name.trim()) return;
    const list = editId
      ? travelers.map(t => t.id === editId ? { ...t, ...form } : t)
      : [...travelers, { id: uid(), name: form.name.trim(), dept: form.dept.trim() }];
    save(list);
    setForm({ name: "", dept: "" });
    setEditId(null);
  };

  const startEdit = (t) => { setForm({ name: t.name, dept: t.dept }); setEditId(t.id); };
  const cancelEdit = () => { setForm({ name: "", dept: "" }); setEditId(null); };

  return (
    <Modal title="👤 出張者マスタ" onClose={onClose}>
      <div style={{display:"flex",flexDirection:"column",gap:12}}>
        <div style={{background:"var(--surf2)",borderRadius:9,padding:12}}>
          <div style={{fontSize:12,fontWeight:700,color:"var(--text3)",marginBottom:8}}>
            {editId ? "✏ 編集" : "＋ 新規追加"}
          </div>
          <div className="g2" style={{marginBottom:8}}>
            <div>
              <label className="lbl">氏名 *</label>
              <input className="inp" value={form.name} onChange={e=>setForm(f=>({...f,name:e.target.value}))}
                placeholder="山田 太郎" onKeyDown={e=>e.key==="Enter"&&add()} autoFocus />
            </div>
            <div>
              <label className="lbl">部署</label>
              <input className="inp" value={form.dept} onChange={e=>setForm(f=>({...f,dept:e.target.value}))}
                placeholder="営業部" onKeyDown={e=>e.key==="Enter"&&add()} />
            </div>
          </div>
          <div className="row" style={{gap:6}}>
            <Btn cls="btn-blue" sm onClick={add} disabled={!form.name.trim()} style={{flex:1,justifyContent:"center"}}>
              {editId ? "更新" : "追加"}
            </Btn>
            {editId && <Btn sm onClick={cancelEdit}>キャンセル</Btn>}
          </div>
        </div>

        <div className="card" style={{marginBottom:0,maxHeight:320,overflowY:"auto"}}>
          {travelers.length === 0 && (
            <div className="empty" style={{padding:24}}>
              <p style={{fontSize:13}}>出張者が登録されていません</p>
            </div>
          )}
          {travelers.map(t => (
            <div key={t.id} style={{padding:"10px 14px",borderBottom:"1px solid var(--border)",display:"flex",alignItems:"center",gap:10}}>
              <div style={{flex:1}}>
                <div style={{fontWeight:700,fontSize:14}}>{t.name}</div>
                {t.dept && <div style={{fontSize:12,color:"var(--text3)",marginTop:1}}>{t.dept}</div>}
              </div>
              {confirmId === t.id ? (
                <div className="row" style={{gap:5}}>
                  <span style={{fontSize:12,color:"var(--text2)"}}>削除？</span>
                  <Btn xs cls="btn-red" onClick={()=>{save(travelers.filter(x=>x.id!==t.id));setConfirmId(null);}}>はい</Btn>
                  <Btn xs onClick={()=>setConfirmId(null)}>いいえ</Btn>
                </div>
              ) : (
                <div className="row" style={{gap:4}}>
                  <Btn xs onClick={()=>startEdit(t)}>✏</Btn>
                  <Btn xs onClick={()=>setConfirmId(t.id)}>🗑</Btn>
                </div>
              )}
            </div>
          ))}
        </div>
        <div style={{fontSize:11,color:"var(--text3)",textAlign:"center"}}>
          {travelers.length}名登録済み
        </div>
      </div>
    </Modal>
  );
}

function TravelerPicker({ selected, onChange, travelers, onOpenMaster }) {
  if (travelers.length === 0) return (
    <div style={{textAlign:"center",padding:"10px 0"}}>
      <span style={{fontSize:13,color:"var(--text3)"}}>出張者が未登録です。</span>
      <button onClick={onOpenMaster} style={{background:"none",border:"none",color:"var(--accent)",fontSize:13,cursor:"pointer",textDecoration:"underline",marginLeft:4}}>
        マスタ登録
      </button>
    </div>
  );
  return (
    <div>
      <div style={{display:"flex",flexWrap:"wrap",gap:6}}>
        {travelers.map(t => {
          const on = selected.includes(t.id);
          return (
            <div key={t.id} onClick={()=>onChange(on ? selected.filter(i=>i!==t.id) : [...selected,t.id])}
              style={{
                padding:"6px 12px",borderRadius:20,cursor:"pointer",fontSize:13,fontWeight:600,
                border:`1px solid ${on?"var(--accent)":"var(--border2)"}`,
                background: on?"rgba(59,130,246,.15)":"var(--surf2)",
                color: on?"var(--accent)":"var(--text2)",
                transition:"all .12s",userSelect:"none",
              }}>
              {on && <span style={{marginRight:4}}>✓</span>}
              {t.name}
              {t.dept && <span style={{fontSize:11,opacity:.6,marginLeft:4}}>{t.dept}</span>}
            </div>
          );
        })}
      </div>
      {travelers.length > 0 && (
        <button onClick={onOpenMaster} style={{background:"none",border:"none",color:"var(--text3)",fontSize:11,cursor:"pointer",marginTop:6,textDecoration:"underline"}}>
          ＋ マスタ管理
        </button>
      )}
    </div>
  );
}


function TripList({ trips, travelers, onCreate, onSelect, onRemove, onOpenMaster, onExportAll, onImport }) {
  const [confirmId, setConfirmId] = useState(null);
  const [tab, setTab] = useState("active");

  const sorted = [...trips].sort((a, b) => (a.date || "").localeCompare(b.date || ""));
  const active = sorted.filter(t => t.status !== "completed");
  const done   = sorted.filter(t => t.status === "completed");
  const list   = tab === "active" ? active : done;

  const tabStyle = (t) => ({
    flex: 1, padding: "8px 0", textAlign: "center", fontSize: 13, fontWeight: 700,
    cursor: "pointer", borderRadius: 7, transition: "all .15s",
    background: tab === t ? "var(--surf3)" : "transparent",
    color: tab === t ? "var(--text)" : "var(--text3)",
    border: "none",
  });

  const TripCard = ({ t }) => {
    const st = STATUS[t.status] || STATUS.planned;
    const tripTravelers = (t.travelerIds||[]).map(id=>travelers.find(x=>x.id===id)).filter(Boolean);
    return (
      <div className="trip-item" onClick={()=>onSelect(t.id)}>
        <div style={{padding:"11px 14px",display:"flex",alignItems:"center",gap:10,justifyContent:"space-between"}}>
          <div style={{flex:1,minWidth:0}}>
            <div className="row" style={{gap:7,marginBottom:4}}>
              <span className="badge" style={{background:st.bg,color:st.color}}>{st.label}</span>
              <span style={{fontWeight:700,fontSize:15,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>{t.name}</span>
            </div>
            <div style={{fontSize:12,color:"var(--text3)",display:"flex",gap:8,flexWrap:"wrap"}}>
              <span>{fmtDate(t.date)}{t.points?.length ? ` · ${t.points.length}地点` : ""}</span>
              {tripTravelers.length > 0 && <span style={{color:"var(--text2)"}}>👤 {tripTravelers.map(x=>x.name).join("・")}</span>}
            </div>
          </div>
          {confirmId === t.id ? (
            <div className="row" style={{gap:6,flexShrink:0}} onClick={e=>e.stopPropagation()}>
              <span style={{fontSize:12,color:"var(--text2)"}}>削除しますか？</span>
              <Btn xs cls="btn-red" onClick={()=>{setConfirmId(null);onRemove(t.id);}}>はい</Btn>
              <Btn xs onClick={()=>setConfirmId(null)}>いいえ</Btn>
            </div>
          ) : (
            <Btn xs onClick={e=>{e.stopPropagation();setConfirmId(t.id)}} style={{flexShrink:0}}>🗑</Btn>
          )}
        </div>
        {t.points?.length > 0 && (
          <div style={{padding:"8px 14px",borderTop:"1px solid var(--border)",display:"flex",gap:5,flexWrap:"wrap"}}>
            {t.points.map(p => (
              <span key={p.id} className="chip" style={{background:PT[p.type]?.bg,color:PT[p.type]?.color}}>
                {PT[p.type]?.icon} {p.name||"未設定"}
              </span>
            ))}
          </div>
        )}
      </div>
    );
  };

  return (
    <div className="pad">
      <div className="row mb2" style={{justifyContent:"space-between",flexWrap:"wrap",gap:6}}>
        <span style={{color:"var(--text3)",fontSize:12}}>{trips.length}件の出張</span>
        <div className="row" style={{gap:5,flexWrap:"wrap"}}>
          <Btn sm onClick={onOpenMaster}>👤 出張者</Btn>
          <Btn sm onClick={onImport}>⬇ 読込</Btn>
          {trips.length > 0 && <Btn sm onClick={onExportAll}>⬆ 全共有</Btn>}
          <Btn cls="btn-blue" sm onClick={onCreate}>＋ 新規出張</Btn>
        </div>
      </div>

      <div style={{display:"flex",gap:4,background:"var(--surf2)",padding:4,borderRadius:9,marginBottom:12}}>
        <button style={tabStyle("active")} onClick={()=>setTab("active")}>
          進行中・予定 {active.length > 0 && <span style={{fontSize:11,opacity:.7}}>({active.length})</span>}
        </button>
        <button style={tabStyle("done")} onClick={()=>setTab("done")}>
          完了済み {done.length > 0 && <span style={{fontSize:11,opacity:.7}}>({done.length})</span>}
        </button>
      </div>

      {list.length === 0 && (
        <div className="empty">
          <div style={{fontSize:36,marginBottom:12}}>{tab === "active" ? "✈" : "✓"}</div>
          <p>{tab === "active" ? "出張がありません" : "完了した出張はありません"}</p>
          {tab === "active" && <p style={{fontSize:12,marginTop:6,color:"var(--text3)"}}>上のボタンで出張を作成してください</p>}
        </div>
      )}

      {list.map(t => <TripCard key={t.id} t={t} />)}
    </div>
  );
}

function NewTripModal({ onClose, onCreate, travelers, onOpenMaster }) {
  const today = new Date().toISOString().slice(0,10);
  const [f, setF] = useState({ name: "", date: today, travelerIds: [] });
  return (
    <Modal title="新規出張を作成" onClose={onClose}
      footer={<><Btn onClick={onClose}>キャンセル</Btn><Btn cls="btn-blue" onClick={()=>onCreate(f)} disabled={!f.name.trim()}>作成</Btn></>}
    >
      <div style={{display:"flex",flexDirection:"column",gap:12}}>
        <div>
          <label className="lbl">出張名 *</label>
          <input className="inp" value={f.name} onChange={e=>setF({...f,name:e.target.value})}
            placeholder="例: 東京出張 2026年5月" autoFocus />
        </div>
        <div>
          <label className="lbl">日付</label>
          <input className="inp" type="date" value={f.date} onChange={e=>setF({...f,date:e.target.value})} />
        </div>
        <div>
          <label className="lbl">出張者</label>
          <TravelerPicker selected={f.travelerIds} onChange={ids=>setF({...f,travelerIds:ids})}
            travelers={travelers} onOpenMaster={onOpenMaster} />
        </div>
      </div>
    </Modal>
  );
}

function AddPointModal({ onClose, onAdd }) {
  const [f, setF] = useState({ type:"visit", name:"", address:"", lat:null, lng:null, plannedArrival:"", plannedDeparture:"" });
  const [showAddr, setShowAddr] = useState(false);
  return (
    <>
    <Modal title="地点を追加" onClose={onClose}
      footer={<><Btn onClick={onClose}>キャンセル</Btn><Btn cls="btn-blue" onClick={()=>onAdd(f)} disabled={!f.name.trim()}>追加</Btn></>}
    >
      <div style={{display:"flex",flexDirection:"column",gap:12}}>
        <div>
          <label className="lbl">種別</label>
          <select className="inp" value={f.type} onChange={e=>setF({...f,type:e.target.value})}>
            {Object.entries(PT).map(([k,v])=><option key={k} value={k}>{v.icon} {v.label}</option>)}
          </select>
        </div>
        <div>
          <label className="lbl">場所名 *</label>
          <input className="inp" value={f.name} onChange={e=>setF({...f,name:e.target.value})} placeholder="会社名・施設名など" />
        </div>
        <div>
          <label className="lbl">住所</label>
          <div className="row">
            <input className="inp flex1" value={f.address} onChange={e=>setF({...f,address:e.target.value,lat:null,lng:null})} placeholder="住所（任意）" />
            <Btn sm onClick={()=>setShowAddr(true)}>🔍</Btn>
          </div>
          {f.lat && <div style={{fontSize:11,color:"var(--text3)",marginTop:4}} className="mono">📡 {f.lat.toFixed(5)}, {f.lng.toFixed(5)}</div>}
        </div>
        {f.type !== "waypoint" && (
          <div className="g2">
            <div>
              <label className="lbl">予定 到着</label>
              <input className="inp" type="time" value={f.plannedArrival} onChange={e=>setF({...f,plannedArrival:e.target.value})} />
            </div>
            <div>
              <label className="lbl">予定 出発</label>
              <input className="inp" type="time" value={f.plannedDeparture} onChange={e=>setF({...f,plannedDeparture:e.target.value})} />
            </div>
          </div>
        )}
        {f.type === "waypoint" && (
          <div>
            <label className="lbl">予定 通過時刻</label>
            <input className="inp" type="time" value={f.plannedArrival} onChange={e=>setF({...f,plannedArrival:e.target.value})} />
          </div>
        )}
      </div>
    </Modal>
    {showAddr && <AddrSearchModal onSelect={r=>{setF({...f,address:r.address,lat:r.lat,lng:r.lng});setShowAddr(false)}} onClose={()=>setShowAddr(false)} />}
    </>
  );
}

function EditPointModal({ point, onClose, onSave }) {
  const [f, setF] = useState({
    type: point.type,
    name: point.name || "",
    address: point.address || "",
    lat: point.lat || null,
    lng: point.lng || null,
    plannedArrival: point.plannedArrival || "",
    plannedDeparture: point.plannedDeparture || "",
  });
  const [showAddr, setShowAddr] = useState(false);
  const isWaypoint = f.type === "waypoint";
  return (
    <>
    <Modal title="地点を編集" onClose={onClose}
      footer={<><Btn onClick={onClose}>キャンセル</Btn><Btn cls="btn-blue" onClick={()=>onSave(f)} disabled={!f.name.trim()}>保存</Btn></>}
    >
      <div style={{display:"flex",flexDirection:"column",gap:12}}>
        <div>
          <label className="lbl">種別</label>
          <select className="inp" value={f.type} onChange={e=>setF({...f,type:e.target.value})}>
            {Object.entries(PT).map(([k,v])=><option key={k} value={k}>{v.icon} {v.label}</option>)}
          </select>
        </div>
        <div>
          <label className="lbl">場所名 *</label>
          <input className="inp" value={f.name} onChange={e=>setF({...f,name:e.target.value})} autoFocus />
        </div>
        <div>
          <label className="lbl">住所</label>
          <div className="row">
            <input className="inp flex1" value={f.address} onChange={e=>setF({...f,address:e.target.value,lat:null,lng:null})} placeholder="住所（任意）" />
            <Btn sm onClick={()=>setShowAddr(true)}>🔍</Btn>
          </div>
          {f.lat && <div style={{fontSize:11,color:"var(--text3)",marginTop:4}} className="mono">📡 {f.lat.toFixed(5)}, {f.lng.toFixed(5)}</div>}
        </div>
        {isWaypoint ? (
          <div>
            <label className="lbl">予定 通過時刻</label>
            <input className="inp" type="time" value={f.plannedArrival} onChange={e=>setF({...f,plannedArrival:e.target.value})} />
          </div>
        ) : (
          <div className="g2">
            <div>
              <label className="lbl">{f.type==="departure" ? "予定 出発" : "予定 到着"}</label>
              <input className="inp" type="time" value={f.plannedArrival} onChange={e=>setF({...f,plannedArrival:e.target.value})} />
            </div>
            <div>
              <label className="lbl">{f.type==="arrival" ? "予定 到着" : "予定 出発"}</label>
              <input className="inp" type="time" value={f.plannedDeparture} onChange={e=>setF({...f,plannedDeparture:e.target.value})} />
            </div>
          </div>
        )}
      </div>
    </Modal>
    {showAddr && <AddrSearchModal onSelect={r=>{setF({...f,address:r.address,lat:r.lat,lng:r.lng});setShowAddr(false)}} onClose={()=>setShowAddr(false)} />}
    </>
  );
}

function TripDetail({ trip, travelers, onBack, onUpdate, onSelectPoint, onOpenMaster }) {
  const [showAdd, setShowAdd] = useState(false);
  const [editingPoint, setEditingPoint] = useState(null);
  const [editingTrip, setEditingTrip] = useState(false);
  const [tripEdit, setTripEdit] = useState({ name: trip.name, date: trip.date });
  const [confirmPtId, setConfirmPtId] = useState(null);

  const addPoint = (f) => {
    const p = { id: uid(), type: f.type, name: f.name, address: f.address, lat: f.lat||null, lng: f.lng||null,
      plannedArrival: f.plannedArrival, plannedDeparture: f.plannedDeparture,
      actualArrival: null, actualDeparture: null, actualPassage: null,
      stayDuration: null, memo: "" };
    onUpdate({ ...trip, points: [...(trip.points||[]), p] });
    setShowAdd(false);
  };

  const saveEdit = (f) => {
    onUpdate({ ...trip, points: trip.points.map(p =>
      p.id === editingPoint.id
        ? { ...p, type:f.type, name:f.name, address:f.address, lat:f.lat, lng:f.lng,
            plannedArrival:f.plannedArrival, plannedDeparture:f.plannedDeparture }
        : p
    )});
    setEditingPoint(null);
  };

  const movePoint = (id, dir) => {
    const pts = [...trip.points];
    const i = pts.findIndex(p => p.id === id);
    const j = i + dir;
    if (j < 0 || j >= pts.length) return;
    [pts[i], pts[j]] = [pts[j], pts[i]];
    onUpdate({ ...trip, points: pts });
  };

  const removePoint = (id) => {
    onUpdate({ ...trip, points: trip.points.filter(p => p.id !== id) });
    setConfirmPtId(null);
  };

  const setStatus = (s) => onUpdate({ ...trip, status: s });

  const pts = trip.points || [];
  const done = pts.filter(p => p.actualArrival || p.actualDeparture || p.actualPassage).length;

  return (
    <>
      <div className="pad">
        <div className="card">
          <div className="card-hdr">
            {editingTrip ? (
              <div style={{flex:1,display:"flex",flexDirection:"column",gap:8}}>
                <input className="inp" value={tripEdit.name}
                  onChange={e=>setTripEdit(p=>({...p,name:e.target.value}))}
                  placeholder="出張名" autoFocus
                  style={{fontSize:15,fontWeight:700}} />
                <div className="row" style={{gap:8}}>
                  <input className="inp flex1" type="date" value={tripEdit.date}
                    onChange={e=>setTripEdit(p=>({...p,date:e.target.value}))} />
                  <Btn cls="btn-blue" sm
                    disabled={!tripEdit.name.trim()}
                    onClick={()=>{onUpdate({...trip,...tripEdit});setEditingTrip(false);}}>
                    保存
                  </Btn>
                  <Btn sm onClick={()=>{setTripEdit({name:trip.name,date:trip.date});setEditingTrip(false);}}>
                    キャンセル
                  </Btn>
                </div>
              </div>
            ) : (
              <>
                <div style={{flex:1}}>
                  <div style={{fontWeight:700,fontSize:15,marginBottom:3}}>{trip.name}</div>
                  <div style={{fontSize:12,color:"var(--text3)"}}>{fmtDate(trip.date)}</div>
                </div>
                <div className="row" style={{gap:6}}>
                  <Btn xs onClick={()=>{setTripEdit({name:trip.name,date:trip.date});setEditingTrip(true);}}>✏</Btn>
                  {trip.status === "planned" && <Btn cls="btn-green" sm onClick={()=>setStatus("active")}>▶ 開始</Btn>}
                  {trip.status === "active" && <Btn cls="btn-ghost" sm onClick={()=>setStatus("completed")}>✓ 完了</Btn>}
                </div>
              </>
            )}
          </div>
          <div style={{padding:"8px 14px",display:"flex",gap:16,alignItems:"center"}}>
            <span style={{fontSize:12,color:"var(--text3)"}}>進捗</span>
            <div style={{flex:1,height:5,background:"var(--surf2)",borderRadius:3,overflow:"hidden"}}>
              <div style={{height:"100%",background:"var(--accent)",width:pts.length ? `${Math.round(done/pts.length*100)}%` : "0%",borderRadius:3,transition:"width .3s"}} />
            </div>
            <span style={{fontSize:12,color:"var(--text3)",minWidth:40,textAlign:"right"}}>{done}/{pts.length}</span>
          </div>
          <div style={{padding:"10px 14px",borderTop:"1px solid var(--border)"}}>
            <div style={{fontSize:11,fontWeight:700,color:"var(--text3)",marginBottom:8,letterSpacing:".06em",textTransform:"uppercase"}}>出張者</div>
            <TravelerPicker
              selected={trip.travelerIds||[]}
              onChange={ids=>onUpdate({...trip,travelerIds:ids})}
              travelers={travelers}
              onOpenMaster={onOpenMaster}
            />
          </div>
        </div>

        <div className="card">
          <div className="sec-title">行程 ({pts.length}地点)</div>
          {pts.length === 0
            ? <div className="empty" style={{padding:24}}><p style={{fontSize:13}}>地点を追加してください</p></div>
            : pts.map((p, i) => {
              const pt = PT[p.type];
              const isWp = p.type === "waypoint";
              const isDep = p.type === "departure";
              const isArr = p.type === "arrival";
              const stampNow = (field) => {
                const t = nowISO();
                const changes = { [field]: t };
                if (field === "actualDeparture" && p.actualArrival) changes.stayDuration = stayMin(p.actualArrival, t);
                if (field === "actualArrival" && p.actualDeparture) changes.stayDuration = stayMin(t, p.actualDeparture);
                onUpdate({ ...trip, points: trip.points.map(x => x.id === p.id ? { ...x, ...changes } : x) });
              };
              const arrDiff = diffMin(p.plannedArrival, p.actualArrival);
              const depDiff = diffMin(p.plannedDeparture, p.actualDeparture);
              const pasDiff = diffMin(p.plannedArrival, p.actualPassage);
              const DiffTag = ({ d }) => {
                if (d === null) return null;
                const color = d === 0 ? "var(--text3)" : d > 0 ? "var(--red)" : "var(--green)";
                return <span style={{fontSize:10,color,fontFamily:"monospace",fontWeight:700}}>{d>0?`+${d}m`:d<0?`${d}m`:"定刻"}</span>;
              };
              return (
                <div key={p.id}>
                  {/* Main info row */}
                  <div style={{padding:"10px 14px",display:"flex",alignItems:"center",gap:10}}>
                    <div className="picon" style={{background:pt?.bg,flexShrink:0}}><span style={{color:pt?.color}}>{pt?.icon}</span></div>
                    <div style={{flex:1,minWidth:0}}>
                      <div className="row" style={{gap:6,marginBottom:2}}>
                        <span style={{color:pt?.color,fontSize:11,fontWeight:700}}>{pt?.label}</span>
                        <span style={{fontWeight:700,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>{p.name||"未設定"}</span>
                      </div>
                      <div style={{fontSize:11,color:"var(--text3)",display:"flex",gap:8,flexWrap:"wrap",alignItems:"center"}}>
                        {p.plannedArrival && <span className="mono">予定 {p.plannedArrival}</span>}
                        {p.actualArrival && <span className="mono" style={{color:"var(--green)"}}>到着 {fmtTime(p.actualArrival)}</span>}
                        {p.actualPassage && <span className="mono" style={{color:"var(--green)"}}>通過 {fmtTime(p.actualPassage)}</span>}
                        {p.actualDeparture && <span className="mono" style={{color:"#fbbf24"}}>出発 {fmtTime(p.actualDeparture)}</span>}
                        <DiffTag d={isWp ? pasDiff : isDep ? depDiff : arrDiff} />
                      </div>
                    </div>
                    {/* Edit/reorder controls */}
                    <div className="row" style={{gap:3,flexShrink:0}} onClick={e=>e.stopPropagation()}>
                      {confirmPtId === p.id ? (
                        <>
                          <span style={{fontSize:11,color:"var(--text2)"}}>削除？</span>
                          <Btn xs cls="btn-red" onClick={()=>removePoint(p.id)}>はい</Btn>
                          <Btn xs onClick={()=>setConfirmPtId(null)}>いいえ</Btn>
                        </>
                      ) : (
                        <>
                          <div style={{display:"flex",flexDirection:"column",gap:2}}>
                            <Btn xs onClick={()=>movePoint(p.id,-1)} disabled={i===0} style={{padding:"2px 5px",lineHeight:1}}>↑</Btn>
                            <Btn xs onClick={()=>movePoint(p.id,1)} disabled={i===pts.length-1} style={{padding:"2px 5px",lineHeight:1}}>↓</Btn>
                          </div>
                          <Btn xs onClick={()=>setEditingPoint(p)} style={{alignSelf:"center"}}>✏</Btn>
                          <Btn xs onClick={()=>setConfirmPtId(p.id)} style={{alignSelf:"center"}}>🗑</Btn>
                        </>
                      )}
                      <span style={{color:"var(--text3)",fontSize:16,cursor:"pointer",alignSelf:"center"}} onClick={()=>onSelectPoint(p.id)}>›</span>
                    </div>
                  </div>
                  {/* Quick action buttons */}
                  <div style={{padding:"0 14px 10px",display:"flex",gap:6,flexWrap:"wrap"}}>
                    {(p.address||p.name) && (
                      <Btn xs cls="btn-blue" onClick={()=>openNav(p)}>🗺 ナビ</Btn>
                    )}
                    {isWp && !p.actualPassage && (
                      <Btn xs cls="btn-green" onClick={()=>stampNow("actualPassage")}>◆ 通過</Btn>
                    )}
                    {isWp && p.actualPassage && (
                      <span style={{fontSize:11,color:"var(--green)",fontFamily:"monospace",display:"flex",alignItems:"center",gap:4}}>◆ {fmtTime(p.actualPassage)} <DiffTag d={pasDiff} /></span>
                    )}
                    {!isWp && !isArr && !p.actualDeparture && (
                      <Btn xs cls="btn-amber" onClick={()=>stampNow("actualDeparture")}>▲ 出発</Btn>
                    )}
                    {!isWp && !isArr && p.actualDeparture && (
                      <span style={{fontSize:11,color:"#fbbf24",fontFamily:"monospace",display:"flex",alignItems:"center",gap:4}}>▲ {fmtTime(p.actualDeparture)} <DiffTag d={depDiff} /></span>
                    )}
                    {!isWp && !isDep && !p.actualArrival && (
                      <Btn xs cls="btn-green" onClick={()=>stampNow("actualArrival")}>▼ 到着</Btn>
                    )}
                    {!isWp && !isDep && p.actualArrival && (
                      <span style={{fontSize:11,color:"var(--green)",fontFamily:"monospace",display:"flex",alignItems:"center",gap:4}}>▼ {fmtTime(p.actualArrival)} <DiffTag d={arrDiff} /></span>
                    )}
                  </div>
                  {i < pts.length-1 && <div style={{width:2,height:12,background:"var(--border)",margin:"0 auto"}} />}
                </div>
              );
            })
          }
          <div style={{padding:"10px 14px",borderTop: pts.length > 0 ? "1px solid var(--border)" : "none"}}>
            <Btn cls="btn-ghost" style={{width:"100%",justifyContent:"center"}} onClick={()=>setShowAdd(true)}>＋ 地点を追加</Btn>
          </div>
        </div>
      </div>
      {showAdd && <AddPointModal onClose={()=>setShowAdd(false)} onAdd={addPoint} />}
      {editingPoint && <EditPointModal point={editingPoint} onClose={()=>setEditingPoint(null)} onSave={saveEdit} />}
    </>
  );
}

function PointDetail({ trip, point, onUpdate }) {
  const [media, setMedia] = useState({ photos: [], recordings: [] });
  const [isRec, setIsRec] = useState(false);
  const [recDur, setRecDur] = useState(0);
  const [lightbox, setLightbox] = useState(null);
  const [showAddr, setShowAddr] = useState(false);
  const [localMemo, setLocalMemo] = useState(point.memo || "");
  const mrRef = useRef(null);
  const chunksRef = useRef([]);
  const timerRef = useRef(null);
  const mediaRef = useRef(media);

  useEffect(() => { mediaRef.current = media; }, [media]);
  useEffect(() => { loadMedia(point.id).then(setMedia); }, [point.id]);
  useEffect(() => { setLocalMemo(point.memo || ""); }, [point.id]);
  useEffect(() => () => clearInterval(timerRef.current), []);

  const upd = useCallback((changes) => {
    const np = { ...point, ...changes };
    onUpdate({ ...trip, points: trip.points.map(p => p.id === point.id ? np : p) });
  }, [point, trip, onUpdate]);

  const stamp = (field) => {
    const t = nowISO();
    const changes = { [field]: t };
    if (field === "actualDeparture" && point.actualArrival) changes.stayDuration = stayMin(point.actualArrival, t);
    if (field === "actualArrival" && point.actualDeparture) changes.stayDuration = stayMin(t, point.actualDeparture);
    upd(changes);
  };

  const updMedia = (fn) => {
    setMedia(prev => {
      const n = fn(prev);
      saveMedia(point.id, n);
      return n;
    });
  };

  const addPhotos = async (files) => {
    const arr = [];
    for (const f of files) { arr.push({ id: uid(), data: await compress(f), ts: nowISO() }); }
    updMedia(p => ({ ...p, photos: [...p.photos, ...arr] }));
  };

  const startRec = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const mr = new MediaRecorder(stream);
      chunksRef.current = [];
      mr.ondataavailable = e => chunksRef.current.push(e.data);
      mr.start(1000);
      mrRef.current = mr;
      setIsRec(true);
      setRecDur(0);
      timerRef.current = setInterval(() => setRecDur(d => d+1), 1000);
    } catch { alert("マイクへのアクセスが許可されていません"); }
  };

  const stopRec = () => {
    const mr = mrRef.current;
    if (!mr) return;
    const dur = recDur;
    mr.onstop = () => {
      clearInterval(timerRef.current);
      const blob = new Blob(chunksRef.current, { type: "audio/webm" });
      const reader = new FileReader();
      reader.onload = e => {
        const rec = { id: uid(), data: e.target.result, dur, ts: nowISO() };
        updMedia(prev => ({ ...prev, recordings: [...prev.recordings, rec] }));
        setIsRec(false);
        setRecDur(0);
      };
      reader.readAsDataURL(blob);
    };
    mr.stop();
    mr.stream.getTracks().forEach(t => t.stop());
  };

  const pt = PT[point.type] || PT.visit;
  const isWaypoint = point.type === "waypoint";
  const isDep = point.type === "departure";
  const isArr = point.type === "arrival";

  const arrDiff = diffMin(point.plannedArrival, point.actualArrival);
  const depDiff = diffMin(point.plannedDeparture, point.actualDeparture);
  const passDiff = diffMin(point.plannedArrival, point.actualPassage);
  const stay = stayMin(point.actualArrival, point.actualDeparture);

  const DiffBadge = ({ d }) => {
    if (d === null) return null;
    const cls = d === 0 ? "diff-ok" : d > 0 ? "diff-late" : "diff-early";
    const label = d === 0 ? "定刻" : d > 0 ? `${d}分遅れ` : `${Math.abs(d)}分早め`;
    return <div className={`time-diff ${cls}`}>{label}</div>;
  };

  return (
    <div className="pad">
      {/* Header card */}
      <div className="card">
        <div className="card-hdr">
          <div className="row" style={{gap:10}}>
            <div className="picon" style={{background:pt.bg,width:38,height:38,borderRadius:10}}>
              <span style={{color:pt.color,fontSize:18}}>{pt.icon}</span>
            </div>
            <div>
              <div style={{color:pt.color,fontSize:11,fontWeight:700,marginBottom:2}}>{pt.label}</div>
              <div style={{fontWeight:700,fontSize:16}}>{point.name || "未設定"}</div>
            </div>
          </div>
          <Btn cls="btn-blue" sm onClick={()=>openNav(point)} disabled={!point.address&&!point.name}>
            🗺 ナビ
          </Btn>
        </div>
        {(point.address || point.lat) && (
          <div style={{padding:"8px 14px",borderTop:"1px solid var(--border)",fontSize:12,color:"var(--text3)",display:"flex",alignItems:"center",gap:8,flexWrap:"wrap"}}>
            {point.address && <span>📍 {point.address}</span>}
            {point.lat && <span className="mono" style={{fontSize:10}}>({point.lat.toFixed(4)}, {point.lng.toFixed(4)})</span>}
          </div>
        )}
      </div>

      {/* Location edit */}
      <div className="card">
        <div className="sec-title">場所情報</div>
        <div className="card-body" style={{display:"flex",flexDirection:"column",gap:10}}>
          <div>
            <label className="lbl">種別</label>
            <select className="inp" value={point.type} onChange={e=>upd({type:e.target.value})}>
              {Object.entries(PT).map(([k,v])=><option key={k} value={k}>{v.icon} {v.label}</option>)}
            </select>
          </div>
          <div>
            <label className="lbl">場所名</label>
            <input className="inp" value={point.name} onChange={e=>upd({name:e.target.value})} />
          </div>
          <div>
            <label className="lbl">住所</label>
            <div className="row">
              <input className="inp flex1" value={point.address||""} onChange={e=>upd({address:e.target.value,lat:null,lng:null})} placeholder="住所（任意）" />
              <Btn sm onClick={()=>setShowAddr(true)}>🔍</Btn>
            </div>
            {point.lat && <div style={{fontSize:11,color:"var(--text3)",marginTop:4}} className="mono">📡 {point.lat.toFixed(5)}, {point.lng.toFixed(5)}</div>}
          </div>
        </div>
      </div>

      {/* Times */}
      <div className="card">
        <div className="sec-title">時刻管理</div>
        <div className="card-body">
          {/* Planned times */}
          {!isWaypoint ? (
            <div className="g2 mb3">
              <div>
                <label className="lbl">{isDep ? "予定 出発" : "予定 到着"}</label>
                <input className="inp" type="time" value={isDep ? (point.plannedDeparture||"") : (point.plannedArrival||"")}
                  onChange={e => upd(isDep ? {plannedDeparture:e.target.value} : {plannedArrival:e.target.value})} />
              </div>
              {!isDep && !isArr && (
                <div>
                  <label className="lbl">予定 出発</label>
                  <input className="inp" type="time" value={point.plannedDeparture||""} onChange={e=>upd({plannedDeparture:e.target.value})} />
                </div>
              )}
              {isArr && (
                <div>
                  <label className="lbl">予定 到着</label>
                  <input className="inp" type="time" value={point.plannedArrival||""} onChange={e=>upd({plannedArrival:e.target.value})} />
                </div>
              )}
            </div>
          ) : (
            <div style={{marginBottom:12}}>
              <label className="lbl">予定 通過時刻</label>
              <input className="inp" type="time" value={point.plannedArrival||""} onChange={e=>upd({plannedArrival:e.target.value})} />
            </div>
          )}

          {/* Stamp buttons */}
          {isWaypoint ? (
            <div className="time-box">
              <div className="time-lbl">通過時刻</div>
              <div className="time-val">{fmtTime(point.actualPassage)}</div>
              <DiffBadge d={passDiff} />
              <Btn cls="btn-green" sm style={{width:"100%",justifyContent:"center",marginTop:8}} onClick={()=>stamp("actualPassage")}>◆ 通過打刻</Btn>
              {point.actualPassage && <Btn xs style={{width:"100%",justifyContent:"center",marginTop:4}} onClick={()=>upd({actualPassage:null})}>クリア</Btn>}
            </div>
          ) : (
            <div className={isDep||isArr ? "" : "g2"}>
              {!isDep && (
                <div className="time-box" style={isDep||isArr ? {marginBottom:10} : {}}>
                  <div className="time-lbl">{isArr ? "実際 到着" : "実際 到着"}</div>
                  <div className="time-val">{fmtTime(point.actualArrival)}</div>
                  <DiffBadge d={arrDiff} />
                  <Btn cls="btn-green" sm style={{width:"100%",justifyContent:"center",marginTop:8}} onClick={()=>stamp("actualArrival")}>▼ 到着打刻</Btn>
                  {point.actualArrival && <Btn xs style={{width:"100%",justifyContent:"center",marginTop:4}} onClick={()=>upd({actualArrival:null})}>クリア</Btn>}
                </div>
              )}
              {!isArr && (
                <div className="time-box">
                  <div className="time-lbl">{isDep ? "実際 出発" : "実際 出発"}</div>
                  <div className="time-val">{fmtTime(point.actualDeparture)}</div>
                  <DiffBadge d={depDiff} />
                  <Btn cls="btn-amber" sm style={{width:"100%",justifyContent:"center",marginTop:8}} onClick={()=>stamp("actualDeparture")}>▲ 出発打刻</Btn>
                  {point.actualDeparture && <Btn xs style={{width:"100%",justifyContent:"center",marginTop:4}} onClick={()=>upd({actualDeparture:null})}>クリア</Btn>}
                </div>
              )}
            </div>
          )}

          {stay !== null && (
            <div className="stay-bar">
              <span style={{fontSize:12,color:"var(--text3)"}}>滞在時間 </span>
              <span className="mono" style={{fontWeight:700,color:"var(--amber)",fontSize:15}}>{fmtMin(stay)}</span>
            </div>
          )}
        </div>
      </div>

      {/* Memo */}
      <div className="card">
        <div className="sec-title">メモ</div>
        <div className="card-body">
          <textarea className="inp" placeholder="現場でのメモ..." value={localMemo}
            onChange={e=>setLocalMemo(e.target.value)}
            onBlur={()=>upd({memo:localMemo})} />
        </div>
      </div>

      {/* Photos */}
      <div className="card">
        <div className="sec-title" style={{display:"flex",alignItems:"center",justifyContent:"space-between"}}>
          <span>写真 ({media.photos.length})</span>
          <label className="btn btn-ghost btn-xs" style={{cursor:"pointer",margin:"0 0 0 auto"}}>
            📷 追加
            <input type="file" accept="image/*" multiple style={{display:"none"}}
              onChange={e=>e.target.files.length&&addPhotos([...e.target.files])} />
          </label>
        </div>
        <div className="card-body">
          {media.photos.length === 0
            ? <div style={{textAlign:"center",padding:"12px 0",color:"var(--text3)",fontSize:13}}>写真がありません</div>
            : <div className="photo-grid">
                {media.photos.map(ph => (
                  <div key={ph.id} className="photo-wrap">
                    <img src={ph.data} className="photo-img" alt="" onClick={()=>setLightbox(ph.data)} />
                    <button className="photo-del" onClick={()=>updMedia(m=>({...m,photos:m.photos.filter(p=>p.id!==ph.id)}))}>✕</button>
                  </div>
                ))}
              </div>
          }
        </div>
      </div>

      {/* Recordings */}
      <div className="card">
        <div className="sec-title" style={{display:"flex",alignItems:"center",justifyContent:"space-between"}}>
          <span>録音 ({media.recordings.length})</span>
          {!isRec
            ? <Btn cls="btn-red" xs onClick={startRec} style={{margin:"0 0 0 auto"}}>🎙 録音開始</Btn>
            : <div className="row" style={{gap:8}}>
                <span className="pulse mono" style={{color:"var(--red)",fontSize:13}}>● {fmtRecTime(recDur)}</span>
                <Btn xs onClick={stopRec}>■ 停止</Btn>
              </div>
          }
        </div>
        <div className="card-body">
          {media.recordings.length === 0 && !isRec
            ? <div style={{textAlign:"center",padding:"12px 0",color:"var(--text3)",fontSize:13}}>録音がありません</div>
            : media.recordings.map(rec => (
                <div key={rec.id} className="rec-item">
                  <span style={{fontSize:16}}>🎵</span>
                  <div style={{flex:1,minWidth:0}}>
                    <audio src={rec.data} controls />
                    <div style={{fontSize:11,color:"var(--text3)",marginTop:3}}>{fmtDateTime(rec.ts)} · {fmtRecTime(rec.dur)}</div>
                  </div>
                  <Btn xs onClick={()=>updMedia(m=>({...m,recordings:m.recordings.filter(r=>r.id!==rec.id)}))}>🗑</Btn>
                </div>
              ))
          }
        </div>
      </div>

      {lightbox && (
        <div className="lightbox" onClick={()=>setLightbox(null)}>
          <img src={lightbox} alt="" />
        </div>
      )}
      {showAddr && <AddrSearchModal onSelect={r=>{upd({address:r.address,lat:r.lat,lng:r.lng});setShowAddr(false)}} onClose={()=>setShowAddr(false)} />}
    </div>
  );
}



// ════════════════════════════════════════════════════════
//  APP (Firebase-backed)
// ════════════════════════════════════════════════════════
function App() {
  // ── Setup state ──────────────────────────────────────
  const [setupDone, setSetupDone] = useState(false);
  const [teamId, setTeamId] = useState('');

  useEffect(() => {
    const team = localStorage.getItem('fb_team');
    if (team) {
      const ok = initFirebase(BAKED_CONFIG, team);
      if (ok) { setTeamId(team); setSetupDone(true); }
    }
  }, []);

  const handleSetupComplete = (cfg, team) => {
    setTeamId(team); setSetupDone(true);
  };

  const resetSetup = () => {
    localStorage.removeItem('fb_config');
    localStorage.removeItem('fb_team');
    setSetupDone(false); setTeamId('');
  };

  if (!setupDone) return <SetupScreen onComplete={handleSetupComplete} />;
  return <MainApp teamId={teamId} onResetSetup={resetSetup} />;
}

function MainApp({ teamId, onResetSetup }) {
  const [trips, setTrips] = useState([]);
  const [travelers, setTravelers] = useState([]);
  const [screen, setScreen] = useState("list");
  const [tripId, setTripId] = useState(null);
  const [pointId, setPointId] = useState(null);
  const [showNew, setShowNew] = useState(false);
  const [showMaster, setShowMaster] = useState(false);
  const [showTeamMenu, setShowTeamMenu] = useState(false);
  const [importMsg, setImportMsg] = useState(null);
  const [online, setOnline] = useState(true);
  const [ready, setReady] = useState(false);
  const importRef = useRef(null);

  // ── Firestore real-time listeners ────────────────────
  useEffect(() => {
    if (!_db) return;

    const unsubTrips = tripsCol().onSnapshot(
      snap => {
        setTrips(snap.docs.map(d => ({ ...d.data(), id: d.id })));
        setOnline(true); setReady(true);
      },
      () => { setOnline(false); setReady(true); }
    );

    const unsubTv = tvDoc().onSnapshot(
      snap => { setTravelers(snap.data()?.list || []); },
      () => {}
    );

    return () => { unsubTrips(); unsubTv(); };
  }, [teamId]);

  const trip  = trips.find(t => t.id === tripId);
  const point = trip?.points?.find(p => p.id === pointId);

  const updTrip = useCallback(async (t) => {
    setTrips(prev => prev.map(x => x.id === t.id ? t : x));
    await fsaveTrip(t);
  }, []);

  const createTrip = async ({ name, date, travelerIds }) => {
    const t = { id: uid(), name, date, status: "planned", points: [], travelerIds: travelerIds||[] };
    await fsaveTrip(t);
    setTripId(t.id); setScreen("trip"); setShowNew(false);
  };

  const removeTrip = async (id) => {
    setTrips(p => p.filter(t => t.id !== id));
    await fdeleteTrip(id);
  };

  // ── Export ───────────────────────────────────────────
  const exportTrips = async (targetTrips) => {
    const tripsWithMedia = await Promise.all(targetTrips.map(async t => {
      const media = {};
      for (const p of t.points||[]) {
        const m = await loadMedia(p.id);
        if (m.photos.length || m.recordings.length) media[p.id] = m;
      }
      return { ...t, _media: media };
    }));
    const data = {
      version: 2, exportedAt: nowISO(), appName: "出張報告書管理",
      travelers, trips: tripsWithMedia,
    };
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    const label = targetTrips.length === 1 ? targetTrips[0].name : `出張データ_${new Date().toISOString().slice(0,10)}`;
    a.href = url; a.download = `${label}.json`; a.click();
    URL.revokeObjectURL(url);
  };

  // ── Import ───────────────────────────────────────────
  const handleImportFile = (file) => {
    const reader = new FileReader();
    reader.onload = async (e) => {
      try {
        const data = JSON.parse(e.target.result);
        if (data.appName !== "出張報告書管理") { setImportMsg({ type:"err", text:"対応していないファイル形式です" }); return; }
        let addedTrips=0, updatedTrips=0, addedTv=0;
        const currentIds = trips.map(t => t.id);
        const currentNames = travelers.map(x => x.name);
        const newTvs = [...travelers];
        for (const tv of data.travelers||[]) {
          if (!currentNames.includes(tv.name)) { newTvs.push({ ...tv, id: uid() }); addedTv++; }
        }
        await fsaveTvs(newTvs);
        for (const t of data.trips||[]) {
          const { _media, ...tripData } = t;
          if (currentIds.includes(tripData.id)) updatedTrips++; else addedTrips++;
          await fsaveTrip(tripData);
          for (const [pid, m] of Object.entries(_media||{})) await saveMedia(pid, m);
        }
        setImportMsg({ type:"ok", text:`完了：出張 ${addedTrips}件追加・${updatedTrips}件更新、出張者 ${addedTv}名追加` });
      } catch { setImportMsg({ type:"err", text:"ファイルの読み込みに失敗しました" }); }
    };
    reader.readAsText(file);
  };

  const headers = {
    list:  { title: "✈ 出張報告書管理", back: null },
    trip:  { title: trip?.name || "出張詳細", back: () => setScreen("list") },
    point: { title: point?.name || "地点詳細", back: () => setScreen("trip") },
  };
  const h = headers[screen] || headers.list;

  if (!ready) return (
    <>
      <style>{css}</style>
      <div id="root" style={{justifyContent:"center",alignItems:"center",display:"flex",flexDirection:"column",gap:12}}>
        <span style={{fontSize:24}}>✈</span>
        <span style={{color:"#4a5570",fontSize:13}}>Firestore に接続中…</span>
      </div>
    </>
  );

  return (
    <>
      <style>{css}</style>
      <div id="root">
        <div className="hdr">
          {h.back && <Btn sm onClick={h.back}>← 戻る</Btn>}
          <h1 style={{flex:1}}>{h.title}</h1>
          {/* Online indicator */}
          <span style={{fontSize:10,padding:"2px 7px",borderRadius:10,fontWeight:700,
            background: online ? "rgba(74,222,128,.12)" : "rgba(248,113,113,.12)",
            color: online ? "#4ade80" : "#f87171"}}>
            {online ? "● オンライン" : "○ オフライン"}
          </span>
          {/* Team badge */}
          <span onClick={()=>setShowTeamMenu(v=>!v)} style={{
            fontSize:11,padding:"3px 8px",borderRadius:8,cursor:"pointer",
            background:"var(--surf2)",color:"var(--text3)",border:"1px solid var(--border2)",
            whiteSpace:"nowrap",maxWidth:90,overflow:"hidden",textOverflow:"ellipsis",
          }}>👥 {teamId}</span>
          {screen === "trip" && trip && (
            <Btn xs cls="btn-ghost" onClick={()=>exportTrips([trip])}>⬆ 共有</Btn>
          )}
        </div>

        {/* Team menu */}
        {showTeamMenu && (
          <div style={{background:"var(--surf2)",borderBottom:"1px solid var(--border)",padding:"12px 16px",display:"flex",flexDirection:"column",gap:8}}>
            <div style={{fontSize:12,color:"var(--text3)"}}>現在のチーム: <strong style={{color:"var(--text)"}}>{teamId}</strong></div>
            <div style={{fontSize:11,color:"var(--text3)"}}>写真・録音はこのデバイスにのみ保存されます</div>
            <div className="row" style={{gap:6}}>
              <Btn xs onClick={()=>setShowTeamMenu(false)}>閉じる</Btn>
              <Btn xs cls="btn-red" onClick={()=>{setShowTeamMenu(false);onResetSetup();}}>チーム変更・再設定</Btn>
            </div>
          </div>
        )}

        {importMsg && (
          <div style={{margin:"8px 14px 0",padding:"10px 14px",borderRadius:8,
            background: importMsg.type==="ok" ? "rgba(74,222,128,.1)" : "rgba(248,113,113,.1)",
            border: `1px solid ${importMsg.type==="ok" ? "rgba(74,222,128,.25)" : "rgba(248,113,113,.25)"}`,
            color: importMsg.type==="ok" ? "#4ade80" : "#f87171",
            fontSize:13,display:"flex",alignItems:"center",justifyContent:"space-between",gap:8}}>
            <span>{importMsg.type==="ok" ? "✓ " : "✕ "}{importMsg.text}</span>
            <Btn xs onClick={()=>setImportMsg(null)}>✕</Btn>
          </div>
        )}

        <div className="scrl">
          {screen === "list" && (
            <TripList trips={trips} travelers={travelers} onCreate={()=>setShowNew(true)}
              onSelect={id=>{setTripId(id);setScreen("trip")}} onRemove={removeTrip}
              onOpenMaster={()=>setShowMaster(true)}
              onExportAll={()=>exportTrips(trips)}
              onImport={()=>importRef.current?.click()} />
          )}
          {screen === "trip" && trip && (
            <TripDetail trip={trip} travelers={travelers} onBack={()=>setScreen("list")}
              onUpdate={updTrip} onOpenMaster={()=>setShowMaster(true)}
              onSelectPoint={id=>{setPointId(id);setScreen("point")}} />
          )}
          {screen === "point" && trip && point && (
            <PointDetail trip={trip} point={point} onUpdate={updTrip} />
          )}
        </div>

        {showNew && <NewTripModal onClose={()=>setShowNew(false)} onCreate={createTrip}
          travelers={travelers} onOpenMaster={()=>setShowMaster(true)} />}
        {showMaster && (
          <TravelerMasterModal
            onClose={async () => { setShowMaster(false); }}
            overrideSave={async (list) => { await fsaveTvs(list); }}
          />
        )}
        <input ref={importRef} type="file" accept=".json" style={{display:"none"}}
          onChange={e=>{e.target.files[0]&&handleImportFile(e.target.files[0]);e.target.value="";}} />
      </div>
    </>
  );
}

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(React.createElement(App));
</script>
</body>
</html>
