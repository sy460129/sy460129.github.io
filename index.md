---
layout: page
title: whoami
permalink: /
---

<!-- 깜빡이는 커서 · 터미널 텍스트 · CVE 보드 스타일 -->
<style>
  .term-symbol {
    color: #ffffff !important;
    font-weight: 600 !important;
  }
  .term-cmd {
    color: #7dd3fc !important;
    font-weight: 600 !important;
  }
  .blink {
    display: inline-block !important;
    width: 8px !important;
    height: 0.95rem !important;
    background-color: #ffffff !important;
    margin-left: 6px !important;
    vertical-align: middle !important;
    animation: blinking 1.2s infinite !important;
  }
  @keyframes blinking {
    0%, 49% { opacity: 0.9; }
    50%, 100% { opacity: 0; }
  }

  /* CVE 보드 (스크린샷 느낌의 표) */
  .cve-board { overflow-x: auto; margin: 1rem 0 0.5rem; }
  .cve-table {
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;
    font-size: 0.9rem;
  }
  .cve-table thead th {
    text-align: left;
    font-weight: 600;
    font-size: 0.72rem;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--text-muted-color, #8a8a8a);
    padding: 0.5rem 0.9rem;
    border-bottom: 1px solid var(--main-border-color, #e6e6e6);
    white-space: nowrap;
  }
  .cve-table tbody td {
    padding: 0.68rem 0.9rem;
    border-bottom: 1px solid var(--main-border-color, #eee);
    vertical-align: middle;
  }
  .cve-table tbody tr:last-child td { border-bottom: none; }
  .cve-table tbody tr { transition: background 0.12s ease; }
  .cve-table tbody tr:hover { background: rgba(128, 128, 128, 0.07); }
  .cve-table .cid {
    font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
    font-weight: 600;
    white-space: nowrap;
  }
  .cve-table .cid a { text-decoration: none; }
  .sev {
    display: inline-block;
    padding: 0.15rem 0.5rem;
    border-radius: 6px;
    font-size: 0.7rem;
    font-weight: 700;
    white-space: nowrap;
  }
  .sev-crit { background: rgba(239, 68, 68, 0.18); color: #ef4444; }
  .sev-high { background: rgba(249, 115, 22, 0.18); color: #f97316; }
  .sev-med  { background: rgba(234, 179, 8, 0.22);  color: #ca8a04; }
  .sev-low  { background: rgba(34, 197, 94, 0.18);  color: #22c55e; }
</style>

# <span class="term-symbol">$</span> <span class="term-cmd">uname -a</span><span class="blink"></span>
> * `Seungyeon Park (sy46)`
> * `Inha University Computer Engineering`
> * `GPA`  `4.18/4.5`
> * `OPIc IH` (26.01)

---

# <span class="term-symbol">$</span> <span class="term-cmd">Published CVE</span>

<div class="cve-board" markdown="0">
<table class="cve-table">
  <thead>
    <tr>
      <th>CVE</th>
      <th>Severity</th>
      <th>Weakness</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="cid"><a href="https://www.cve.org/CVERecord?id=CVE-2025-51005">CVE-2025-51005</a></td>
      <td><span class="sev sev-high">7.5 · High</span></td>
      <td>CWE-122</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.cve.org/CVERecord?id=CVE-2025-51006">CVE-2025-51006</a></td>
      <td><span class="sev sev-high">7.8 · High</span></td>
      <td>CWE-415</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.cve.org/CVERecord?id=CVE-2026-27625">CVE-2026-27625</a></td>
      <td><span class="sev sev-high">8.1 · High</span></td>
      <td>CWE-22, CWE-23</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.cve.org/CVERecord?id=CVE-2026-27691">CVE-2026-27691</a></td>
      <td><span class="sev sev-med">6.2 · Medium</span></td>
      <td>CWE-190, CWE-681</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.cve.org/CVERecord?id=CVE-2026-28354">CVE-2026-28354</a></td>
      <td><span class="sev sev-med">5.7 · Medium</span></td>
      <td>CWE-639, CWE-863</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.zerodayinitiative.com/advisories/ZDI-26-507/">[Pwn2Own] ZDI-CAN-29108 · CVE-2026-44095</a></td>
      <td><span class="sev sev-high">7.8 · High</span></td>
      <td>CWE-78</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.zerodayinitiative.com/advisories/ZDI-26-508/">[Pwn2Own] ZDI-CAN-29109 · CVE-2026-44096</a></td>
      <td><span class="sev sev-high">7.8 · High</span></td>
      <td>CWE-78</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://www.zerodayinitiative.com/advisories/ZDI-26-513/">[Pwn2Own] ZDI-CAN-29110 · CVE-2026-44097</a></td>
      <td><span class="sev sev-low">2.4 · Low</span></td>
      <td>CWE-434</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://assets.phoenixcontact.com/file/e789620a-b5fb-4618-9374-86c78948f26a/media/original?pcsa-2026-00006_vde-2026-008.pdf">[Pwn2Own] CVE-2026-44107</a></td>
      <td><span class="sev sev-med">6.5 · Medium</span></td>
      <td>CWE-749</td>
    </tr>
    <tr>
      <td class="cid"><a href="https://assets.phoenixcontact.com/file/e789620a-b5fb-4618-9374-86c78948f26a/media/original?pcsa-2026-00006_vde-2026-008.pdf">[Pwn2Own] CVE-2026-44108</a></td>
      <td><span class="sev sev-med">6.4 · Medium</span></td>
      <td>CWE-696</td>
    </tr>
  </tbody>
</table>
</div>

---

# <span class="term-symbol">$</span> <span class="term-cmd">history</span>
> * `Inha Univ. Computer Science & Engineering` **(2020.03 ~ 2027.02)** 
> * `WhiteHat School 2nd` **(2024.03 ~ 2024.09)**
> * `BoB 14th Vulnerability Analysis Track` **(2025.07 ~ 2026.02)**

---

# <span class="term-symbol">$</span> <span class="term-cmd">Awards</span>
> * `Team BoB::Takedown | Pwn2own Automotive 2026 Winner (Charx-SEC-31XX, Grizzl-e)`
> * `Team Three-Piece | Inha Univ Capstone`
