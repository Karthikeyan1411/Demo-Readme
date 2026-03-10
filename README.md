import { useState, useEffect, useRef } from "react";
import { Line, Pie } from "react-chartjs-2";
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  ArcElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from "chart.js";

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  ArcElement,
  Title,
  Tooltip,
  Legend,
  Filler,
);

// Inject global keyframes once
const _style = document.createElement("style");
// .fs-btn:hover  { background: rgba(45,212,191,0.15) !important; border-color: rgba(45,212,191,0.4) !important; color: #2DD4BF !important; }
_style.textContent = `
  @keyframes spin    { to { transform: rotate(360deg); } }
  @keyframes fadeIn  { from { opacity: 0; } to { opacity: 1; } }
  @keyframes slideUp { from { opacity: 0; transform: translateY(28px) scale(0.97); } to { opacity: 1; transform: translateY(0) scale(1); } }
  .close-btn:hover { background: rgba(248,113,113,0.15) !important; border-color: rgba(248,113,113,0.4) !important; color: #F87171 !important; }
  .hide-scroll { scrollbar-width: 5px ; -ms-overflow-style: none; }
  .hide-scroll::-webkit-scrollbar { display: none; }
`;
if (!document.head.querySelector("#dash-keyframes")) {
  _style.id = "dash-keyframes";
  document.head.appendChild(_style);
}

const CATEGORY_COLORS = {
  "Food Waste Gen": "#4CAF50",
  "Dry Waste Gen": "#2196F3",
  "Garden Waste Gen": "#808000",
  "Coconut Shell Gen": "#8B4513",
  "Tender Coconut Gen": "#FF9800",
  "Bio Hazardous Waste Gen": "#FF6B6B",
  "C&D Gen": "#E0E0E0",
  "Sugarcane Trash Gen": "#9C27B0",
  "Direct Residual Gen": "#C62828",
  "Residual Food Waste": "#E57373",
  "Residual Dry Waste": "#EF9A9A",
};

const CATEGORY_COLOR_VALUES = Object.values(CATEGORY_COLORS);

function getCategoryColor(label, index) {
  return (
    CATEGORY_COLORS[label] ??
    CATEGORY_COLOR_VALUES[index % CATEGORY_COLOR_VALUES.length]
  );
}

// ─── Date helpers ──────────────────────────────────────────────────────────────
const MONTHS = [
  "Jan",
  "Feb",
  "Mar",
  "Apr",
  "May",
  "Jun",
  "Jul",
  "Aug",
  "Sep",
  "Oct",
  "Nov",
  "Dec",
];
const DAYS = ["Su", "Mo", "Tu", "We", "Th", "Fr", "Sa"];

function parseRecordDate(str) {
  if (!str || str === "Unknown") return null;
  const [day, mon, yr] = str.split("-");
  return new Date(`${mon} ${day} ${yr}`);
}

function formatDisplay(date) {
  if (!date) return "";
  return `${String(date.getDate()).padStart(2, "0")} ${MONTHS[date.getMonth()]} ${date.getFullYear()}`;
}

function sameDay(a, b) {
  return (
    a &&
    b &&
    a.getFullYear() === b.getFullYear() &&
    a.getMonth() === b.getMonth() &&
    a.getDate() === b.getDate()
  );
}

function inRange(d, start, end) {
  if (!d || !start || !end) return false;
  const t = d.getTime(),
    s = start.getTime(),
    e = end.getTime();
  return t >= Math.min(s, e) && t <= Math.max(s, e);
}

function filterByDateRange(records, dateRange) {
  if (!dateRange.start) return records;
  return records.filter((r) => {
    const d = parseRecordDate(r.Date_field);
    if (!d) return false;
    const s = dateRange.start.getTime();
    const e = (dateRange.end || dateRange.start).getTime();
    const t = d.getTime();
    return t >= Math.min(s, e) && t <= Math.max(s, e) + 86400000 - 1;
  });
}

