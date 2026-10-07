/*!
 * Motor de Dashboards · versão do CLIENTE
 * Relatório de uma página só, para o dono do negócio: camada de negócios,
 * evolução por mês, funil, campanhas e próximos passos. Lê o mesmo
 * window.DASH e as mesmas planilhas do motor interno (Motor Github.js).
 */
(function () {
'use strict';

var VERSION = '1.0.0';
var D = window.DASH || {};
var ROOT = document.getElementById(D.elemento || 'dash');
if (!ROOT) return;

/* ============================== ARMAZENAMENTO ============================== */
var CLIENT_KEY = String(D.cliente || 'cliente').normalize('NFD').replace(/[\u0300-\u036f]/g, '').replace(/\W+/g, '_').toLowerCase();
var store = {
  k: function (n) { return 'cli_' + CLIENT_KEY + '_' + n; },
  get: function (n, def) { try { var v = localStorage.getItem(this.k(n)); return v == null ? def : JSON.parse(v); } catch (e) { return def; } },
  set: function (n, v) { try { localStorage.setItem(this.k(n), JSON.stringify(v)); } catch (e) {} },
  del: function (n) { try { localStorage.removeItem(this.k(n)); } catch (e) {} }
};

/* ============================== IDIOMA E FORMATO ============================== */
var LANGS = (D.idiomas && D.idiomas.length) ? D.idiomas : ['pt'];
var LANG = store.get('lang', LANGS[0]);
if (LANGS.indexOf(LANG) < 0) LANG = LANGS[0];
function L(pt, es) { return (LANG === 'es' && es != null) ? es : pt; }
function LOC() { return LANG === 'es' ? 'es-AR' : 'pt-BR'; }
var CUR = D.moeda || 'BRL';

function ok(v) { return v != null && isFinite(v); }
function nf(v, d) { d = d || 0; return new Intl.NumberFormat(LOC(), { minimumFractionDigits: d, maximumFractionDigits: d }).format(v || 0); }
function money(v) {
  if (!ok(v)) return '—';
  var a = Math.abs(v), d = a < 100 ? 2 : 0;
  try { return new Intl.NumberFormat(LOC(), { style: 'currency', currency: CUR, minimumFractionDigits: d, maximumFractionDigits: d }).format(v); }
  catch (e) { return CUR + ' ' + nf(v, d); }
}
function moneyShort(v) {
  if (!ok(v)) return '—';
  var a = Math.abs(v);
  if (a >= 1e6) return money(v / 1e6).replace(/[,.]00(?=\D*$)/, '') + 'M';
  if (a >= 1e4) return money(Math.round(v / 1e3)).replace(/[,.]00(?=\D*$)/, '') + 'k';
  return money(v);
}
function count(v) { if (!ok(v)) return '—'; return nf(v, (Math.abs(v - Math.round(v)) > 0.05 && Math.abs(v) < 100) ? 1 : 0); }
function pctf(v, d) { if (!ok(v)) return '—'; if (d == null) d = Math.abs(v) < 0.1 ? 2 : 1; return nf(v * 100, d) + '%'; }
function xf(v) { return ok(v) ? nf(v, 2) + 'x' : '—'; }
function deltaTxt(c, p) {
  if (!ok(c) || !ok(p) || p === 0) return null;
  var d = (c - p) / Math.abs(p);
  return { v: d, txt: (d > 0 ? '+' : '') + nf(d * 100, 0) + '%' };
}
function esc(s) { return String(s == null ? '' : s).replace(/[&<>"']/g, function (c) { return { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]; }); }
function norm(s) { return String(s == null ? '' : s).normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/\s+/g, ' ').trim(); }
function $(s, c) { return (c || ROOT).querySelector(s); }
function $$(s, c) { return Array.prototype.slice.call((c || ROOT).querySelectorAll(s)); }

/* ============================== DATAS (sempre ISO AAAA-MM-DD) ============================== */
function p2(n) { return String(n).padStart(2, '0'); }
function isoOf(d) { return d.getFullYear() + '-' + p2(d.getMonth() + 1) + '-' + p2(d.getDate()); }
function todayISO() { return isoOf(new Date()); }
function addDays(iso, n) { var a = iso.split('-').map(Number); return isoOf(new Date(a[0], a[1] - 1, a[2] + n)); }
function daysBetween(a, b) { return Math.round((new Date(b + 'T00:00:00') - new Date(a + 'T00:00:00')) / 864e5); }
function monthStart(iso) { return iso.slice(0, 8) + '01'; }
function monthEnd(iso) { var y = +iso.slice(0, 4), m = +iso.slice(5, 7); return iso.slice(0, 8) + p2(new Date(y, m, 0).getDate()); }
function prevMonthStart(iso) { var y = +iso.slice(0, 4), m = +iso.slice(5, 7); return isoOf(new Date(y, m - 2, 1)); }
function fmtD(iso, year) { if (!iso) return '—'; var s = iso.slice(8, 10) + '/' + iso.slice(5, 7); return year ? s + '/' + iso.slice(0, 4) : s; }
function parseDate(v) {
  if (v == null) return null;
  var s = String(v).trim(), m;
  if (!s) return null;
  if ((m = s.match(/^(\d{4})-(\d{1,2})-(\d{1,2})/))) return m[1] + '-' + p2(m[2]) + '-' + p2(m[3]);
  if ((m = s.match(/^(\d{1,2})[\/\-.](\d{1,2})[\/\-.](\d{2,4})/))) { var y = +m[3]; if (y < 100) y += 2000; return y + '-' + p2(m[2]) + '-' + p2(m[1]); }
  var d = new Date(s);
  return isNaN(d) ? null : isoOf(d);
}


/* ============================== CRM: VALORES VAZIOS ============================== */
function valorVazio(v) {
  var t = String(v == null ? '' : v).trim();
  if (!t) return true;
  if (/^\{\{.*\}\}$/.test(t) || /^\$\{.*\}$/.test(t)) return true;   // template não substituído
  var n = norm(t).replace(/[()\[\]]/g, '').trim();
  return ['unknown', 'undefined', 'null', 'none', 'not set', 'no set', 'nao definido', 'não definido', 'sin definir',
          'other', 'others', 'outro', 'outros', 'n/a', 'na', '-', '--', '0', 'desconhecido', 'desconocido', 'sem informacao'].indexOf(n) > -1;
}

/* ============================== CSV ============================== */
function detectDelim(text) {
  var lines = text.split(/\r?\n/).slice(0, 8), best = ',', bestScore = -1;
  [',', ';', '\t'].forEach(function (d) {
    var counts = lines.map(function (l) { return l.split(d).length - 1; }).filter(function (n) { return n > 0; });
    var score = counts.length ? counts.reduce(function (a, b) { return a + b; }, 0) / counts.length : 0;
    if (score > bestScore) { bestScore = score; best = d; }
  });
  return best;
}
function csvRows(text, delim) {
  var rows = [], row = [], cell = '', q = false;
  for (var i = 0; i < text.length; i++) {
    var c = text[i], n = text[i + 1];
    if (q) { if (c === '"' && n === '"') { cell += '"'; i++; } else if (c === '"') q = false; else cell += c; }
    else if (c === '"') q = true;
    else if (c === delim) { row.push(cell); cell = ''; }
    else if (c === '\n' || c === '\r') { if (c === '\r' && n === '\n') i++; row.push(cell); rows.push(row); row = []; cell = ''; }
    else cell += c;
  }
  row.push(cell); if (row.length > 1 || row[0] !== '') rows.push(row);
  return rows.filter(function (r) { return r.some(function (x) { return String(x).trim() !== ''; }); });
}

/* ============================== MAPA DE COLUNAS ============================== */
var NOT_COST = /custo|cost|costo|\bcpc\b|\bcpm\b|\bctr\b|taxa|tasa|\brate\b|\/ /;
var RULES = {
  date: { exact: ['dia', 'data', 'fecha', 'day', 'date', 'data real', 'created time', 'created_time', 'data de cadastro', 'data de inscricao', 'timestamp', 'horario de envio', 'submitted at'], re: [/^(dia|data|fecha|date|day)\b/, /created|timestamp|cadastro|inscri|envio/], not: [/atualiz|nascim/] },
  campaign: { exact: ['campanha', 'nome da campanha', 'campana', 'nombre de la campana', 'campaign', 'campaign name'], re: [/campanha|campana|campaign/], not: [/^id|\bid\b|tipo|objetiv|objective|status|estado|orcamento|budget/] },
  account: { exact: ['conta', 'nome da conta', 'nome da conta de anuncios', 'account', 'account name', 'cuenta', 'nombre de la cuenta'], re: [/nome da conta|account name|nombre de la cuenta/], not: [/\bid\b/] },
  objective: { exact: ['objetivo', 'objective', 'objetivo da campanha', 'objetivo de la campana'], re: [/^objetivo/, /objective/] },
  adset: { exact: ['nome do conjunto de anuncios', 'conjunto de anuncios', 'nombre del conjunto de anuncios', 'ad set name', 'grupo de anuncios'], re: [/conjunto de anuncio|ad set name|grupo de anuncio/], not: [/\bid\b/] },
  ad: { exact: ['nome do anuncio', 'anuncio', 'nombre del anuncio', 'ad name'], re: [/nome do anuncio|nombre del anuncio|^ad name$/], not: [/\bid\b/] },
  spend: { exact: ['valor gasto', 'valor gasto (brl)', 'valor gasto (ars)', 'valor usado', 'valor usado (brl)', 'importe gastado', 'importe gastado (ars)', 'amount spent', 'custo', 'cost', 'investimento', 'inversion'], re: [/^valor (gasto|usado)/, /^importe gastado/, /^custo$/, /^cost$/, /amount spent/, /^investimento$/], not: [/por |\/|cpc|cpm|conv|lead|resultado/] },
  impressions: { exact: ['impr', 'impr.', 'impressoes', 'impresiones', 'impressions'], re: [/^impr/, /impression/], not: [/%|ctr|custo|cost|cpm|taxa|parcela|share/] },
  clicks: { exact: ['cliques no link', 'clics en el enlace', 'link clicks', 'cliques', 'clics', 'clicks'], re: [/cliques no link|clics en el enlace|link clicks/, /^cliques/, /^clics/, /^clicks/], not: [/unicos|unique|ctr|custo|costo|cost|cpc|taxa|saida|outbound/] },
  reach: { exact: ['alcance', 'reach'], re: [/^alcance/, /^reach/], not: [/custo|cost|costo|por /] },
  frequency: { exact: ['frequencia', 'frecuencia', 'frequency'], re: [/^frequen|^frecuen/] },
  lpv: { exact: ['visualizacoes da pagina de destino', 'visualizaciones de la pagina de destino', 'landing page views'], re: [/pagina de destino|landing page view/], not: [NOT_COST] },
  conversations: { exact: ['conversas por mensagem iniciadas', 'conversas por mensagens iniciadas', 'conversas iniciadas por mensagem', 'conversas iniciadas por mensagens', 'conversas iniciadas', 'conversas', 'messaging conversations started', 'messaging conversation started', 'conversations started', 'conversaciones con mensajes iniciadas', 'conversaciones iniciadas'], re: [/\bconversas?\b|conversaciones|mensag|messaging conv/], not: [NOT_COST, /valor|value|compra|conversao|conversion/] },
  leads: { exact: ['leads', 'lead', 'cadastros', 'leads de formulario', 'leads no formulario', 'form leads', 'clientes potenciales', 'registros'], re: [/^leads?\b/, /cadastro/, /clientes? potencial/], not: [NOT_COST, /qualific|conversa|mensag/] },
  purchases: { exact: ['compras', 'purchases', 'conversoes', 'conversiones', 'conversions', 'conv.', 'action omni purchase', 'omni purchase', 'website purchases', 'compras no site'], re: [/^compras$/, /^purchases$/, /^conversoes$/, /^conversiones$/, /^conversions$/, /^conv\.?$/, /omni purchase$/], not: [NOT_COST, /valor|value|vista|view/] },
  revenue: { exact: ['valor conv', 'valor conv.', 'valor de conversao', 'valor de conversao da compra', 'valor de conversion de compras', 'conversion value', 'purchase conversion value', 'receita', 'action value omni purchase'], re: [/valor (de )?conv/, /conversion value/, /^receita$/, /value omni purchase/], not: [/carrinho|carrito|cart|finaliza|checkout|\/ ?cust|\/ ?cost|por cust|pagina|page/] },
  cart: { exact: ['adicoes ao carrinho', 'articulos agregados al carrito', 'adds to cart', 'agregar al carrito'], re: [/carrinho|carrito|add to cart|adds to cart/], not: [/valor|custo|costo|cost|por /] },
  checkout: { exact: ['finalizacoes de compra iniciadas', 'pagos iniciados', 'checkouts initiated', 'inicio de pago'], re: [/finaliza|checkout|pagos iniciados/], not: [/valor|custo|costo|cost|por /] },
  views: { exact: ['media insights total views', 'total views', 'page insights media view', 'visualizacoes', 'visualizaciones', 'views', 'media insights views', 'impressoes do perfil'], re: [/total views|media view|^visualizac|^views$/], not: [/custo|cost|taxa|rate|skip/] },
  interactions: { exact: ['media insights total interactions', 'total interactions', 'page insights post engagements', 'action post engagement', 'post engagements', 'interacoes', 'interacciones', 'engajamento', 'engagement'], re: [/total interactions|post engagement|^interac|^engaj/], not: [/custo|cost|page engagement|taxa|rate/] },
  pageEng: { exact: ['action page engagement', 'page engagement', 'page insights engagement'], re: [/page engagement/], not: [/custo|cost|post/] },
  likes: { exact: ['media insights total likes', 'total likes', 'curtidas', 'me gusta', 'likes'], re: [/total likes|^curtidas|^likes$/], not: [/custo|cost|page like/] },
  comments: { exact: ['media insights total comments', 'total comments', 'action post comments', 'comentarios', 'comments'], re: [/total comments|post comments|^comentario|^comments$/], not: [/custo|cost/] },
  shares: { exact: ['media insights shares', 'shares', 'compartilhamentos', 'compartidos'], re: [/^shares$|compartilh|compartid/], not: [/custo|cost/] },
  saves: { exact: ['media insights saved', 'saved', 'saves', 'salvamentos', 'guardados', 'action post save (onsite conversion)', 'action post save'], re: [/^saved$|^saves$|salvament|guardado|post save/], not: [/custo|cost/] },
  followersNew: { exact: ['new followers (last 30 days only)', 'new followers', 'novos seguidores', 'seguidores ganhos', 'action page likes', 'page likes'], re: [/new followers|novos seguidores|seguidores ganhos|page likes/], not: [/custo|cost|total/] },
  followersTotal: { exact: ['total followers (all time)', 'total followers', 'page insights follows', 'seguidores', 'followers', 'follows'], re: [/total followers|page insights follows|^seguidores$|^followers$|^follows$/], not: [/custo|cost|new|novos|ganhos/] },
  convAction: { exact: ['acao de conversao', 'tipo de conversao', 'accion de conversion', 'tipo de conversion', 'conversion action', 'conversion type'], re: [/acao de conv|tipo de conv|accion de conv|conversion (action|type)/] }
};
// Ordem importa: valor de conversão e compras são resolvidos antes de conversas/cadastros
var MEDIA_FIELDS = ['date', 'campaign', 'account', 'objective', 'adset', 'ad', 'spend', 'impressions', 'clicks', 'reach', 'frequency', 'lpv', 'revenue', 'purchases', 'cart', 'checkout', 'conversations', 'leads'];
var NUM_FIELDS = ['spend', 'impressions', 'clicks', 'reach', 'frequency', 'lpv', 'conversations', 'leads', 'purchases', 'revenue', 'cart', 'checkout'];
/* Redes sociais: uma linha por dia. followersTotal é um retrato (não se soma),
   o resto são contagens do dia (somam). */
var SOC_FIELDS = ['views', 'interactions', 'pageEng', 'likes', 'comments', 'shares', 'saves', 'followersNew', 'followersTotal'];
var SOC_SUM = ['views', 'interactions', 'pageEng', 'likes', 'comments', 'shares', 'saves', 'followersNew'];

function mapColumns(header, wanted) {
  var normed = header.map(norm), map = {}, used = {};
  wanted.forEach(function (f) {
    var rule = RULES[f]; if (!rule) return;
    var idx = -1;
    for (var i = 0; i < rule.exact.length && idx < 0; i++) { var j = normed.indexOf(rule.exact[i]); if (j > -1 && !used[j]) idx = j; }
    if (idx < 0) for (var r = 0; r < rule.re.length && idx < 0; r++) {
      for (var k = 0; k < normed.length; k++) {
        if (used[k]) continue;
        if (rule.re[r].test(normed[k]) && !(rule.not || []).some(function (nr) { return nr.test(normed[k]); })) { idx = k; break; }
      }
    }
    if (idx > -1) { map[f] = idx; used[idx] = 1; }
  });
  return map;
}
function findHeaderRow(rows) {
  for (var i = 0; i < Math.min(rows.length, 12); i++) {
    var normed = rows[i].map(norm), filled = normed.filter(Boolean).length;
    if (filled >= 2 && normed.some(function (h) { return RULES.date.exact.indexOf(h) > -1 || /^(dia|data|fecha|date|day)\b|created|timestamp/.test(h); })) return i;
  }
  return 0;
}
/* "12.686" é ambíguo: pode ser doze mil e seiscentos em pt-BR ou doze vírgula
   seiscentos em inglês. Já "4272778.196689775" e "1.234.567,89" não deixam
   dúvida. Por isso a evidência inequívoca vale 4 e a ambígua vale 1: basta um
   punhado de números com muitas casas para decidir o arquivo inteiro. */
function detectLocale(values) {
  var comma = 0, dot = 0;
  values.forEach(function (v) {
    var s = String(v).replace(/[^\d,.\-]/g, '');
    if (!s || !/\d/.test(s)) return;
    if (/^-?\d{1,3}(\.\d{3})+,\d+$/.test(s)) { comma += 4; return; }   // 1.234.567,89
    if (/^-?\d{1,3}(,\d{3})+\.\d+$/.test(s)) { dot += 4; return; }     // 1,234,567.89
    if (/^-?\d+,\d{4,}$/.test(s)) { comma += 4; return; }               // 123,456789
    if (/^-?\d+\.\d{4,}$/.test(s)) { dot += 4; return; }                // 123.456789
    if (/^-?\d+,\d{1,2}$/.test(s)) { comma += 2; return; }              // 123,45
    if (/^-?\d+\.\d{1,2}$/.test(s)) { dot += 2; return; }               // 123.45
    if (/^-?\d{1,3}(\.\d{3})+$/.test(s)) { comma += 1; return; }        // 12.686 (ambíguo)
    if (/^-?\d{1,3}(,\d{3})+$/.test(s)) { dot += 1; return; }           // 12,686 (ambíguo)
  });
  return comma > dot ? 'comma' : 'dot';
}
function makeNum(locale) {
  return function (v) {
    if (v == null) return 0;
    var s = String(v).trim(); if (!s || s === '-' || s === '--') return 0;
    s = s.replace(/[^\d,.\-]/g, '');
    if (locale === 'comma') s = s.replace(/\./g, '').replace(',', '.'); else s = s.replace(/,/g, '');
    var n = parseFloat(s); return isFinite(n) ? n : 0;
  };
}

/* ============================== CLASSIFICAÇÃO POR OBJETIVO ============================== */
/* Tipos de venda (opcional, DASH.tiposDeVenda): divide "Vendas no site" em
   funis com nome próprio (ex: associados e ingressos de eventos). Cada tipo
   vira um funil de venda completo, com rótulos, termos de campanha e, no
   Google, as ações de conversão que contam como resultado. */
var SALE_TYPES = (function () {
  var t = D.tiposDeVenda || {}, out = {};
  Object.keys(t).forEach(function (id) {
    var x = t[id] || {}, r = x.resultado || [];
    if (typeof r === 'string') r = [r, r];
    var lst = function (v) { return (v || []).map(norm).filter(Boolean); };
    out[id] = { id: id, nome: x.nome || id, um: r[0] || 'venda', varios: r[1] || r[0] || 'vendas', acao: x.etapaFinal || null,
      termos: lst(x.termos), receita: x.receita !== false, evento: x.evento === 'leads' ? 'leads' : 'purchases', etapas: x.etapas || {},
      mensal: +x.mensalidade || 0, gFinal: lst(x.acoesGoogle), gCheckout: lst(x.acoesGoogleCheckout), gCart: lst(x.acoesGoogleCarrinho) };
  });
  return out;
})();
var SALE_IDS = Object.keys(SALE_TYPES);
function isSale(f) { return f === 'vendas' || !!SALE_TYPES[f]; }
var FUNNEL_ORDER = SALE_IDS.concat(['vendas', 'cadastro', 'whatsapp', 'trafego', 'outros']);
function saleTypeByName(name) {
  var n = norm(name), hit = null;
  SALE_IDS.some(function (id) { if (SALE_TYPES[id].termos.some(function (t) { return n.indexOf(t) > -1; })) { hit = id; return true; } });
  return hit;
}
function matchAny(action, list) { var n = norm(action); return list.some(function (t) { return n.indexOf(t) > -1; }); }
function funnelFromText(t) {
  var n = norm(t);
  if (!n) return null;
  if (/whats|wpp|\bzap\b|mensag|messag|conversa|direct|\bdm\b|\bmsg\b/.test(n)) return 'whatsapp';
  if (/lead|cadastro|formul|potencial|registro|captac/.test(n)) return 'cadastro';
  if (/engag|engaj|interac|awareness|reconhec|alcance|reach|video|view|seguidor|perfil|brand|branding/.test(n)) return 'outros';
  if (/traffic|trafego|trafico|landing|visita/.test(n)) return 'trafego';
  if (/sales|\bvendas\b|\bventas\b|compras?\b|catalog|shopping|pmax|performance max|ecommerce|e-commerce|conversion|conversao|conversiones/.test(n)) return 'vendas';
  return null;
}
/* Funis que fazem sentido para este cliente: lista em DASH.funis, ou automático pelas colunas existentes. */
function allowedFunnels() {
  if (D.funis && D.funis.length) { var l = D.funis.concat(['outros']); if (l.indexOf('vendas') > -1) l = l.concat(SALE_IDS); return l; }
  var a = ['cadastro', 'whatsapp', 'outros'];
  if (STATE.has.purchases || SALE_IDS.length) a = a.concat(['vendas'], SALE_IDS);
  if (STATE.has.lpv) a.push('trafego');
  return a;
}
function classifyCampaigns(rows) {
  var agg = {};
  rows.forEach(function (r) {
    var k = r.key, a = agg[k] || (agg[k] = { campaign: r.campaign, platform: r.platform, objective: '', purchases: 0, leads: 0, conversations: 0, lpv: 0, clicks: 0, forced: r.forcedFunnel || null });
    a.purchases += r.purchases; a.leads += r.leads; a.conversations += r.conversations; a.lpv += r.lpv; a.clicks += r.clicks;
    if (r.objective) a.objective = r.objective;
  });
  var overrides = D.objetivos || {}, out = {}, allow = allowedFunnels();
  Object.keys(agg).forEach(function (k) {
    var a = agg[k], f = null, why = '';
    // 1) regra manual  2) fonte  3) resultado que a campanha gerou  4) nome  5) objetivo da planilha
    Object.keys(overrides).some(function (sub) { if (norm(a.campaign).indexOf(norm(sub)) > -1) { f = overrides[sub]; why = 'regra da configuração'; return true; } });
    // 1b) tipo de venda pelo nome (ignora campanhas de alcance/engajamento)
    if (!f && SALE_IDS.length && funnelFromText(a.campaign) !== 'outros') { var tv = saleTypeByName(a.campaign); if (tv) { f = tv; why = 'tipo de venda pelo nome'; } }
    if (!f && a.forced) { f = a.forced; why = 'definido na fonte'; }
    if (!f) {
      var res = [['vendas', a.purchases], ['cadastro', a.leads], ['whatsapp', a.conversations]].filter(function (x) { return allow.indexOf(x[0]) > -1; }).sort(function (x, y) { return y[1] - x[1]; });
      if (res.length && res[0][1] > 0) { f = res[0][0]; why = 'resultado gerado'; }
    }
    if (!f) { f = funnelFromText(a.campaign); if (f) why = 'nome da campanha'; }
    if (!f) { f = funnelFromText(a.objective); if (f) why = 'objetivo da campanha'; }
    if (!f) { f = a.lpv > 0 ? 'trafego' : 'outros'; why = 'sem resultado de conversão'; }
    if (allow.indexOf(f) < 0) { why += ' → ' + FNAME(f) + ' não se aplica a este cliente'; f = 'outros'; }
    out[k] = { funnel: f, why: why };
  });
  return out;
}

/* ============================== ESTADO ============================== */
var STATE = {
  sources: [], rows: [], crm: [], gconv: [], social: [], has: {}, hasSoc: {}, socRedes: {}, camp: {},
  preset: store.get('preset', 'mtd'), from: null, to: null, incToday: false,
  plat: 'all', acct: 'all', chartMetric: 'spend', sort: { col: 'spendNow', dir: -1 },
  loadedAt: null
};

/* ============================== PROCESSAMENTO POR FONTE ============================== */
function processMedia(src, text) {
  var st = src.status;
  var rows = csvRows(text, detectDelim(text));
  if (!rows.length) { st.error = L('Planilha vazia.', 'Planilla vacía.'); return []; }
  var h = findHeaderRow(rows), header = rows[h], map = mapColumns(header, MEDIA_FIELDS);
  st.headers = header; st.map = map; st.headerRow = h;
  var miss = ['date', 'campaign', 'spend'].filter(function (f) { return map[f] == null; });
  if (miss.length) { st.error = L('Colunas essenciais não encontradas: ', 'Columnas esenciales no encontradas: ') + miss.join(', '); return []; }
  var body = rows.slice(h + 1), sample = [];
  NUM_FIELDS.forEach(function (f) { if (map[f] != null) body.slice(0, 300).forEach(function (r) { if (r[map[f]]) sample.push(r[map[f]]); }); });
  st.locale = src.decimal || detectLocale(sample);
  var num = makeNum(st.locale), out = [], seen = {}, dup = 0, skipped = 0;
  var platform = src.plataforma || 'meta', have = {};
  NUM_FIELDS.forEach(function (f) { have[f] = map[f] != null; });
  if (src.conversaoComo === 'cadastro') { have.leads = have.leads || have.purchases; have.purchases = false; have.revenue = false; }
  body.forEach(function (r) {
    var date = parseDate(r[map.date]);
    if (!date) { skipped++; return; }
    var o = {
      date: date, platform: platform,
      account: (map.account != null && r[map.account]) ? String(r[map.account]).trim() : (src.conta || (platform === 'google' ? 'Google Ads' : 'Meta Ads')),
      campaign: String(r[map.campaign] || '(sem nome)').trim(),
      objective: map.objective != null ? String(r[map.objective] || '') : '',
      adset: map.adset != null ? String(r[map.adset] || '') : '',
      ad: map.ad != null ? String(r[map.ad] || '') : '',
      forcedFunnel: src.funil || null, _h: have
    };
    NUM_FIELDS.forEach(function (f) { o[f] = map[f] != null ? num(r[map[f]]) : 0; });
    if (src.conversaoComo === 'cadastro') { o.leads += o.purchases; o.purchases = 0; o.revenue = 0; }
    o.key = o.platform + '||' + o.account + '||' + o.campaign;
    var dk = [date, o.key, o.adset, o.ad].join('|'), sig = NUM_FIELDS.map(function (f) { return Math.round(o[f] * 100); }).join(',');
    if (seen[dk] === sig) { dup++; return; }
    seen[dk] = sig;
    out.push(o);
  });
  NUM_FIELDS.forEach(function (f) { if (map[f] != null) STATE.has[f] = true; });
  if (src.conversaoComo === 'cadastro' && map.purchases != null) STATE.has.leads = true;
  st.rows = out.length; st.dup = dup; st.skipped = skipped;
  return out;
}
var SALE_RE = /vend|vendid|fechad|ganh|\bwon\b|compr|contrat|matricul|cerrad|cliente fechado/;
var QUAL_RE = /qualific|visita|agend|proposta|negoci|oportun|reuni|atendid|interesse|calificad/;
function processCRM(src, text) {
  var st = src.status;
  var rows = csvRows(text, detectDelim(text));
  if (!rows.length) { st.error = L('Planilha vazia.', 'Planilla vacía.'); return []; }
  var h = findHeaderRow(rows), header = rows[h], normed = header.map(norm);
  var map = mapColumns(header, ['date']);
  /* O nome da coluna não basta: já apareceu planilha com "Data de entrada"
     totalmente vazia e a data real em "Data de qualificação". Aqui o motor
     testa quantas linhas cada coluna candidata consegue virar data de verdade
     e fica com a melhor. Sem isso, a fonte inteira sumiria em silêncio. */
  var amostra = rows.slice(h + 1, h + 400);
  function taxaData(idx) {
    if (idx == null || idx < 0) return 0;
    var vivos = 0, bons = 0;
    amostra.forEach(function (r) {
      var t = String(r[idx] == null ? '' : r[idx]).trim();
      if (!t) return;
      vivos++;
      if (parseDate(t)) bons++;
    });
    return amostra.length ? bons / amostra.length : 0;
  }
  if (src.colunaData) {
    var forc = normed.indexOf(norm(src.colunaData));
    if (forc > -1) map.date = forc;
  } else {
    var melhor = map.date, melhorTaxa = taxaData(map.date);
    if (melhorTaxa < 0.5) {
      header.forEach(function (_, j) {
        var t = taxaData(j);
        if (t > melhorTaxa + 0.05) { melhorTaxa = t; melhor = j; }
      });
      map.date = melhor;
    }
    st.dateScore = melhorTaxa;
  }
  st.dateCol = map.date != null && map.date > -1 ? header[map.date] : null;
  /* "Data de qualificação" contém "qualific" e era escolhida como coluna de
     status. Agora: primeiro quem tem "status" no nome, depois os outros termos,
     sempre descartando colunas de data. */
  var naoStatus = function (x) { return /^data|fecha|^date|horario|timestamp/.test(x); };
  var statusIdx = -1;
  if (src.colunaStatus) statusIdx = normed.indexOf(norm(src.colunaStatus));
  else {
    statusIdx = normed.findIndex(function (x) { return /status/.test(x) && !naoStatus(x); });
    if (statusIdx < 0) statusIdx = normed.findIndex(function (x) { return /etapa|situac|fase|stage|qualific|resultado/.test(x) && !naoStatus(x); });
  }
  // quando a própria planilha JÁ é a lista de qualificados (ou de vendas), não há
  // coluna de status: a etapa vem declarada na fonte
  var forcaQual = src.etapa === 'qualificado' || src.etapa === 'qual';
  var forcaVenda = src.etapa === 'venda' || src.etapa === 'vendido';
  var utmIdx = src.colunaCampanha ? normed.indexOf(norm(src.colunaCampanha)) : normed.findIndex(function (x) { return /utm[_ ]?campaign|utm[_ ]?campanha|^campanha$|^campaign$|origem da campanha/.test(x); });
  var valueIdx = src.colunaValor ? normed.indexOf(norm(src.colunaValor)) : normed.findIndex(function (x) { return /valor da venda|valor venda|receita|ticket|valor fechado/.test(x); });
  st.headers = header; st.map = { date: map.date, status: statusIdx > -1 ? statusIdx : undefined, value: valueIdx > -1 ? valueIdx : undefined }; st.headerRow = h;
  if (map.date == null) { st.error = L('Nenhuma coluna de data encontrada.', 'No se encontró columna de fecha.'); return []; }
  var sampleC = [];
  rows.slice(h + 1, h + 200).forEach(function (r) { r.forEach(function (cell) { if (cell) sampleC.push(cell); }); });
  var num = makeNum(src.decimal === 'comma' || src.decimal === 'dot' ? src.decimal : detectLocale(sampleC));
  var out = [];
  rows.slice(h + 1).forEach(function (r) {
    var d = parseDate(r[map.date]); if (!d) return;
    var s = statusIdx > -1 ? norm(r[statusIdx]) : '';
    var sale = forcaVenda || (statusIdx > -1 && SALE_RE.test(s));
    var qual = sale || forcaQual || (statusIdx > -1 && QUAL_RE.test(s));
    var raw = {}; header.forEach(function (hh, j) { raw[hh] = r[j] == null ? '' : r[j]; });
    out.push({ date: d, funnel: src.funil || 'cadastro', sale: sale, qual: qual, value: (sale && valueIdx > -1) ? num(r[valueIdx]) : 0, status: statusIdx > -1 ? String(r[statusIdx]).trim() : '', utm: (utmIdx > -1 && !valorVazio(r[utmIdx])) ? String(r[utmIdx]).trim() : '', etapa: src.etapa || '', raw: raw, src: src.nome || '' });
  });
  st.rows = out.length; st.hasStatus = statusIdx > -1 || forcaQual || forcaVenda;
  st.map.campanha = utmIdx > -1 ? utmIdx : undefined;
  st.etapa = src.etapa || '';
  return out;
}
function processGConv(src, text) {
  var st = src.status, rows = csvRows(text, detectDelim(text));
  if (!rows.length) { st.error = L('Planilha vazia.', 'Planilla vacía.'); return []; }
  var h = findHeaderRow(rows), header = rows[h], map = mapColumns(header, ['date', 'campaign', 'convAction', 'purchases', 'revenue']);
  st.headers = header; st.map = map; st.headerRow = h;
  if (map.date == null || map.convAction == null) { st.error = L('Faltam colunas de data ou ação de conversão.', 'Faltan columnas de fecha o acción de conversión.'); return []; }
  var body = rows.slice(h + 1), num = makeNum(detectLocale(body.slice(0, 300).map(function (r) { return r[map.purchases]; })));
  var out = [], seen = {}, dup = 0;
  body.forEach(function (r) {
    var d = parseDate(r[map.date]); if (!d) return;
    var o = { date: d, campaign: map.campaign != null ? String(r[map.campaign] || '').trim() : '', action: String(r[map.convAction] || '—').trim(), conv: map.purchases != null ? num(r[map.purchases]) : 0, value: map.revenue != null ? num(r[map.revenue]) : 0 };
    // mesma dedupe da mídia: planilha que soma em vez de substituir dobra os números
    var dk = [o.date, o.campaign, o.action].join('|'), sig = Math.round(o.conv * 100) + ',' + Math.round(o.value * 100);
    if (seen[dk] === sig) { dup++; return; }
    seen[dk] = sig;
    out.push(o);
  });
  st.rows = out.length; st.dup = dup;
  return out;
}

/* ============================== REDES SOCIAIS ============================== */
function processSocial(src, text) {
  var st = src.status, rows = csvRows(text, detectDelim(text));
  if (!rows.length) { st.error = L('Planilha vazia.', 'Planilla vacía.'); return []; }
  var h = findHeaderRow(rows), header = rows[h], map = mapColumns(header, ['date'].concat(SOC_FIELDS));
  st.headers = header; st.map = map; st.headerRow = h;
  if (map.date == null) { st.error = L('Não encontrei a coluna de data.', 'No encontré la columna de fecha.'); return []; }

  var body = rows.slice(h + 1), sample = [];
  SOC_FIELDS.forEach(function (f) { if (map[f] != null) body.slice(0, 300).forEach(function (r) { if (r[map[f]]) sample.push(r[map[f]]); }); });
  var loc = src.decimal === 'comma' || src.decimal === 'dot' ? src.decimal : detectLocale(sample);
  st.locale = loc;
  var num = makeNum(loc);

  var origem = src.origem || (src.tipo === 'engajamento' ? 'pago' : 'organico');
  var rede = norm(src.rede || (origem === 'pago' ? 'meta' : '')) || 'rede';
  var byDate = {}, dup = 0;

  body.forEach(function (r) {
    var d = parseDate(r[map.date]); if (!d) return;
    var o = { date: d, rede: rede, origem: origem, fonte: src.nome || src.rede || src.tipo };
    SOC_FIELDS.forEach(function (f) { o[f] = map[f] != null ? num(r[map[f]]) : null; });
    // sem coluna de total, engajamento é a soma das partes que existirem
    if (o.interactions == null) {
      var partes = ['likes', 'comments', 'shares', 'saves'].filter(function (f) { return ok(o[f]); });
      if (partes.length) o.interactions = partes.reduce(function (a, f) { return a + o[f]; }, 0);
    }
    // uma linha por dia por fonte: se vier repetida (planilha que sobrescreve
    // em vez de substituir), a última vence e a anterior é contada como duplicata
    if (byDate[d]) dup++;
    byDate[d] = o;
  });

  var out = Object.keys(byDate).sort().map(function (k) { return byDate[k]; });
  // o campo de total de seguidores às vezes vem congelado (a extração repete o
  // total de hoje em todas as linhas). Nesse caso ele só serve como retrato.
  var tots = out.map(function (x) { return x.followersTotal; }).filter(function (v) { return v != null && v > 0; });
  st.followersFrozen = tots.length > 2 && tots.every(function (v) { return v === tots[0]; });
  st.rows = out.length; st.dup = dup; st.origem = origem; st.rede = rede;
  SOC_FIELDS.forEach(function (f) { if (map[f] != null) STATE.hasSoc[f] = true; });
  if (out.length && out.some(function (x) { return ok(x.interactions); })) STATE.hasSoc.interactions = true;
  if (out.length) STATE.socRedes[rede] = true;
  return out;
}
function socIn(from, to, filter) {
  return STATE.social.filter(function (r) { return inRange(r.date, from, to) && (!filter || filter(r)); });
}
function socAgg(rows) {
  var o = { n: rows.length };
  SOC_SUM.forEach(function (f) { o[f] = 0; });
  var lastTot = {};
  rows.forEach(function (r) {
    SOC_SUM.forEach(function (f) { if (ok(r[f])) o[f] += r[f]; });
    if (ok(r.followersTotal) && r.followersTotal > 0) lastTot[r.rede] = r.followersTotal;
  });
  o.followersTotal = Object.keys(lastTot).reduce(function (a, k) { return a + lastTot[k]; }, 0);
  return o;
}
/** Ganho de seguidores: usa o campo diário quando existe; senão, a diferença do retrato. */
function followersGain(rede, from, to) {
  var rows = socIn(from, to, function (r) { return r.rede === rede; });
  if (!rows.length) return null;
  var hasNew = rows.some(function (r) { return ok(r.followersNew); });
  if (hasNew) return rows.reduce(function (a, r) { return a + (ok(r.followersNew) ? r.followersNew : 0); }, 0);
  var tot = rows.filter(function (r) { return ok(r.followersTotal) && r.followersTotal > 0; });
  if (tot.length < 2) return null;
  var d = tot[tot.length - 1].followersTotal - tot[0].followersTotal;
  return d;
}
function hasSocial() { return STATE.social.length > 0; }

/* ============================== CARREGAMENTO ============================== */
function initSources() {
  STATE.sources = (D.fontes || []).map(function (f, i) {
    var tipo = f.tipo || (f.plataforma ? 'midia' : 'crm');
    return Object.assign({}, f, { id: 'f' + i, tipo: tipo, status: {} });
  });
}
function fetchText(url) {
  var u = url + (url.indexOf('?') > -1 ? '&' : '?') + '_=' + Date.now();
  return fetch(u, { cache: 'no-store' }).then(function (r) { if (!r.ok) throw new Error('HTTP ' + r.status); return r.text(); });
}
function loadAll() {
  STATE.rows = []; STATE.crm = []; STATE.gconv = []; STATE.social = []; STATE.has = {}; STATE.hasSoc = {}; STATE.socRedes = {};
  return Promise.all(STATE.sources.map(function (src) {
    src.status = {};
    var pasted = store.get('paste_' + src.id, '');
    var p = pasted ? Promise.resolve(pasted).then(function (t) { src.status.fromPaste = true; return t; }) : fetchText(src.url);
    return p.then(function (text) {
      if (src.tipo === 'midia') STATE.rows = STATE.rows.concat(processMedia(src, text));
      else if (src.tipo === 'conversoes_google') STATE.gconv = STATE.gconv.concat(processGConv(src, text));
      else if (src.tipo === 'social' || src.tipo === 'engajamento') STATE.social = STATE.social.concat(processSocial(src, text));
      else STATE.crm = STATE.crm.concat(processCRM(src, text));
      src.status.ok = !src.status.error;
    }).catch(function (e) { src.status.ok = false; src.status.error = L('Não foi possível ler a planilha publicada: ', 'No se pudo leer la planilla publicada: ') + e.message; });
  })).then(function () {
    STATE.camp = classifyCampaigns(STATE.rows);
    STATE.rows.forEach(function (r) { r.funnel = STATE.camp[r.key].funnel; });
    applySaleTypes();
    STATE.loadedAt = new Date();
  });
}

/* Google: a coluna "Conversões" soma todas as ações principais (carrinho,
   checkout, compra de outro produto...). Quando existe a planilha de tipo de
   conversão e o tipo de venda lista as ações dele, o resultado da campanha
   passa a ser só a soma dessas ações, no mesmo dia e campanha. */
function applySaleTypes() {
  var gc = {};
  STATE.gconv.forEach(function (c) { var k = c.date + '|' + norm(c.campaign); (gc[k] = gc[k] || []).push(c); });
  var done = {};
  STATE.rows.forEach(function (r) {
    var T = SALE_TYPES[r.funnel]; if (!T) return;
    if (!T.receita) r.revenue = 0;
    /* Tipo cujo pixel registra o resultado como Lead (ex: assinatura com
       cadastro): o lead vira o resultado do funil, sem contar em dobro. */
    if (T.evento === 'leads' && r.platform === 'meta') { r.purchases = r.leads; r.leads = 0; r.cart = 0; r.checkout = 0; }
    if (r.platform !== 'google' || !STATE.gconv.length || !(T.gFinal.length || T.gCheckout.length || T.gCart.length)) return;
    var k = r.date + '|' + norm(r.campaign), list = done[k] ? [] : (gc[k] || []);
    done[k] = 1;
    var sum = function (terms, fld) { return list.filter(function (c) { return matchAny(c.action, terms); }).reduce(function (a, c) { return a + c[fld]; }, 0); };
    var h = Object.assign({}, r._h);
    if (T.gFinal.length) { r.purchases = sum(T.gFinal, 'conv'); r.revenue = T.receita ? sum(T.gFinal, 'value') : 0; h.purchases = true; }
    if (T.gCheckout.length) { r.checkout = sum(T.gCheckout, 'conv'); h.checkout = true; }
    if (T.gCart.length) { r.cart = sum(T.gCart, 'conv'); h.cart = true; }
    r._h = h;
  });
}
function gconvType(action) {
  var out = null;
  SALE_IDS.some(function (id) {
    var T = SALE_TYPES[id];
    if (matchAny(action, T.gFinal)) out = { f: id, papel: T.um };
    else if (matchAny(action, T.gCheckout)) out = { f: id, papel: L('início do pagamento', 'inicio del pago') };
    else if (matchAny(action, T.gCart)) out = { f: id, papel: L('carrinho', 'carrito') };
    return !!out;
  });
  return out;
}

/* ============================== AGREGAÇÃO ============================== */
function blank() { var o = { n: 0, freqW: 0, freqImpr: 0 }; NUM_FIELDS.forEach(function (f) { o[f] = 0; }); return o; }
function agg(rows) {
  var s = blank();
  rows.forEach(function (r) {
    NUM_FIELDS.forEach(function (f) { if (f !== 'frequency') s[f] += r[f]; });
    if (r.frequency > 0) { s.freqW += r.frequency * r.impressions; s.freqImpr += r.impressions; }
    s.n++;
  });
  s.freq = s.freqImpr > 0 ? s.freqW / s.freqImpr : null;
  s.ctr = s.impressions > 0 ? s.clicks / s.impressions : null;
  s.cpm = s.impressions > 0 ? s.spend / s.impressions * 1000 : null;
  s.cpc = s.clicks > 0 ? s.spend / s.clicks : null;
  s.roas = s.spend > 0 && s.revenue > 0 ? s.revenue / s.spend : null;
  return s;
}
function inRange(d, from, to) { return d >= from && d <= to; }
function filtered(from, to, extra) {
  return STATE.rows.filter(function (r) {
    return inRange(r.date, from, to) && unitOk(r.campaign) && (STATE.plat === 'all' || r.platform === STATE.plat) && (STATE.acct === 'all' || r.account === STATE.acct) && (!extra || extra(r));
  });
}
function crmIn(from, to, funnel) { return STATE.crm.filter(function (c) { return inRange(c.date, from, to) && (!funnel || c.funnel === funnel) && (STATE.unit === 'all' || (c.utm && unitOk(c.utm))); }); }
function funnelsPresent() {
  var per = resolvePeriod(), set = {};
  STATE.rows.forEach(function (r) {
    if ((STATE.plat !== 'all' && r.platform !== STATE.plat) || (STATE.acct !== 'all' && r.account !== STATE.acct) || !unitOk(r.campaign)) return;
    if (r.spend > 0 && (inRange(r.date, per.from, per.to) || inRange(r.date, per.pFrom, per.pTo))) set[r.funnel] = 1;
  });
  return FUNNEL_ORDER.filter(function (f) { return set[f]; });
}

/* ============================== PERÍODOS ============================== */
function lastDataDate() { var m = ''; STATE.rows.forEach(function (r) { if (r.date > m) m = r.date; }); return m || null; }
function resolvePeriod() {
  var today = todayISO(), end = STATE.incToday ? today : addDays(today, -1), p = STATE.preset, from, to, pFrom, pTo;
  if (p === 'custom' && STATE.from && STATE.to) { from = STATE.from; to = STATE.to; var n = daysBetween(from, to) + 1; pTo = addDays(from, -1); pFrom = addDays(pTo, -(n - 1)); }
  else if (p === 'mtd') { from = monthStart(today); to = end < from ? from : end; pFrom = prevMonthStart(today); pTo = addDays(from, -1); }
  else if (p === 'lastmonth') { to = addDays(monthStart(today), -1); from = monthStart(to); pTo = addDays(from, -1); pFrom = monthStart(pTo); }
  else { var len = parseInt(p, 10) || 30; to = end; from = addDays(to, -(len - 1)); pTo = addDays(from, -1); pFrom = addDays(pTo, -(len - 1)); }
  return { from: from, to: to, pFrom: pFrom, pTo: pTo, len: daysBetween(from, to) + 1, pLen: daysBetween(pFrom, pTo) + 1, isMtd: p === 'mtd' };
}
function daySeries(from, to, extra) {
  var map = {}, out = [];
  filtered(from, to, extra).forEach(function (r) { (map[r.date] = map[r.date] || []).push(r); });
  for (var d = from; d <= to; d = addDays(d, 1)) out.push({ date: d, s: agg(map[d] || []) });
  return out;
}

/* ============================== FUNIS ============================== */
function FNAME(f) {
  if (SALE_TYPES[f]) return SALE_TYPES[f].nome;
  if (D.nomesFunis && D.nomesFunis[f]) return D.nomesFunis[f];
  return {
    vendas: L('Vendas no site', 'Ventas en el sitio'), cadastro: L('Cadastros', 'Registros'), whatsapp: 'WhatsApp',
    trafego: L('Tráfego para o site', 'Tráfico al sitio'), outros: L('Alcance e engajamento', 'Alcance e interacción')
  }[f] || f;
}
function RESULT(f) {
  if (SALE_TYPES[f]) return 'purchases';
  return { vendas: 'purchases', cadastro: 'leads', whatsapp: 'conversations', trafego: 'lpv', outros: null }[f];
}
function RNAME(f, plural) {
  if (SALE_TYPES[f]) return plural === false ? SALE_TYPES[f].um : SALE_TYPES[f].varios;
  var n = { vendas: [L('compra', 'compra'), L('compras', 'compras')], cadastro: [L('cadastro', 'registro'), L('cadastros', 'registros')], whatsapp: [L('conversa', 'conversación'), L('conversas', 'conversaciones')], trafego: [L('visita', 'visita'), L('visitas', 'visitas')] }[f];
  return n ? n[plural === false ? 0 : 1] : L('resultados', 'resultados');
}
/* Referências de mercado (Meta). Faixas amplas, servem de sanidade — o histórico da conta vale mais. */
var BENCH = {
  ctr: [0.01, 0.02], lpvRate: [0.70, 0.85], cartRate: [0.04, 0.10], chkRate: [0.45, 0.70], buyRate: [0.30, 0.50],
  qualRate: [0.15, 0.30], saleRate: [0.15, 0.30]
};

/* Monta as etapas de um funil com números do período atual e anterior. */
function buildFunnel(f, per, camp) {
  var ext = function (r) { return r.funnel === f && (!camp || r.key === camp); };
  var cur = agg(filtered(per.from, per.to, ext)), prev = agg(filtered(per.pFrom, per.pTo, ext));
  var crmSrc = STATE.sources.some(function (s) { return s.tipo === 'crm' && (s.funil || 'cadastro') === f; });
  var crmStatus = STATE.sources.some(function (s) { return s.tipo === 'crm' && (s.funil || 'cadastro') === f && s.status.hasStatus; });
  /* Com uma campanha selecionada, os contatos só entram se a UTM registrada
     no cadastro bater com o nome dela. Quem chegou sem UTM não é atribuível a
     campanha nenhuma e fica de fora, por isso a ressalva no cabeçalho. */
  var nomeCamp = camp && STATE.camp[camp] ? norm(camp.split('||')[2] || '') : '';
  var casaUtm = function (x) {
    if (!camp) return true;
    var u = norm(x.utm || '');
    if (!u || !nomeCamp) return false;
    return u === nomeCamp || u.indexOf(nomeCamp) > -1 || nomeCamp.indexOf(u) > -1;
  };
  var cc = crmIn(per.from, per.to, f).filter(casaUtm), cp = crmIn(per.pFrom, per.pTo, f).filter(casaUtm);
  var crmCur = { n: cc.length, qual: cc.filter(function (x) { return x.qual; }).length, sale: cc.filter(function (x) { return x.sale; }).length, value: cc.reduce(function (a, x) { return a + x.value; }, 0) };
  var crmPrev = { n: cp.length, qual: cp.filter(function (x) { return x.qual; }).length, sale: cp.filter(function (x) { return x.sale; }).length, value: cp.reduce(function (a, x) { return a + x.value; }, 0) };
  var useLpv = STATE.has.lpv && cur.lpv + prev.lpv > 0 && (isSale(f) || f === 'trafego' || (f === 'cadastro' && cur.lpv > 0));
  var S = [];
  function st(key, label, c, p, from, bench, extra) { S.push(Object.assign({ key: key, label: label, cur: c, prev: p, from: from, bench: bench || null }, extra || {})); }
  st('impressions', L('Aparições do anúncio', 'Apariciones del anuncio'), cur.impressions, prev.impressions, null);
  st('clicks', L('Cliques', 'Clics'), cur.clicks, prev.clicks, 'impressions', f === 'outros' ? null : BENCH.ctr, { rateKey: 'ctr' });
  if (useLpv) st('lpv', L('Chegaram na página', 'Llegaron a la página'), cur.lpv, prev.lpv, 'clicks', BENCH.lpvRate, { rateKey: 'lpv' });
  var base = useLpv ? 'lpv' : 'clicks';
  if (isSale(f)) {
    /* Etapa sem nenhum registro nos dois períodos não é medida neste funil
       (ex: Google sem carrinho, associação sem carrinho): some do desenho em
       vez de aparecer como zero, que parece problema. */
    var T = SALE_TYPES[f] || { etapas: {} };
    var hasCart = STATE.has.cart && cur.cart + prev.cart > 0, hasChk = STATE.has.checkout && cur.checkout + prev.checkout > 0;
    /* Etapa intermediária com MENOS registros que a compra final não está
       sendo medida de verdade (ex: pixel sem evento de checkout): esconder. */
    var fin = cur.purchases + prev.purchases;
    if (cur.checkout + prev.checkout < fin) hasChk = false;
    if (cur.cart + prev.cart < fin) hasCart = false;
    if (T.etapas.carrinho === false) hasCart = false;
    if (T.etapas.pagamento === false) hasChk = false;
    if (hasCart) st('cart', (typeof T.etapas.carrinho === 'string' ? T.etapas.carrinho : L('Colocaram no carrinho', 'Agregaron al carrito')), cur.cart, prev.cart, base, useLpv ? BENCH.cartRate : null, { rateKey: 'cart' });
    if (hasChk) st('checkout', (typeof T.etapas.pagamento === 'string' ? T.etapas.pagamento : L('Foram para o pagamento', 'Fueron al pago')), cur.checkout, prev.checkout, hasCart ? 'cart' : base, hasCart ? BENCH.chkRate : null, { rateKey: 'checkout' });
    st('purchases', T.acao || L('Compraram', 'Compraron'), cur.purchases, prev.purchases, hasChk ? 'checkout' : (hasCart ? 'cart' : base), hasChk ? BENCH.buyRate : null, { rateKey: 'purchases', final: true, sale: true });
  } else if (f === 'cadastro') {
    st('leads', L('Preencheram o formulário', 'Completaron el formulario'), cur.leads, prev.leads, base, null, { rateKey: 'leads', final: true });
  } else if (f === 'whatsapp') {
    st('conversations', L('Chamaram no WhatsApp', 'Escribieron por WhatsApp'), cur.conversations, prev.conversations, base, null, { rateKey: 'conversations', final: true });
  }
  var resKey = RESULT(f);
  if (f === 'cadastro' || f === 'whatsapp') {
    if (crmSrc) st('crm', L('Chegaram ao comercial', 'Llegaron a ventas'), crmCur.n, crmPrev.n, resKey, null, { rateKey: 'crm' });
    else st('crm', L('Chegaram ao comercial', 'Llegaron a ventas'), null, null, resKey, null, { missing: 'crm' });
    if (crmStatus) {
      var fonteQual = STATE.sources.filter(function (x) { return x.tipo === 'crm' && x.status && x.status.etapa; })[0];
      st('qual', L('Qualificados', 'Calificados') + (fonteQual ? nota(L('Os qualificados vêm de uma lista própria (', 'Los calificados vienen de una lista propia (') + esc(fonteQual.nome || fonteQual.etapa) + L('), separada da lista de cadastros. Se um contato qualificado não estiver também na lista de cadastros, a taxa de passagem entre as duas etapas fica subestimada.', '), separada de la lista de registros. Si un contacto calificado no está también en la lista de registros, la tasa de paso entre las dos etapas queda subestimada.')) : ''), crmCur.qual, crmPrev.qual, 'crm', BENCH.qualRate, { rateKey: 'qual' });
      st('sale', L('Compraram', 'Compraron'), crmCur.sale, crmPrev.sale, 'qual', BENCH.saleRate, { rateKey: 'sale', sale: true });
    } else {
      st('qual', L('Qualificados', 'Calificados'), null, null, 'crm', null, { missing: 'qual' });
      st('sale', L('Compraram', 'Compraron'), null, null, 'qual', null, { missing: 'sale' });
    }
  }
  var byKey = {}; S.forEach(function (s) { byKey[s.key] = s; });
  var curRows = filtered(per.from, per.to, ext), prevRows = filtered(per.pFrom, per.pTo, ext);
  // Taxa entre duas etapas só usa linhas de fontes que medem AS DUAS etapas
  // (ex: Google sem "visitas à página" não entra no cálculo clique → visita).
  function pairRate(rows, k, b) {
    var num = 0, den = 0;
    rows.forEach(function (r) { if (r._h[k] && r._h[b]) { num += r[k]; den += r[b]; } });
    return { rate: den > 0 ? num / den : null, base: den };
  }
  var MEDIA_KEYS = { impressions: 1, clicks: 1, lpv: 1, cart: 1, checkout: 1, purchases: 1, leads: 1, conversations: 1 };
  S.forEach(function (s) {
    var b = s.from && byKey[s.from];
    if (b && MEDIA_KEYS[s.key] && MEDIA_KEYS[b.key]) {
      var c = pairRate(curRows, s.key, b.key), p = pairRate(prevRows, s.key, b.key);
      s.rate = c.rate; s.pRate = p.rate; s.base = c.base; s.pBase = p.base;
    } else {
      s.rate = b && ok(s.cur) && b.cur > 0 ? s.cur / b.cur : null;
      s.pRate = b && ok(s.prev) && b.prev > 0 ? s.prev / b.prev : null;
      s.base = b ? b.cur : null; s.pBase = b ? b.prev : null;
    }
  });
  var level = 1;
  if (resKey && cur[resKey] + prev[resKey] > 0) level = 2;
  if (crmSrc && (f === 'cadastro' || f === 'whatsapp')) level = 3;
  if (isSale(f) && cur.purchases + prev.purchases > 0) level = 4;
  if (crmStatus && crmCur.sale + crmPrev.sale > 0) level = 4;
  var salesReal = isSale(f) ? cur.purchases : (crmStatus ? crmCur.sale : null);
  var revenueReal = isSale(f) ? (cur.revenue || null) : (crmStatus && crmCur.value ? crmCur.value : null);
  return { f: f, cur: cur, prev: prev, stages: S, by: byKey, level: level, crmSrc: crmSrc, crmStatus: crmStatus, crmCur: crmCur, crmPrev: crmPrev, resKey: resKey, salesReal: salesReal, revenueReal: revenueReal };
}


/* ============================== CLIENTE: UTILIDADES ============================== */
function nota() { return ''; }
function cap(s) { s = String(s || ''); return s.charAt(0).toUpperCase() + s.slice(1); }
function plural(n, one, many) { return nf(n, 0) + ' ' + (Math.round(n) === 1 ? one : many); }
function pct(v) { if (!ok(v)) return '—'; var x = v * 100; return nf(x, Math.abs(x) < 10 ? 1 : 0) + '%'; }
function perDay(v) { return ok(v) ? nf(v, v < 10 ? 1 : 0) : '—'; }
function missingLine(key) {
  return ({ crm: 'Sem lista de contatos conectada', qual: 'Ainda não registrado', sale: 'Venda ainda não registrada' })[key] || 'Não medido';
}
function ic(id, cls) { return '<svg class="i' + (cls ? ' ' + cls : '') + '"><use href="#i-' + id + '"/></svg>'; }

/* Unidades (opcional): DASH.unidades = ['Paulista', { nome: 'Vila Sonia', termos: ['vila sonia'] }]
   A unidade é reconhecida por um trecho do nome da campanha. */
var UNITS = (D.unidades || []).map(function (u) {
  if (typeof u === 'string') u = { nome: u };
  return { nome: u.nome, termos: (u.termos && u.termos.length ? u.termos : [u.nome]).map(norm) };
});
STATE.unit = 'all';
function unitOk(name) {
  if (STATE.unit === 'all') return true;
  var u = UNITS[+STATE.unit]; if (!u) return true;
  var n = norm(name);
  return u.termos.some(function (t) { return t && n.indexOf(t) > -1; });
}

/* Apelido da campanha: tira marcações internas (agência, objetivo, canal) e
   deixa o que o cliente reconhece. DASH.apelidos = { 'trecho do nome': 'Apelido' }
   corrige qualquer caso. O nome original fica sempre a um clique. */
var NICK_DROP = /^(fitmark|atom|atom lab|lp|landing|meta|meta ads|google|google ads|ads|search|pesquisa|rede de pesquisa|display|pmax|performance max|msg|mensagem|wpp|whatsapp|zap|leads?|cadastros?|vendas?|conversao|conversoes|trafego|eng|engajamento|alcance|reach|video|videos|reels?|cbo|abo|adv\+?|advantage\+?|asc|teste|test|campanha|remarketing rmk|\d{1,3})$/;
var NICK_ACC = { socios: 'Sócios', socio: 'Sócio', associacao: 'Associação', associados: 'Associados', conexoes: 'Conexões', expansao: 'Expansão', 'sao paulo': 'São Paulo', brasilia: 'Brasília', maringa: 'Maringá', unidade: 'Unidade', inscricoes: 'Inscrições', inscricao: 'Inscrição', promocao: 'Promoção', matricula: 'Matrícula', matriculas: 'Matrículas', verao: 'Verão' };
var NICK_KEEP = ['ACAD', 'UACAD', 'BR', 'SP', 'RJ', 'PR', 'MG', 'CNX', 'EAD', 'CRM', 'B2B'].concat(D.siglas || []);
var SMALL = { de: 1, da: 1, do: 1, das: 1, dos: 1, e: 1, em: 1, para: 1, com: 1, a: 1, o: 1 };
function titleWord(w, i) {
  if (!w) return w;
  if (NICK_KEEP.indexOf(w.toUpperCase()) > -1 && w === w.toUpperCase()) return w;
  var low = w.toLowerCase();
  if (i > 0 && SMALL[low]) return low;
  if (NICK_ACC[norm(low)]) return NICK_ACC[norm(low)];
  return low.charAt(0).toUpperCase() + low.slice(1);
}
function niceToken(t) {
  var hasLower = /[a-zà-ú]/.test(t);
  if (hasLower) return NICK_ACC[norm(t)] || t;
  var full = NICK_ACC[norm(t)];
  if (full) return full;
  return t.split(/\s+/).map(titleWord).join(' ');
}
function nick(name) {
  var map = D.apelidos || {}, n = norm(name), hit = null;
  Object.keys(map).some(function (k) { if (n.indexOf(norm(k)) > -1) { hit = map[k]; return true; } });
  if (hit) return hit;
  var s = String(name || '').replace(/[\u{1F000}-\u{1FAFF}\u{2600}-\u{27BF}\u{2B00}-\u{2BFF}\u{FE0F}]/gu, ' ');
  var parts = [];
  s.split(/[\[\]|]+/).forEach(function (p) { p.split(/\s+-\s+|\s+–\s+/).forEach(function (q) { q = q.replace(/_/g, ' ').replace(/\s+/g, ' ').trim(); if (q) parts.push(q); }); });
  var seen = {}, out = [];
  parts.forEach(function (p) {
    var k = norm(p); if (!k || seen[k] || NICK_DROP.test(k)) return;
    seen[k] = 1; out.push(niceToken(p));
  });
  var res = out.join(' · ');
  res = res || String(name || '').trim();
  return res.charAt(0).toUpperCase() + res.slice(1);
}

/* ============================== CLIENTE: ESTILO ============================== */
var PRIMARY = D.corDestaque || '#ff0561';
var FUNDO = D.fundo || '#e7e7e7';
function injectStyle() {
  if (!document.getElementById('acl-font')) {
    var l = document.createElement('link'); l.id = 'acl-font'; l.rel = 'stylesheet';
    l.href = 'https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap';
    document.head.appendChild(l);
  }
  var css = '\
.acl{--background:' + FUNDO + ';--foreground:#1c1c22;--card:#ffffff;--popover:#ffffff;--primary:' + PRIMARY + ';--secondary:#f4f4f5;--muted-foreground:#71717a;--grouped:#f4f4f6;--destructive:#e5484d;--border:#e4e4e7;--ring:' + PRIMARY + ';\
--success:#047857;--success-soft:#d1fae5;--warning:#b45309;--glass-surface:rgba(255,255,255,.72);--glass-border:rgba(255,255,255,.4);\
--shadow-card:0 1px 2px 0 rgba(0,0,0,.05);--shadow-modal:0 20px 25px -5px rgba(0,0,0,.1),0 8px 10px -6px rgba(0,0,0,.1);\
--r-sm:8px;--r-md:10px;--r-lg:12px;--r-2xl:16px;--chip-ok:#34d399;--chip-bad:#f87171;\
container-type:inline-size;background:var(--background);color:var(--foreground);font-family:Inter,system-ui,-apple-system,sans-serif;font-size:14px;line-height:20px;-webkit-font-smoothing:antialiased;padding:8px 16px 40px;box-sizing:border-box}\
.acl *{box-sizing:border-box}.acl button{font:inherit;color:inherit}.acl :focus-visible{outline:2px solid var(--ring);outline-offset:2px}\
.acl .i{width:16px;height:16px;flex:none;stroke:currentColor;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}\
.acl .num{font-variant-numeric:tabular-nums}.acl [hidden]{display:none!important}\
.acl main{max-width:1120px;margin:0 auto;padding-top:16px;display:flex;flex-direction:column;gap:40px}\
.acl .section-label{font-size:12px;line-height:16px;font-weight:600;letter-spacing:.025em;text-transform:uppercase;color:var(--muted-foreground)}\
.acl h1{margin:0;font-size:24px;line-height:32px;font-weight:600;letter-spacing:-.025em}\
.acl h2{margin:0;font-size:18px;line-height:28px;font-weight:600;letter-spacing:-.01em}\
.acl h3{margin:0;font-size:16px;line-height:24px;font-weight:600}.acl p{margin:0}\
.acl .hint{color:var(--muted-foreground);font-size:13px;line-height:20px}\
.acl .pagehead{display:flex;flex-wrap:wrap;align-items:flex-end;justify-content:space-between;gap:16px}\
.acl .titles{display:flex;flex-direction:column;gap:4px}\
.acl .filters{display:flex;flex-wrap:wrap;align-items:flex-start;justify-content:flex-end;gap:12px 16px}\
.acl .fwrap{display:flex;flex-direction:column;align-items:flex-end;gap:6px}.acl .fwrap.l{align-items:flex-start}\
.acl .pbox{position:relative}\
.acl .period{display:inline-flex;align-items:center;gap:10px;min-height:40px;padding:4px 12px;cursor:pointer;border:1px solid var(--border);border-radius:var(--r-md);background:var(--card);box-shadow:var(--shadow-card);font-weight:500;text-align:left;transition:border-color 150ms}\
.acl .period:hover{border-color:color-mix(in srgb,var(--primary) 40%,var(--border))}.acl .period .i{color:var(--muted-foreground)}\
.acl .period .rng{color:var(--muted-foreground);font-weight:400;font-size:13px}\
.acl .menu{position:absolute;right:0;top:calc(100% + 8px);z-index:30;width:280px;max-width:calc(100vw - 32px);background:var(--popover);border:1px solid var(--border);border-radius:var(--r-lg);box-shadow:var(--shadow-modal);padding:4px;display:flex;flex-direction:column;gap:2px}\
.acl .menu.l{left:0;right:auto}\
.acl .menu button{display:flex;align-items:center;justify-content:space-between;gap:12px;width:100%;min-height:40px;padding:6px 12px;border:0;background:transparent;border-radius:var(--r-sm);cursor:pointer;text-align:left}\
.acl .menu button:hover{background:color-mix(in srgb,var(--primary) 10%,transparent)}\
.acl .menu button .o{display:flex;flex-direction:column}.acl .menu button .o b{font-weight:500;font-size:14px}.acl .menu button .o span{font-size:12px;line-height:16px;color:var(--muted-foreground)}\
.acl .menu button[aria-selected=true]{color:var(--primary)}.acl .menu button .i{color:var(--primary);visibility:hidden}.acl .menu button[aria-selected=true] .i{visibility:visible}\
.acl .menu hr{border:0;border-top:1px solid var(--border);margin:4px 0;width:100%}\
.acl .menu .custom{display:flex;flex-direction:column;gap:8px;padding:8px 12px 10px}\
.acl .menu .custom .row{display:flex;gap:8px;align-items:center}\
.acl .menu .custom #aclApply{justify-content:center;background:var(--primary);color:#fff;font-weight:600;min-height:36px}.acl .menu .custom #aclApply:hover{filter:brightness(1.05);background:var(--primary)}\
.acl .menu input[type=date]{flex:1;min-width:0;height:34px;border:1px solid var(--border);border-radius:var(--r-sm);padding:0 8px;font:inherit;font-size:13px;background:var(--card);color:var(--foreground)}\
.acl .cmpline{margin:0;font-size:12px;line-height:16px;color:var(--muted-foreground);font-variant-numeric:tabular-nums}\
.acl .sr{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0 0 0 0);white-space:nowrap}\
.acl .badge{display:inline-flex;align-items:center;gap:4px;height:22px;padding-inline:8px;border-radius:9999px;font-size:12px;line-height:16px;font-weight:500;white-space:nowrap;background:var(--secondary);color:var(--foreground)}\
.acl .badge .i{width:12px;height:12px}.acl .badge.ok{background:var(--success-soft);color:var(--success)}\
.acl .badge.warn{background:color-mix(in srgb,var(--warning) 14%,transparent);color:var(--warning)}\
.acl .badge.pk{background:color-mix(in srgb,var(--primary) 10%,transparent);color:var(--primary)}\
.acl .badge.plain{background:transparent;border:1px solid var(--border);color:var(--muted-foreground)}\
.acl .hero{position:relative;isolation:isolate}\
.acl .hero::before{content:"";position:absolute;z-index:-1;inset:-60px -40px -40px;background:radial-gradient(closest-side at 82% 40%,color-mix(in srgb,var(--primary) 13%,transparent),transparent),radial-gradient(closest-side at 12% 80%,color-mix(in srgb,var(--primary) 6%,transparent),transparent);pointer-events:none}\
.acl .glass{background:var(--glass-surface);border:1px solid var(--glass-border);-webkit-backdrop-filter:blur(12px);backdrop-filter:blur(12px);border-radius:var(--r-2xl);box-shadow:var(--shadow-card)}\
.acl .hero-card{padding:24px;display:flex;flex-direction:column;gap:16px;outline:1px solid var(--border);outline-offset:-1px}\
.acl .flow{display:flex;align-items:stretch}\
.acl .flow.res{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%,190px),1fr));gap:12px}\
.acl .blk{flex:1 1 0;min-width:0;background:var(--grouped);border-radius:var(--r-2xl);padding:20px 16px;display:flex;flex-direction:column;align-items:center;text-align:center;gap:4px}\
.acl .blk.inv{background:color-mix(in srgb,var(--primary) 8%,var(--card))}\
.acl .blk .k{display:flex;align-items:center;justify-content:center;min-height:32px;font-size:12px;line-height:16px;font-weight:600;letter-spacing:.025em;text-transform:uppercase;color:var(--muted-foreground);text-wrap:balance}\
.acl .blk .v{font-size:36px;line-height:44px;font-weight:600;letter-spacing:-.03em;white-space:nowrap}\
.acl .flow.res .blk .v{font-size:32px;line-height:40px}\
.acl .blk .v.dash{color:var(--muted-foreground)}\
.acl .blk .s{font-size:13px;line-height:20px;color:var(--muted-foreground)}\
.acl .blk .s b{color:var(--foreground);font-weight:600}\
.acl .var{margin-top:8px;display:flex;align-items:center;justify-content:center;gap:6px;font-size:13px;font-weight:500;flex-wrap:wrap}\
.acl .var .i{width:14px;height:14px}.acl .neu{color:var(--muted-foreground)}.acl .dn{color:var(--destructive)}.acl .upc{color:var(--success)}\
.acl .conn{flex:none;width:96px;margin-inline:-10px;position:relative;z-index:1;align-self:center;min-height:60px;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:6px 18px 6px 12px;background:var(--foreground);color:#fff;text-align:center;clip-path:polygon(0 0,calc(100% - 14px) 0,100% 50%,calc(100% - 14px) 100%,0 100%);border-radius:8px 0 0 8px}\
.acl .conn .cv{font-size:13px;line-height:16px;font-weight:600;white-space:nowrap}.acl .conn .cv+.cs{margin-top:6px}\
.acl .conn .cs{display:flex;align-items:center;gap:3px;font-size:10px;line-height:12px;font-weight:600;white-space:nowrap}\
.acl .conn .cs .i{width:10px;height:10px;stroke-width:3}.acl .conn .cs .i.ok{color:var(--chip-ok)}.acl .conn .cs .i.bad{color:var(--chip-bad)}\
.acl .conn .cp{font-size:10px;line-height:12px;font-weight:500;white-space:nowrap;color:rgba(255,255,255,.75)}\
.acl .conn .lbl{font-size:9.5px;line-height:12px;font-weight:500;color:rgba(255,255,255,.75);margin-bottom:2px}\
.acl .note-line{display:flex;gap:8px;align-items:flex-start;font-size:12px;line-height:16px;color:var(--muted-foreground)}.acl .note-line .i{width:14px;height:14px;margin-top:1px}\
.acl .sec{display:flex;flex-direction:column;gap:16px}\
.acl .sec-head{display:flex;flex-wrap:wrap;align-items:flex-end;justify-content:space-between;gap:12px 16px}\
.acl .sec-head .t{display:flex;flex-direction:column;gap:4px;min-width:0}\
.acl .panel{background:var(--card);border:1px solid var(--border);border-radius:var(--r-2xl);box-shadow:var(--shadow-card);padding:24px}\
.acl .toolbar{display:flex;flex-wrap:wrap;align-items:center;gap:8px 12px;margin-bottom:16px}\
.acl .seg{display:inline-flex;flex-wrap:wrap;padding:3px;gap:2px;background:var(--secondary);border-radius:var(--r-md)}\
.acl .seg button{min-height:30px;padding-inline:12px;border:0;border-radius:8px;background:transparent;cursor:pointer;font-size:13px;font-weight:500;color:var(--muted-foreground)}\
.acl .seg button:hover{color:var(--foreground)}.acl .seg button[aria-pressed=true]{background:var(--card);color:var(--primary);box-shadow:var(--shadow-card)}\
.acl .btn-outline{display:inline-flex;align-items:center;gap:8px;height:32px;padding-inline:12px;cursor:pointer;border:1px solid var(--border);background:var(--card);border-radius:var(--r-md);font-size:13px;font-weight:500}\
.acl .btn-outline:hover{background:color-mix(in srgb,var(--primary) 10%,var(--card));color:var(--primary)}\
.acl .spacer{flex:1}\
.acl .legend{display:flex;flex-wrap:wrap;gap:4px 16px;font-size:13px}.acl .legend span{display:inline-flex;align-items:center;gap:8px}.acl .legend svg{width:24px;height:8px}\
.acl .chartbox{position:relative}.acl .chartbox svg{display:block;width:100%;touch-action:pan-y;overflow:visible}\
.acl .tip{position:absolute;z-index:5;pointer-events:none;min-width:168px;max-width:250px;background:var(--popover);border:1px solid var(--border);border-radius:var(--r-md);box-shadow:var(--shadow-modal);padding:8px 12px;font-size:12px;line-height:16px;display:flex;flex-direction:column;gap:4px}\
.acl .tip b{font-size:13px;font-weight:600}.acl .tip .r{display:flex;justify-content:space-between;gap:12px}\
.acl .tip .r span:first-child{display:inline-flex;align-items:center;gap:6px;color:var(--muted-foreground)}\
.acl .tip .sw{width:10px;height:3px;border-radius:2px;display:inline-block}.acl .tip .d{padding-top:4px;border-top:1px solid var(--border);font-weight:500}\
.acl .tablewrap{overflow-x:auto}\
.acl table{width:100%;border-collapse:collapse;font-variant-numeric:tabular-nums}\
.acl th{text-align:left;padding:8px 12px;font-size:12px;line-height:16px;font-weight:600;letter-spacing:.025em;text-transform:uppercase;color:var(--muted-foreground);background:color-mix(in srgb,var(--secondary) 60%,transparent)}\
.acl th.n,.acl td.n{text-align:right}.acl td{padding:8px 12px;border-top:1px solid color-mix(in srgb,var(--border) 60%,transparent)}\
.acl .fixed-note{display:flex;flex-wrap:nowrap;align-items:flex-start;gap:8px;margin-top:16px;padding-top:16px;border-top:1px solid var(--border);font-size:12px;line-height:16px;color:var(--muted-foreground)}\
.acl .funnels{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%,420px),1fr));gap:16px}\
.acl .funnel{background:var(--card);border:1px solid var(--border);border-radius:var(--r-2xl);box-shadow:var(--shadow-card);padding:24px;display:flex;flex-direction:column;gap:16px;min-width:0}\
.acl .funnel header{display:flex;align-items:baseline;justify-content:space-between;gap:8px;flex-wrap:wrap}\
.acl .steps{display:flex;flex-direction:column;align-items:center}\
.acl .bar{display:flex;align-items:center;justify-content:space-between;gap:12px;min-height:44px;padding:8px 14px;border-radius:var(--r-md);background:var(--grouped);border:1px solid var(--border);max-width:100%}\
.acl .bar .l{font-size:13px;line-height:16px;font-weight:500;min-width:0;text-align:left}\
.acl .bar .vv{font-size:16px;line-height:24px;font-weight:600;letter-spacing:-.01em;white-space:nowrap}\
.acl .bar.first{background:color-mix(in srgb,var(--primary) 10%,var(--card));border-color:color-mix(in srgb,var(--primary) 28%,var(--border))}.acl .bar.first .vv{color:var(--primary)}\
.acl .bar.miss{background:transparent;border:1px dashed var(--border);color:var(--muted-foreground)}\
.acl .bar.miss .vv{font-size:12.5px;font-weight:500;color:var(--muted-foreground);text-align:right;white-space:normal}\
.acl .link{min-height:28px;display:flex;align-items:center;justify-content:center;flex-wrap:wrap;gap:2px 6px;font-size:12px;line-height:16px;color:var(--muted-foreground);padding-block:2px;text-align:center}\
.acl .link b{color:var(--foreground);font-weight:600}.acl .link .i{width:12px;height:12px}\
.acl .callout{display:flex;gap:12px;align-items:flex-start;padding:16px;border-radius:var(--r-lg);background:color-mix(in srgb,var(--warning) 9%,var(--card));border:1px solid color-mix(in srgb,var(--warning) 20%,transparent)}\
.acl .callout .i{color:var(--warning);margin-top:2px}.acl .callout p b{font-weight:600}\
.acl .camps{display:flex;flex-direction:column;gap:8px}\
.acl .camp{display:grid;grid-template-columns:minmax(0,2.2fr) minmax(0,1.2fr) minmax(0,1.1fr) minmax(0,1.5fr);gap:16px;align-items:center;background:var(--card);border:1px solid var(--border);border-radius:var(--r-lg);box-shadow:var(--shadow-card);padding:16px}\
.acl .camp .nm{display:flex;flex-direction:column;gap:8px;min-width:0}\
.acl .camp .nm .t{display:flex;align-items:center;gap:6px;position:relative;font-weight:600;font-size:14px}.acl .camp .nm .t>span{overflow-wrap:anywhere}\
.acl .camp .tags{display:flex;flex-wrap:wrap;gap:4px}\
.acl .infobtn{flex:none;width:24px;height:24px;display:grid;place-items:center;cursor:pointer;padding:0;border:0;border-radius:9999px;background:transparent;color:var(--muted-foreground)}\
.acl .infobtn:hover,.acl .infobtn[aria-expanded=true]{background:color-mix(in srgb,var(--primary) 10%,transparent);color:var(--primary)}\
.acl .pop{position:absolute;left:0;top:calc(100% + 8px);z-index:25;width:320px;max-width:calc(100vw - 64px);background:var(--popover);border:1px solid var(--border);border-radius:var(--r-md);box-shadow:var(--shadow-modal);padding:12px;display:flex;flex-direction:column;gap:8px;font-weight:400;font-size:12px;line-height:16px}\
.acl .pop .cap{color:var(--muted-foreground);font-weight:600;letter-spacing:.025em;text-transform:uppercase}\
.acl .pop code{font-family:ui-monospace,SFMono-Regular,Menlo,monospace;font-size:12px;line-height:18px;overflow-wrap:anywhere;background:var(--grouped);border-radius:var(--r-sm);padding:8px;display:block}\
.acl .pop button{align-self:flex-start}\
.acl .camp .cell{display:flex;flex-direction:column;gap:4px;min-width:0}\
.acl .camp .cap{font-size:12px;line-height:16px;color:var(--muted-foreground)}\
.acl .camp .val{font-size:16px;line-height:24px;font-weight:600;letter-spacing:-.01em}\
.acl .camp .cmp{display:flex;flex-wrap:wrap;align-items:center;gap:4px 8px;font-size:12px;color:var(--muted-foreground)}\
.acl .meter{height:6px;border-radius:9999px;background:var(--secondary);overflow:hidden}.acl .meter i{display:block;height:100%;border-radius:9999px;background:var(--primary)}\
.acl .camps-foot{font-size:12px;line-height:16px;color:var(--muted-foreground)}\
.acl .two{display:grid;grid-template-columns:repeat(auto-fit,minmax(min(100%,420px),1fr));gap:16px}\
.acl .group{background:var(--card);border:1px solid var(--border);border-radius:var(--r-2xl);box-shadow:var(--shadow-card);padding:24px;display:flex;flex-direction:column;gap:16px;min-width:0}\
.acl .list{display:flex;flex-direction:column;gap:8px;margin:0;padding:0;list-style:none}\
.acl .list li{display:flex;gap:12px;align-items:flex-start;background:var(--grouped);border-radius:var(--r-lg);padding:12px 16px}\
.acl .list li .body{display:flex;flex-direction:column;gap:4px;min-width:0}.acl .list li .body b{font-weight:600}.acl .list li .body span{color:var(--muted-foreground)}\
.acl .circ{width:20px;height:20px;flex:none;border-radius:9999px;border:2px solid var(--border);background:var(--card)}\
.acl footer{border-top:1px solid color-mix(in srgb,var(--foreground) 12%,transparent);padding-top:16px;display:flex;flex-wrap:wrap;gap:4px 24px;justify-content:space-between;font-size:12px;line-height:16px;color:var(--muted-foreground)}\
.acl .state{text-align:center;padding:80px 20px;color:var(--muted-foreground)}.acl .state h2{color:var(--foreground);margin-bottom:6px}\
.acl .spin{width:28px;height:28px;border:2.5px solid var(--border);border-top-color:var(--primary);border-radius:50%;animation:aclsp .8s linear infinite;margin:0 auto 16px}\
@keyframes aclsp{to{transform:rotate(360deg)}}\
@container (max-width:980px){.acl .flow:not(.res){flex-direction:column;align-items:stretch}\
.acl .flow:not(.res) .conn{width:100%;max-width:150px;align-self:center;margin-inline:0;margin-block:-8px;min-height:68px;padding:6px 16px 16px;border-radius:8px 8px 0 0;clip-path:polygon(0 0,100% 0,100% calc(100% - 12px),50% 100%,0 calc(100% - 12px))}}\
@container (max-width:760px){.acl main{gap:32px}.acl .hero-card,.acl .panel,.acl .funnel,.acl .group{padding:16px}.acl .bar{width:100%!important}\
.acl .camp{grid-template-columns:1fr 1fr}.acl .camp .nm{grid-column:1 / -1}.acl .filters,.acl .fwrap{width:100%;align-items:stretch}.acl .pbox,.acl .period{width:100%}.acl .period{justify-content:space-between}\
.acl .menu{left:0;right:0;width:auto;max-width:none}.acl .blk .v{font-size:30px;line-height:38px}}\
@media (prefers-reduced-motion:reduce){.acl *{transition:none!important;animation:none!important}}\
';
  var st = document.getElementById('acl-style');
  if (!st) { st = document.createElement('style'); st.id = 'acl-style'; document.head.appendChild(st); }
  st.textContent = css;
  try { document.documentElement.style.background = FUNDO; document.body.style.background = FUNDO; document.body.style.margin = '0'; } catch (e) {}
}
var SPRITE = '<svg width="0" height="0" style="position:absolute" aria-hidden="true"><defs>' +
  '<symbol id="i-cal" viewBox="0 0 24 24"><rect x="3" y="4" width="18" height="18" rx="2"/><path d="M16 2v4M8 2v4M3 10h18"/></symbol>' +
  '<symbol id="i-chev" viewBox="0 0 24 24"><path d="m6 9 6 6 6-6"/></symbol>' +
  '<symbol id="i-check" viewBox="0 0 24 24"><path d="M20 6 9 17l-5-5"/></symbol>' +
  '<symbol id="i-alert" viewBox="0 0 24 24"><path d="m21.73 18-8-14a2 2 0 0 0-3.48 0l-8 14A2 2 0 0 0 4 21h16a2 2 0 0 0 1.73-3"/><path d="M12 9v4M12 17h.01"/></symbol>' +
  '<symbol id="i-info" viewBox="0 0 24 24"><circle cx="12" cy="12" r="10"/><path d="M12 16v-4M12 8h.01"/></symbol>' +
  '<symbol id="i-table" viewBox="0 0 24 24"><rect width="18" height="18" x="3" y="3" rx="2"/><path d="M3 9h18M3 15h18M12 3v18"/></symbol>' +
  '<symbol id="i-chart" viewBox="0 0 24 24"><path d="M3 3v16a2 2 0 0 0 2 2h16"/><path d="m19 9-5 5-4-4-3 3"/></symbol>' +
  '<symbol id="i-down" viewBox="0 0 24 24"><path d="M12 5v14M19 12l-7 7-7-7"/></symbol>' +
  '<symbol id="i-up" viewBox="0 0 24 24"><path d="M12 19V5M5 12l7-7 7 7"/></symbol>' +
  '<symbol id="i-copy" viewBox="0 0 24 24"><rect width="14" height="14" x="8" y="8" rx="2"/><path d="M4 16c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2h10c1.1 0 2 .9 2 2"/></symbol>' +
  '<symbol id="i-pin" viewBox="0 0 24 24"><path d="M20 10c0 6-8 12-8 12s-8-6-8-12a8 8 0 0 1 16 0Z"/><circle cx="12" cy="10" r="3"/></symbol>' +
  '</defs></svg>';

/* ============================== CLIENTE: ALTURA NO IFRAME ============================== */
function reportHeight() {
  try {
    var h = Math.ceil(ROOT.getBoundingClientRect().height);
    if (window.frameElement) window.frameElement.style.height = h + 'px';
    if (window.parent && window.parent !== window) window.parent.postMessage({ type: 'dash-height', height: h }, '*');
  } catch (e) {}
}

/* ============================== CLIENTE: PERÍODO ============================== */
var PRESETS = [['7', '7 dias'], ['14', '14 dias'], ['30', '30 dias'], ['mtd', 'Este mês'], ['lastmonth', 'Mês passado']];
function periodFor(k) { var old = STATE.preset; STATE.preset = k; var p = resolvePeriod(); STATE.preset = old; return p; }
function presetLabel() { var p = PRESETS.filter(function (x) { return x[0] === STATE.preset; })[0]; return p ? p[1] : 'Período escolhido'; }
function rangeTxt(a, b) { return fmtD(a) + ' a ' + fmtD(b); }

/* ============================== CLIENTE: BLOCOS DE NEGÓCIO ============================== */
function chg(c, p) { if (!ok(c) || !ok(p) || p === 0) return null; var r = c / p - 1; return { v: r, p: Math.round(Math.abs(r) * 100), dir: r >= 0 ? 'up' : 'down' }; }
function varLine(c, p, prevTxt, mode) {
  // mode: 'neutral' | 'more' (mais é melhor) | 'less' (menos é melhor)
  var d = chg(c, p); if (!d) return '';
  if (d.p === 0) return '<span class="var neu">Igual ao período anterior</span>';
  var good = mode === 'less' ? d.dir === 'down' : d.dir === 'up';
  var cls = mode === 'neutral' ? 'neu' : (good ? 'upc' : 'dn');
  return '<span class="var" title="Comparado com a média por dia do período anterior">' + ic(d.dir, cls) + '<span class="num ' + cls + '">' + d.p + '%</span><span class="neu num">(' + prevTxt + ')</span></span>';
}
function blk(k, val, sub, varHtml, cls) {
  return '<div class="blk' + (cls ? ' ' + cls : '') + '"><span class="k">' + k + '</span><span class="v num' + (val === '—' ? ' dash' : '') + '">' + val + '</span><span class="s num">' + sub + '</span>' + (varHtml || '') + '</div>';
}
function connHtml(lbl, val, d, good, ptext) {
  return '<div class="conn">' + (lbl ? '<span class="lbl">' + lbl + '</span>' : '') + '<span class="cv num">' + val + '</span>' +
    (d ? '<span class="cs">' + ic(d.dir, good ? 'ok' : 'bad') + '<span class="num">' + d + '</span><span class="sr">, ' + (good ? 'melhor' : 'pior') + ' que no período anterior</span></span>' : '') +
    (ptext ? '<span class="cp num">' + ptext + '</span>' : '') + '</div>';
}
function connCost(cost, pCost) {
  var d = chg(cost, pCost);
  if (!d || d.p === 0) return connHtml('custo por contato', money(cost), null, true, pCost ? '(' + money(pCost) + ')' : '');
  return '<div class="conn"><span class="lbl">custo por contato</span><span class="cv num">' + money(cost) + '</span><span class="cs">' + ic(d.dir, d.dir === 'down' ? 'ok' : 'bad') + '<span class="num">' + d.p + '%</span></span><span class="cp num">(' + money(pCost) + ')</span></div>';
}
function connRate(lbl, rate, prev) {
  if (!ok(rate)) return '<div class="conn"><span class="cv">—</span></div>';
  var h = '<div class="conn"><span class="lbl">' + lbl + '</span><span class="cv num">' + pct(rate) + '</span>';
  if (ok(prev)) {
    var diff = (rate - prev) * 100, a = Math.abs(diff);
    h += '<span class="cs">' + ic(diff >= 0 ? 'up' : 'down', diff >= 0 ? 'ok' : 'bad') + '<span class="num">' + nf(a, a < 10 ? 1 : 0) + ' p.p.</span></span><span class="cp num">(' + pct(prev) + ')</span>';
  }
  return h + '</div>';
}
function bizFunnels() { return funnelsPresent().filter(function (f) { return f !== 'outros' && f !== 'trafego' && RESULT(f); }); }
function salesMeasured() { return STATE.crm.some(function (c) { return c.sale; }); }

function heroHTML(per) {
  var fs = bizFunnels(), L1 = per.len, L0 = per.pLen;
  var tot = agg(filtered(per.from, per.to)), totP = agg(filtered(per.pFrom, per.pTo));
  var invBlk = blk('Investido', money(tot.spend), money(tot.spend / L1) + ' por dia', varLine(tot.spend / L1, totP.spend / L0, money(totP.spend / L0), 'neutral'), 'inv');
  var notes = [];
  if (!fs.length) return { html: '<div class="flow res">' + invBlk + '</div>', notes: ['Nenhuma campanha com resultado de conversão no período.'] };
  var salesMode = fs.some(isSale);
  if (salesMode) {
    var h = invBlk;
    fs.forEach(function (f) {
      var F = buildFunnel(f, per), rk = F.resKey, c = F.cur[rk], p = F.prev[rk];
      var cpr = c > 0 ? F.cur.spend / c : null, ppr = p > 0 ? F.prev.spend / p : null;
      var sub = cpr ? '<b>' + money(cpr) + '</b> cada' : 'sem resultado no período';
      if (isSale(f) && F.cur.revenue > 0 && (!SALE_TYPES[f] || SALE_TYPES[f].receita)) sub += ' · ' + money(F.cur.revenue) + ' vendidos';
      var vl = cpr && ppr ? varLine(cpr, ppr, money(ppr), 'less').replace('<span class="var"', '<span class="var" title="Custo por resultado comparado com o período anterior"') : varLine(c / L1, p / L0, perDay(p / L0) + '/dia', 'more');
      h += blk(FNAME(f), count(c), sub, vl);
    });
    var outros = agg(filtered(per.from, per.to, function (r) { return r.funnel === 'outros' || r.funnel === 'trafego'; })).spend;
    if (outros > 0) notes.push('Inclui ' + money(outros) + ' em campanhas de alcance e engajamento, que não geram resultados contáveis.');
    notes.push('Nos resultados, a variação compara o custo de cada um com o período anterior.');
    return { html: '<div class="flow res">' + h + '</div>', notes: notes };
  }
  // modo contatos: Investido > Contatos > Qualificados > Vendas
  var FF = fs.map(function (f) { return buildFunnel(f, per); });
  var cS = 0, cN = 0, pS = 0, pN = 0;
  FF.forEach(function (F) { cS += F.cur.spend; cN += F.cur[F.resKey]; pS += F.prev.spend; pN += F.prev[F.resKey]; });
  var cost = cN > 0 ? cS / cN : null, pCost = pN > 0 ? pS / pN : null;
  var h2 = invBlk + connCost(cost, pCost);
  var tipos = fs.map(function (f) { return f === 'whatsapp' ? 'conversas no WhatsApp' : RNAME(f); }).join(' e ');
  h2 += blk('Contatos gerados', count(cN), perDay(cN / L1) + ' por dia', varLine(cN / L1, pN / L0, perDay(pN / L0) + '/dia', 'more'));
  var QF = FF.filter(function (F) { return F.crmStatus; });
  if (QF.length) {
    var q = 0, qp = 0, base = 0, baseP = 0;
    QF.forEach(function (F) { q += F.crmCur.qual; qp += F.crmPrev.qual; base += F.cur[F.resKey]; baseP += F.prev[F.resKey]; });
    var parcial = QF.length < FF.length;
    h2 += connRate('qualificados', base > 0 ? q / base : null, baseP > 0 ? qp / baseP : null);
    h2 += blk('Contatos qualificados', count(q), parcial ? 'Só ' + QF.map(function (F) { return FNAME(F.f).toLowerCase(); }).join(' e ') : perDay(q / L1) + ' por dia',
      parcial ? '<span class="var"><span class="badge warn">' + ic('info') + 'Medição parcial</span></span>' : varLine(q / L1, qp / L0, perDay(qp / L0) + '/dia', 'more'));
    if (parcial) notes.push('A qualificação só é registrada para ' + QF.map(function (F) { return FNAME(F.f).toLowerCase(); }).join(' e ') + '. A taxa usa só esses contatos.');
    if (salesMeasured()) {
      var v = 0, vp = 0; QF.forEach(function (F) { v += F.crmCur.sale; vp += F.crmPrev.sale; });
      h2 += connRate('compraram', q > 0 ? v / q : null, qp > 0 ? vp / qp : null);
      h2 += blk('Vendas', count(v), perDay(v / L1) + ' por dia', varLine(v / L1, vp / L0, perDay(vp / L0) + '/dia', 'more'));
    } else {
      h2 += '<div class="conn"><span class="cv">—</span></div>' + blk('Vendas', '—', 'Ainda não registradas', '<span class="var"><span class="badge warn">' + ic('info') + 'Não medido</span></span>');
      notes.push('As vendas ainda não são registradas, por isso o custo por venda não aparece.');
    }
  } else {
    h2 += '<div class="conn"><span class="cv">—</span></div>' + blk('Contatos qualificados', '—', 'Ainda não registrados', '<span class="var"><span class="badge warn">' + ic('info') + 'Não medido</span></span>');
    h2 += '<div class="conn"><span class="cv">—</span></div>' + blk('Vendas', '—', 'Ainda não registradas', '<span class="var"><span class="badge warn">' + ic('info') + 'Não medido</span></span>');
    notes.push('Qualificação e vendas ainda não são registradas. Com esse registro, mostramos aqui quanto custa cada venda.');
  }
  var outros2 = tot.spend - cS;
  if (outros2 > 1) notes.unshift('Contatos somam ' + tipos + '. O custo por contato considera só as campanhas que geram contato (' + money(cS) + ' de ' + money(tot.spend) + ').');
  return { html: '<div class="flow">' + h2 + '</div>', notes: notes };
}

/* ============================== CLIENTE: EVOLUÇÃO POR MÊS ============================== */
var MES = ['Jan', 'Fev', 'Mar', 'Abr', 'Mai', 'Jun', 'Jul', 'Ago', 'Set', 'Out', 'Nov', 'Dez'];
var MESL = ['Janeiro', 'Fevereiro', 'Março', 'Abril', 'Maio', 'Junho', 'Julho', 'Agosto', 'Setembro', 'Outubro', 'Novembro', 'Dezembro'];
var EVO = { metric: 'custo', table: false, f: null };
function evoGroups() {
  var fs = bizFunnels();
  if (!fs.length) return [];
  if (!fs.some(isSale)) return [{ k: 'contatos', n: 'Contatos', one: 'contato', many: 'contatos', fs: fs }];
  return fs.map(function (f) { return { k: f, n: FNAME(f), one: RNAME(f, false), many: RNAME(f), fs: [f] }; });
}
function monthSeries(G, year) {
  var last = lastDataDate() || todayISO(), out = { custo: [], dia: [], spend: [], n: [] };
  for (var m = 0; m < 12; m++) {
    var from = year + '-' + p2(m + 1) + '-01', to = monthEnd(from);
    if (from > last) { out.custo.push(null); out.dia.push(null); out.spend.push(null); out.n.push(null); continue; }
    var end = to > last ? last : to, days = daysBetween(from, end) + 1;
    var sp = 0, n = 0;
    STATE.rows.forEach(function (r) {
      if (r.date < from || r.date > end || G.fs.indexOf(r.funnel) < 0 || !unitOk(r.campaign)) return;
      sp += r.spend; n += r[RESULT(r.funnel)] || 0;
    });
    var has = sp > 0 || n > 0;
    out.spend.push(has ? sp : null); out.n.push(has ? n : null);
    out.custo.push(has && n > 0 ? sp / n : null);
    out.dia.push(has ? n / days : null);
  }
  return out;
}
function axisOf(maxVal) {
  if (!(maxVal > 0)) return { step: 1, max: 4 };
  var p = Math.pow(10, Math.floor(Math.log10(maxVal / 4))), cand = [1, 2, 2.5, 5, 10].map(function (x) { return x * p; });
  var step = cand.filter(function (x) { return x >= maxVal / 4; })[0] || cand[4];
  return { step: step, max: Math.ceil(maxVal / step) * step };
}
function evoHTML() {
  var GS = evoGroups(); if (!GS.length) return '';
  if (!EVO.f || !GS.some(function (g) { return g.k === EVO.f; })) EVO.f = GS[0].k;
  var G = GS.filter(function (g) { return g.k === EVO.f; })[0];
  var h = '<section class="sec" aria-labelledby="h-evo"><div class="sec-head"><div class="t"><span class="section-label">Evolução do trabalho</span><h2 id="h-evo">Este ano contra o ano passado</h2></div>' +
    '<span class="badge plain">' + ic('info') + 'Fixo: não muda com o período</span></div><div class="panel"><div class="toolbar">';
  if (GS.length > 1) h += '<div class="seg" role="group" aria-label="Resultado">' + GS.map(function (g) { return '<button type="button" data-evof="' + g.k + '" aria-pressed="' + (g.k === EVO.f) + '">' + esc(g.n) + '</button>'; }).join('') + '</div>';
  h += '<div class="seg" role="group" aria-label="Métrica"><button type="button" data-evom="custo" aria-pressed="' + (EVO.metric === 'custo') + '">Custo por ' + esc(G.one) + '</button><button type="button" data-evom="dia" aria-pressed="' + (EVO.metric === 'dia') + '">' + esc(cap(G.many)) + ' por dia</button></div>' +
    '<div class="spacer"></div><button class="btn-outline" type="button" id="aclView">' + ic(EVO.table ? 'chart' : 'table') + '<span>' + (EVO.table ? 'Ver gráfico' : 'Ver tabela') + '</span></button></div>' +
    '<div class="legend" id="aclLegend"' + (EVO.table ? ' hidden' : '') + '></div><div class="chartbox" id="aclPlot" style="margin-top:12px"' + (EVO.table ? ' hidden' : '') + '></div><div class="tablewrap" id="aclTbl"' + (EVO.table ? '' : ' hidden') + '></div>' +
    '<div class="fixed-note" id="aclEvoNote"></div></div></section>';
  return h;
}
function evoData() {
  var GS = evoGroups(), G = GS.filter(function (g) { return g.k === EVO.f; })[0] || GS[0];
  var last = lastDataDate() || todayISO(), Y = +last.slice(0, 4), cm = +last.slice(5, 7) - 1;
  var A = monthSeries(G, Y), B = monthSeries(G, Y - 1);
  var a = A[EVO.metric], b = B[EVO.metric];
  var fmt = EVO.metric === 'custo' ? money : function (v) { return perDay(v); };
  var good = function (p) { return EVO.metric === 'custo' ? p <= 0 : p >= 0; };
  var delta = function (p) {
    var q = Math.abs(p);
    if (EVO.metric === 'custo') return p <= 0 ? q + '% mais barato que em ' + (Y - 1) : q + '% mais caro que em ' + (Y - 1);
    return p >= 0 ? q + '% a mais que em ' + (Y - 1) : q + '% a menos que em ' + (Y - 1);
  };
  return { G: G, Y: Y, cm: cm, a: a, b: b, fmt: fmt, good: good, delta: delta, hasPrev: b.some(ok), label: EVO.metric === 'custo' ? 'Custo por ' + G.one : cap(G.many) + ' por dia' };
}
function drawEvo() {
  var box = $('#aclPlot'); if (!box || box.hidden) return;
  var E = evoData(), a = E.a, b = E.b;
  var vals = a.concat(b).filter(ok), ax = axisOf((vals.length ? Math.max.apply(null, vals) : 1) * 1.08);
  var w = Math.max(280, box.clientWidth || 800), hgt = w < 520 ? 250 : 320, m = { t: 16, r: 44, b: 28, l: w < 520 ? 52 : 64 };
  var iw = w - m.l - m.r, ih = hgt - m.t - m.b;
  var X = function (i) { return m.l + iw * (i / 11); }, Yp = function (v) { return m.t + ih * (1 - v / ax.max); };
  var tick = EVO.metric === 'custo' ? function (t) { return t === 0 ? '0' : moneyShort(t); } : function (t) { return nf(t, t < 10 && t % 1 ? 1 : 0); };
  var s = '<svg viewBox="0 0 ' + w + ' ' + hgt + '" width="' + w + '" height="' + hgt + '" role="img" aria-label="' + esc(E.label) + ' por mês. Use Ver tabela para ler os valores.">';
  for (var t = 0; t <= ax.max + 1e-9; t += ax.step) {
    s += '<line x1="' + m.l + '" x2="' + (w - m.r) + '" y1="' + Yp(t) + '" y2="' + Yp(t) + '" stroke="var(--border)"' + (t === 0 ? '' : ' stroke-dasharray="2 4"') + '/>' +
      '<text x="' + (m.l - 8) + '" y="' + (Yp(t) + 4) + '" text-anchor="end" font-size="11" fill="var(--muted-foreground)" style="font-variant-numeric:tabular-nums">' + tick(t) + '</text>';
  }
  MES.forEach(function (n, i) { s += '<text x="' + X(i) + '" y="' + (hgt - 8) + '" text-anchor="middle" font-size="11" fill="var(--muted-foreground)">' + n + '</text>'; });
  function path(arr, upto) {
    var d = '', on = false;
    arr.forEach(function (v, i) { if (upto != null && i > upto) return; if (!ok(v)) { on = false; return; } d += (on ? 'L' : 'M') + X(i).toFixed(1) + ' ' + Yp(v).toFixed(1) + ' '; on = true; });
    return d;
  }
  if (E.hasPrev) s += '<path d="' + path(b) + '" fill="none" stroke="var(--muted-foreground)" stroke-width="2" stroke-dasharray="5 4" stroke-linecap="round" stroke-linejoin="round"/>';
  var cm = E.cm, solidTo = cm - 1;
  var solid = path(a, solidTo);
  // área sob os meses fechados
  var idx = []; a.forEach(function (v, i) { if (ok(v) && i <= solidTo) idx.push(i); });
  if (idx.length > 1) s += '<path d="' + solid + 'L' + X(idx[idx.length - 1]) + ' ' + Yp(0) + ' L' + X(idx[0]) + ' ' + Yp(0) + ' Z" fill="var(--primary)" opacity="0.08"/>';
  s += '<path d="' + solid + '" fill="none" stroke="var(--primary)" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>';
  if (ok(a[cm])) {
    if (cm > 0 && ok(a[cm - 1])) s += '<path d="M' + X(cm - 1) + ' ' + Yp(a[cm - 1]) + ' L' + X(cm) + ' ' + Yp(a[cm]) + '" fill="none" stroke="var(--primary)" stroke-width="2.5" stroke-dasharray="1 5" stroke-linecap="round"/>';
    s += '<circle cx="' + X(cm) + '" cy="' + Yp(a[cm]) + '" r="5" fill="var(--card)" stroke="var(--primary)" stroke-width="2.5"/>';
    s += '<text x="' + (X(cm) + 10) + '" y="' + (Yp(a[cm]) + 4) + '" font-size="12" font-weight="600" fill="var(--foreground)">' + E.Y + '</text>';
  }
  a.forEach(function (v, i) { if (ok(v) && i < cm) s += '<circle cx="' + X(i) + '" cy="' + Yp(v) + '" r="3" fill="var(--primary)"/>'; });
  if (E.hasPrev) { var li = -1; b.forEach(function (v, i) { if (ok(v)) li = i; }); if (li > -1) s += '<text x="' + (X(li) + 8) + '" y="' + (Yp(b[li]) + 4) + '" font-size="12" font-weight="600" fill="var(--muted-foreground)">' + (E.Y - 1) + '</text>'; }
  s += '<line id="aclX" y1="' + m.t + '" y2="' + (m.t + ih) + '" stroke="var(--muted-foreground)" visibility="hidden"/><circle id="aclH0" r="4.5" fill="var(--card)" stroke="var(--muted-foreground)" stroke-width="2" visibility="hidden"/><circle id="aclH1" r="4.5" fill="var(--primary)" stroke="var(--card)" stroke-width="2" visibility="hidden"/>' +
    '<rect id="aclHit" x="' + (m.l - 12) + '" y="' + m.t + '" width="' + (iw + 24) + '" height="' + ih + '" fill="transparent"/></svg><div class="tip" id="aclTip" hidden></div>';
  box.innerHTML = s;
  var svg = box.querySelector('svg'), tip = $('#aclTip'), cross = $('#aclX'), h0 = $('#aclH0'), h1 = $('#aclH1');
  function show(e) {
    var r = svg.getBoundingClientRect(), px = (e.clientX - r.left) * (w / (r.width || w)), i = Math.max(0, Math.min(11, Math.round((px - m.l) / iw * 11)));
    var v1 = a[i], v0 = b[i];
    if (!ok(v1) && !ok(v0)) return hide();
    cross.setAttribute('x1', X(i)); cross.setAttribute('x2', X(i)); cross.setAttribute('visibility', 'visible');
    if (ok(v0)) { h0.setAttribute('cx', X(i)); h0.setAttribute('cy', Yp(v0)); h0.setAttribute('visibility', 'visible'); } else h0.setAttribute('visibility', 'hidden');
    if (ok(v1)) { h1.setAttribute('cx', X(i)); h1.setAttribute('cy', Yp(v1)); h1.setAttribute('visibility', 'visible'); } else h1.setAttribute('visibility', 'hidden');
    var html = '<b>' + MESL[i] + '</b>';
    if (ok(v1)) html += '<div class="r"><span><i class="sw" style="background:var(--primary)"></i>' + E.Y + (i === E.cm ? ' (em andamento)' : '') + '</span><b class="num">' + E.fmt(v1) + '</b></div>';
    if (ok(v0)) html += '<div class="r"><span><i class="sw" style="background:var(--muted-foreground)"></i>' + (E.Y - 1) + '</span><b class="num">' + E.fmt(v0) + '</b></div>';
    if (ok(v1) && ok(v0) && v0 > 0) { var p = Math.round((v1 / v0 - 1) * 100); html += '<div class="d" style="color:' + (E.good(p) ? 'var(--success)' : 'var(--warning)') + '">' + E.delta(p) + '</div>'; }
    tip.innerHTML = html; tip.hidden = false;
    var tw = tip.offsetWidth, left = X(i) + 14; if (left + tw > w) left = X(i) - tw - 14;
    tip.style.left = Math.max(0, left) + 'px'; tip.style.top = (m.t + 4) + 'px';
  }
  function hide() { tip.hidden = true; [cross, h0, h1].forEach(function (x) { x.setAttribute('visibility', 'hidden'); }); }
  var hit = $('#aclHit');
  hit.addEventListener('pointermove', show); hit.addEventListener('pointerdown', show); hit.addEventListener('pointerleave', hide);
}
function fillEvo() {
  if (!$('#aclEvoNote')) return;
  var E = evoData();
  $('#aclLegend').innerHTML = '<span><svg viewBox="0 0 24 8"><line x1="0" y1="4" x2="24" y2="4" stroke="var(--primary)" stroke-width="2.5" stroke-linecap="round"/></svg>' + E.Y + '</span>' +
    (E.hasPrev ? '<span><svg viewBox="0 0 24 8"><line x1="0" y1="4" x2="24" y2="4" stroke="var(--muted-foreground)" stroke-width="2" stroke-dasharray="5 4" stroke-linecap="round"/></svg>' + (E.Y - 1) + '</span>' : '') +
    '<span class="hint">' + MESL[E.cm] + ' de ' + E.Y + ' está em andamento</span>';
  var rows = MES.map(function (n, i) {
    var v1 = E.a[i], v0 = E.b[i], d = '—';
    if (ok(v1) && ok(v0) && v0 > 0) { var p = Math.round((v1 / v0 - 1) * 100); d = (p > 0 ? '+' : '') + p + '%'; }
    if (!ok(v1) && !ok(v0)) return '';
    return '<tr><td>' + MESL[i] + (i === E.cm ? ' (em andamento)' : '') + '</td>' + (E.hasPrev ? '<td class="n">' + (ok(v0) ? E.fmt(v0) : '—') + '</td>' : '') + '<td class="n">' + (ok(v1) ? E.fmt(v1) : '—') + '</td>' + (E.hasPrev ? '<td class="n">' + d + '</td>' : '') + '</tr>';
  }).join('');
  $('#aclTbl').innerHTML = '<table><thead><tr><th>Mês</th>' + (E.hasPrev ? '<th class="n">' + (E.Y - 1) + '</th>' : '') + '<th class="n">' + E.Y + '</th>' + (E.hasPrev ? '<th class="n">Variação</th>' : '') + '</tr></thead><tbody>' + rows + '</tbody></table>';
  var first = null; STATE.rows.forEach(function (r) { if (!first || r.date < first) first = r.date; });
  $('#aclEvoNote').innerHTML = ic('info') + '<span>' + (E.hasPrev ? 'Meses sem investimento ficam em branco.' : 'Ainda não há dados de ' + (E.Y - 1) + ' nas planilhas: a comparação aparece quando o histórico existir.') +
    (first ? ' Histórico disponível desde ' + fmtD(first, 1) + '.' : '') + ' O mês atual aparece pontilhado porque ainda está em andamento.</span>';
  drawEvo();
}

/* ============================== CLIENTE: FUNIL ============================== */
var WIDTHS = [100, 92, 84, 76, 68, 60, 54];
function funnelsHTML(per) {
  var fs = bizFunnels(); if (!fs.length) return '';
  var missing = false;
  var cards = fs.map(function (f) {
    var F = buildFunnel(f, per), h = '<div class="funnel"><header><h3>' + esc(FNAME(f)) + '</h3><span class="hint num">Investido: ' + money(F.cur.spend) + '</span></header><div class="steps">';
    F.stages.forEach(function (s, i) {
      if (s.missing) missing = true;
      if (i > 0) {
        if (s.missing || F.stages[i - 1].missing || !ok(s.rate)) h += '<div class="link" aria-hidden="true">' + ic('down') + '</div>';
        else {
          var extra = '';
          if (s.final || s.key === 'crm' || s.key === 'qual' || s.key === 'sale') extra = '<span class="num">· ' + perDay(s.cur / per.len) + ' por dia</span>';
          h += '<div class="link">' + ic('down') + '<span><b class="num">' + pct(s.rate) + '</b> seguiram</span>' + (ok(s.pRate) ? '<span class="num">· antes ' + pct(s.pRate) + '</span>' : '') + extra + '</div>';
        }
      }
      var lbl = String(s.label).replace(/<[^>]+>/g, '');
      if (s.missing) h += '<div class="bar miss" style="width:' + WIDTHS[Math.min(i, 6)] + '%"><span class="l">' + esc(lbl) + '</span><span class="vv">' + missingLine(s.missing) + '</span></div>';
      else h += '<div class="bar' + (i === 0 ? ' first' : '') + '" style="width:' + WIDTHS[Math.min(i, 6)] + '%"><span class="l">' + esc(lbl) + '</span><span class="vv num">' + count(s.cur) + '</span></div>';
    });
    return h + '</div></div>';
  }).join('');
  var co = missing ? '<div class="callout">' + ic('alert') + '<p><b>Falta medir o que acontece depois do contato.</b> Se a equipe registrar quem foi qualificado e quem comprou, mostramos aqui quanto custa cada venda.</p></div>' : '';
  return '<section class="sec" aria-labelledby="h-funil"><div class="sec-head"><div class="t"><span class="section-label">Do anúncio ao resultado</span><h2 id="h-funil">Para onde vai o seu investimento</h2></div><p class="hint">As barras mostram as etapas, não estão em escala.</p></div><div class="funnels">' + cards + '</div>' + co + '</section>';
}

/* ============================== CLIENTE: CAMPANHAS ============================== */
var CAMP = { all: false, list: [] };
function campRows(per) {
  function by(rows) { var m = {}; rows.forEach(function (r) { var c = m[r.key] || (m[r.key] = { key: r.key, name: r.campaign, plat: r.platform, f: r.funnel, spend: 0, res: 0 }); c.spend += r.spend; var rk = RESULT(r.funnel); c.res += rk ? r[rk] : 0; }); return m; }
  var now = by(filtered(per.from, per.to)), prev = by(filtered(per.pFrom, per.pTo));
  return Object.keys(now).map(function (k) {
    var n = now[k], p = prev[k];
    return { name: n.name, nick: nick(n.name), plat: n.plat, f: n.f, inv: n.spend, res: n.res, cost: n.res > 0 ? n.spend / n.res : null, prev: p && p.res > 0 ? p.spend / p.res : null, nova: !p || p.spend <= 0 };
  }).filter(function (c) { return c.inv > 0; }).sort(function (a, b) { return b.inv - a.inv; });
}
function campsHTML(per) {
  var rows = campRows(per); CAMP.list = rows;
  if (!rows.length) return '';
  var show = CAMP.all ? rows : rows.slice(0, 5), maxInv = rows[0].inv;
  var h = show.map(function (c, idx) {
    var rk = RESULT(c.f), cmp;
    if (!rk) cmp = '<span>Campanha de alcance</span>';
    else if (c.cost && c.prev) {
      var p = Math.round((1 - c.cost / c.prev) * 100);
      cmp = (p > 0 ? '<span class="badge ok">' + ic('check') + p + '% mais barato</span>' : p < 0 ? '<span class="badge warn">' + ic('alert') + Math.abs(p) + '% mais caro</span>' : '<span class="badge">Igual</span>') + '<span class="num">Antes: ' + money(c.prev) + '</span>';
    } else if (c.nova) cmp = '<span class="badge pk">Nova</span><span>Sem histórico</span>';
    else cmp = '<span>Sem resultado no período anterior</span>';
    return '<article class="camp"><div class="nm"><div class="t"><span>' + esc(c.nick) + '</span>' +
      '<button type="button" class="infobtn" data-i="' + idx + '" aria-expanded="false" aria-label="Ver o nome original da campanha" title="Ver o nome original">' + ic('info') + '</button>' +
      '<div class="pop" hidden><span class="cap">Nome original</span><code>' + esc(c.name) + '</code><button type="button" class="btn-outline" data-copy="' + idx + '">' + ic('copy') + '<span>Copiar nome</span></button></div></div>' +
      '<span class="tags"><span class="badge plain">' + (c.plat === 'google' ? 'Google' : 'Meta') + '</span><span class="badge">' + esc(FNAME(c.f)) + '</span>' + (c.nova ? '<span class="badge pk">Nova</span>' : '') + '</span></div>' +
      '<div class="cell"><span class="cap">Investido</span><span class="val num">' + money(c.inv) + '</span><div class="meter" aria-hidden="true"><i style="width:' + Math.max(2, Math.round(c.inv / maxInv * 100)) + '%"></i></div></div>' +
      '<div class="cell"><span class="cap">Resultado</span><span class="val num">' + (rk ? count(c.res) : '—') + '</span><span class="cap">' + (rk ? RNAME(c.f, Math.round(c.res) !== 1) : 'alcance e engajamento') + '</span></div>' +
      '<div class="cell"><span class="cap">Custo por resultado</span><span class="val num">' + (rk ? money(c.cost) : '—') + '</span><div class="cmp">' + cmp + '</div></div></article>';
  }).join('');
  var tot = rows.reduce(function (a, c) { return a + c.inv; }, 0), top = rows.slice(0, 5).reduce(function (a, c) { return a + c.inv; }, 0);
  var foot = CAMP.all ? 'Todas as ' + rows.length + ' campanhas do período.' : (rows.length > 5 ? 'As cinco campanhas somam ' + money(top) + ' dos ' + money(tot) + ' investidos.' : 'Todas as campanhas do período.');
  foot += ' Custos comparados com o período anterior quando a campanha já existia. O ícone ao lado do nome mostra o nome original.';
  return '<section class="sec" aria-labelledby="h-camp"><div class="sec-head"><div class="t"><span class="section-label">Campanhas</span><h2 id="h-camp">Onde o investimento rendeu mais</h2></div><p class="hint">' + (CAMP.all ? 'Todas as campanhas' : 'As com maior investimento') + ' no período</p></div>' +
    '<div class="camps" id="aclCamps">' + h + '</div>' +
    (rows.length > 5 ? '<button type="button" class="btn-outline" id="aclAll" style="align-self:flex-start">' + (CAMP.all ? 'Mostrar só as cinco maiores' : 'Ver todas as campanhas (' + rows.length + ')') + '</button>' : '') +
    '<p class="camps-foot">' + foot + '</p></section>';
}

/* ============================== CLIENTE: O QUE MUDOU E PRÓXIMOS PASSOS ============================== */
function notesHTML(per) {
  var fs = bizFunnels(); if (!fs.length) return '';
  var FF = fs.map(function (f) { return buildFunnel(f, per); }).sort(function (a, b) { return b.cur.spend - a.cur.spend; });
  var items = [], steps = [], rows = CAMP.list.length ? CAMP.list : campRows(per);
  FF.slice(0, 3).forEach(function (F) {
    var rk = F.resKey, c = F.cur[rk], p = F.prev[rk], cpr = c > 0 ? F.cur.spend / c : null, ppr = p > 0 ? F.prev.spend / p : null, nm = RNAME(F.f, false);
    if (!cpr || !ppr) return;
    var d = cpr / ppr - 1;
    if (Math.abs(d) < 0.03) items.push({ ok: true, t: 'Custo por ' + nm + ' estável', d: money(cpr) + ', praticamente igual ao período anterior (' + money(ppr) + ').' });
    else items.push({ ok: d < 0, t: 'Custo por ' + nm + (d < 0 ? ' caiu ' : ' subiu ') + Math.round(Math.abs(d) * 100) + '%', d: money(cpr) + ' por ' + nm + ', contra ' + money(ppr) + ' no período anterior.' });
  });
  var main = FF[0], cand = rows.filter(function (c) { return c.f === main.f && c.res >= 3; }).sort(function (a, b) { return a.cost - b.cost; });
  if (cand.length > 1) items.push({ ok: true, t: cand[0].nick + ' é a mais eficiente', d: plural(cand[0].res, RNAME(main.f, false), RNAME(main.f)) + ' a ' + money(cand[0].cost) + ' cada, em ' + FNAME(main.f).toLowerCase() + '.' });
  items = items.slice(0, 4);
  // próximos passos
  var leadF = FF.filter(function (F) { return F.f === 'cadastro' || F.f === 'whatsapp'; });
  if (leadF.length && !salesMeasured()) steps.push({ t: 'Registrar quem comprou', d: 'Com esse registro, o relatório mostra quanto custa cada venda.' });
  if (leadF.some(function (F) { return !F.crmStatus; })) steps.push({ t: 'Registrar quem foi qualificado', d: 'Hoje não sabemos quais ' + leadF.filter(function (F) { return !F.crmStatus; }).map(function (F) { return RNAME(F.f); }).join(' e ') + ' tinham perfil.' });
  var avg = main.cur[main.resKey] > 0 ? main.cur.spend / main.cur[main.resKey] : null;
  var worst = rows.filter(function (c) { return c.f === main.f && c.cost && avg && c.cost > avg * 1.3 && c.inv >= main.cur.spend * 0.1; }).sort(function (a, b) { return b.cost - a.cost; })[0];
  if (worst) steps.push({ t: 'Rever ' + worst.nick, d: 'Custo por ' + RNAME(main.f, false) + ' de ' + money(worst.cost) + ', acima da média de ' + money(avg) + '.' });
  var zero = rows.filter(function (c) { return RESULT(c.f) && c.res === 0 && avg && c.inv > avg; }).sort(function (a, b) { return b.inv - a.inv; })[0];
  if (zero) steps.push({ t: 'Verificar ' + zero.nick, d: money(zero.inv) + ' investidos sem resultado registrado no período.' });
  var novas = rows.filter(function (c) { return c.nova && RESULT(c.f); }).map(function (c) { return c.nick; });
  if (novas.length) steps.push({ t: 'Acompanhar as campanhas novas', d: (novas.length > 2 ? novas.slice(0, 2).join(', ') + ' e mais ' + (novas.length - 2) : novas.join(' e ')) + ' começaram neste período e ainda não têm histórico.' });
  if (!steps.length) steps.push({ t: 'Manter a estratégia atual', d: 'Os custos estão dentro do esperado para o período.' });
  steps = steps.slice(0, 3);
  if (!items.length && !steps.length) return '';
  return '<section class="two" aria-label="Resumo e próximos passos"><div class="group"><h3>O que mudou</h3><ul class="list">' +
    (items.length ? items.map(function (it) { return '<li>' + ic(it.ok ? 'check' : 'alert').replace('class="i"', 'class="i" style="color:var(--' + (it.ok ? 'success' : 'warning') + ');margin-top:2px"') + '<div class="body"><b>' + esc(it.t) + '</b><span>' + esc(it.d) + '</span></div></li>'; }).join('') : '<li><div class="body"><span>Sem base de comparação suficiente no período anterior.</span></div></li>') +
    '</ul></div><div class="group"><h3>Próximos passos sugeridos</h3><ul class="list">' +
    steps.map(function (s) { return '<li><span class="circ" aria-hidden="true"></span><div class="body"><b>' + esc(s.t) + '</b><span>' + esc(s.d) + '</span></div></li>'; }).join('') + '</ul></div></section>';
}

/* ============================== CLIENTE: MONTAGEM ============================== */
function headHTML(per) {
  var h = '<div class="pagehead"><div class="titles"><span class="section-label">' + esc(D.cliente || '') + (STATE.unit !== 'all' ? ' · ' + esc(UNITS[+STATE.unit].nome) : '') + '</span><h1>' + esc(D.titulo || 'Resultados de mídia') + '</h1></div><div class="filters">';
  if (UNITS.length) {
    h += '<div class="fwrap l"><div class="pbox"><button class="period" id="aclUbtn" type="button" aria-haspopup="listbox" aria-expanded="false">' + ic('pin') + '<span><span class="rng">Unidade</span> ' + (STATE.unit === 'all' ? 'Todas as unidades' : esc(UNITS[+STATE.unit].nome)) + '</span>' + ic('chev') + '</button>' +
      '<div class="menu l" id="aclUmenu" role="listbox" hidden>' + ['all'].concat(UNITS.map(function (_, i) { return String(i); })).map(function (k, i) {
        return (i === 1 ? '<hr>' : '') + '<button type="button" role="option" data-u="' + k + '" aria-selected="' + (STATE.unit === k) + '"><span class="o"><b>' + (k === 'all' ? 'Todas as unidades' : esc(UNITS[+k].nome)) + '</b></span>' + ic('check') + '</button>';
      }).join('') + '</div></div><p class="cmpline">Pelo nome das campanhas</p></div>';
  }
  h += '<div class="fwrap"><div class="pbox"><button class="period" id="aclPbtn" type="button" aria-haspopup="listbox" aria-expanded="false">' + ic('cal') + '<span>' + presetLabel() + ' <span class="rng num">· ' + rangeTxt(per.from, per.to) + '</span></span>' + ic('chev') + '</button>' +
    '<div class="menu" id="aclPmenu" role="listbox" hidden>' + PRESETS.map(function (p) {
      var pp = periodFor(p[0]);
      return '<button type="button" role="option" data-p="' + p[0] + '" aria-selected="' + (STATE.preset === p[0]) + '"><span class="o"><b>' + p[1] + '</b><span class="num">' + rangeTxt(pp.from, pp.to) + '</span></span>' + ic('check') + '</button>';
    }).join('') + '<hr><div class="custom"><b style="font-weight:500">Escolher datas</b><div class="row"><input type="date" id="aclFrom" value="' + (STATE.preset === 'custom' ? per.from : '') + '"><input type="date" id="aclTo" value="' + (STATE.preset === 'custom' ? per.to : '') + '"></div><button type="button" class="btn-outline" id="aclApply">Aplicar</button></div></div></div>' +
    '<p class="cmpline">Comparado com ' + rangeTxt(per.pFrom, per.pTo) + ' · média por dia</p></div></div></div>';
  return h;
}
function footHTML() {
  var plats = {}; STATE.rows.forEach(function (r) { plats[r.platform === 'google' ? 'Google Ads' : 'Meta Ads'] = 1; });
  var ld = lastDataDate(), d = STATE.loadedAt;
  return '<footer><span>' + (d ? 'Dados lidos em ' + d.toLocaleDateString('pt-BR', { day: 'numeric', month: 'long', year: 'numeric' }) + ', às ' + d.toLocaleTimeString('pt-BR', { hour: '2-digit', minute: '2-digit' }) + '.' : '') + (ld ? ' Último dia com dados: ' + fmtD(ld, 1) + '.' : '') + '</span><span>Fontes: ' + (Object.keys(plats).join(' e ') || '—') + '</span></footer>';
}
function render() {
  var per = resolvePeriod();
  var hero = heroHTML(per);
  ROOT.innerHTML = SPRITE + '<main>' + headHTML(per) +
    '<section class="hero" aria-labelledby="h-neg"><div class="sec-head" style="margin-bottom:16px"><h2 id="h-neg">Camada de Negócios</h2></div><div class="glass hero-card">' + hero.html +
    (hero.notes.length ? hero.notes.map(function (n) { return '<div class="note-line">' + ic('info') + '<span>' + n + '</span></div>'; }).join('') : '') + '</div></section>' +
    evoHTML() + funnelsHTML(per) + campsHTML(per) + notesHTML(per) + footHTML() + '</main>';
  bind(per);
  fillEvo();
  setTimeout(reportHeight, 30);
}
function closeAll(except) {
  $$('.menu').forEach(function (m) { if (m !== except) m.hidden = true; });
  $$('.period').forEach(function (b) { b.setAttribute('aria-expanded', 'false'); });
  $$('.pop').forEach(function (p) { if (p !== except) { p.hidden = true; var b = p.parentNode.querySelector('.infobtn'); if (b) b.setAttribute('aria-expanded', 'false'); } });
}
function bind(per) {
  function menu(btnSel, menuSel) {
    var b = $(btnSel), m = $(menuSel); if (!b || !m) return;
    b.onclick = function (e) { e.stopPropagation(); var open = m.hidden; closeAll(); m.hidden = !open; b.setAttribute('aria-expanded', String(open)); reportHeight(); };
    m.onclick = function (e) { e.stopPropagation(); };
  }
  menu('#aclPbtn', '#aclPmenu'); menu('#aclUbtn', '#aclUmenu');
  $$('[data-p]').forEach(function (b) { b.onclick = function () { STATE.preset = b.dataset.p; store.set('preset', STATE.preset); render(); }; });
  if ($('#aclApply')) $('#aclApply').onclick = function () {
    var f = $('#aclFrom').value, t = $('#aclTo').value;
    if (!f || !t) return;
    if (f > t) { var x = f; f = t; t = x; }
    STATE.preset = 'custom'; STATE.from = f; STATE.to = t; render();
  };
  $$('[data-u]').forEach(function (b) { b.onclick = function () { STATE.unit = b.dataset.u; CAMP.all = false; render(); }; });
  $$('[data-evof]').forEach(function (b) { b.onclick = function () { EVO.f = b.dataset.evof; render(); }; });
  $$('[data-evom]').forEach(function (b) { b.onclick = function () { EVO.metric = b.dataset.evom; render(); }; });
  if ($('#aclView')) $('#aclView').onclick = function () { EVO.table = !EVO.table; render(); };
  if ($('#aclAll')) $('#aclAll').onclick = function () { CAMP.all = !CAMP.all; render(); };
  var cs = $('#aclCamps');
  if (cs) cs.onclick = function (e) {
    e.stopPropagation();
    var cp = e.target.closest('[data-copy]');
    if (cp) {
      var name = CAMP.list[+cp.dataset.copy].name, lab = cp.querySelector('span');
      var done = function () { lab.textContent = 'Copiado'; setTimeout(function () { lab.textContent = 'Copiar nome'; }, 1500); };
      try { navigator.clipboard.writeText(name).then(done, function () { lab.textContent = 'Selecione e copie'; }); } catch (err) { lab.textContent = 'Selecione e copie'; }
      return;
    }
    var ib = e.target.closest('.infobtn'); if (!ib) return;
    var pop = ib.parentNode.querySelector('.pop'), open = pop.hidden;
    closeAll(pop); pop.hidden = !open; ib.setAttribute('aria-expanded', String(open)); reportHeight();
  };
  if (!ROOT._aclDoc) {
    ROOT._aclDoc = 1;
    document.addEventListener('click', function () { closeAll(); });
    document.addEventListener('keydown', function (e) { if (e.key === 'Escape') closeAll(); });
  }
}
function boot(msg) { ROOT.innerHTML = '<main><div class="state"><div class="spin"></div>' + (msg || 'Lendo os dados das campanhas…') + '</div></main>'; reportHeight(); }
function init() {
  injectStyle();
  ROOT.classList.add('acl');
  initSources();
  boot();
  loadAll().then(function () {
    if (!STATE.rows.length) {
      var errs = STATE.sources.filter(function (s) { return s.status && s.status.error; }).map(function (s) { return esc(s.nome || s.conta || s.plataforma) + ': ' + esc(s.status.error); });
      ROOT.innerHTML = '<main><div class="state"><h2>Não foi possível carregar os dados</h2><p>Tente atualizar a página em alguns minutos. Se continuar, fale com a ATOM LAB.</p>' + (errs.length ? '<p class="hint" style="margin-top:12px">' + errs.join('<br>') + '</p>' : '') + '</div></main>';
      reportHeight(); return;
    }
    render();
  }).catch(function (e) { ROOT.innerHTML = '<main><div class="state"><h2>Erro ao montar o relatório</h2><p class="hint">' + esc(e.message) + '</p></div></main>'; reportHeight(); if (window.console) console.error(e); });
  if (window.ResizeObserver) { new ResizeObserver(function () { reportHeight(); }).observe(ROOT); var t; new ResizeObserver(function () { clearTimeout(t); t = setTimeout(drawEvo, 120); }).observe(ROOT); }
  window.addEventListener('resize', function () { drawEvo(); reportHeight(); });
  setInterval(reportHeight, 2000);
}
window.DASH_CLIENTE = { version: VERSION, state: STATE, nick: nick };
if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', init); else init();
})();
