<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Wireshark — Network Reconnaissance Detection</title>
<link href="https://fonts.googleapis.com/css2?family=Share+Tech+Mono&family=Rajdhani:wght@300;400;500;600;700&family=Orbitron:wght@400;700;900&display=swap" rel="stylesheet">
<style>
  :root {
    --bg: #030a0e;
    --bg2: #060f15;
    --surface: #0a1a22;
    --surface2: #0d2030;
    --border: #0f3a50;
    --accent: #00d4ff;
    --accent2: #00ff88;
    --danger: #ff3b3b;
    --warn: #ffaa00;
    --muted: #3a6a80;
    --text: #c8e8f0;
    --text2: #7ab8cc;
    --font-mono: 'Share Tech Mono', monospace;
    --font-body: 'Rajdhani', sans-serif;
    --font-display: 'Orbitron', monospace;
  }

  * { margin: 0; padding: 0; box-sizing: border-box; }

  html { scroll-behavior: smooth; }

  body {
    background: var(--bg);
    color: var(--text);
    font-family: var(--font-body);
    font-size: 16px;
    line-height: 1.7;
    overflow-x: hidden;
  }

  /* Scanline overlay */
  body::before {
    content: '';
    position: fixed;
    inset: 0;
    background: repeating-linear-gradient(
      0deg,
      transparent,
      transparent 2px,
      rgba(0,212,255,0.015) 2px,
      rgba(0,212,255,0.015) 4px
    );
    pointer-events: none;
    z-index: 1000;
  }

  /* Noise texture */
  body::after {
    content: '';
    position: fixed;
    inset: 0;
    background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.9' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)' opacity='0.03'/%3E%3C/svg%3E");
    pointer-events: none;
    z-index: 999;
    opacity: 0.4;
  }

  /* ─── LAYOUT ─────────────────────────────── */
  .container {
    max-width: 1100px;
    margin: 0 auto;
    padding: 0 2rem;
  }

  /* ─── HEADER ─────────────────────────────── */
  header {
    position: relative;
    padding: 4rem 0 3rem;
    border-bottom: 1px solid var(--border);
    overflow: hidden;
  }

  .header-grid {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 2rem;
    align-items: end;
  }

  .header-bg {
    position: absolute;
    inset: 0;
    background: 
      radial-gradient(ellipse 60% 100% at 80% 50%, rgba(0,212,255,0.06) 0%, transparent 60%),
      radial-gradient(ellipse 40% 80% at 10% 50%, rgba(0,255,136,0.03) 0%, transparent 60%);
  }

  .badge-row {
    display: flex;
    gap: 0.6rem;
    flex-wrap: wrap;
    margin-bottom: 1.2rem;
  }

  .badge {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    padding: 0.2rem 0.7rem;
    border: 1px solid;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    animation: fadeIn 0.5s ease both;
  }
  .badge-blue { border-color: var(--accent); color: var(--accent); background: rgba(0,212,255,0.07); }
  .badge-green { border-color: var(--accent2); color: var(--accent2); background: rgba(0,255,136,0.07); }
  .badge-red { border-color: var(--danger); color: var(--danger); background: rgba(255,59,59,0.07); }
  .badge-warn { border-color: var(--warn); color: var(--warn); background: rgba(255,170,0,0.07); }

  h1 {
    font-family: var(--font-display);
    font-size: clamp(1.4rem, 3vw, 2rem);
    font-weight: 900;
    color: #fff;
    letter-spacing: 0.04em;
    line-height: 1.2;
    text-shadow: 0 0 40px rgba(0,212,255,0.3);
    animation: fadeInUp 0.6s ease both;
  }

  h1 span { color: var(--accent); }

  .header-sub {
    font-family: var(--font-mono);
    font-size: 0.75rem;
    color: var(--text2);
    margin-top: 0.8rem;
    letter-spacing: 0.05em;
    animation: fadeInUp 0.7s ease both;
  }

  .header-sub span { color: var(--accent); }

  .capture-stats {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.5rem;
    min-width: 220px;
    animation: fadeIn 0.8s ease both;
  }

  .stat-box {
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 0.7rem 1rem;
    text-align: center;
    position: relative;
    overflow: hidden;
  }

  .stat-box::before {
    content: '';
    position: absolute;
    top: 0; left: 0; right: 0;
    height: 2px;
    background: var(--accent);
    opacity: 0.5;
  }

  .stat-val {
    font-family: var(--font-display);
    font-size: 1.1rem;
    font-weight: 700;
    color: var(--accent);
    display: block;
  }

  .stat-lbl {
    font-family: var(--font-mono);
    font-size: 0.6rem;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.1em;
  }

  /* ─── NAV ─────────────────────────────────── */
  nav {
    position: sticky;
    top: 0;
    z-index: 100;
    background: rgba(3,10,14,0.92);
    backdrop-filter: blur(12px);
    border-bottom: 1px solid var(--border);
    padding: 0.6rem 0;
  }

  .nav-inner {
    display: flex;
    gap: 0.3rem;
    overflow-x: auto;
    scrollbar-width: none;
  }

  .nav-inner::-webkit-scrollbar { display: none; }

  .nav-link {
    font-family: var(--font-mono);
    font-size: 0.68rem;
    color: var(--muted);
    text-decoration: none;
    padding: 0.3rem 0.8rem;
    border: 1px solid transparent;
    white-space: nowrap;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    transition: all 0.2s;
  }

  .nav-link:hover {
    color: var(--accent);
    border-color: var(--border);
    background: var(--surface);
  }

  /* ─── SECTIONS ───────────────────────────── */
  .section {
    padding: 3rem 0 2rem;
    border-bottom: 1px solid rgba(15,58,80,0.5);
    opacity: 0;
    transform: translateY(20px);
    transition: opacity 0.6s ease, transform 0.6s ease;
  }

  .section.visible {
    opacity: 1;
    transform: translateY(0);
  }

  .section-header {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 2rem;
  }

  .step-num {
    font-family: var(--font-display);
    font-size: 0.6rem;
    font-weight: 700;
    color: var(--accent);
    background: rgba(0,212,255,0.1);
    border: 1px solid rgba(0,212,255,0.3);
    padding: 0.3rem 0.6rem;
    letter-spacing: 0.1em;
    white-space: nowrap;
  }

  h2 {
    font-family: var(--font-display);
    font-size: 1rem;
    font-weight: 700;
    color: #fff;
    letter-spacing: 0.06em;
  }

  .menu-path {
    font-family: var(--font-mono);
    font-size: 0.7rem;
    color: var(--muted);
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 0.2rem 0.6rem;
    margin-left: auto;
  }

  .menu-path span { color: var(--accent2); }

  /* ─── TABLES ─────────────────────────────── */
  .table-wrap { overflow-x: auto; margin: 1.5rem 0; }

  table {
    width: 100%;
    border-collapse: collapse;
    font-family: var(--font-mono);
    font-size: 0.8rem;
  }

  thead tr {
    background: var(--surface2);
    border-bottom: 2px solid var(--accent);
  }

  th {
    padding: 0.7rem 1rem;
    text-align: left;
    color: var(--accent);
    font-size: 0.65rem;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    font-weight: 400;
  }

  td {
    padding: 0.6rem 1rem;
    border-bottom: 1px solid rgba(15,58,80,0.6);
    color: var(--text);
    vertical-align: top;
  }

  tr:hover td { background: rgba(0,212,255,0.03); }

  .tag {
    display: inline-block;
    font-size: 0.6rem;
    padding: 0.1rem 0.5rem;
    border: 1px solid;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    white-space: nowrap;
  }
  .tag-atk { border-color: var(--danger); color: var(--danger); background: rgba(255,59,59,0.1); }
  .tag-vic { border-color: var(--warn); color: var(--warn); background: rgba(255,170,0,0.1); }
  .tag-int { border-color: var(--muted); color: var(--muted); background: rgba(58,106,128,0.1); }
  .tag-ext { border-color: #666; color: #888; }

  /* ─── FILTER CARDS ───────────────────────── */
  .filter-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 1rem;
    margin: 1.5rem 0;
  }

  .filter-card {
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 1.2rem;
    position: relative;
    overflow: hidden;
    transition: border-color 0.2s, transform 0.2s;
    cursor: default;
  }

  .filter-card:hover {
    border-color: var(--accent);
    transform: translateY(-2px);
  }

  .filter-card::before {
    content: '';
    position: absolute;
    left: 0; top: 0; bottom: 0;
    width: 3px;
    background: var(--accent);
  }

  .filter-card.danger::before { background: var(--danger); }
  .filter-card.warn::before { background: var(--warn); }
  .filter-card.green::before { background: var(--accent2); }

  .filter-id {
    font-family: var(--font-display);
    font-size: 0.6rem;
    color: var(--muted);
    letter-spacing: 0.15em;
    margin-bottom: 0.5rem;
  }

  .filter-expr {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--accent2);
    background: rgba(0,0,0,0.4);
    padding: 0.5rem 0.8rem;
    border: 1px solid rgba(0,255,136,0.15);
    word-break: break-all;
    margin: 0.5rem 0;
    line-height: 1.6;
  }

  .filter-result {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-top: 0.8rem;
  }

  .result-count {
    font-family: var(--font-display);
    font-size: 1.3rem;
    font-weight: 700;
    color: #fff;
  }

  .result-count.zero { color: var(--danger); }
  .result-count.low { color: var(--warn); }
  .result-count.high { color: var(--accent); }

  .result-label {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }

  /* ─── HANDSHAKE DIAGRAM ──────────────────── */
  .diagram-wrap {
    background: var(--surface);
    border: 1px solid var(--border);
    padding: 2rem;
    margin: 1.5rem 0;
    position: relative;
    overflow: hidden;
  }

  .diagram-wrap::after {
    content: 'TCP HANDSHAKE ANALYSIS';
    position: absolute;
    top: 0.6rem; right: 1rem;
    font-family: var(--font-mono);
    font-size: 0.55rem;
    color: var(--muted);
    letter-spacing: 0.15em;
  }

  .handshake-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
  }

  .hs-col-title {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    color: var(--muted);
    text-transform: uppercase;
    letter-spacing: 0.12em;
    margin-bottom: 1rem;
    padding-bottom: 0.5rem;
    border-bottom: 1px solid var(--border);
  }

  .hs-col-title.expected { color: var(--accent2); border-color: rgba(0,255,136,0.3); }
  .hs-col-title.observed { color: var(--danger); border-color: rgba(255,59,59,0.3); }

  .hs-nodes {
    display: flex;
    justify-content: space-between;
    margin-bottom: 0.5rem;
    font-family: var(--font-mono);
    font-size: 0.65rem;
    color: var(--text2);
  }

  .hs-line {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin: 0.6rem 0;
    font-family: var(--font-mono);
    font-size: 0.7rem;
    position: relative;
  }

  .hs-arrow {
    flex: 1;
    height: 1px;
    position: relative;
    display: flex;
    align-items: center;
  }

  .hs-arrow::before {
    content: '';
    position: absolute;
    left: 0; right: 0;
    height: 1px;
    background: currentColor;
  }

  .hs-arrow::after {
    content: '▶';
    position: absolute;
    right: -4px;
    font-size: 0.5rem;
  }

  .hs-arrow.left::after { display: none; }
  .hs-arrow.left::before { content: ''; }
  .hs-arrow.left-arr::before {
    content: '◀';
    position: absolute;
    left: -4px;
    font-size: 0.5rem;
  }

  .arrow-label {
    font-size: 0.62rem;
    white-space: nowrap;
    padding: 0 0.3rem;
    background: var(--surface);
    position: relative;
    z-index: 1;
  }

  .hs-pkt {
    font-family: var(--font-mono);
    font-size: 0.62rem;
    padding: 0.15rem 0.5rem;
    border: 1px solid;
    white-space: nowrap;
  }

  .pkt-syn { border-color: var(--accent); color: var(--accent); }
  .pkt-synack { border-color: var(--accent2); color: var(--accent2); }
  .pkt-ack { border-color: #888; color: #aaa; }
  .pkt-rst { border-color: var(--danger); color: var(--danger); }
  .pkt-miss { border-color: #333; color: #444; font-style: italic; }

  .hs-note {
    font-family: var(--font-mono);
    font-size: 0.62rem;
    color: var(--danger);
    background: rgba(255,59,59,0.06);
    border: 1px solid rgba(255,59,59,0.2);
    padding: 0.4rem 0.7rem;
    margin-top: 1rem;
  }

  /* ─── TIMELINE ───────────────────────────── */
  .timeline {
    position: relative;
    padding-left: 2rem;
    margin: 1.5rem 0;
  }

  .timeline::before {
    content: '';
    position: absolute;
    left: 0.4rem;
    top: 0; bottom: 0;
    width: 1px;
    background: linear-gradient(to bottom, var(--accent), var(--accent2), transparent);
  }

  .tl-item {
    position: relative;
    margin-bottom: 1.5rem;
  }

  .tl-item::before {
    content: '';
    position: absolute;
    left: -1.65rem;
    top: 0.45rem;
    width: 8px; height: 8px;
    background: var(--accent);
    border: 2px solid var(--bg);
    transform: rotate(45deg);
  }

  .tl-item.phase2::before { background: var(--accent2); }

  .tl-phase {
    font-family: var(--font-display);
    font-size: 0.65rem;
    color: var(--accent);
    letter-spacing: 0.12em;
    text-transform: uppercase;
    margin-bottom: 0.3rem;
  }

  .tl-item.phase2 .tl-phase { color: var(--accent2); }

  .tl-time {
    font-family: var(--font-mono);
    font-size: 0.7rem;
    color: var(--warn);
    margin-bottom: 0.3rem;
  }

  .tl-desc {
    font-family: var(--font-body);
    font-size: 0.9rem;
    color: var(--text2);
    line-height: 1.5;
  }

  /* ─── IOC BLOCK ──────────────────────────── */
  .ioc-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 1rem;
    margin: 1.5rem 0;
  }

  .ioc-card {
    background: rgba(255,59,59,0.04);
    border: 1px solid rgba(255,59,59,0.2);
    padding: 1rem 1.2rem;
    position: relative;
  }

  .ioc-card::before {
    content: 'IOC';
    position: absolute;
    top: 0.5rem; right: 0.7rem;
    font-family: var(--font-mono);
    font-size: 0.55rem;
    color: rgba(255,59,59,0.4);
    letter-spacing: 0.15em;
  }

  .ioc-card p {
    font-family: var(--font-body);
    font-size: 0.85rem;
    color: var(--text);
    line-height: 1.5;
  }

  /* ─── CONCLUSION ─────────────────────────── */
  .conclusion-box {
    background: var(--surface);
    border: 1px solid var(--border);
    border-top: 3px solid var(--danger);
    padding: 2rem;
    margin: 1.5rem 0;
  }

  .verdict {
    display: flex;
    align-items: center;
    gap: 1.5rem;
    margin-bottom: 1.5rem;
    padding-bottom: 1.5rem;
    border-bottom: 1px solid var(--border);
  }

  .verdict-icon {
    font-family: var(--font-display);
    font-size: 2rem;
    color: var(--danger);
    text-shadow: 0 0 20px rgba(255,59,59,0.5);
    flex-shrink: 0;
  }

  .verdict-text h3 {
    font-family: var(--font-display);
    font-size: 1rem;
    font-weight: 700;
    color: var(--danger);
    letter-spacing: 0.06em;
    margin-bottom: 0.3rem;
  }

  .verdict-text p {
    font-size: 0.9rem;
    color: var(--text2);
  }

  .sig-box {
    background: rgba(0,0,0,0.5);
    border: 1px solid rgba(0,212,255,0.15);
    padding: 1rem 1.2rem;
    margin: 1rem 0;
  }

  .sig-box p {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--text2);
    line-height: 1.8;
  }

  .sig-box strong { color: var(--accent); }

  /* ─── HOST SUMMARY CARDS ─────────────────── */
  .host-cards {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
    margin: 1.5rem 0;
  }

  .host-card {
    background: var(--surface);
    border: 1px solid;
    padding: 1.2rem 1.5rem;
    position: relative;
    overflow: hidden;
  }

  .host-card.attacker { border-color: rgba(255,59,59,0.4); }
  .host-card.victim { border-color: rgba(255,170,0,0.4); }

  .host-card::before {
    content: '';
    position: absolute;
    inset: 0;
    opacity: 0.03;
  }

  .host-card.attacker::before { background: var(--danger); }
  .host-card.victim::before { background: var(--warn); }

  .host-role {
    font-family: var(--font-display);
    font-size: 0.6rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    margin-bottom: 0.5rem;
  }

  .host-card.attacker .host-role { color: var(--danger); }
  .host-card.victim .host-role { color: var(--warn); }

  .host-ip {
    font-family: var(--font-mono);
    font-size: 1.3rem;
    color: #fff;
    margin-bottom: 0.3rem;
  }

  .host-mac {
    font-family: var(--font-mono);
    font-size: 0.7rem;
    color: var(--muted);
  }

  /* ─── PROTOCOL BAR ───────────────────────── */
  .proto-bars { margin: 1.5rem 0; }

  .proto-row {
    display: flex;
    align-items: center;
    gap: 1rem;
    margin-bottom: 1rem;
  }

  .proto-name {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--text2);
    width: 3rem;
    text-align: right;
  }

  .proto-bar-wrap {
    flex: 1;
    height: 24px;
    background: var(--surface2);
    border: 1px solid var(--border);
    position: relative;
    overflow: hidden;
  }

  .proto-bar {
    height: 100%;
    display: flex;
    align-items: center;
    padding-left: 0.5rem;
    font-family: var(--font-mono);
    font-size: 0.62rem;
    color: rgba(0,0,0,0.8);
    font-weight: bold;
    transition: width 1.5s cubic-bezier(0.16, 1, 0.3, 1);
    position: relative;
  }

  .proto-bar::after {
    content: '';
    position: absolute;
    right: 0; top: 0; bottom: 0;
    width: 2px;
    background: rgba(255,255,255,0.3);
  }

  .bar-tcp { background: linear-gradient(90deg, #0077a8, #00d4ff); }
  .bar-arp { background: linear-gradient(90deg, #005a30, #00ff88); }
  .bar-other { background: linear-gradient(90deg, #2a2a2a, #444); }

  .proto-count {
    font-family: var(--font-mono);
    font-size: 0.7rem;
    color: var(--text2);
    width: 5rem;
  }

  /* ─── UTILITY ─────────────────────────────── */
  .mono { font-family: var(--font-mono); }
  .accent { color: var(--accent); }
  .danger { color: var(--danger); }
  .warn { color: var(--warn); }
  .green { color: var(--accent2); }
  .muted { color: var(--muted); }

  code {
    font-family: var(--font-mono);
    font-size: 0.8em;
    color: var(--accent2);
    background: rgba(0,255,136,0.08);
    padding: 0.1em 0.4em;
    border: 1px solid rgba(0,255,136,0.15);
  }

  p { font-size: 0.95rem; color: var(--text2); line-height: 1.7; margin: 0.5rem 0; }

  h3 {
    font-family: var(--font-display);
    font-size: 0.8rem;
    font-weight: 600;
    color: var(--text);
    letter-spacing: 0.08em;
    margin: 1.5rem 0 0.8rem;
  }

  /* ─── FOOTER ─────────────────────────────── */
  footer {
    padding: 2rem 0;
    border-top: 1px solid var(--border);
    text-align: center;
  }

  footer p {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    color: var(--muted);
    letter-spacing: 0.1em;
  }

  /* ─── ANIMATIONS ─────────────────────────── */
  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  @keyframes fadeInUp {
    from { opacity: 0; transform: translateY(16px); }
    to { opacity: 1; transform: translateY(0); }
  }

  @keyframes pulse-border {
    0%, 100% { border-color: rgba(255,59,59,0.3); }
    50% { border-color: rgba(255,59,59,0.7); }
  }

  .host-card.attacker { animation: pulse-border 3s ease infinite; }

  /* ─── SCROLLBAR ──────────────────────────── */
  ::-webkit-scrollbar { width: 4px; height: 4px; }
  ::-webkit-scrollbar-track { background: var(--bg); }
  ::-webkit-scrollbar-thumb { background: var(--border); }

  /* ─── RESPONSIVE ─────────────────────────── */
  @media (max-width: 640px) {
    .header-grid { grid-template-columns: 1fr; }
    .capture-stats { grid-template-columns: repeat(4, 1fr); }
    .handshake-grid { grid-template-columns: 1fr; }
    .host-cards { grid-template-columns: 1fr; }
  }
</style>
</head>
<body>

<!-- HEADER -->
<header>
  <div class="header-bg"></div>
  <div class="container">
    <div class="header-grid">
      <div>
        <div class="badge-row">
          <span class="badge badge-red">Active Reconnaissance</span>
          <span class="badge badge-blue">Network Forensics</span>
          <span class="badge badge-warn">Half-Open SYN Scan</span>
          <span class="badge badge-green">Wireshark</span>
        </div>
        <h1>Wireshark<br><span>Network Reconnaissance</span><br>Detection Analysis</h1>
        <div class="header-sub">
          Capture: <span>2026-03-05 18:51:02</span> &mdash; <span>2026-03-05 19:00:13</span>
          &nbsp;&bull;&nbsp; Elapsed: <span>00:09:10</span>
        </div>
      </div>
      <div class="capture-stats">
        <div class="stat-box">
          <span class="stat-val">4,819</span>
          <span class="stat-lbl">Packets</span>
        </div>
        <div class="stat-box">
          <span class="stat-val">100%</span>
          <span class="stat-lbl">Displayed</span>
        </div>
        <div class="stat-box">
          <span class="stat-val">2,008</span>
          <span class="stat-lbl">SYN Pkts</span>
        </div>
        <div class="stat-box">
          <span class="stat-val">531</span>
          <span class="stat-lbl">ARP Pkts</span>
        </div>
      </div>
    </div>
  </div>
</header>

<!-- NAV -->
<nav>
  <div class="container">
    <div class="nav-inner">
      <a class="nav-link" href="#s1">01 &mdash; File Props</a>
      <a class="nav-link" href="#s2">02 &mdash; Protocol Hierarchy</a>
      <a class="nav-link" href="#s3">03 &mdash; Conversations</a>
      <a class="nav-link" href="#s4">04 &mdash; Time Settings</a>
      <a class="nav-link" href="#s5">05 &mdash; Display Filters</a>
      <a class="nav-link" href="#s6">Conclusion</a>
    </div>
  </div>
</nav>

<div class="container">

  <!-- STEP 01 -->
  <section class="section" id="s1">
    <div class="section-header">
      <span class="step-num">STEP 01</span>
      <h2>Capture File Properties</h2>
      <div class="menu-path"><span>Statistics</span> → Capture File Properties</div>
    </div>

    <div class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>Property</th>
            <th>Value</th>
          </tr>
        </thead>
        <tbody>
          <tr><td class="muted mono">First Packet</td><td class="accent">2026-03-05 18:51:02</td></tr>
          <tr><td class="muted mono">Last Packet</td><td class="accent">2026-03-05 19:00:13</td></tr>
          <tr><td class="muted mono">Elapsed Time</td><td>00:09:10</td></tr>
          <tr><td class="muted mono">Total Packets</td><td>4,819 captured &nbsp;<span class="badge badge-green">100.0% displayed</span></td></tr>
        </tbody>
      </table>
    </div>
  </section>

  <!-- STEP 02 -->
  <section class="section" id="s2">
    <div class="section-header">
      <span class="step-num">STEP 02</span>
      <h2>Protocol Hierarchy</h2>
      <div class="menu-path"><span>Statistics</span> → Protocol Hierarchy</div>
    </div>

    <p>Primary transport layer: <strong class="accent">IPv4</strong> — 4,283 of 4,819 packets</p>

    <div class="proto-bars">
      <div class="proto-row">
        <span class="proto-name">TCP</span>
        <div class="proto-bar-wrap">
          <div class="proto-bar bar-tcp" style="width:0%" data-target="84.2%">4,059 pkts &mdash; 84.2%</div>
        </div>
        <span class="proto-count accent">4,059</span>
      </div>
      <div class="proto-row">
        <span class="proto-name">ARP</span>
        <div class="proto-bar-wrap">
          <div class="proto-bar bar-arp" style="width:0%" data-target="12.4%">597 pkts &mdash; 12.4%</div>
        </div>
        <span class="proto-count green">597</span>
      </div>
      <div class="proto-row">
        <span class="proto-name">Other</span>
        <div class="proto-bar-wrap">
          <div class="proto-bar bar-other" style="width:0%" data-target="3.4%">163 pkts &mdash; 3.4%</div>
        </div>
        <span class="proto-count muted">163</span>
      </div>
    </div>
  </section>

  <!-- STEP 03 -->
  <section class="section" id="s3">
    <div class="section-header">
      <span class="step-num">STEP 03</span>
      <h2>Conversation Analysis</h2>
      <div class="menu-path"><span>Statistics</span> → Conversations</div>
    </div>

    <div class="host-cards">
      <div class="host-card attacker">
        <div class="host-role">Attacker</div>
        <div class="host-ip">192.168.1.116</div>
        <div class="host-mac">MAC: 08:00:27:63:b0:05</div>
      </div>
      <div class="host-card victim">
        <div class="host-role">Victim</div>
        <div class="host-ip">192.168.1.113</div>
        <div class="host-mac">Internal Host — Target</div>
      </div>
    </div>

    <h3>IPv4 Hosts Identified</h3>
    <div class="table-wrap">
      <table>
        <thead>
          <tr><th>IP Address</th><th>Classification</th></tr>
        </thead>
        <tbody>
          <tr><td class="mono accent">192.168.1.103</td><td><span class="tag tag-int">Internal Host</span></td></tr>
          <tr><td class="mono accent">192.168.1.109</td><td><span class="tag tag-int">Internal Host</span></td></tr>
          <tr><td class="mono danger">192.168.1.113</td><td><span class="tag tag-vic">Victim</span></td></tr>
          <tr><td class="mono danger">192.168.1.116</td><td><span class="tag tag-atk">Attacker</span></td></tr>
          <tr><td class="mono accent">192.168.1.254</td><td><span class="tag tag-int">Gateway</span></td></tr>
          <tr><td class="mono muted">2 external addresses</td><td><span class="tag tag-ext">External</span></td></tr>
        </tbody>
      </table>
    </div>

    <h3>TCP Port Probe Sequence</h3>
    <p>Single-packet probes from <code>192.168.1.116</code> targeting port <code>1</code> on <code>192.168.1.113</code>:</p>

    <div class="table-wrap">
      <table>
        <thead>
          <tr><th>Attacker Source Port</th><th>Victim Destination Port</th><th>Packets</th></tr>
        </thead>
        <tbody>
          <tr><td class="mono">39640</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">39641</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">39642</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">39896</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">39897</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">39898</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono">57214</td><td class="mono accent">1</td><td class="mono muted">1</td></tr>
          <tr><td class="mono warn">41761</td><td class="mono accent">1 — 65,389</td><td class="mono warn">Multiple (60B / 54B)</td></tr>
          <tr><td class="mono warn">41858</td><td class="mono accent">1 — 65,389</td><td class="mono warn">Multiple (60B / 54B)</td></tr>
        </tbody>
      </table>
    </div>
  </section>

  <!-- STEP 04 -->
  <section class="section" id="s4">
    <div class="section-header">
      <span class="step-num">STEP 04</span>
      <h2>Time Settings</h2>
      <div class="menu-path"><span>View</span> → Time Display Format → UTC</div>
    </div>
    <p>
      Switch to <strong class="accent">UTC Date and Time of Day</strong> format for accurate cross-timezone log correlation and incident timeline reconstruction. Enable the delta time column to measure per-packet intervals for burst detection.
    </p>
  </section>

  <!-- STEP 05 -->
  <section class="section" id="s5">
    <div class="section-header">
      <span class="step-num">STEP 05</span>
      <h2>Display Filter Analysis</h2>
    </div>

    <div class="filter-grid">
      <div class="filter-card">
        <div class="filter-id">FILTER 01 — SYN from Attacker</div>
        <div class="filter-expr">ip.src==192.168.1.116 &amp;&amp; tcp.flags.syn==1</div>
        <div class="filter-result">
          <span class="result-count high">2,008</span>
          <span class="result-label">packets</span>
        </div>
      </div>
      <div class="filter-card danger">
        <div class="filter-id">FILTER 02 — SYN-ACK from Victim</div>
        <div class="filter-expr">(ip.src==192.168.1.113) &amp;&amp; (ip.dst==192.168.1.116) &amp;&amp; (tcp.flags.ack==1 &amp;&amp; tcp.flags.syn==1)</div>
        <div class="filter-result">
          <span class="result-count zero">0</span>
          <span class="result-label">packets — no open ports</span>
        </div>
      </div>
      <div class="filter-card warn">
        <div class="filter-id">FILTER 03 — RST-ACK from Victim</div>
        <div class="filter-expr">(ip.src==192.168.1.113) &amp;&amp; (ip.dst==192.168.1.116) &amp;&amp; (tcp.flags.ack==1 &amp;&amp; tcp.flags.reset==1)</div>
        <div class="filter-result">
          <span class="result-count warn">2,012</span>
          <span class="result-label">packets — ports rejected</span>
        </div>
      </div>
      <div class="filter-card green">
        <div class="filter-id">FILTER 04 — ACK from Attacker</div>
        <div class="filter-expr">ip.src==192.168.1.116 &amp;&amp; tcp.flags.ack==1</div>
        <div class="filter-result">
          <span class="result-count low">4</span>
          <span class="result-label">packets — handshake never completed</span>
        </div>
      </div>
      <div class="filter-card" style="grid-column: span 2">
        <div class="filter-id">FILTER 06 — ARP from Attacker MAC</div>
        <div class="filter-expr">eth.addr == 08:00:27:63:b0:05 &amp;&amp; arp</div>
        <div class="filter-result">
          <span class="result-count high">531</span>
          <span class="result-label">ARP packets &nbsp;&bull;&nbsp; timeline: 12:52:25 — 12:53:38</span>
        </div>
      </div>
    </div>

    <h3>TCP Handshake — Expected vs. Observed</h3>

    <div class="diagram-wrap">
      <div class="handshake-grid">
        <div>
          <div class="hs-col-title expected">Expected — Normal Connection</div>
          <div class="hs-nodes"><span>Client (Attacker)</span><span>Server (Victim)</span></div>
          <div class="hs-line">
            <span class="hs-pkt pkt-syn">SYN</span>
            <span class="hs-arrow" style="color:var(--accent)"></span>
          </div>
          <div class="hs-line">
            <span class="hs-arrow left-arr" style="color:var(--accent2); transform:scaleX(-1)"></span>
            <span class="hs-pkt pkt-synack">SYN-ACK</span>
          </div>
          <div class="hs-line">
            <span class="hs-pkt pkt-ack">ACK</span>
            <span class="hs-arrow" style="color:#666"></span>
          </div>
          <div style="font-family:var(--font-mono);font-size:0.65rem;color:var(--accent2);margin-top:0.8rem;padding:0.4rem 0.7rem;border:1px solid rgba(0,255,136,0.2);">
            Connection established
          </div>
        </div>
        <div>
          <div class="hs-col-title observed">Observed — Half-Open Scan</div>
          <div class="hs-nodes"><span>Attacker (116)</span><span>Victim (113)</span></div>
          <div class="hs-line">
            <span class="hs-pkt pkt-syn">SYN x2,008</span>
            <span class="hs-arrow" style="color:var(--danger)"></span>
          </div>
          <div class="hs-line">
            <span class="hs-arrow" style="color:var(--danger);transform:scaleX(-1)"></span>
            <span class="hs-pkt pkt-rst">RST-ACK x2,012</span>
          </div>
          <div class="hs-line">
            <span class="hs-pkt pkt-miss">ACK &mdash; NOT SENT</span>
            <span class="hs-arrow" style="color:#333"></span>
          </div>
          <div class="hs-note">
            Handshake never completed &mdash; stealth scan confirmed
          </div>
        </div>
      </div>
    </div>
  </section>

  <!-- CONCLUSION -->
  <section class="section" id="s6">
    <div class="section-header">
      <span class="step-num">CONCLUSION</span>
      <h2>Attack Assessment</h2>
    </div>

    <div class="conclusion-box">
      <div class="verdict">
        <div class="verdict-icon">!</div>
        <div class="verdict-text">
          <h3>Half-Open SYN Port Scan Detected</h3>
          <p>Consistent with <strong>Nmap -sS</strong> or equivalent stealth scanning tool. No open ports discovered on the victim host.</p>
        </div>
      </div>

      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>Attribute</th><th>Value</th></tr>
          </thead>
          <tbody>
            <tr><td class="muted mono">Attacker IP</td><td class="danger mono">192.168.1.116</td></tr>
            <tr><td class="muted mono">Attacker MAC</td><td class="mono">08:00:27:63:b0:05</td></tr>
            <tr><td class="muted mono">Victim IP</td><td class="warn mono">192.168.1.113</td></tr>
            <tr><td class="muted mono">Attack Type</td><td>Half-Open SYN Port Scan (Stealth)</td></tr>
            <tr><td class="muted mono">Scan Range</td><td>TCP ports 1 — 65,389</td></tr>
            <tr><td class="muted mono">ARP Phase</td><td>531 broadcast requests (host discovery)</td></tr>
            <tr><td class="muted mono">Open Ports Found</td><td class="green">None &mdash; all ports returned RST-ACK</td></tr>
          </tbody>
        </table>
      </div>
    </div>

    <h3>Attack Timeline</h3>
    <div class="timeline">
      <div class="tl-item">
        <div class="tl-phase">Phase 1 — ARP Discovery</div>
        <div class="tl-time">12:52:25 &mdash; 12:53:38</div>
        <div class="tl-desc">192.168.1.116 broadcasts 531 ARP requests targeting 192.168.1.113, resolving the victim's MAC address via the gateway at 192.168.1.254.</div>
      </div>
      <div class="tl-item phase2">
        <div class="tl-phase">Phase 2 — TCP SYN Port Scan</div>
        <div class="tl-time">12:52:33 &mdash; 12:55:08</div>
        <div class="tl-desc">192.168.1.116 sends 2,008 SYN packets sweeping ports 1–65,389. The victim responds with RST-ACK on every port. The attacker never completes the three-way handshake, confirming a half-open stealth scan.</div>
      </div>
    </div>

    <h3>Indicators of Compromise</h3>
    <div class="ioc-grid">
      <div class="ioc-card">
        <p>Sequential SYN packets targeting a wide port range from a single source IP within a compressed time window.</p>
      </div>
      <div class="ioc-card">
        <p>Absence of ACK packets following SYN-ACK responses — TCP handshake is never completed.</p>
      </div>
      <div class="ioc-card">
        <p>High-volume ARP broadcasts immediately preceding TCP scan activity — host discovery before enumeration.</p>
      </div>
      <div class="ioc-card">
        <p>Constant packet size pattern: <strong class="accent">60B outbound / 54B reply</strong> — characteristic of automated scanning tools.</p>
      </div>
    </div>

    <h3>Detection Signatures</h3>
    <div class="sig-box">
      <p><strong>tcp.flags.syn == 1 &amp;&amp; !tcp.flags.ack</strong> from a single source exceeding threshold volume</p>
      <p><strong>Source port increments</strong> (39640, 39641, 39642...) &mdash; sequential socket allocation from scanning tool</p>
      <p><strong>RST-ACK volume</strong> proportional to attacker SYN volume with zero established sessions</p>
      <p><strong>ARP broadcast burst</strong> preceding TCP scan by ~8 seconds &mdash; automated discovery workflow</p>
    </div>
  </section>

</div>

<!-- FOOTER -->
<footer>
  <div class="container">
    <p>Wireshark Network Reconnaissance Detection Analysis &mdash; 2026-03-05 &mdash; Forensic Report</p>
  </div>
</footer>

<script>
  // Intersection observer for section reveal
  const sections = document.querySelectorAll('.section');
  const observer = new IntersectionObserver((entries) => {
    entries.forEach(e => {
      if (e.isIntersecting) {
        e.target.classList.add('visible');
        // Animate protocol bars when step 2 is visible
        if (e.target.id === 's2') {
          setTimeout(() => {
            document.querySelectorAll('.proto-bar').forEach(bar => {
              bar.style.width = bar.dataset.target;
            });
          }, 200);
        }
      }
    });
  }, { threshold: 0.1 });

  sections.forEach(s => observer.observe(s));

  // Active nav link highlight
  const navLinks = document.querySelectorAll('.nav-link');
  const sectionEls = document.querySelectorAll('section[id]');

  window.addEventListener('scroll', () => {
    let current = '';
    sectionEls.forEach(s => {
      if (window.scrollY >= s.offsetTop - 120) current = s.id;
    });
    navLinks.forEach(a => {
      a.style.color = a.getAttribute('href') === '#' + current ? 'var(--accent)' : '';
    });
  });
</script>
</body>
</html>