// ─── Range Calendar ────────────────────────────────────────────────────────────
function RangeCalendar({ onApply, initialStart, initialEnd }) {
  const today = new Date();
  const [viewYear, setViewYear] = useState(
    initialStart ? initialStart.getFullYear() : today.getFullYear(),
  );
  const [viewMonth, setViewMonth] = useState(
    initialStart ? initialStart.getMonth() : today.getMonth(),
  );
  const [selecting, setSelecting] = useState(null);
  const [hovered, setHovered] = useState(null);
  const [start, setStart] = useState(initialStart || null);
  const [end, setEnd] = useState(initialEnd || null);

  function prevMonth() {
    if (viewMonth === 0) {
      setViewMonth(11);
      setViewYear((y) => y - 1);
    } else setViewMonth((m) => m - 1);
  }
  function nextMonth() {
    if (viewMonth === 11) {
      setViewMonth(0);
      setViewYear((y) => y + 1);
    } else setViewMonth((m) => m + 1);
  }

  function handleDay(d) {
    if (!selecting) {
      // First click: start selection, immediately apply as single date
      setSelecting(d);
      setStart(d);
      setEnd(null);
      onApply(d, d);
    } else {
      // Second click: complete the range, auto-apply
      const s = selecting < d ? selecting : d;
      const e = selecting < d ? d : selecting;
      setStart(s);
      setEnd(e);
      setSelecting(null);
      onApply(s, e);
    }
  }

  const firstDay = new Date(viewYear, viewMonth, 1).getDay();
  const daysInMonth = new Date(viewYear, viewMonth + 1, 0).getDate();
  const cells = [];
  for (let i = 0; i < firstDay; i++) cells.push(null);
  for (let d = 1; d <= daysInMonth; d++)
    cells.push(new Date(viewYear, viewMonth, d));

  const previewEnd =
    selecting && hovered ? (hovered > selecting ? hovered : selecting) : end;
  const previewStart =
    selecting && hovered ? (hovered < selecting ? hovered : selecting) : start;

  return (
    <div style={cal.wrap}>
      <div style={cal.nav}>
        <button style={cal.navBtn} onClick={prevMonth}>
          ‹
        </button>
        <span style={cal.navTitle}>
          {MONTHS[viewMonth]} {viewYear}
        </span>
        <button style={cal.navBtn} onClick={nextMonth}>
          ›
        </button>
      </div>
      <div style={cal.grid}>
        {DAYS.map((d) => (
          <div key={d} style={cal.dayLabel}>
            {d}
          </div>
        ))}
        {cells.map((d, i) => {
          if (!d) return <div key={"e" + i} />;
          const isStart = sameDay(d, start),
            isEnd = sameDay(d, previewEnd),
            isToday = sameDay(d, today);
          const isInRange =
            inRange(d, previewStart, previewEnd) && !isStart && !isEnd;
          const isSelecting = sameDay(d, selecting);
          return (
            <div
              key={i}
              onClick={() => handleDay(d)}
              onMouseEnter={() => setHovered(d)}
              onMouseLeave={() => setHovered(null)}
              style={{
                ...cal.day,
                ...(isStart || isSelecting ? cal.dayStart : {}),
                ...(isEnd && !isSelecting ? cal.dayEnd : {}),
                ...(isInRange ? cal.dayRange : {}),
                ...(isToday && !isStart && !isEnd ? cal.dayToday : {}),
              }}
            >
              {d.getDate()}
            </div>
          );
        })}
      </div>
      <div style={cal.selectedLabel}>
        {selecting && (
          <span style={{ color: "#2DD4BF" }}>Now select end date…</span>
        )}
        {start && end && !selecting && (
          <span>
            <b>{formatDisplay(start)}</b> → <b>{formatDisplay(end)}</b>
          </span>
        )}
        {start && !end && !selecting && (
          <span>
            <b>{formatDisplay(start)}</b>
          </span>
        )}
        {!start && (
          <span style={{ color: "#475569" }}>Click a date to select</span>
        )}
      </div>
    </div>
  );
}

const cal = {
  wrap: {
    background: "#1e293b",
    border: "1px solid rgba(45,212,191,0.25)",
    borderRadius: "0.75rem",
    padding: "1rem",
    width: 280,
    boxShadow: "0 8px 40px rgba(0,0,0,0.6)",
    fontFamily: "'DM Sans',sans-serif",
  },
  nav: {
    display: "flex",
    alignItems: "center",
    justifyContent: "space-between",
    marginBottom: "0.6rem",
  },
  navBtn: {
    background: "none",
    border: "none",
    color: "#94a3b8",
    cursor: "pointer",
    fontSize: "1.2rem",
    padding: "0 0.4rem",
    lineHeight: 1,
  },
  navTitle: { color: "#f1f5f9", fontWeight: 600, fontSize: "0.9rem" },
  grid: {
    display: "grid",
    gridTemplateColumns: "repeat(7,1fr)",
    gap: "2px",
    marginBottom: "0.5rem",
  },
  dayLabel: {
    textAlign: "center",
    color: "#475569",
    fontSize: "0.7rem",
    padding: "4px 0",
    userSelect: "none",
  },
  day: {
    textAlign: "center",
    fontSize: "0.78rem",
    padding: "5px 2px",
    borderRadius: "6px",
    cursor: "pointer",
    color: "#94a3b8",
    userSelect: "none",
    transition: "background 0.1s,color 0.1s",
  },
  dayStart: {
    background: "#2DD4BF",
    color: "#0f172a",
    fontWeight: 700,
    borderRadius: "6px",
  },
  dayEnd: {
    background: "#0891b2",
    color: "#fff",
    fontWeight: 700,
    borderRadius: "6px",
  },
  dayRange: {
    background: "rgba(45,212,191,0.15)",
    color: "#2DD4BF",
    borderRadius: "0",
  },
  dayToday: { border: "1px solid rgba(45,212,191,0.5)", color: "#2DD4BF" },
  selectedLabel: {
    fontSize: "0.75rem",
    color: "#94a3b8",
    textAlign: "center",
    minHeight: "1.4rem",
    marginTop: "0.4rem",
  },
};

