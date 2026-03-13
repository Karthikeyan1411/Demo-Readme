// Inject global keyframes once
const _style = document.createElement("style");
_style.textContent = `
  @keyframes spin    { to { transform: rotate(360deg); } }
  @keyframes fadeIn  { from { opacity: 0; } to { opacity: 1; } }
  @keyframes slideUp { from { opacity: 0; transform: translateY(28px) scale(0.97); } to { opacity: 1; transform: translateY(0) scale(1); } }
  .fs-btn:focus, .fs-btn:focus-visible { outline: none !important; }
  .close-btn:focus, .close-btn:focus-visible { outline: none !important; }
  .close-btn:hover { background: rgba(248,113,113,0.15) !important; border-color: rgba(248,113,113,0.4) !important; color: #F87171 !important; }
  .fs-btn:hover { background: rgba(148,163,184,0.25) !important; border-color: rgba(148,163,184,0.4) !important; color: #475569 !important; }
  .hide-scroll { scrollbar-width: 5px ; -ms-overflow-style: none; }
  .hide-scroll::-webkit-scrollbar { display: none; }

  /* ── Tablet (≤ 1024px) ── */
  @media (max-width: 1024px) {
    .dash-header { padding: 1rem 1.25rem 0.85rem !important; }
    .dash-title  { font-size: 1.35rem !important; }
    .dash-grid   { grid-template-columns: repeat(2,1fr) !important; padding: 1rem 1.25rem !important; gap: 1rem !important; }
  }

  /* ── Large Mobile / Small Tablet (≤ 768px) ── */
  @media (max-width: 768px) {
    .dash-header {
      flex-wrap: wrap !important;
      gap: 0.6rem !important;
      padding: 0.85rem 1rem 0.75rem !important;
    }
    .dash-header-left  { min-width: unset !important; order: 1; }
    .dash-header-title { order: 2; flex: 1 1 100% !important; text-align: center !important; font-size: 1.2rem !important; }
    .dash-header-right { min-width: unset !important; order: 3; }
    .dash-grid   { grid-template-columns: 1fr !important; padding: 0.85rem 1rem !important; gap: 0.85rem !important; }
    .dash-card   { padding: 0.75rem 0.9rem 0.85rem !important; }
    .dash-chart-inner { height: 220px !important; }
  }

  /* ── Mobile (≤ 480px) ── */
  @media (max-width: 480px) {
    .dash-header {
      flex-direction: column !important;
      align-items: flex-start !important;
      gap: 0.5rem !important;
      padding: 0.75rem 0.85rem !important;
    }
    .dash-header-title { font-size: 1.05rem !important; text-align: left !important; }
    .dash-grid   { padding: 0.75rem 0.85rem !important; gap: 0.75rem !important; }
    .dash-chart-inner { height: 195px !important; }
    .dash-card-title  { font-size: 0.85rem !important; }
    .cal-wrap    { width: calc(100vw - 2rem) !important; max-width: 290px; }
  }
`;
if (!document.head.querySelector("#dash-keyframes")) {
  _style.id = "dash-keyframes";
  document.head.appendChild(_style);
}
