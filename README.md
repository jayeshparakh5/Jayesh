        <h3 id="productsTitle">Products</h3>
        <div id="productsGrid" class="grid"></div>
      </section>

      <section class="card" id="about" style="margin-top:12px">
        <h3 id="aboutTitle">About</h3>
        <div class="muted small" id="aboutText">This is a demo marketplace built as a single HTML file (localStorage). For real use add server, DB and payment gateway.</div>
      </section>

      <section class="card" id="ordersCard" style="margin-top:12px;display:none">
        <h3 id="purchaseTitle">Your Purchases</h3>
        <div id="ordersList"></div>
      </section>

    </main>

    <aside>
      <div class="card">
        <h4 id="liveTitle">Live Delivery Videos</h4>
        <div class="video-stack" id="liveVideos"></div>
        <div style="margin-top:8px" class="small muted">Videos are public YouTube embeds. Admin can update links.</div>
      </div>

      <div style="height:12px"></div>

      <div class="card">
        <div style="font-weight:700" id="sellersTitle">Sellers</div>
        <div id="asideSellers" class="muted small" style="margin-top:8px"></div>
      </div>

      <div style="height:12px"></div>

      <div class="card">
        <div style="font-weight:700" id="ordersTitle">Recent Orders</div>
        <div id="asideOrders" class="muted small" style="margin-top:8px"></div>
      </div>
    </aside>
  </div>

  <footer>© <span id="year"></span> <span id="siteFooterTitle">Anu Krupa Marketplace</span> — Demo (local)</footer>
</div>

<!-- modal root -->
<div id="modalRoot" style="display:none"></div>

<script>
/*
  Single-file marketplace demo (client-side).
  - Registration/Login (Buyer, Seller)
  - Seller dashboard: add/edit products (images via file=>dataURL or image URL)
  - Buyer: browse, buy (UPI simulated), attach delivery video URL
  - Admin: site settings, approve sellers, manage products, attach videos, confirm delivery & payout simulation
  - Multi-language (en/hi) and currency select
  - All data stored in localStorage under keys prefixed with ak_
*/