// ─── Date Picker Button + Popover ──────────────────────────────────────────────
function DateRangePicker({ dateRange, onApply }) {
  const [open, setOpen] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    function handler(e) {
      if (ref.current && !ref.current.contains(e.target)) setOpen(false);
    }
    document.addEventListener("mousedown", handler);
    return () => document.removeEventListener("mousedown", handler);
  }, []);

  const label = dateRange.start
    ? sameDay(dateRange.start, dateRange.end)
      ? formatDisplay(dateRange.start)
      : `${formatDisplay(dateRange.start)} → ${formatDisplay(dateRange.end)}`
    : formatDisplay(new Date());

  return (
    <div ref={ref} style={{ position: "relative" }}>
      <button
        onClick={() => setOpen((o) => !o)}
        style={{
          display: "flex",
          alignItems: "center",
          gap: "0.5rem",
          background: dateRange.start
            ? "rgba(45,212,191,0.15)"
            : "rgba(30,41,59,0.9)",
          border: `1px solid ${dateRange.start ? "rgba(45,212,191,0.5)" : "rgba(148,163,184,0.2)"}`,
          color: dateRange.start ? "#2DD4BF" : "#94a3b8",
          borderRadius: "8px",
          padding: "0.4rem 0.9rem",
          cursor: "pointer",
          fontFamily: "'DM Sans',sans-serif",
          fontSize: "0.82rem",
          fontWeight: 500,
          transition: "all 0.2s",
        }}
      >
        <span>📅</span>
        <span>{label}</span>
        <span style={{ fontSize: "0.65rem", opacity: 0.6 }}>
          {open ? "▲" : "▼"}
        </span>
      </button>
      {open && (
        <div
          style={{
            position: "absolute",
            top: "calc(100% + 8px)",
            right: 0,
            zIndex: 1000,
          }}
        >
          <RangeCalendar
            initialStart={dateRange.start}
            initialEnd={dateRange.end}
            onApply={(s, e) => {
              onApply(s, e);
            }}
          />
        </div>
      )}
    </div>
  );
}

function processData(records) {
  const byDate = {};
  records.forEach((r) => {
    const date = r.Date_field || "Unknown";
    byDate[date] = (byDate[date] || 0) + parseFloat(r.Total_Quantity_Kg || 0);
  });
  const sortedDates = Object.keys(byDate).sort((a, b) => {
    const parse = (d) => {
      const [day, mon, yr] = d.split("-");
      return new Date(`${mon} ${day} ${yr}`);
    };
    return parse(a) - parse(b);
  });

  // Build M-N labels: last date = "M", second-to-last = "M-1", etc.
  const n = sortedDates.length;
  const mLabels = sortedDates.map((_, i) => {
    const diff = n - 1 - i;
    return diff === 0 ? "M" : `M-${diff}`;
  });

  const byWaste = {},
    byMaterial = {},
    byRegister = {};
  records.forEach((r) => {
    const cat = r.Waste_Category || "Unknown";
    byWaste[cat] = (byWaste[cat] || 0) + parseFloat(r.Total_Quantity_Kg || 0);
    const mat = r.Material_Waste_Category?.Category || "Unknown";
    byMaterial[mat] =
      (byMaterial[mat] || 0) + parseFloat(r.Total_Quantity_Kg || 0);
    const rt = r.Register_Type || "Unknown";
    byRegister[rt] =
      (byRegister[rt] || 0) + parseFloat(r.Total_Quantity_Kg || 0);
  });
  return {
    line: {
      labels: mLabels,
      realDates: sortedDates,
      values: sortedDates.map((d) => byDate[d]),
    },
    waste: { labels: Object.keys(byWaste), values: Object.values(byWaste) },
    material: {
      labels: Object.keys(byMaterial),
      values: Object.values(byMaterial),
    },
    register: {
      labels: Object.keys(byRegister),
      values: Object.values(byRegister),
    },
  };
}

