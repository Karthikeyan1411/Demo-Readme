const _style = document.createElement("style");
_style.textContent = `
  @keyframes spin    { to { transform: rotate(360deg); } }
  @keyframes fadeIn  { from { opacity: 0; } to { opacity: 1; } }
  @keyframes slideUp { from { opacity: 0; transform: translateY(28px) scale(0.97); } to { opacity: 1; transform: translateY(0) scale(1); } }
  .fs-btn:focus-visible { outline: none !important; }
  .fs-btn:hover, .close-btn:hover { background: rgba(248,113,113,0.15) !important; border-color: rgba(248,113,113,0.4) !important; color: #F87171 !important; }
  .hide-scroll { scrollbar-width: 5px ; -ms-overflow-style: none; }
  .hide-scroll::-webkit-scrollbar { display: none; }

  /* ── Responsive ── */
  .dash-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 1.25rem; padding: 1.25rem 2rem; }
  .dash-header { display: flex; justify-content: space-between; align-items: center; padding: 1.25rem 2rem 1rem; border-bottom: 1px solid #EA457F; }
  .dash-title { margin: 0; font-weight: 600; color: #35335D; letter-spacing: -0.02em; font-size: calc(1.325rem + .9vw); text-align: center; flex: 1; }

  /* Tablet: 768px - 1024px */
  @media (max-width: 1024px) {
    .dash-grid { padding: 1rem 1.25rem; gap: 1rem; }
    .dash-header { padding: 1rem 1.25rem 0.85rem; }
    .dash-title { font-size: 1.4rem; }
    .dash-chart-inner { height: 220px !important; }
  }

  /* Mobile: up to 767px — single column */
  @media (max-width: 767px) {
    .dash-grid { grid-template-columns: 1fr !important; padding: 0.75rem 1rem; gap: 0.85rem; }
    .dash-header { flex-direction: column; align-items: flex-start; gap: 0.75rem; padding: 0.85rem 1rem; }
    .dash-title { font-size: 1.2rem; text-align: left; flex: none; }
    .dash-chart-inner { height: 200px !important; }
    .dash-card { padding: 0.75rem 0.9rem 0.85rem !important; }
    .dash-modal-panel { width: 96vw !important; height: 92vh !important; }
    .dash-modal-body { flex-direction: column !important; }
    .dash-modal-chart { flex: 0 0 45% !important; padding: 0.75rem !important; }
    .dash-modal-divider { width: 100% !important; height: 1px !important; }
    .dash-modal-table { flex: 1 !important; }
    .dash-date-picker { width: 100%; }
  }

  /* Small mobile: up to 480px */
  @media (max-width: 480px) {
    .dash-title { font-size: 1rem; }
    .dash-chart-inner { height: 170px !important; }
    .dash-header-right { flex-wrap: wrap; gap: 0.5rem; }
  }
`;
if (!document.head.querySelector("#dash-keyframes")) {
  _style.id = "dash-keyframes";
  document.head.appendChild(_style);
}