// ---------- Utilities ----------
const $ = id => document.getElementById(id);
function setStore(k,v){ localStorage.setItem(k, JSON.stringify(v)); }
function getStore(k, fallback){ const s = localStorage.getItem(k); return s?JSON.parse(s):fallback; }
function uid(prefix='u'){ return prefix + Date.now().toString(36) + Math.random().toString(36).slice(2,6); }
function showModal(html){ const root = $('modalRoot'); root.style.display='block'; root.innerHTML = `<div class="modal-back" onclick="closeModal()"><div class="modal" onclick="event.stopPropagation()">${html}</div></div>`; }
function closeModal(){ const root = $('modalRoot'); root.style.display='none'; root.innerHTML=''; }
function esc(s){ if(!s) return ''; return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function youtubeId(url){ try{ const u=new URL(url); const v=u.searchParams.get('v'); if(v) return v; const parts=u.pathname.split('/'); return parts[parts.length-1]; }catch(e){ return ''; } }

// ---------- Seed data (once) ----------
(function seed(){
  if(getStore('ak_seeded')) return;
  // admin
  setStore('ak_admin', {email:'jayesh.parakh@anukrupa.demo', password:'Admin@1234', name:'Jayesh Parakh'});
  // sellers (5)
  const sellers = [];
  for(let i=1;i<=5;i++){
    sellers.push({
      id:`s${i}`,
      email:`seller${i}@anukrupa.demo`,
      password:`Seller@${i}23`,
      name:`Seller ${i}`,
      shopName:`Shop ${i}`,
      sellerType: (i%2===0) ? 'gst' : 'nogst',
      pan: `PAN${i}XXX`,
      aadhaar: `AAD${i}YYY`,
      bankAcc: `ACC00${i}`,
      ifsc: `IFSC000${i}`,
      kycStatus: 'verified',
      approved: true
    });
  }
  setStore('ak_sellers', sellers);

  // products: 5 each
  const imgs = [
    'https://picsum.photos/seed/p1/800/600',
    'https://picsum.photos/seed/p2/800/600',
    'https://picsum.photos/seed/p3/800/600',
    'https://picsum.photos/seed/p4/800/600',
    'https://picsum.photos/seed/p5/800/600',
    'https://picsum.photos/seed/p6/800/600'
  ];
  const products = [];
  let pid = 1;
  sellers.forEach((s,si)=>{
    for(let j=1;j<=5;j++){
      products.push({
        id: 'p' + pid,
        sellerId: s.id,
        sellerEmail: s.email,
        sellerName: s.shopName,
        name: `Item ${si+1}-${j}`,
        price: 700 + (si+1)*j*110,
        category: ['Doors','Tiles','Windows','Fittings','Tools'][j%5],
        img: imgs[(pid-1)%imgs.length],
        published: true,
        currency: 'INR'
      });
      pid++;
    }
  });
  setStore('ak_products', products);

  // admin config
  setStore('ak_config', {
    siteTitle: 'Anu Krupa Marketplace',
    logoText: 'AK',
    heroTitle: 'Anu Krupa — Trusted Marketplace',
    heroSub: 'Sellers KYC • Delivery Video Proof • Admin-controlled payouts',
    primary: '#0ea5a4',
    accent: '#7c3aed',
    commission: 10,
    liveVideos: [
      'https://www.youtube.com/watch?v=4NRXx6U8ABQ',
      'https://www.youtube.com/watch?v=kXYiU_JCYtU',
      'https://www.youtube.com/watch?v=JGwWNGJdvx8',
      'https://www.youtube.com/watch?v=YykjpeuMNEk',
      'https://www.youtube.com/watch?v=uelHwf8o7_U'
    ],
    currencies: { // simple demo rates (base INR)
      INR: {symbol:'₹', rate:1},
      USD: {symbol:'$', rate:0.012},
      AED: {symbol:'د.إ', rate:0.044},
      EUR: {symbol:'€', rate:0.011}
    }
  });

  setStore('ak_users', []); // buyers
  setStore('ak_orders', []);
  setStore('ak_seeded', true);
})();

// ---------- UI helpers ----------
function applyConfigUI(){
  const cfg = getStore('ak_config', {});
  document.documentElement.style.setProperty('--primary', cfg.primary || '#0ea5a4');
  document.documentElement.style.setProperty('--accent', cfg.accent || '#7c3aed');
  $('logoText').innerText = cfg.logoText || 'AK';
  $('siteTitle').innerText = cfg.siteTitle || 'Anu Krupa Marketplace';
  $('heroTitle').innerText = cfg.heroTitle || 'Anu Krupa — Trusted Marketplace';
  $('heroSub').innerText = cfg.heroSub || '';
  $('siteFooterTitle').innerText = cfg.siteTitle || 'Anu Krupa Marketplace';
  // currency select
  const curSel = $('currency'); curSel.innerHTML = '';
  const currencies = cfg.currencies || {INR:{symbol:'₹',rate:1}};
  Object.keys(currencies).forEach(k=>{
    const o = document.createElement('option'); o.value = k; o.innerText = k; curSel.appendChild(o);
  });
}
applyConfigUI();

// language dictionary (simple)
const DICT = {
  en: {
    login: 'Login', register: 'Register', sellerOnboard: 'Seller Onboard', admin: 'Admin', products:'Products',
    liveVideos:'Live Delivery Videos', sellers:'Sellers', recentOrders:'Recent Orders', buy:'Buy', price:'Price',
    commission:'Commission', net:'Net', quantity:'Quantity', buyNow:'Buy Now (UPI)', pay:'Pay via UPI', attachVideo:'Attach Delivery Video',
    approve:'Approve', reject:'Reject', settings:'Settings', siteSettings:'Site Settings', save:'Save',
    orders:'Orders', purchases:'Purchases', myProducts:'My Products', addProduct:'Add Product', edit:'Edit'
  },
  hi: {
    login: 'लॉगिन', register: 'रजिस्टर', sellerOnboard: 'सेलर ऑनबोर्ड', admin: 'एडमिन', products:'प्रोडक्ट्स',
    liveVideos:'डिलीवरी वीडियो (लाइव)', sellers:'सेलर्स', recentOrders:'हाल के ऑर्डर', buy:'खरीदें', price:'मूल्य',
    commission:'कमीशन', net:'नेट', quantity:'मात्रा', buyNow:'खरीदें (UPI)', pay:'UPI से पे करें', attachVideo:'डिलीवरी वीडियो जोड़ें',
    approve:'स्वीकृत करें', reject:'अस्वीकृत', settings:'सेटिंग्स', siteSettings:'साइट सेटिंग्स', save:'सहेजें',
    orders:'ऑर्डर', purchases:'खरीद', myProducts:'मेरे प्रोडक्ट', addProduct:'प्रोडक्ट जोड़ें', edit:'संपादित करें'
  }
};
let LANG = localStorage.getItem('ak_lang') || 'en';
function t(key){ return (DICT[LANG] && DICT[LANG][key]) || key; }
function setLang(l){ LANG = l; localStorage.setItem('ak_lang', l); renderAll(); }

// ---------- Render functions ----------
function renderNav(){
  const admin = getStore('ak_current_admin');
  const user = getStore('ak_current_user');
  const nav = $('navRight'); nav.innerHTML = '';
  if(admin){ nav.innerHTML = `<span class="badge">${esc(admin.name)} (Admin)</span> <button class="btn btn-ghost" onclick="adminLogout()">Logout</button>`; }
  else if(user){ nav.innerHTML = `<span class="badge">${esc(user.name||user.email)}</span> <button class="btn btn-ghost" onclick="userLogout()">Logout</button>`; if(user.role==='seller') nav.innerHTML += ` <button class="btn btn-ghost" onclick="openSellerDashboard()">Seller</button>`; }
  else { nav.innerHTML = `<button class="btn btn-ghost" onclick="showLogin()">${t('login')}</button>`; }
}

function renderProducts(filterQ=''){
  const products = getStore('ak_products', []);
  const cfg = getStore('ak_config', {});
  const currency = $('currency').value || 'INR';
  const rate = (cfg.currencies && cfg.currencies[currency] && cfg.currencies[currency].rate) || 1;
  const sym = (cfg.currencies && cfg.currencies[currency] && cfg.currencies[currency].symbol) || '';
  const grid = $('productsGrid'); grid.innerHTML = '';
  products.filter(p=>p.published).filter(p=>{
    if(!filterQ) return true;
    const q = filterQ.toLowerCase();
    return (p.name||'').toLowerCase().includes(q) || (p.category||'').toLowerCase().includes(q) || (p.sellerName||'').toLowerCase().includes(q);
  }).forEach(p=>{
    const commission = Math.round((p.price * cfg.commission)/100);
    const net = p.price - commission;
    const priceShow = `${sym} ${Math.round(p.price * rate)}`;
    const comShow = `${sym} ${Math.round(commission * rate)}`;
    const netShow = `${sym} ${Math.round(net * rate)}`;
    const card = document.createElement('div'); card.className='card';
    card.innerHTML = `
      <div class="product-img">${p.img?`<img src="${esc(p.img)}" alt="${esc(p.name)}">`:'No Image'}</div>
      <div style="margin-top:8px;font-weight:700">${esc(p.name)}</div>
      <div class="muted small">${esc(p.category)} • ${esc(p.sellerName)}</div>
      <div style="font-weight:800;margin-top:6px">${priceShow}</div>
      <div class="muted small">${t('commission')}: ${cfg.commission}% → ${comShow} • ${t('net')}: ${netShow}</div>
      <div style="display:flex;gap:8px;justify-content:flex-end;margin-top:8px">
        <button class="btn btn-primary" onclick="openProduct('${p.id}')">${t('buy')}</button>
      </div>
    `;
    grid.appendChild(card);
  });
  renderAsideSellers(); renderAsideOrders();
}

function renderAsideSellers(){
  const sellers = getStore('ak_sellers', []);
  $('asideSellers').innerHTML = sellers.map(s=>`<div style="padding:6px 0;border-bottom:1px solid #f2f6fb"><b>${esc(s.shopName)}</b><div class="small muted">${esc(s.email)} • ${esc(s.sellerType)}</div></div>`).join('');
}

function renderAsideOrders(){
  const orders = getStore('ak_orders', []);
  const html = orders.slice().reverse().slice(0,6).map(o=>`<div style="padding:6px 0;border-bottom:1px solid #f2f6fb"><b>${esc(o.productName)}</b><div class="small muted">${esc(o.id)} • ${esc(o.status)} • ₹${o.price}</div></div>`).join('');
  $('asideOrders').innerHTML = html || '<div class="small muted">No orders yet</div>';
}

function renderLiveVideos(){
  const cfg = getStore('ak_config', {});
  const vids = cfg.liveVideos || [];
  const wrap = $('liveVideos'); wrap.innerHTML = '';
  vids.slice(0,5).forEach(u=>{
    const id = youtubeId(u);
    const iframe = document.createElement('iframe');
    iframe.className = 'video-frame';
    iframe.src = id ? `https://www.youtube.com/embed/${id}` : '';
    iframe.allow='accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture';
    wrap.appendChild(iframe);
  });
}

function renderOrdersForUser(){
  const cur = getStore('ak_current_user');
  if(!cur) { $('ordersCard').style.display='none'; return; }
  const orders = getStore('ak_orders', []).filter(o=>o.buyer === cur.email);
  if(!orders.length){ $('ordersCard').style.display='none'; return; }
  $('ordersCard').style.display='block';
  $('ordersList').innerHTML = orders.map(o=>`<div style="padding:8px;border-bottom:1px solid #f2f6fb"><b>Order ${esc(o.id)}</b><div class="small muted">${esc(o.productName)} • Qty ${o.qty} • ₹${o.price}</div><div style="margin-top:6px">Status: <b>${esc(o.status)}</b></div><div style="margin-top:6px">${o.deliveryVideoUrl?`<a href="${esc(o.deliveryVideoUrl)}" target="_blank">View Delivery Video</a>`:`<button class="btn btn-ghost" onclick="attachVideoPrompt('${o.id}')">${t('attachVideo')}</button>`}</div></div>`).join('');
}

// ---------- Search ----------
function onSearch(){ const q = $('search').value.trim(); renderProducts(q); }

// ---------- Auth: Login / Register / Seller Onboard ----------
function showLogin(){
  showModal(`<div style="font-weight:700">${t('login')}</div>
    <label>Email</label><input id="liEmail" />
    <label>Password</label><input id="liPass" type="password" />
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Close</button><button class="btn btn-primary" onclick="doLogin()">Login</button></div>
    <div style="margin-top:8px" class="small muted">New user? <a href="#" onclick="showRegister()">Register</a> | <a href='#' onclick='showSellerOnboard()'>${t('sellerOnboard')}</a></div>`);
}

function doLogin(){
  const email = $('liEmail').value.trim(); const pass = $('liPass').value;
  if(!email || !pass) return alert('Enter credentials');
  // admin?
  const admin = getStore('ak_admin');
  if(admin && admin.email === email && admin.password === pass){ setStore('ak_current_admin', admin); closeModal(); renderAll(); return alert('Admin logged in'); }
  // sellers
  const sellers = getStore('ak_sellers', []);
  const s = sellers.find(x=>x.email === email && x.password === pass);
  if(s){ if(!s.approved) return alert('Your seller profile pending admin approval'); setStore('ak_current_user', {email:s.email, name:s.name, role:'seller', sellerId:s.id}); closeModal(); renderAll(); return alert('Seller logged in'); }
  // buyers
  const users = getStore('ak_users', []);
  const u = users.find(x=>x.email === email && x.password === pass);
  if(u){ setStore('ak_current_user', {email:u.email, name:u.name, role:'buyer'}); closeModal(); renderAll(); return alert('Logged in'); }
  alert('Invalid credentials');
}

function showRegister(){
  showModal(`<div style="font-weight:700">${t('register')}</div>
    <label>Full Name</label><input id="rgName" />
    <label>Email</label><input id="rgEmail" />
    <label>Password</label><input id="rgPass" type="password" />
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Close</button><button class="btn btn-primary" onclick="doRegister()">Register</button></div>`);
}
function doRegister(){
  const name = $('rgName').value.trim(), email = $('rgEmail').value.trim(), pass = $('rgPass').value;
  if(!name||!email||!pass) return alert('Complete details');
  const users = getStore('ak_users', []);
  if(users.find(x=>x.email === email)) return alert('Email already used');
  users.push({id: uid('u'), name, email, password: pass});
  setStore('ak_users', users);
  setStore('ak_current_user', {email, name, role:'buyer'});
  closeModal(); renderAll(); alert('Registered & logged in');
}

function showSellerOnboard(){
  showModal(`<div style="font-weight:700">${t('sellerOnboard')}</div>
    <label>Full Name</label><input id="soName" />
    <label>Email</label><input id="soEmail" />
    <label>Password</label><input id="soPass" type="password" />
    <label>Shop/Business Name</label><input id="soShop" />
    <label>Seller Type</label><select id="soType"><option value="gst">GST</option><option value="nogst">No GST</option></select>
    <label>PAN</label><input id="soPan" /><label>Aadhaar</label><input id="soAad" />
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Cancel</button><button class="btn btn-primary" onclick="doSellerOnboard()">Submit</button></div>`);
}
function doSellerOnboard(){
  const name=$('soName').value.trim(), email=$('soEmail').value.trim(), pass=$('soPass').value, shop=$('soShop').value.trim(), type=$('soType').value, pan=$('soPan').value, aad=$('soAad').value;
  if(!name||!email||!pass||!shop) return alert('Complete details');
  const sellers = getStore('ak_sellers', []);
  const id = 's' + (sellers.length + 1);
  sellers.push({id, email, password: pass, name, shopName: shop, sellerType:type, pan, aad, approved:false, kycStatus:'pending'});
  setStore('ak_sellers', sellers);
  setStore('ak_current_user', {email, name, role:'seller', sellerId:id});
  closeModal(); renderAll(); alert('Seller submitted for approval (admin will verify).');
}

function userLogout(){ localStorage.removeItem('ak_current_user'); renderAll(); alert('Logged out'); }
function adminLogout(){ localStorage.removeItem('ak_current_admin'); renderAll(); alert('Admin logged out'); }

// ---------- Admin Panel ----------
function showAdminLogin(){
  showModal(`<div style="font-weight:700">Admin Login</div><label>Email</label><input id="admE" /><label>Password</label><input id="admP" type="password" /><div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Close</button><button class="btn btn-primary" onclick="doAdminLogin()">Login</button></div>`);
}
function doAdminLogin(){
  const e = $('admE').value.trim(), p = $('admP').value;
  const admin = getStore('ak_admin');
  if(admin && admin.email === e && admin.password === p){ setStore('ak_current_admin', admin); closeModal(); renderAll(); alert('Admin logged in'); } else alert('Invalid admin credentials');
}

function showAdmin(){
  const admin = getStore('ak_current_admin');
  if(!admin) return showAdminLogin();
  const sellers = getStore('ak_sellers', []);
  const products = getStore('ak_products', []);
  const orders = getStore('ak_orders', []);
  const cfg = getStore('ak_config', {});
  let html = `<div style="font-weight:700">Admin Panel — ${esc(admin.name)}</div>`;
  html += `<h3 style="margin-top:10px">${t('siteSettings')}</h3>
    <label>Site Title</label><input id="adm_siteTitle" value="${esc(cfg.siteTitle||'')}" />
    <label>Logo Text</label><input id="adm_logoText" value="${esc(cfg.logoText||'')}" />
    <label>Hero Title</label><input id="adm_heroTitle" value="${esc(cfg.heroTitle||'')}" />
    <label>Hero Sub</label><input id="adm_heroSub" value="${esc(cfg.heroSub||'')}" />
    <label>Primary Color</label><input id="adm_primary" value="${esc(cfg.primary||'')}" />
    <label>Accent Color</label><input id="adm_accent" value="${esc(cfg.accent||'')}" />
    <label>Commission %</label><input id="adm_commission" value="${esc(cfg.commission||10)}" />
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Close</button><button class="btn btn-primary" onclick="saveAdminSettings()">Save</button></div>`;
  html += `<h3 style="margin-top:14px">Pending Sellers</h3>`;
  sellers.filter(s=>!s.approved).forEach(s=>{ html += `<div style="padding:8px;border-bottom:1px solid #f2f6fb"><b>${esc(s.shopName)}</b><div class="small muted">${esc(s.email)}</div><div style="margin-top:6px"><button class="btn btn-primary" onclick="approveSeller('${s.id}')">${t('approve')}</button> <button class="btn btn-ghost" onclick="rejectSeller('${s.id}')">${t('reject')}</button></div></div>`; });
  html += `<h3 style="margin-top:12px">Products</h3>`;
  products.forEach(p=>{ html += `<div style="padding:8px;border-bottom:1px solid #f2f6fb"><b>${esc(p.name)}</b> — ₹${p.price}<div class="small muted">${esc(p.sellerName)}</div><div style="margin-top:6px"><button class="btn btn-ghost" onclick="editProductAdmin('${p.id}')">${t('edit')}</button></div></div>`; });
  html += `<h3 style="margin-top:12px">Orders</h3>`;
  orders.forEach(o=>{ html += `<div style="padding:8px;border-bottom:1px solid #f2f6fb"><b>${esc(o.productName)}</b> — ${esc(o.id)} — ₹${o.price} — ${esc(o.status)}<div style="margin-top:6px">${o.deliveryVideoUrl?`<a href="${esc(o.deliveryVideoUrl)}" target="_blank">Video</a>`:`<button class="btn btn-ghost" onclick="attachVideoAdminPrompt('${o.id}')">Attach Video</button>`} <button class="btn btn-primary" onclick="confirmDeliveryAdmin('${o.id}')">Confirm & Payout</button></div></div>`; });
  showModal(html);
}

function saveAdminSettings(){
  const cfg = getStore('ak_config', {});
  cfg.siteTitle = $('adm_siteTitle').value; cfg.logoText = $('adm_logoText').value; cfg.heroTitle = $('adm_heroTitle').value;
  cfg.heroSub = $('adm_heroSub').value; cfg.primary = $('adm_primary').value; cfg.accent = $('adm_accent').value;
  cfg.commission = parseFloat($('adm_commission').value) || cfg.commission;
  setStore('ak_config', cfg);
  applyConfigUI(); renderAll(); closeModal(); alert('Saved');
}
function approveSeller(id){ const sellers = getStore('ak_sellers',[]); const s = sellers.find(x=>x.id===id); if(s){ s.approved = true; s.kycStatus='verified'; setStore('ak_sellers', sellers); alert('Approved'); renderAll(); showAdmin(); } }
function rejectSeller(id){ let sellers = getStore('ak_sellers',[]); sellers = sellers.filter(x=>x.id!==id); setStore('ak_sellers', sellers); alert('Rejected'); renderAll(); showAdmin(); }
function editProductAdmin(id){
  const p = (getStore('ak_products',[])).find(x=>x.id===id); if(!p) return alert('Not found');
  showModal(`<div style="font-weight:700">Edit Product</div><label>Name</label><input id="ep_name" value="${esc(p.name)}" /><label>Price</label><input id="ep_price" value="${esc(p.price)}" /><label>Category</label><input id="ep_cat" value="${esc(p.category)}" /><label>Image URL</label><input id="ep_img" value="${esc(p.img)}" /><div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Cancel</button><button class="btn btn-primary" onclick="saveEditProduct('${p.id}')">Save</button></div>`);
}
function saveEditProduct(id){ const products = getStore('ak_products',[]); const p = products.find(x=>x.id===id); p.name = $('ep_name').value; p.price = parseFloat($('ep_price').value)||p.price; p.category = $('ep_cat').value; p.img = $('ep_img').value; setStore('ak_products', products); closeModal(); alert('Saved'); renderAll(); }
function attachVideoAdminPrompt(orderId){ const url = prompt('Paste YouTube URL'); if(!url) return; const orders = getStore('ak_orders',[]); const o = orders.find(x=>x.id===orderId); if(!o) return alert('Not found'); o.deliveryVideoUrl = url; o.status = 'delivered'; setStore('ak_orders', orders); alert('Video attached'); renderAll(); showAdmin(); }
function confirmDeliveryAdmin(orderId){ const orders = getStore('ak_orders',[]); const o = orders.find(x=>x.id===orderId); if(!o) return alert('Not found'); o.status = 'delivered'; o.paid = true; setStore('ak_orders', orders); alert('Delivery confirmed & payout simulated'); renderAll(); showAdmin(); }

// ---------- Seller Dashboard ----------
function openSellerDashboard(){
  const cur = getStore('ak_current_user'); if(!cur || cur.role!=='seller') return alert('Login as seller');
  const sellers = getStore('ak_sellers',[]); const seller = sellers.find(s=>s.email===cur.email);
  if(!seller) return alert('Seller profile not found');
  const products = getStore('ak_products',[]).filter(p=>p.sellerId===seller.id);
  let html = `<div style="font-weight:700">Seller Dashboard — ${esc(seller.shopName)}</div>`;
  html += `<h4 style="margin-top:10px">${t('myProducts')}</h4><div style="margin-top:6px"><button class="btn btn-primary" onclick="showAddProductForm('${seller.id}')">${t('addProduct')}</button></div>`;
  products.forEach(p=>{ html += `<div style="padding:8px;border-bottom:1px solid #f2f6fb"><b>${esc(p.name)}</b> — ₹${p.price}<div class="small muted">${esc(p.category)}</div><div style="margin-top:6px"><button class="btn btn-ghost" onclick="editProductSeller('${p.id}','${seller.id}')">${t('edit')}</button> <button class="btn btn-ghost" onclick="deleteProductSeller('${p.id}','${seller.id}')">Delete</button></div></div>`; });
  showModal(html);
}

function showAddProductForm(sellerId){
  showModal(`<div style="font-weight:700">Add Product</div>
    <label>Name</label><input id="np_name" />
    <label>Price (INR)</label><input id="np_price" />
    <label>Category</label><input id="np_cat" />
    <label>Image file (optional)</label><input id="np_imgfile" type="file" accept="image/*" />
    <label>Or Image URL</label><input id="np_imgurl" />
    <label>Publish?</label><select id="np_pub"><option value="1">Yes</option><option value="0">No</option></select>
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Cancel</button><button class="btn btn-primary" onclick="doAddProduct('${sellerId}')">Add</button></div>`);
}

function doAddProduct(sellerId){
  const name = $('np_name').value.trim(); const price = parseFloat($('np_price').value)||0; const category = $('np_cat').value.trim()||'General'; const pub = $('np_pub').value==='1';
  if(!name || !price) return alert('Complete details');
  const fileInput = $('np_imgfile'); const urlInput = $('np_imgurl').value.trim();
  if(fileInput.files && fileInput.files[0]){
    const f = fileInput.files[0]; const reader = new FileReader();
    reader.onload = function(e){ const data = e.target.result; saveNewProduct(sellerId, name, price, category, data, pub); };
    reader.readAsDataURL(f);
  } else {
    saveNewProduct(sellerId, name, price, category, urlInput||'', pub);
  }
  closeModal();
}
function saveNewProduct(sellerId, name, price, category, img, pub){
  const products = getStore('ak_products', []);
  const id = 'p' + (products.length + 1000);
  const sellers = getStore('ak_sellers',[]); const s = sellers.find(x=>x.id===sellerId);
  products.push({id, sellerId, sellerEmail: s.email, sellerName: s.shopName, name, price, category, img, published: pub});
  setStore('ak_products', products); alert('Product added'); renderAll();
}

function editProductSeller(pid, sellerId){
  const products = getStore('ak_products', []); const p = products.find(x=>x.id===pid); if(!p) return alert('Not found');
  showModal(`<div style="font-weight:700">Edit Product</div><label>Name</label><input id="sp_name" value="${esc(p.name)}" /><label>Price</label><input id="sp_price" value="${esc(p.price)}" /><label>Category</label><input id="sp_cat" value="${esc(p.category)}" /><label>Image URL / Data</label><input id="sp_img" value="${esc(p.img)}" /><label>Published</label><select id="sp_pub"><option value="1" ${p.published? 'selected':''}>Yes</option><option value="0" ${!p.published? 'selected':''}>No</option></select><div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Cancel</button><button class="btn btn-primary" onclick="saveProductSeller('${pid}')">Save</button></div>`);
}
function saveProductSeller(pid){
  const products = getStore('ak_products', []); const p = products.find(x=>x.id===pid);
  p.name = $('sp_name').value; p.price = parseFloat($('sp_price').value)||p.price; p.category = $('sp_cat').value; p.img = $('sp_img').value; p.published = $('sp_pub').value==='1';
  setStore('ak_products', products); closeModal(); alert('Saved'); renderAll();
}
function deleteProductSeller(pid, sellerId){
  if(!confirm('Delete this product?')) return;
  let products = getStore('ak_products', []); products = products.filter(x=>x.id!==pid); setStore('ak_products', products); alert('Deleted'); renderAll();
}

// ---------- Product view & buy ----------
function openProduct(pid){
  const p = (getStore('ak_products', [])).find(x=>x.id===pid); if(!p) return alert('Not found');
  const cfg = getStore('ak_config', {}); const currency = $('currency').value; const rate = (cfg.currencies && cfg.currencies[currency] && cfg.currencies[currency].rate)||1; const sym = (cfg.currencies && cfg.currencies[currency] && cfg.currencies[currency].symbol)||'';
  const commission = Math.round((p.price * cfg.commission)/100); const net = p.price - commission;
  const priceShow = `${sym} ${Math.round(p.price * rate)}`;
  const html = `<div style="display:flex;gap:12px"><div style="flex:1"><img src="${esc(p.img)}" style="width:100%;height:360px;object-fit:cover;border-radius:8px"></div><div style="width:420px"><h3>${esc(p.name)}</h3><div class="muted small">${esc(p.category)} • ${esc(p.sellerName)}</div><div style="font-weight:800;margin-top:8px">${priceShow}</div><div class="muted small">Commission ${cfg.commission}% → ${sym} ${Math.round(commission*rate)} • Net ${sym} ${Math.round(net*rate)}</div><label>${t('quantity')}</label><input id="buyQty" value="1" /><label>UPI ID for payment (eg: yourname@upi)</label><input id="buyUpi" placeholder="payer@upi" /><div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Cancel</button><button class="btn btn-primary" onclick="doBuy('${p.id}')">${t('pay')}</button></div></div></div>`;
  showModal(html);
}

function doBuy(pid){
  const qty = parseInt($('buyQty').value) || 1; const upi = $('buyUpi').value.trim();
  if(!upi) return alert('Enter UPI ID to simulate payment');
  const cur = getStore('ak_current_user'); if(!cur) return alert('Login to buy');
  const products = getStore('ak_products', []); const p = products.find(x=>x.id===pid);
  const cfg = getStore('ak_config', {}); const commission = Math.round((p.price * cfg.commission)/100); const net = p.price - commission;
  const order = { id: 'o'+Date.now(), productId: pid, productName: p.name, price: p.price*qty, qty, buyer: cur.email, sellerId: p.sellerId, commission, net, status: 'paid_via_upi', deliveryVideoUrl:'', paid: true, createdAt: new Date().toISOString(), upi };
  const orders = getStore('ak_orders', []); orders.push(order); setStore('ak_orders', orders);
  closeModal(); renderAll(); alert('Payment simulated successful. Order ID: ' + order.id); openOrderDetail(order.id);
}

function openOrderDetail(id){
  const o = (getStore('ak_orders', [])).find(x=>x.id===id); if(!o) return alert('Not found');
  const html = `<div style="font-weight:700">Order ${esc(o.id)}</div><div class="muted small">${esc(o.productName)} • Qty ${o.qty}</div><div style="margin-top:8px">Status: <b>${esc(o.status)}</b></div><div style="margin-top:8px">Delivery Video: ${o.deliveryVideoUrl?`<a href="${esc(o.deliveryVideoUrl)}" target="_blank">View</a>`:`<button class="btn btn-ghost" onclick="attachVideoPrompt('${o.id}')">${t('attachVideo')}</button>`}</div>`;
  showModal(html);
}

function attachVideoPrompt(orderId){
  const url = prompt('Paste YouTube URL (public)'); if(!url) return; const orders = getStore('ak_orders', []); const o = orders.find(x=>x.id===orderId); if(!o) return alert('Not found'); o.deliveryVideoUrl = url; setStore('ak_orders', orders); alert('Video saved'); renderAll();
}

// ---------- Settings (Public accessible but meant for admin) ----------
function showSettings(){
  const cfg = getStore('ak_config', {});
  showModal(`<div style="font-weight:700">${t('settings')}</div>
    <label>Site Title</label><input id="cfg_siteTitle" value="${esc(cfg.siteTitle||'')}" />
    <label>Logo Text</label><input id="cfg_logoText" value="${esc(cfg.logoText||'')}" />
    <label>Hero Title</label><input id="cfg_heroTitle" value="${esc(cfg.heroTitle||'')}" />
    <label>Hero Sub</label><input id="cfg_heroSub" value="${esc(cfg.heroSub||'')}" />
    <label>Primary Color</label><input id="cfg_primary" value="${esc(cfg.primary||'')}" />
    <label>Accent Color</label><input id="cfg_accent" value="${esc(cfg.accent||'')}" />
    <label>Commission %</label><input id="cfg_commission" value="${esc(cfg.commission||10)}" />
    <label>Live Video URLs (comma separated)</label><textarea id="cfg_live">${(cfg.liveVideos||[]).join(',')}</textarea>
    <div style="margin-top:8px;display:flex;gap:8px;justify-content:flex-end"><button class="btn btn-ghost" onclick="closeModal()">Close</button><button class="btn btn-primary" onclick="saveSettingsPublic()">Save</button></div>`);
}
function saveSettingsPublic(){
  const cfg = getStore('ak_config', {});
  cfg.siteTitle = $('cfg_siteTitle').value; cfg.logoText = $('cfg_logoText').value; cfg.heroTitle = $('cfg_heroTitle').value; cfg.heroSub = $('cfg_heroSub').value;
  cfg.primary = $('cfg_primary').value; cfg.accent = $('cfg_accent').value; cfg.commission = parseFloat($('cfg_commission').value)||cfg.commission;
  const live = ($('cfg_live').value||'').split(',').map(s=>s.trim()).filter(Boolean); cfg.liveVideos = live;
  setStore('ak_config', cfg); applyConfigUI(); renderAll(); closeModal(); alert('Settings saved (local demo)');
}

// ---------- Admin actions for orders etc ----------
function confirmDeliveryAdmin(orderId){ const orders = getStore('ak_orders', []); const o = orders.find(x=>x.id===orderId); if(!o) return alert('Not found'); o.status = 'delivered'; o.paid = true; setStore('ak_orders', orders); alert('Confirmed & payout simulated'); renderAll(); }
function attachVideoToOrder(orderId, url){ const orders = getStore('ak_orders', []); const o = orders.find(x=>x.id===orderId); if(!o) return; o.deliveryVideoUrl = url; setStore('ak_orders', orders); renderAll(); }

// ---------- Currency / Language ----------
function onCurrencyChange(){ renderProducts($('search').value.trim()); }
function setLangFromStorage(){ const l = localStorage.getItem('ak_lang')||'en'; $('lang').value = l; setLang(l); }
setLangFromStorage();

// ---------- Render everything ----------
function renderAll(){
  applyConfigUI(); renderNav(); renderProducts($('search').value.trim()); renderLiveVideos(); renderOrdersForUser();
  const admin = getStore('ak_current_admin'); if(admin) { /* admin logged */ }
}
renderAll();

// ---------- small helpers and expose for debugging ----------
window.showLogin = showLogin; window.showRegister = showRegister; window.showSellerOnboard = showSellerOnboard;
window.showAdmin = showAdmin; window.showSettings = showSettings; window.openProduct = openProduct;
window.openSellerDashboard = openSellerDashboard; window.onSearch = onSearch; window.onCurrencyChange = onCurrencyChange;
window.attachVideoPrompt = attachVideoPrompt; window.openOrderDetail = openOrderDetail; window.setLang = setLang;

// initialize UI values
document.addEventListener('DOMContentLoaded', ()=>{
  $('year').innerText = new Date().getFullYear();
  // ensure currency select default
  const cfg = getStore('ak_config'); if(cfg && cfg.currencies){ $('currency').value = Object.keys(cfg.currencies)[0]; }
  // set lang
  const lang = localStorage.getItem('ak_lang') || 'en'; $('lang').value = lang; setLang(lang);
  renderAll();
});
</script>
</body>
</html>