function buildLineData(labels, values, hidden, realDates = []) {
  return {
    labels,
    datasets: [
      {
        label: "Total Quantity (kg)",
        data: labels.map((l, i) => (hidden.has(l) ? null : values[i])),
        realDates, // stored for tooltip access
        // borderColor: "#2DD4BF",
        backgroundColor: (ctx) => {
          const { chartArea, ctx: c } = ctx.chart;
          if (!chartArea) return "rgba(45,212,191,0.3)";
          const g = c.createLinearGradient(
            0,
            chartArea.top,
            0,
            chartArea.bottom,
          );
          g.addColorStop(0, "rgba(45,212,191,0.55)");
          g.addColorStop(0.6, "rgba(45,212,191,0.15)");
          g.addColorStop(1, "rgba(45,212,191,0.01)");
          return g;
        },
        pointBackgroundColor: labels.map((l) =>
          hidden.has(l) ? "transparent" : "#2DD4BF",
        ),
        pointBorderColor: labels.map((l) =>
          hidden.has(l) ? "transparent" : "#0f172a",
        ),
        pointBorderWidth: 2,
        pointRadius: 5,
        pointHoverRadius: 8,
        tension: 0.4,
        fill: "origin",
        borderWidth: 2.5,
        spanGaps: true,
      },
    ],
  };
}

function buildPieData(labels, values, hidden, colorOffset = 0) {
  const vis = labels.filter((l) => !hidden.has(l));
  return {
    labels: vis,
    datasets: [
      {
        data: vis.map((l) => values[labels.indexOf(l)]),
        backgroundColor: vis.map((l, i) =>
          getCategoryColor(l, i + colorOffset),
        ),
        borderWidth: 2,
        // borderColor: "#0f172a",
        datalabels: { display: false },
      },
    ],
  };
}

// Custom plugin: draws percentage labels directly on pie slices
const piePercentLabelPlugin = {
  id: "piePercentLabels",
  afterDatasetsDraw(chart) {
    // Only run on pie/doughnut charts
    if (chart.config.type !== "pie" && chart.config.type !== "doughnut") return;

    const { ctx, data } = chart;
    const dataset = data.datasets[0];
    if (!dataset) return;

    const total = dataset.data.reduce((a, b) => a + b, 0);
    const meta = chart.getDatasetMeta(0);

    meta.data.forEach((arc, i) => {
      const value = dataset.data[i];
      if (!value || value === 0) return;

      const pct = (value / total) * 100;
      // Skip tiny slices to avoid clutter (< 3%)
      if (pct < 3) return;

      const label = pct.toFixed(1) + "%";
      const { x, y } = arc.tooltipPosition();

      ctx.save();
      ctx.font = `bold ${pct < 6 ? 9 : 11}px 'DM Sans', sans-serif`;
      ctx.fillStyle = "#ffffff";
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.shadowColor = "rgba(0,0,0,0.7)";
      ctx.shadowBlur = 4;
      ctx.fillText(label, x, y);
      ctx.restore();
    });
  },
};

// Register the plugin globally
ChartJS.register(piePercentLabelPlugin);

const sharedPieOptions = (title) => ({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    title: {
      display: true,
      text: title,
      color: "#484575",
      font: { size: 14, weight: "600", family: "'DM Sans', sans-serif" },
      padding: { bottom: 10 },
    },
    tooltip: {
      callbacks: {
        label: (ctx) => {
          const total = ctx.dataset.data.reduce((a, b) => a + b, 0);
          const pct =
            total > 0 ? ((ctx.parsed / total) * 100).toFixed(1) : "0.0";
          return ` ${ctx.label}: ${pct}%`;
        },
      },
    },
    // Disable chartjs-plugin-datalabels globally for pie charts
    datalabels: { display: false },
    piePercentLabels: {},
  },
});

const lineOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    datalabels: { display: false },
    title: {
      display: true,
      text: "Waste Quantity Trend",
      color: "#484575",
      font: { size: 14, weight: "600", family: "'DM Sans', sans-serif" },
      padding: { bottom: 10 },
    },
    tooltip: {
      callbacks: {
        title: (items) => {
          const idx = items[0]?.dataIndex;
          const rd = items[0]?.dataset?.realDates;
          const real = rd && rd[idx] ? rd[idx] : items[0]?.label;
          return real;
        },
        label: (ctx) =>
          ctx.parsed.y !== null ? ` ${ctx.parsed.y.toFixed(2)} kg` : " Hidden",
      },
    },
  },
  scales: {
    x: {
      ticks: { color: "#94a3b8", font: { size: 10 }, maxRotation: 0 },
      grid: { color: "rgba(148,163,184,0.08)" },
    },
    y: {
      ticks: { color: "#94a3b8", font: { size: 9 } },
      grid: { color: "rgba(148,163,184,0.08)" },
    },
  },
};

// ─── Fullscreen Icon SVG ───────────────────────────────────────────────────────
function FullscreenIcon() {
  return (
    <svg
      width="15"
      height="15"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth="2"
      strokeLinecap="round"
      strokeLinejoin="round"
    >
      <polyline points="15 3 21 3 21 9" />
      <polyline points="9 21 3 21 3 15" />
      <line x1="21" y1="3" x2="14" y2="10" />
      <line x1="3" y1="21" x2="10" y2="14" />
    </svg>
  );
}

function CloseIcon() {
  return (
    <svg
      width="18"
      height="18"
      viewBox="0 0 24 24"
      fill="none"
      stroke="currentColor"
      strokeWidth="2.5"
      strokeLinecap="round"
    >
      <line x1="18" y1="6" x2="6" y2="18" />
      <line x1="6" y1="6" x2="18" y2="18" />
    </svg>
  );
}

// ─── Shared Table ──────────────────────────────────────────────────────────────
function DataTable({
  labels,
  values,
  hidden,
  onToggle,
  colorOffset = 0,
  compact = false,
  isPie = false,
  displayLabels = null,
}) {
  // Always show Kg(%) — for line chart use total of all values, for pie same
  const total = values.reduce((a, b) => a + b, 0);
  // displayLabels overrides what's shown in the Label column (used for line chart to show real dates)
  const shownLabels = displayLabels || labels;

  return (
    <div
      className="hide-scroll"
      style={{ ...styles.tableWrapper, maxHeight: compact ? "190px" : "100%" }}
    >
      <table style={styles.table}>
        <thead>
          <tr>
            {/* <th style={styles.th}></th> */}
            {/* <th style={styles.th}>Label</th> */}
            {/* <th style={{ ...styles.th, textAlign: "right" }}>Kg (%)</th> */}
          </tr>
        </thead>
        <tbody>
          {labels.map((label, i) => {
            const isHidden = hidden.has(label);
            const pct =
              total > 0 ? ((values[i] / total) * 100).toFixed(1) : "0.0";
            return (
              <tr
                key={label}
                onClick={() => onToggle(label)}
                style={{
                  ...(i % 2 === 0 ? styles.trEven : styles.trOdd),
                  cursor: "pointer",
                  opacity: isHidden ? 0.35 : 1,
                  transition: "opacity 0.2s, background 0.15s",
                }}
                onMouseEnter={(e) => {
                  e.currentTarget.style.background = "#777777";
                }}
                onMouseLeave={(e) => {
                  e.currentTarget.style.background = "#ffffff";
                }}
              >
                <td style={{ ...styles.td, width: 28 }}>
                  <span
                    style={{
                      display: "inline-block",
                      width: 10,
                      height: 10,
                      borderRadius: "50%",
                      background: isHidden
                        ? "#db1515ff"
                        : getCategoryColor(label, i + colorOffset),
                      transition: "background 0.2s",
                    }}
                  />
                </td>
                <td
                  style={{
                    ...styles.td,
                    textDecoration: isHidden ? "line-through" : "none",
                    color: isHidden ? "#475569" : "#414141",
                  }}
                >
                  {shownLabels[i]}
                </td>
                <td
                  style={{
                    ...styles.td,
                    textAlign: "right",
                    color: isHidden ? "#475569" : "#414141",
                  }}
                >
                  {values[i].toFixed(2)}{" "}
                  <span
                    style={{
                      color: isHidden ? "#334155" : "#2DD4BF",
                      fontWeight: 600,
                      fontSize: "0.72rem",
                    }}
                  >
                    ({pct}%)
                  </span>
                </td>
              </tr>
            );
          })}
        </tbody>
      </table>
    </div>
  );
}

