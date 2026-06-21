/* =========================================================================
   GOLDENBET — DEMO FRONT-END LOGIC
   All data below is MOCK. Wire the marked sections to your real:
   - odds/data feed (replace MATCHES + the simulateOddsMovement loop)
   - authentication provider (LOGIN section)
   - payment processor (WALLET section)
   - bet placement / settlement API (BET SLIP + settlement simulation)
   - KYC / age verification provider (AGE GATE)
   ========================================================================= */

(function () {
"use strict";

/* ---------------------------- helpers ---------------------------- */
const $ = (sel, root = document) => root.querySelector(sel);
const $$ = (sel, root = document) => Array.from(root.querySelectorAll(sel));
const fmt = (n) => `$${n.toFixed(2)}`;
const fmtOdds = (n) => n.toFixed(2);
const rand = (min, max) => Math.random() * (max - min) + min;
const randInt = (min, max) => Math.floor(rand(min, max + 1));
const pick = (arr) => arr[randInt(0, arr.length - 1)];
const uid = (() => { let n = 0; return (p = "id") => `${p}_${Date.now()}_${n++}`; })();
const clamp = (n, min, max) => Math.max(min, Math.min(max, n));

function toast(msg, variant = "") {
  const root = $("#toastRoot");
  const el = document.createElement("div");
  el.className = `toast ${variant}`;
  el.textContent = msg;
  root.appendChild(el);
  setTimeout(() => el.remove(), 3600);
}

function openOverlay(id) { $(id).classList.remove("hidden"); }
function closeOverlay(id) { $(id).classList.add("hidden"); }

/* ---------------------------- persistence ---------------------------- */
const STORE = {
  get(key, fallback) {
    try { const v = localStorage.getItem(key); return v === null ? fallback : JSON.parse(v); }
    catch { return fallback; }
  },
  set(key, val) { try { localStorage.setItem(key, JSON.stringify(val)); } catch {} },
};

let state = {
  balance: STORE.get("gb_balance", 1000),
  favLeagues: STORE.get("gb_favs", []),
  openBets: STORE.get("gb_open_bets", []),
  settledBets: STORE.get("gb_settled_bets", []),
  tx: STORE.get("gb_tx", [
    { id: uid("tx"), label: "Welcome bonus credit", amt: 1000, ts: Date.now() - 86400000 },
  ]),
  selfExcluded: STORE.get("gb_self_excluded", false),
  slip: {},                // key -> selection
  slipMode: "single",
  currentSport: "football",
  currentView: "sportsbook",
  filter: "all",
  searchTerm: "",
};

function persist() {
  STORE.set("gb_balance", state.balance);
  STORE.set("gb_favs", state.favLeagues);
  STORE.set("gb_open_bets", state.openBets);
  STORE.set("gb_settled_bets", state.settledBets);
  STORE.set("gb_tx", state.tx);
  STORE.set("gb_self_excluded", state.selfExcluded);
}

/* ============================================================
   MOCK SPORTS DATA — replace with calls to your real odds feed
   ============================================================ */
const SPORTS = [
  { id: "football", name: "Football", icon: "⚽" },
  { id: "basketball", name: "Basketball", icon: "🏀" },
  { id: "tennis", name: "Tennis", icon: "🎾" },
  { id: "esports", name: "Esports", icon: "🎮" },
  { id: "baseball", name: "Baseball", icon: "⚾" },
  { id: "hockey", name: "Hockey", icon: "🏒" },
];

const LEAGUE_POOLS = {
  football: {
    "Premier League": ["Arsenal", "Chelsea", "Liverpool", "Man City", "Man United", "Tottenham", "Newcastle", "Aston Villa"],
    "La Liga": ["Real Madrid", "Barcelona", "Atletico Madrid", "Sevilla", "Valencia", "Villarreal"],
    "Serie A": ["Inter Milan", "AC Milan", "Juventus", "Napoli", "Roma", "Lazio"],
  },
  basketball: {
    "NBA": ["Lakers", "Celtics", "Warriors", "Bucks", "Nuggets", "Heat", "Suns", "76ers"],
    "EuroLeague": ["Real Madrid BC", "Olympiacos", "Fenerbahce", "Panathinaikos"],
  },
  tennis: {
    "ATP Tour": ["Djokovic", "Alcaraz", "Sinner", "Medvedev", "Zverev", "Rublev", "Tsitsipas", "Ruud"],
    "WTA Tour": ["Swiatek", "Sabalenka", "Gauff", "Rybakina", "Pegula", "Vondrousova"],
  },
  esports: {
    "CS2 Major": ["Navi", "Vitality", "FaZe", "G2", "Astralis", "Liquid"],
    "LoL Champions": ["T1", "G2 Esports", "JDG", "Fnatic", "Cloud9", "MAD Lions"],
  },
  baseball: {
    "MLB": ["Yankees", "Dodgers", "Red Sox", "Astros", "Braves", "Mets", "Giants", "Cubs"],
  },
  hockey: {
    "NHL": ["Maple Leafs", "Bruins", "Oilers", "Avalanche", "Rangers", "Panthers", "Stars", "Kings"],
  },
};

const HAS_DRAW = new Set(["football"]);

function genOdds() { return +rand(1.35, 5.5).toFixed(2); }
function genLine(spread) { return +(Math.round(rand(spread ? -8 : 190, spread ? 8 : 230) * 2) / 2).toFixed(1); }

function buildMatch(sport, league, home, away) {
  const isLive = Math.random() < 0.32;
  const base = {
    id: uid("m"),
    sport, league, home, away,
    isLive,
    minute: isLive ? `${randInt(1, 89)}'` : null,
    scoreHome: isLive ? randInt(0, 4) : null,
    scoreAway: isLive ? randInt(0, 4) : null,
    startLabel: isLive ? "LIVE" : pick(["Today 18:00", "Today 20:30", "Tomorrow 15:00", "Tomorrow 19:45", "Sat 17:30"]),
    fav: false,
  };

  if (HAS_DRAW.has(sport)) {
    base.markets = {
      oneXtwo: { home: genOdds(), draw: +rand(2.8, 4.2).toFixed(2), away: genOdds() },
      ou: { line: pick([1.5, 2.5, 3.5]), over: +rand(1.6, 2.3).toFixed(2), under: +rand(1.6, 2.3).toFixed(2) },
    };
  } else {
    const spreadLine = +rand(1.5, 9.5).toFixed(1);
    base.markets = {
      moneyline: { home: genOdds(), away: genOdds() },
      spread: { line: spreadLine, home: +rand(1.75, 2.05).toFixed(2), away: +rand(1.75, 2.05).toFixed(2) },
      total: { line: genLine(false) / 10 + 180, over: +rand(1.75, 2.05).toFixed(2), under: +rand(1.75, 2.05).toFixed(2) },
    };
  }
  return base;
}

function generateAllMatches() {
  const matches = [];
  for (const sport of Object.keys(LEAGUE_POOLS)) {
    for (const [league, teams] of Object.entries(LEAGUE_POOLS[sport])) {
      const shuffled = [...teams].sort(() => Math.random() - 0.5);
      const count = Math.min(3, Math.floor(shuffled.length / 2));
      for (let i = 0; i < count; i++) {
        matches.push(buildMatch(sport, league, shuffled[i * 2], shuffled[i * 2 + 1]));
      }
    }
  }
  return matches;
}

let MATCHES = generateAllMatches();

/* ============================================================
   ICONS (inline, no external icon library needed)
   ============================================================ */
const starIcon = (filled) =>
  `<svg viewBox="0 0 24 24" class="star ${filled ? "filled" : ""}"><path d="M12 2l3 6.5 7 1-5 5 1.3 7L12 18l-6.3 3.5L7 14l-5-5 7-1z"/></svg>`;

/* ============================================================
   RENDER: SPORT NAV
   ============================================================ */
function renderSportNav() {
  const nav = $("#sportNav");
  const liveBtn = `<button class="sport-tab live-tab ${state.currentView === "live" ? "active" : ""}" data-nav-view="live">
      <span class="live-dot"></span> Live
    </button>`;
  const tabs = SPORTS.map(
    (s) => `<button class="sport-tab ${state.currentView === "sportsbook" && state.currentSport === s.id ? "active" : ""}" data-sport="${s.id}">
        <span class="ic">${s.icon}</span>${s.name}
      </button>`
  ).join("");
  nav.innerHTML = liveBtn + tabs;
}

/* ============================================================
   RENDER: SIDEBAR LEAGUES
   ============================================================ */
function renderSidebar() {
  const heading = $("#leagueHeading");
  const sportName = SPORTS.find((s) => s.id === state.currentSport)?.name || "Leagues";
  heading.textContent = `${sportName} Leagues`;

  const leagues = Object.keys(LEAGUE_POOLS[state.currentSport] || {});
  $("#leagueList").innerHTML = leagues
    .map(
      (lg) => `<li class="league-item" data-league="${lg}">
        <span>${lg}</span>
        <button class="star-btn ${state.favLeagues.includes(lg) ? "filled" : ""}" data-fav-league="${lg}">
          ${starIcon(state.favLeagues.includes(lg))}
        </button>
      </li>`
    )
    .join("");

  $("#favList").innerHTML = state.favLeagues
    .map((lg) => `<li class="league-item active"><span>${lg}</span></li>`)
    .join("");
}

/* ============================================================
   RENDER: MATCH CARDS
   ============================================================ */
function matchPassesFilters(m) {
  if (state.filter === "live" && !m.isLive) return false;
  if (state.filter === "upcoming" && m.isLive) return false;
  if (state.searchTerm) {
    const t = state.searchTerm.toLowerCase();
    return (m.home + " " + m.away + " " + m.league).toLowerCase().includes(t);
  }
  return true;
}

function selKey(matchId, marketKey) { return `${matchId}__${marketKey}`; }

function oddBtnHTML(matchId, marketKey, side, label, oddsVal, sub) {
  const key = selKey(matchId, marketKey);
  const selected = state.slip[key] && state.slip[key].side === side;
  return `<button class="odd-btn ${selected ? "selected" : ""}"
      data-match="${matchId}" data-market="${marketKey}" data-side="${side}" data-odds="${oddsVal}">
    <span class="odd-label">${label}</span>
    <span class="odd-value-wrap">
      <span class="odd-value" data-oddcell="${matchId}|${marketKey}|${side}">${fmtOdds(oddsVal)}</span>
      <span class="odd-arrow"></span>
    </span>
    ${sub ? `<span class="odd-label" style="font-size:9.5px">${sub}</span>` : ""}
  </button>`;
}

function renderMatchCard(m) {
  const teamRow = (name, score, leading) =>
    `<div class="team-row ${leading ? "leading" : ""}"><span>${name}</span>${
      score !== null ? `<span class="score">${score}</span>` : ""
    }</div>`;
  const leadingHome = m.isLive && m.scoreHome > m.scoreAway;
  const leadingAway = m.isLive && m.scoreAway > m.scoreHome;

  let marketsHTML;
  if (HAS_DRAW.has(m.sport)) {
    const mk = m.markets.oneXtwo;
    marketsHTML = `
      <div class="markets" style="grid-template-columns:repeat(3,1fr)">
        ${oddBtnHTML(m.id, "oneXtwo", "home", "1", mk.home)}
        ${oddBtnHTML(m.id, "oneXtwo", "draw", "X", mk.draw)}
        ${oddBtnHTML(m.id, "oneXtwo", "away", "2", mk.away)}
      </div>
      <div class="more-markets" data-toggle-more="${m.id}">More markets ▾</div>
      <div class="markets hidden" id="more_${m.id}" style="grid-template-columns:repeat(2,1fr); margin-top:8px;">
        ${oddBtnHTML(m.id, "ou", "over", `O ${m.markets.ou.line}`, m.markets.ou.over)}
        ${oddBtnHTML(m.id, "ou", "under", `U ${m.markets.ou.line}`, m.markets.ou.under)}
      </div>`;
  } else {
    const { moneyline, spread, total } = m.markets;
    marketsHTML = `
      <div class="markets" style="grid-template-columns:repeat(3,1fr); margin-bottom:6px;">
        <div class="odd-label" style="text-align:center">Moneyline</div>
        <div class="odd-label" style="text-align:center">Spread</div>
        <div class="odd-label" style="text-align:center">Total</div>
      </div>
      <div class="markets" style="grid-template-columns:repeat(3,1fr)">
        ${oddBtnHTML(m.id, "moneyline", "home", m.home.slice(0, 3).toUpperCase(), moneyline.home)}
        ${oddBtnHTML(m.id, "spread", "home", `${spread.line > 0 ? "-" : "+"}${Math.abs(spread.line)}`, spread.home)}
        ${oddBtnHTML(m.id, "total", "over", `O ${total.line}`, total.over)}
      </div>
      <div class="markets" style="grid-template-columns:repeat(3,1fr); margin-top:8px;">
        ${oddBtnHTML(m.id, "moneyline", "away", m.away.slice(0, 3).toUpperCase(), moneyline.away)}
        ${oddBtnHTML(m.id, "spread", "away", `${spread.line > 0 ? "+" : "-"}${Math.abs(spread.line)}`, spread.away)}
        ${oddBtnHTML(m.id, "total", "under", `U ${total.line}`, total.under)}
      </div>`;
  }

  return `<article class="match-card" data-card="${m.id}">
    <div class="match-top">
      <span class="match-status ${m.isLive ? "live" : ""}">
        ${m.isLive ? `<span class="live-dot"></span> LIVE ${m.minute}` : m.startLabel}
      </span>
      <button class="star-btn ${m.fav ? "filled" : ""}" data-fav-match="${m.id}">${starIcon(m.fav)}</button>
    </div>
    <div class="teams">
      ${teamRow(m.home, m.scoreHome, leadingHome)}
      ${teamRow(m.away, m.scoreAway, leadingAway)}
    </div>
    ${marketsHTML}
  </article>`;
}

function renderLeagueGroupedList(targetEl, matches) {
  if (!matches.length) {
    targetEl.innerHTML = `<div class="empty-state">
      <svg viewBox="0 0 24 24"><circle cx="11" cy="11" r="7"/><path d="M21 21l-4.3-4.3"/></svg>
      <p>No matches found. Try a different filter or search term.</p>
    </div>`;
    return;
  }
  const byLeague = {};
  matches.forEach((m) => { (byLeague[m.league] = byLeague[m.league] || []).push(m); });
  targetEl.innerHTML = Object.entries(byLeague)
    .map(([league, list]) => `<div class="league-group-label">🏆 ${league}</div>${list.map(renderMatchCard).join("")}`)
    .join("");
}

function renderMatchList() {
  $("#contentTitle").textContent = SPORTS.find((s) => s.id === state.currentSport)?.name || "";
  const matches = MATCHES.filter((m) => m.sport === state.currentSport).filter(matchPassesFilters);
  renderLeagueGroupedList($("#matchList"), matches);
}

function renderLiveList() {
  const matches = MATCHES.filter((m) => m.isLive).filter((m) => matchPassesFilters({ ...m, isLive: true }));
  renderLeagueGroupedList($("#liveList"), matches);
}

function renderFilterChips() {
  $("#filterChips").innerHTML = ["all", "live", "upcoming"]
    .map((f) => `<button class="chip ${state.filter === f ? "active" : ""}" data-filter="${f}">${f[0].toUpperCase() + f.slice(1)}</button>`)
    .join("");
}

/* ============================================================
   TICKER
   ============================================================ */
function renderTicker() {
  const live = MATCHES.filter((m) => m.isLive).slice(0, 14);
  const html = (live.length ? live : MATCHES.slice(0, 8))
    .map((m) => {
      const odd = HAS_DRAW.has(m.sport) ? m.markets.oneXtwo.home : m.markets.moneyline.home;
      return `<span class="tick-item">${m.isLive ? '<span class="live-dot"></span>' : ""}
        ${m.home} <b>${m.isLive ? m.scoreHome : "-"}</b>:<b>${m.isLive ? m.scoreAway : "-"}</b> ${m.away}
        <span class="tick-odds">${fmtOdds(odd)}</span></span>`;
    })
    .join("");
  $("#ticker").innerHTML = html;
  $("#tickerDup").innerHTML = html;
}

/* ============================================================
   PROMO STRIP (static demo content)
   ============================================================ */
function renderPromoStrip() {
  $("#promoStrip").innerHTML = `
    <div class="promo-card gold"><h4>🎁 100% Welcome Bonus</h4><p>Demo credit on your first deposit, up to $200</p></div>
    <div class="promo-card cyan"><h4>⚡ Live Boosted Odds</h4><p>Selected in-play markets boosted hourly</p></div>
    <div class="promo-card gold"><h4>🔗 Parlay Insurance</h4><p>Get your stake back in credit if one leg lets you down</p></div>`;
  $$(".promo-card").forEach((c) => c.addEventListener("click", () => toast("Demo promo — connect your promotions engine to activate.")));
}

/* ============================================================
   BET SLIP
   ============================================================ */
function slipCount() { return Object.keys(state.slip).length; }

function updateSlipBadges() {
  const n = slipCount();
  $$("#slipCountFab, #slipCountInline").forEach((el) => { el.textContent = n; el.classList.toggle("hidden", n === 0); });
  $("#betslipFab").classList.toggle("hidden", n === 0 || isDesktop());
}

function isDesktop() { return window.matchMedia("(min-width: 981px)").matches; }

function toggleSelection(matchId, marketKey, side, odds, btn) {
  if (state.selfExcluded) { toast("Self-exclusion is active — betting is disabled for this session.", "gold"); return; }
  const key = selKey(matchId, marketKey);
  const m = MATCHES.find((x) => x.id === matchId);
  const existing = state.slip[key];

  if (existing && existing.side === side) {
    delete state.slip[key];
  } else {
    const label = btn.querySelector(".odd-label").textContent;
    state.slip[key] = {
      matchId, marketKey, side, odds,
      matchLabel: `${m.home} vs ${m.away}`,
      selLabel: `${label} · ${marketKey === "oneXtwo" ? "1X2" : marketKey}`,
      stake: 10,
    };
  }
  if (slipCount() < 2) state.slipMode = "single";
  renderMatchList(); renderLiveList(); renderBetSlip();
  if (!isDesktop() && slipCount() > 0) openBetSlipMobile();
}

function renderBetSlip() {
  const items = Object.values(state.slip);
  $("#slipEmpty").classList.toggle("hidden", items.length > 0);
  $("#betslipFooter").classList.toggle("hidden", items.length === 0);
  $("#parlayModeBtn").disabled = items.length < 2;
  $$(".mode-btn").forEach((b) => b.classList.toggle("active", b.dataset.mode === state.slipMode));
  $("#parlayStakeRow").classList.toggle("hidden", state.slipMode !== "parlay");

  $("#slipItems").innerHTML = items
    .map((it) => {
      const key = selKey(it.matchId, it.marketKey);
      const payout = it.stake * it.odds;
      return `<div class="slip-item" data-slip-key="${key}">
        <div class="slip-item-top">
          <div><div class="slip-item-match">${it.matchLabel}</div><div class="slip-item-sel">${it.selLabel}</div></div>
          <button class="slip-remove" data-remove-slip="${key}"><svg viewBox="0 0 24 24"><path d="M6 6l12 12M18 6L6 18"/></svg></button>
        </div>
        <div class="slip-item-bottom">
          <span class="slip-odds">@ ${fmtOdds(it.odds)}</span>
          ${state.slipMode === "single" ? `<div class="stake-input"><span>$</span><input type="number" min="0" step="1" value="${it.stake}" data-stake-key="${key}" /></div>` : ""}
        </div>
        ${state.slipMode === "single" ? `<div class="slip-item-payout">Payout: ${fmt(payout)}</div>` : ""}
      </div>`;
    })
    .join("");

  computeSlipSummary();
  updateSlipBadges();
}

function computeSlipSummary() {
  const items = Object.values(state.slip);
  if (state.slipMode === "parlay") {
    const stake = +($("#parlayStake").value || 0);
    const combo = items.reduce((acc, it) => acc * it.odds, 1);
    $("#comboOdds").textContent = items.length ? fmtOdds(combo) : "—";
    $("#potentialPayout").textContent = fmt(stake * combo);
  } else {
    const totalStake = items.reduce((s, it) => s + (it.stake || 0), 0);
    const totalPayout = items.reduce((s, it) => s + (it.stake || 0) * it.odds, 0);
    $("#comboOdds").textContent = items.length ? `${items.length} singles` : "—";
    $("#potentialPayout").textContent = fmt(totalPayout);
    $("#parlayStakeRow").previousElementSibling; // no-op guard
    void totalStake;
  }
}

function placeBet() {
  if (state.selfExcluded) { toast("Self-exclusion is active — betting is disabled.", "gold"); return; }
  const items = Object.values(state.slip);
  if (!items.length) return;

  if (state.slipMode === "parlay") {
    const stake = +($("#parlayStake").value || 0);
    const combo = items.reduce((acc, it) => acc * it.odds, 1);
    if (stake <= 0) return toast("Enter a stake first.");
    if (stake > state.balance) return toast("Insufficient demo balance — try depositing more funds.", "gold");
    state.balance -= stake;
    state.openBets.unshift({
      id: uid("bet"), type: "Parlay", legs: items.map((i) => i.selLabel), stake, odds: combo,
      potential: stake * combo, placedAt: Date.now(),
    });
  } else {
    let totalStake = 0;
    for (const it of items) totalStake += it.stake || 0;
    if (totalStake <= 0) return toast("Enter at least one stake.");
    if (totalStake > state.balance) return toast("Insufficient demo balance — try depositing more funds.", "gold");
    state.balance -= totalStake;
    items.forEach((it) => {
      if (!it.stake) return;
      state.openBets.unshift({
        id: uid("bet"), type: "Single", legs: [`${it.matchLabel} — ${it.selLabel}`], stake: it.stake,
        odds: it.odds, potential: it.stake * it.odds, placedAt: Date.now(),
      });
    });
  }

  state.slip = {};
  state.slipMode = "single";
  persist();
  renderAll();
  toast("Bet placed (demo) — no real funds were wagered.", "gold");
  closeBetSlipMobile();
}

function openBetSlipMobile() { $("#betslip").classList.remove("hidden"); $("#betslip").classList.add("open"); }
function closeBetSlipMobile() { $("#betslip").classList.remove("open"); }

/* ============================================================
   WALLET
   ============================================================ */
function renderWallet() {
  $("#balanceAmt").textContent = fmt(state.balance);
  $("#walletBalance").textContent = fmt(state.balance);
  $("#txList").innerHTML = state.tx
    .slice(0, 25)
    .map(
      (t) => `<div class="tx-row"><div class="tx-meta"><strong>${t.label}</strong><span>${new Date(t.ts).toLocaleString()}</span></div>
        <span class="${t.amt >= 0 ? "amt-pos" : "amt-neg"}">${t.amt >= 0 ? "+" : ""}${fmt(t.amt)}</span></div>`
    )
    .join("") || `<div class="empty-state"><p>No transactions yet.</p></div>`;
}

/* ============================================================
   MY BETS
   ============================================================ */
function renderBets(tab = "open") {
  const list = tab === "open" ? state.openBets : state.settledBets;
  $("#betsList").innerHTML =
    list
      .map((b) => {
        const statusBadge =
          tab === "open"
            ? `<span class="badge-pill badge-pending">Pending</span>`
            : `<span class="badge-pill ${b.result === "won" ? "badge-win" : "badge-lose"}">${b.result === "won" ? "Won" : "Lost"}</span>`;
        return `<div class="bet-row">
          <div class="bet-meta">
            <strong>${b.type} · ${b.legs.length} ${b.legs.length > 1 ? "legs" : "leg"}</strong>
            <span>${b.legs.join(" • ")}</span>
            <span>Stake ${fmt(b.stake)} @ ${fmtOdds(b.odds)} → ${fmt(b.potential)}</span>
          </div>
          ${statusBadge}
        </div>`;
      })
      .join("") || `<div class="empty-state"><p>No ${tab} bets yet — place one from the sportsbook.</p></div>`;
}

function settleRandomBet() {
  if (!state.openBets.length) return;
  const idx = state.openBets.findIndex((b) => Date.now() - b.placedAt > 9000);
  if (idx === -1) return;
  const bet = state.openBets.splice(idx, 1)[0];
  const impliedProb = clamp(1 / bet.odds, 0.12, 0.85);
  const won = Math.random() < impliedProb;
  bet.result = won ? "won" : "lost";
  state.settledBets.unshift(bet);
  if (won) {
    state.balance += bet.potential;
    state.tx.unshift({ id: uid("tx"), label: `Bet won — ${bet.type}`, amt: bet.potential, ts: Date.now() });
    toast(`🎉 Bet settled: Won ${fmt(bet.potential)}!`, "gold");
  } else {
    toast(`Bet settled: Lost ${fmt(bet.stake)}.`);
  }
  persist();
  if (state.currentView === "bets") renderBets($(".tab-btn.active")?.dataset.tab || "open");
  renderWallet();
}

/* ============================================================
   VIEW ROUTING
   ============================================================ */
function switchView(view) {
  state.currentView = view;
  $$(".view").forEach((v) => v.classList.add("hidden"));
  $(`#view-${view}`).classList.remove("hidden");
  $$(".bn-btn").forEach((b) => b.classList.toggle("active", b.dataset.view === view));
  $("#betslip").classList.toggle("hidden", false);
  renderSportNav();
  if (view === "sportsbook") renderMatchList();
  if (view === "live") renderLiveList();
  if (view === "wallet") renderWallet();
  if (view === "bets") renderBets($(".tab-btn.active")?.dataset.tab || "open");
  if (window.innerWidth <= 980 && view !== "sportsbook" && view !== "live") {
    // keep bet slip reachable but out of the way on secondary views
  }
}

function renderAll() {
  renderSportNav(); renderSidebar(); renderTicker(); renderPromoStrip(); renderFilterChips();
  renderMatchList(); renderLiveList(); renderBetSlip(); renderWallet();
  $("#betslip").classList.remove("hidden");
}

/* ============================================================
   LIVE ODDS SIMULATION (signature feature)
   ============================================================ */
function bumpOdds(market, side) {
  const delta = +rand(0.03, 0.14).toFixed(2) * (Math.random() < 0.5 ? -1 : 1);
  const oldVal = market[side];
  market[side] = clamp(+(oldVal + delta).toFixed(2), 1.05, 15);
  return { oldVal, newVal: market[side] };
}

function simulateOddsMovement() {
  if (!MATCHES.length) return;
  const m = pick(MATCHES);
  const marketKeys = Object.keys(m.markets);
  const marketKey = pick(marketKeys);
  const market = m.markets[marketKey];
  const sides = Object.keys(market).filter((k) => typeof market[k] === "number" && k !== "line");
  if (!sides.length) return;
  const side = pick(sides);
  const { oldVal, newVal } = bumpOdds(market, side);

  const cell = $(`[data-oddcell="${m.id}|${marketKey}|${side}"]`);
  if (cell) {
    cell.textContent = fmtOdds(newVal);
    cell.classList.remove("flip"); void cell.offsetWidth; cell.classList.add("flip");
    const arrow = cell.parentElement.querySelector(".odd-arrow");
    if (arrow) {
      arrow.textContent = newVal > oldVal ? "▲" : "▼";
      arrow.className = `odd-arrow show ${newVal > oldVal ? "up" : "down"}`;
      setTimeout(() => arrow.classList.remove("show"), 2500);
    }
    const btn = cell.closest(".odd-btn");
    if (btn) btn.dataset.odds = newVal;
  }
  // keep slip odds in sync if that exact selection is in the bet slip
  const key = selKey(m.id, marketKey);
  if (state.slip[key] && state.slip[key].side === side) {
    state.slip[key].odds = newVal;
    renderBetSlip();
  }
  if (m.isLive) renderTicker();
}

/* ============================================================
   EVENT WIRING
   ============================================================ */
function wireEvents() {
  $("#ageConfirm").addEventListener("click", () => {
    localStorage.setItem("gb_age_ok", "1");
    closeOverlay("#ageGate");
    $("#app").classList.remove("hidden");
  });

  // sport nav / live nav (delegated)
  $("#sportNav").addEventListener("click", (e) => {
    const sportBtn = e.target.closest("[data-sport]");
    const liveBtn = e.target.closest("[data-nav-view]");
    if (sportBtn) { state.currentSport = sportBtn.dataset.sport; switchView("sportsbook"); renderSidebar(); }
    if (liveBtn) { switchView("live"); }
  });

  $(".brand").addEventListener("click", () => switchView("sportsbook"));

  // bottom nav + sidebar "manage limits" link
  document.addEventListener("click", (e) => {
    const navBtn = e.target.closest("[data-view]");
    if (navBtn) switchView(navBtn.dataset.view);
  });

  $("#walletBtn").addEventListener("click", () => switchView("wallet"));
  $("#acctBtn").addEventListener("click", () => switchView("account"));
  $("#betsNavBtn").addEventListener("click", () => switchView("bets"));

  // search
  $("#searchBtn").addEventListener("click", () => { $("#searchRow").classList.toggle("hidden"); $("#searchInput").focus(); });
  $("#searchClose").addEventListener("click", () => { $("#searchRow").classList.add("hidden"); $("#searchInput").value = ""; state.searchTerm = ""; renderMatchList(); });
  $("#searchInput").addEventListener("input", (e) => { state.searchTerm = e.target.value; renderMatchList(); });

  // filter chips
  $("#filterChips").addEventListener("click", (e) => {
    const chip = e.target.closest("[data-filter]");
    if (!chip) return;
    state.filter = chip.dataset.filter;
    renderFilterChips(); renderMatchList();
  });

  // sidebar leagues: favorite + filter click
  $("#leagueList").addEventListener("click", (e) => {
    const favBtn = e.target.closest("[data-fav-league]");
    const item = e.target.closest("[data-league]");
    if (favBtn) {
      const lg = favBtn.dataset.favLeague;
      state.favLeagues = state.favLeagues.includes(lg) ? state.favLeagues.filter((x) => x !== lg) : [...state.favLeagues, lg];
      persist(); renderSidebar();
      return;
    }
    if (item) {
      $$("#leagueList .league-item").forEach((li) => li.classList.toggle("active", li === item));
    }
  });

  // mobile menu
  $("#menuBtn").addEventListener("click", () => { $("#sidebar").classList.add("open"); $("#sidebarScrim").classList.add("show"); });
  $("#sidebarScrim").addEventListener("click", () => { $("#sidebar").classList.remove("open"); $("#sidebarScrim").classList.remove("show"); });

  // match list delegation (odds, favorite match, more markets) — applies to both lists
  function matchListClick(e) {
    const oddBtn = e.target.closest(".odd-btn");
    const favBtn = e.target.closest("[data-fav-match]");
    const moreBtn = e.target.closest("[data-toggle-more]");
    if (oddBtn) {
      toggleSelection(oddBtn.dataset.match, oddBtn.dataset.market, oddBtn.dataset.side, +oddBtn.dataset.odds, oddBtn);
    } else if (favBtn) {
      const id = favBtn.dataset.favMatch;
      const m = MATCHES.find((x) => x.id === id);
      m.fav = !m.fav;
      renderMatchList(); renderLiveList();
    } else if (moreBtn) {
      const panel = $(`#more_${moreBtn.dataset.toggleMore}`);
      panel.classList.toggle("hidden");
      moreBtn.textContent = panel.classList.contains("hidden") ? "More markets ▾" : "Less markets ▴";
    }
  }
  $("#matchList").addEventListener("click", matchListClick);
  $("#liveList").addEventListener("click", matchListClick);

  // bet slip
  $("#betslipClose").addEventListener("click", closeBetSlipMobile);
  $("#betslipFab").addEventListener("click", openBetSlipMobile);
  $("#slipMode").addEventListener("click", (e) => {
    const btn = e.target.closest(".mode-btn");
    if (!btn || btn.disabled) return;
    state.slipMode = btn.dataset.mode;
    renderBetSlip();
  });
  $("#betslipBody").addEventListener("click", (e) => {
    const rm = e.target.closest("[data-remove-slip]");
    if (rm) { delete state.slip[rm.dataset.removeSlip]; renderMatchList(); renderLiveList(); renderBetSlip(); }
  });
  $("#betslipBody").addEventListener("input", (e) => {
    const stakeInput = e.target.closest("[data-stake-key]");
    if (stakeInput) {
      const it = state.slip[stakeInput.dataset.stakeKey];
      if (it) { it.stake = +stakeInput.value || 0; computeSlipSummary(); $(`[data-slip-key="${stakeInput.dataset.stakeKey}"] .slip-item-payout`).textContent = `Payout: ${fmt(it.stake * it.odds)}`; }
    }
  });
  $("#parlayStake").addEventListener("input", computeSlipSummary);
  $("#placeBetBtn").addEventListener("click", placeBet);

  // my bets tabs
  $("#betsTabs").addEventListener("click", (e) => {
    const tab = e.target.closest(".tab-btn");
    if (!tab) return;
    $$("#betsTabs .tab-btn").forEach((b) => b.classList.toggle("active", b === tab));
    renderBets(tab.dataset.tab);
  });

  // wallet
  $("#depositBtn").addEventListener("click", () => openOverlay("#depositOverlay"));
  $("#withdrawBtn").addEventListener("click", () => {
    const amt = Math.min(50, state.balance);
    if (amt <= 0) return toast("No balance to withdraw.");
    state.balance -= amt;
    state.tx.unshift({ id: uid("tx"), label: "Withdrawal (demo)", amt: -amt, ts: Date.now() });
    persist(); renderWallet();
    toast("Demo withdrawal processed — connect a real payout flow to go live.");
  });
  $$(".quick-amounts .chip").forEach((c) => c.addEventListener("click", () => { $("#depositAmt").value = c.dataset.amt; }));
  $("#depositConfirm").addEventListener("click", () => {
    const amt = +$("#depositAmt").value || 0;
    if (amt <= 0) return;
    state.balance += amt;
    state.tx.unshift({ id: uid("tx"), label: "Deposit (demo)", amt, ts: Date.now() });
    persist(); renderWallet();
    closeOverlay("#depositOverlay");
    toast(`Added ${fmt(amt)} demo funds.`, "gold");
  });

  // login modal
  $("#loginConfirm").addEventListener("click", () => { closeOverlay("#loginOverlay"); location.reload(); });

  // overlays generic close
  $$("[data-close-overlay]").forEach((b) => b.addEventListener("click", () => b.closest(".overlay").classList.add("hidden")));

  // account: responsible gambling controls
  $("#limitSlider").addEventListener("input", (e) => { $("#limitValue").textContent = `$${e.target.value}`; });
  $$("[data-cooloff]").forEach((b) => b.addEventListener("click", () => toast(`${b.dataset.cooloff}-day cool-off activated (demo) — enforce this server-side in production.`, "gold")));
  $("#selfExcludeBtn").addEventListener("click", () => {
    if (!confirm("This will block all betting for this demo session. Continue?")) return;
    state.selfExcluded = true;
    state.slip = {};
    persist();
    renderBetSlip();
    toast("Self-exclusion activated. Betting is disabled for this session.", "gold");
  });
  $("#logoutBtn").addEventListener("click", () => {
    if (!confirm("Log out and reset all demo data?")) return;
    ["gb_balance", "gb_favs", "gb_open_bets", "gb_settled_bets", "gb_tx", "gb_self_excluded"].forEach((k) => localStorage.removeItem(k));
    openOverlay("#loginOverlay");
  });

  // notifications (lightweight demo dropdown via toast)
  $("#notifBtn").addEventListener("click", () => toast("3 new: bonus credited, bet settled, line move on a favourite.", "gold"));

  window.addEventListener("resize", updateSlipBadges);
}

/* ============================================================
   INIT
   ============================================================ */
function init() {
  if (localStorage.getItem("gb_age_ok") === "1") {
    closeOverlay("#ageGate");
    $("#app").classList.remove("hidden");
  }
  if (state.selfExcluded) toast("Self-exclusion is active on this demo account.", "gold");

  wireEvents();
  renderAll();
  switchView("sportsbook");

  setInterval(simulateOddsMovement, 4500);
  setInterval(settleRandomBet, 6000);
}

document.addEventListener("DOMContentLoaded", init);
})();
