(function () {
  'use strict';
  var $ = function (s, r) { return (r || document).querySelector(s); };
  var $$ = function (s, r) { return Array.prototype.slice.call((r || document).querySelectorAll(s)); };
  var SITE = document.body.getAttribute('data-site-url') || location.origin;
  var SITE_TITLE = document.body.getAttribute('data-site-title') || document.title;
  var AR = '٠١٢٣٤٥٦٧٨٩';
  var arNum = function (n) { return String(n).replace(/\d/g, function (d) { return AR[d]; }); };

  function store(k, v) {
    try { if (v === undefined) return localStorage.getItem(k); localStorage.setItem(k, v); } catch (e) { return null; }
  }
  function toast(msg) {
    var t = document.createElement('div'); t.className = 'toast'; t.textContent = msg; t.setAttribute('role', 'status');
    document.body.appendChild(t); setTimeout(function () { t.remove(); }, 1800);
  }
  function copy(text) {
    if (navigator.clipboard && window.isSecureContext) {
      return navigator.clipboard.writeText(text).then(function () { toast('تم النسخ'); }, fallback);
    }
    fallback();
    function fallback() {
      var ta = document.createElement('textarea'); ta.value = text; ta.style.position = 'fixed'; ta.style.opacity = '0';
      document.body.appendChild(ta); ta.select();
      try { document.execCommand('copy'); toast('تم النسخ'); } catch (e) { toast('تعذّر النسخ'); }
      ta.remove();
    }
  }

  // القائمة
  var mb = $('[data-menu]'), nav = $('[data-nav]');
  if (mb && nav) mb.addEventListener('click', function () {
    var o = nav.classList.toggle('open'); mb.setAttribute('aria-expanded', o ? 'true' : 'false');
  });

  // المشاركة
  var modal = $('[data-modal]');
  function openShare(opts) {
    var url = opts.url, text = opts.text;
    $('[data-modal-title]').textContent = opts.heading;
    $('[data-modal-sub]').textContent = opts.sub || '';
    $('[data-share-url]').value = url;
    var enc = encodeURIComponent;
    var full = text ? text + '\n' + url : url;
    var grid = $('[data-share-grid]'); grid.innerHTML = '';
    function link(label, href) { var a = document.createElement('a'); a.textContent = label; a.href = href; a.target = '_blank'; a.rel = 'noopener'; grid.appendChild(a); }
    function btn(label, fn) { var b = document.createElement('button'); b.textContent = label; b.addEventListener('click', fn); grid.appendChild(b); }
    if (navigator.share) btn('مشاركة عبر التطبيقات', function () {
      navigator.share({ title: opts.title || SITE_TITLE, text: text || SITE_TITLE, url: url }).catch(function () {});
    });
    link('فيسبوك', 'https://www.facebook.com/sharer/sharer.php?u=' + enc(url) + (text ? '&quote=' + enc(text) : ''));
    link('واتساب', 'https://wa.me/?text=' + enc(full));
    link('تيليجرام', 'https://t.me/share/url?url=' + enc(url) + '&text=' + enc(text || SITE_TITLE));
    link('إكس (تويتر)', 'https://twitter.com/intent/tweet?url=' + enc(url) + '&text=' + enc(text || SITE_TITLE));
    btn('نسخ النص مع الرابط', function () { copy(full); });
    $('[data-stories-tip]').textContent = 'للقصص (فيسبوك / واتساب / إنستجرام): انسخ الرابط من الأسفل ثم الصقه في ملصق «رابط» داخل القصة، أو اختر «مشاركة عبر التطبيقات» إن ظهر لك. لا يتيح إنستجرام نشر رابط من موقع ويب مباشرةً.';
    modal.hidden = false;
  }
  function closeShare() { modal.hidden = true; }
  if (modal) {
    modal.addEventListener('click', function (e) { if (e.target === modal || e.target.hasAttribute('data-close')) closeShare(); });
    document.addEventListener('keydown', function (e) { if (e.key === 'Escape') closeShare(); });
    $('[data-copy-link]').addEventListener('click', function () { copy($('[data-share-url]').value); });
  }
  document.addEventListener('click', function (e) {
    var t = e.target.closest('button,a'); if (!t) return;
    if (t.hasAttribute('data-share-site')) {
      openShare({ heading: 'انشر الخير وشارك الموقع', sub: 'شارك الموقع ليكون صدقة جارية عن الحاج جمال أبوحطب وجميع موتى المسلمين.', url: SITE + '/', text: SITE_TITLE });
    } else if (t.hasAttribute('data-share-item')) {
      var page = location.pathname.split('/').pop() || '';
      openShare({ heading: 'مشاركة: ' + t.getAttribute('data-title'), url: SITE + '/' + page + '#' + t.getAttribute('data-anchor'), text: t.getAttribute('data-text'), title: t.getAttribute('data-title') });
    } else if (t.hasAttribute('data-copy')) {
      copy(t.getAttribute('data-copy') + '\n' + SITE + '/');
    }
  });

  // العدّادات
  $$('[data-counter]').forEach(function (c) {
    var id = 'cnt:' + c.getAttribute('data-id'), target = parseInt(c.getAttribute('data-target'), 10) || 0;
    var n = parseInt(store(id), 10) || 0, out = $('[data-n]', c);
    function draw() { out.textContent = arNum(n); c.classList.toggle('done', target > 0 && n >= target); }
    draw();
    $('[data-tap]', c).addEventListener('click', function () {
      n += 1; store(id, String(n)); draw();
      if (target && n === target) { toast('أتممتَ العدد، تقبّل الله منك'); if (navigator.vibrate) navigator.vibrate(60); }
    });
    $('[data-reset]', c).addEventListener('click', function () { n = 0; store(id, '0'); draw(); });
  });

  // ───── جلب القرآن (مباشر من Quran.com في وضع الرفع من الموبايل، أو من الملفات المبنية) ─────
  var LIVE = document.body.getAttribute('data-live') === '1';
  var API = document.body.getAttribute('data-quran-api') || 'https://api.quran.com/api/v4';
  function fetchJson(u) { return fetch(u).then(function (r) { if (!r.ok) throw new Error(r.status); return r.json(); }); }
  var idxP = null;
  function loadIndex() {
    if (!idxP) {
      idxP = (LIVE ? fetchJson(API + '/chapters?language=ar').then(function (d) {
        if (!d.chapters || d.chapters.length !== 114) throw new Error('chapters');
        return { chapters: d.chapters.map(function (c) { return { id: c.id, name: c.name_arabic, count: c.verses_count, bismillahPre: !!c.bismillah_pre }; }) };
      }) : fetchJson('data/quran/index.json'));
      idxP.catch(function () { idxP = null; });
    }
    return idxP;
  }
  // آيات من a إلى b مع تحقق صارم من العدد والتسلسل
  function liveVerses(n, a, b) {
    var reqs = [];
    for (var p = Math.ceil(a / 50); p <= Math.ceil(b / 50); p++) reqs.push(fetchJson(API + '/verses/by_chapter/' + n + '?fields=text_uthmani&per_page=50&page=' + p));
    return Promise.all(reqs).then(function (rs) {
      var all = [];
      rs.forEach(function (r) { r.verses.forEach(function (v) { all.push({ n: v.verse_number, t: v.text_uthmani }); }); });
      all = all.filter(function (v) { return v.n >= a && v.n <= b; });
      if (all.length !== b - a + 1) throw new Error('count');
      all.forEach(function (v, i) { if (v.n !== a + i || !v.t) throw new Error('seq'); });
      return all;
    });
  }
  var surahCache = {};
  function loadSurah(n) {
    if (!surahCache[n]) {
      surahCache[n] = loadIndex().then(function (idx) {
        if (!LIVE) return fetchJson('data/quran/' + n + '.json');
        var ch = idx.chapters[n - 1];
        return liveVerses(n, 1, ch.count).then(function (vs) { return { id: n, name: ch.name, bismillahPre: ch.bismillahPre, verses: vs }; });
      });
      surahCache[n].catch(function () { delete surahCache[n]; });
    }
    return surahCache[n];
  }

  // آيات الأدعية والأذكار في الوضع المباشر
  $$('[data-qref]').forEach(function (box) {
    var refs = box.getAttribute('data-qref').split(',');
    loadIndex().then(function (idx) {
      return Promise.all(refs.map(function (ref) {
        var m = /^(\d+):(\d+)(?:-(\d+))?$/.exec(ref), s = +m[1], a = +m[2], b = +(m[3] || m[2]), ch = idx.chapters[s - 1];
        if (b > ch.count) throw new Error('range');
        return liveVerses(s, a, b).then(function (vs) {
          var cite = a === b ? 'سورة ' + ch.name + '، الآية ' + arNum(a) : 'سورة ' + ch.name + '، الآيات ' + arNum(a) + '–' + arNum(b);
          return { text: vs.map(function (v) { return v.t + ' \uFD3F' + arNum(v.n) + '\uFD3E'; }).join(' '), cite: cite, plain: '﴿' + vs.map(function (v) { return v.t; }).join(' ') + '﴾ [' + cite + ']' };
        });
      }));
    }).then(function (rs) {
      box.textContent = '';
      rs.forEach(function (r) {
        var p1 = document.createElement('p'); p1.className = 'ayah'; p1.textContent = r.text;
        var p2 = document.createElement('p'); p2.className = 'cite'; p2.textContent = r.cite;
        box.appendChild(p1); box.appendChild(p2);
      });
      var plain = rs.map(function (r) { return r.plain; }).join('\n');
      var card = box.closest('.card');
      if (card) ['data-copy', 'data-text'].forEach(function (attr) {
        var el = card.querySelector('[' + attr + ']');
        if (el) { var lines = (el.getAttribute(attr) || '').split('\n'); lines.splice(1, 0, plain); el.setAttribute(attr, lines.join('\n')); }
      });
    }).catch(function () {
      box.innerHTML = '<p class="note">تعذّر تحميل الآيات الآن. تحقق من الاتصال وأعد فتح الصفحة، أو افتحها على <a href="https://quran.com" rel="noopener">quran.com</a>.</p>';
    });
  });

  // قارئ القرآن
  var reader = $('[data-reader]');
  if (reader) {
    var sel = $('[data-surah-select]'), mush = $('[data-mushaf]'), index = null, cur = 1;
    var size = parseFloat(store('q-size')) || 1.9;
    function applySize() { mush.style.setProperty('--q-size', size + 'rem'); }
    applySize();
    function show(n, ayah) {
      cur = n; sel.value = String(n); store('q-last', String(n));
      mush.innerHTML = '<p class="muted">جارٍ تحميل السورة…</p>';
      var needBasmala = n !== 1 && index.chapters[n - 1].bismillahPre;
      Promise.all([loadSurah(n), needBasmala ? loadSurah(1) : Promise.resolve(null)]).then(function (r) {
        var s = r[0], html = '<h2>سورة ' + s.name + '</h2>';
        if (needBasmala) html += '<p class="basmala">' + r[1].verses[0].t + '</p>';
        html += '<p>' + s.verses.map(function (v) {
          return '<span class="v" id="v' + v.n + '" data-n="' + v.n + '">' + v.t + ' <span class="vn">﴿' + arNum(v.n) + '﴾</span></span>';
        }).join(' ') + '</p>';
        mush.innerHTML = html;
        history.replaceState(null, '', '#' + n + (ayah ? ':' + ayah : ''));
        if (ayah) { var el = $('#v' + ayah, mush); if (el) { el.classList.add('hl'); el.scrollIntoView({ block: 'center' }); } }
        else window.scrollTo({ top: 0 });
      }).catch(function () {
        mush.innerHTML = '<p>تعذّر تحميل السورة. تحقق من الاتصال ثم أعد المحاولة، أو افتحها على <a href="https://quran.com/' + n + '" rel="noopener">quran.com</a>.</p>';
      });
    }
    loadIndex().then(function (d) {
      index = d;
      sel.innerHTML = d.chapters.map(function (c) { return '<option value="' + c.id + '">' + arNum(c.id) + '. ' + c.name + ' (' + arNum(c.count) + ' آية)</option>'; }).join('');
      var m = /^#(\d{1,3})(?::(\d{1,3}))?$/.exec(location.hash);
      var start = m ? parseInt(m[1], 10) : parseInt(store('q-last'), 10) || 1;
      if (start < 1 || start > 114) start = 1;
      show(start, m && m[2] ? parseInt(m[2], 10) : null);
    }).catch(function () { mush.innerHTML = '<p>تعذّر تحميل فهرس السور.</p>'; });
    sel.addEventListener('change', function () { show(parseInt(sel.value, 10)); });
    $('[data-prev]').addEventListener('click', function () { if (cur > 1) show(cur - 1); });
    $('[data-next]').addEventListener('click', function () { if (cur < 114) show(cur + 1); });
    $('[data-font-plus]').addEventListener('click', function () { size = Math.min(3.2, size + 0.2); store('q-size', String(size)); applySize(); });
    $('[data-font-minus]').addEventListener('click', function () { size = Math.max(1.3, size - 0.2); store('q-size', String(size)); applySize(); });
    mush.addEventListener('click', function (e) {
      var v = e.target.closest('.v'); if (!v) return;
      $$('.hl', mush).forEach(function (x) { x.classList.remove('hl'); });
      v.classList.add('hl');
      var n = v.getAttribute('data-n');
      openShare({ heading: 'مشاركة الآية ' + arNum(n), url: SITE + '/quran.html#' + cur + ':' + n, text: '﴿' + v.firstChild.textContent.trim() + '﴾ [سورة ' + index.chapters[cur - 1].name + '، الآية ' + arNum(n) + ']' });
    });
  }
})();