// ─── Fullscreen Modal ──────────────────────────────────────────────────────────
function FullscreenModal({
  title,
  chartType,
  labels,
  values,
  hidden,
  onToggle,
  colorOffset,
  onClose,
  realDates = [],
}) {
  // Close on Escape key
  useEffect(() => {
    const handler = (e) => {
      if (e.key === "Escape") onClose();
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
  }, [onClose]);

  // Prevent body scroll while open
  useEffect(() => {
    document.body.style.overflow = "hidden";
    return () => {
      document.body.style.overflow = "";
    };
  }, []);

  const chartData =
    chartType === "line"
      ? buildLineData(labels, values, hidden, realDates)
      : buildPieData(labels, values, hidden, colorOffset);

  const chartOptions =
    chartType === "line" ? lineOptions : sharedPieOptions(title);

  const ChartComponent = chartType === "line" ? Line : Pie;

  return (
    <div
      style={modal.overlay}
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose();
      }}
    >
      <div style={modal.panel}>
        {/* Modal header */}
        <div style={modal.header}>
          <h2 style={modal.title}>{title}</h2>
          <button
            className="close-btn"
            style={modal.closeBtn}
            onClick={onClose}
            title="Close (Esc)"
          >
            <CloseIcon />
          </button>
        </div>

        {/* Modal body: chart left, table right */}
        <div style={modal.body}>
          {/* Chart side */}
          <div style={modal.chartSide}>
            <div
              style={{ position: "relative", height: "100%", width: "100%" }}
            >
              <ChartComponent data={chartData} options={chartOptions} />
            </div>
          </div>

          {/* Divider */}
          <div style={modal.divider} />

          {/* Table side */}
          <div style={modal.tableSide}>
            <DataTable
              labels={labels}
              values={values}
              hidden={hidden}
              onToggle={onToggle}
              colorOffset={colorOffset}
              compact={false}
              isPie={chartType !== "line"}
              displayLabels={
                chartType === "line" && realDates.length ? realDates : null
              }
            />
          </div>
        </div>
      </div>
    </div>
  );
}

const modal = {
  overlay: {
    position: "fixed",
    inset: 0,
    zIndex: 9999,
    background: "#ffffff",
    backdropFilter: "blur(6px)",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    padding: "1.5rem",
    animation: "fadeIn 0.18s ease",
  },
  panel: {
    background: "#ffffff",
    border: "1px solid rgba(45,212,191,0.2)",
    borderRadius: "1.25rem",
    boxShadow: "0 24px 80px rgba(0,0,0,0.7), 0 0 0 1px rgba(45,212,191,0.08)",
    width: "min(92vw, 1100px)",
    height: "min(88vh, 680px)",
    display: "flex",
    flexDirection: "column",
    overflow: "hidden",
    animation: "slideUp 0.22s cubic-bezier(0.16,1,0.3,1)",
  },
  header: {
    display: "flex",
    alignItems: "center",
    justifyContent: "space-between",
    padding: "1rem 1.5rem",
    borderBottom: "1px solid rgba(148,163,184,0.1)",
    flexShrink: 0,
  },
  title: {
    margin: 0,
    color: "#484575",
    fontSize: "1.05rem",
    fontWeight: 700,
    fontFamily: "'DM Sans', sans-serif",
    letterSpacing: "-0.01em",
  },
  closeBtn: {
    background: "rgba(148,163,184,0.1)",
    border: "1px solid rgba(148,163,184,0.15)",
    color: "#94a3b8",
    borderRadius: "8px",
    width: 34,
    height: 34,
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    cursor: "pointer",
    transition: "all 0.15s",
    padding: 0,
  },
  body: {
    display: "flex",
    flex: 1,
    overflow: "hidden",
  },
  chartSide: {
    flex: "0 0 58%",
    padding: "1.25rem 1.5rem",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
  },
  divider: {
    width: "1px",
    background: "rgba(148,163,184,0.1)",
    flexShrink: 0,
  },
  tableSide: {
    flex: 1,
    display: "flex",
    flexDirection: "column",
    padding: "1rem 1.25rem",
    overflow: "hidden",
  },
  tableHint: {
    margin: "0 0 0.6rem",
    fontSize: "0.75rem",
    color: "#475569",
    fontFamily: "'DM Sans', sans-serif",
  },
};

// ─── Quadrant Card ─────────────────────────────────────────────────────────────
function QuadrantCard({
  title,
  chartType,
  chartComponent,
  labels,
  values,
  hidden,
  onToggle,
  colorOffset,
  realDates = [],
  isEmpty = false,
}) {
  const [isFullscreen, setIsFullscreen] = useState(false);

  return (
    <>
      <div style={styles.card}>
        <button
          className="fs-btn"
          style={styles.fsBtn}
          onClick={() => setIsFullscreen(true)}
          title="Open fullscreen"
        >
          <FullscreenIcon />
        </button>

        <div style={{ ...styles.chartInner, position: "relative" }}>
          {chartComponent}
        
        </div>

        <DataTable
          labels={labels}
          values={values}
          hidden={hidden}
          onToggle={onToggle}
          colorOffset={colorOffset}
          compact={true}
          isPie={chartType !== "line"}
          displayLabels={
            chartType === "line" && realDates.length ? realDates : null
          }
        />
      </div>

      {isFullscreen && (
        <FullscreenModal
          title={title}
          chartType={chartType}
          labels={labels}
          values={values}
          hidden={hidden}
          onToggle={onToggle}
          colorOffset={colorOffset}
          realDates={realDates}
          onClose={() => setIsFullscreen(false)}
        />
      )}
    </>
  );
}

// ─── App ───────────────────────────────────────────────────────────────────────
export default function App() {
  const [allRecords, setAllRecords] = useState([]);
  const [totalRecords, setTotalRecords] = useState(0);
  const [loading, setLoading] = useState(true);
  const [dateRange, setDateRange] = useState({ start: null, end: null });

  const [hiddenLine, setHiddenLine] = useState(new Set());
  const [hiddenWaste, setHiddenWaste] = useState(new Set());
  const [hiddenMaterial, setHiddenMaterial] = useState(new Set());
  const [hiddenRegister, setHiddenRegister] = useState(new Set());

  const resetHidden = () => {
    setHiddenLine(new Set());
    setHiddenWaste(new Set());
    setHiddenMaterial(new Set());
    setHiddenRegister(new Set());
  };

  const toggle = (setHidden) => (label) => {
    setHidden((prev) => {
      const next = new Set(prev);
      next.has(label) ? next.delete(label) : next.add(label);
      return next;
    });
  };

  useEffect(() => {
    fetch("/data.json")
      .then((r) => r.json())
      .then((json) => {
        const records = json.data || [];
        setAllRecords(records);
        setTotalRecords(records.length);
        setLoading(false);
      })
      .catch(() => setLoading(false));
  }, []);

  const handleApply = (start, end) => {
    setDateRange({ start, end });
    resetHidden();
  };

  const handleClear = () => {
    setDateRange({ start: null, end: null });
    resetHidden();
  };

  const filteredRecords = filterByDateRange(allRecords, dateRange);
  const hasData = filteredRecords.length > 0;
  const raw = hasData
    ? processData(filteredRecords)
    : {
        line: { labels: [], realDates: [], values: [] },
        waste: { labels: [], values: [] },
        material: { labels: [], values: [] },
        register: { labels: [], values: [] },
      };

  if (loading) {
    return (
      <div style={styles.loadingScreen}>
        <div style={styles.spinner} />
        <p style={styles.loadingText}>Loading dashboard…</p>
      </div>
    );
  }

  return (
    <div style={styles.page}>
      <header style={styles.header}>
        <div>
          <h1 style={styles.title}>Waste Management Dashboard</h1>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: "0.75rem" }}>
          {dateRange.start && (
            <button
              onClick={handleClear}
              style={{
                display: "flex",
                alignItems: "center",
                gap: "0.4rem",
                background: "#EA457F",
                border: "1px solid rgba(248,113,113,0.35)",
                color: "#ffffff",
                borderRadius: "8px",
                padding: "0.4rem 0.85rem",
                cursor: "pointer",
                fontFamily: "'DM Sans',sans-serif",
                fontSize: "0.82rem",
                fontWeight: 500,
                transition: "all 0.2s",
              }}
            >
              <span style={{ fontSize: "0.9rem" }}>✕</span>
              <span>Clear filter</span>
            </button>
          )}
          <DateRangePicker dateRange={dateRange} onApply={handleApply} />
        </div>
      </header>

      <div style={styles.grid}>
        <QuadrantCard
          title="Waste Quantity Trend"
          chartType="line"
          chartComponent={
            <Line
              data={buildLineData(
                raw.line.labels,
                raw.line.values,
                hiddenLine,
                raw.line.realDates,
              )}
              options={lineOptions}
            />
          }
          labels={raw.line.labels}
          values={raw.line.values}
          hidden={hiddenLine}
          onToggle={toggle(setHiddenLine)}
          colorOffset={0}
          realDates={raw.line.realDates}
          isEmpty={!hasData}
        />
        <QuadrantCard
          title="Generated Waste Streams"
          chartType="pie"
          chartComponent={
            <Pie
              data={buildPieData(
                raw.waste.labels,
                raw.waste.values,
                hiddenWaste,
                0,
              )}
              options={sharedPieOptions("Generated Waste Streams")}
            />
          }
          labels={raw.waste.labels}
          values={raw.waste.values}
          hidden={hiddenWaste}
          onToggle={toggle(setHiddenWaste)}
          colorOffset={0}
          isEmpty={!hasData}
        />
        <QuadrantCard
          title="Dry Waste Category"
          chartType="pie"
          chartComponent={
            <Pie
              data={buildPieData(
                raw.material.labels,
                raw.material.values,
                hiddenMaterial,
                0,
              )}
              options={sharedPieOptions("Dry Waste Category")}
            />
          }
          labels={raw.material.labels}
          values={raw.material.values}
          hidden={hiddenMaterial}
          onToggle={toggle(setHiddenMaterial)}
          colorOffset={0}
          isEmpty={!hasData}
        />
        <QuadrantCard
          title="Residual Solid Waste"
          chartType="pie"
          chartComponent={
            <Pie
              data={buildPieData(
                raw.register.labels,
                raw.register.values,
                hiddenRegister,
                2,
              )}
              options={sharedPieOptions("Residual Solid Waste")}
            />
          }
          labels={raw.register.labels}
          values={raw.register.values}
          hidden={hiddenRegister}
          onToggle={toggle(setHiddenRegister)}
          colorOffset={0}
          isEmpty={!hasData}
        />
      </div>
    </div>
  );
}

const styles = {
  page: {
    minHeight: "100vh",
    background: "#ffffff",
    fontFamily: "'DM Sans', sans-serif",
    paddingBottom: "2rem",
    border: "none",
    outline: "none",
  },
  header: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    padding: "1.25rem 2rem 1rem",
    borderBottom: "1px solid #EA457F",
  },
  title: {
    marginLeft: "22.5rem",
    fontWeight: 600,
    color: "#35335D", //  #484575
    letterSpacing: "-0.02em",
    fontSize: "calc(1.325rem + .9vw)",
  },
  grid: {
    display: "grid",
    gridTemplateColumns: "repeat(2,1fr)",
    gap: "1.25rem",
    padding: "1.25rem 2rem",
  },
  card: {
    position: "relative",
    background: "#ffffff",
    borderRadius: "1rem",
    border: "1px solid rgba(148,163,184,0.1)",
    boxShadow: "0 2px 12px rgba(0,0,0,0.08)",
    padding: "1rem 1.2rem",
    display: "flex",
    flexDirection: "column",
    backdropFilter: "blur(8px)",
    gap: "2rem",
  },
  fsBtn: {
    position: "absolute",
    top: "0.75rem",
    right: "0.75rem",
    background: "#EA457F",
    color: "#ffffff",
    borderRadius: "7px",
    width: 25,
    height: 25,
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    cursor: "pointer",
    zIndex: 1,
    padding: 0,
    transition: "all 0.15s",
  },
  chartInner: { position: "relative", height: "260px" },
  tableWrapper: {
    overflowX: "auto",
    overflowY: "auto",
    borderRadius: "0.5rem",
    flex: 1,
  },
  table: {
    width: "100%",
    borderCollapse: "collapse",
    fontSize: "0.78rem",
    fontFamily: "'DM Sans',sans-serif",
    // backgroundColor: "rgb(71, 85, 105)",
  },
  th: {
    position: "sticky",
    top: 0,
    borderBottom: "1px solid rgb(234, 69, 127)",
    background: "#ffffff",
    color: "#484575",
    fontWeight: 600,
    padding: "0.45rem 0.75rem",
    textAlign: "left",
    whiteSpace: "nowrap",
    zIndex: 1,
    userSelect: "none",
  },
  td: {
    padding: "0.38rem 0.75rem",
    whiteSpace: "nowrap",
    transition: "color 0.2s",
  },
  trEven: { background: "#ffffff" },
  trOdd: { background: "#ffffff" },
  loadingScreen: {
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
    justifyContent: "center",
    minHeight: "100vh",
    background: "#0f172a",
    gap: "1rem",
  },
  spinner: {
    width: 40,
    height: 40,
    // border: "3px solid rgba(45,212,191,0.2)",
    // borderTop: "3px solid #2DD4BF",
    borderRadius: "50%",
    animation: "spin 0.8s linear infinite",
  },
  loadingText: {
    color: "#64748b",
    fontFamily: "sans-serif",
    fontSize: "0.9rem",
  },
};
