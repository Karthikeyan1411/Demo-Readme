import { useState, useEffect, useRef, useContext } from "react";
import { DataContext } from "../Context/DataContext";
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

const _style = document.createElement("style");
_style.textContent = `
  @keyframes spin    { to { transform: rotate(360deg); } }
  @keyframes fadeIn  { from { opacity: 0; } to { opacity: 1; } }
  @keyframes slideUp { from { opacity: 0; transform: translateY(28px) scale(0.97); } to { opacity: 1; transform: translateY(0) scale(1); } }
  .fs-btn:focus-visible { outline: none !important; }
  .fs-btn:hover, .close-btn:hover { background: rgba(248,113,113,0.15) !important; border-color: rgba(248,113,113,0.4) !important; color: #F87171 !important; }
  .hide-scroll { scrollbar-width: 5px ; -ms-overflow-style: none; }
  .hide-scroll::-webkit-scrollbar { display: none; }
`;
if (!document.head.querySelector("#dash-keyframes")) {
  _style.id = "dash-keyframes";
  document.head.appendChild(_style);
}

const CATEGORY_COLORS = {
  "Direct Residual Gen": "#C62828",
  "Dry Waste Gen": "#2196F3",
  "Food Waste Gen": "#4CAF50",
  "Garden Waste Gen": "#808000",
  "Coconut Shell Gen": "#8B4513",
  "Tender Coconut Gen": "#FF9800",
  "Bio Hazardous Waste Gen": "#FF6B6B",
  "C&D Gen": "#90A4AE",
  "Sugarcane Trash Gen": "#9C27B0",
  "Residual Food Waste": "#E57373",
  "Residual Dry Waste": "#EF9A9A",
  "Inward Register": "#00BCD4",
  "Outward Register": "#FF5722",
  Unknown: "#B0BEC5",
};

const CATEGORY_COLOR_VALUES = Object.values(CATEGORY_COLORS);

function getCategoryColor(label, index) {
  return (
    CATEGORY_COLORS[label] ??
    CATEGORY_COLOR_VALUES[index % CATEGORY_COLOR_VALUES.length]
  );
}

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
  // Records are top-level Zoho objects with a nested Data JSON string.
  // Date filtering happens at the month level inside processData via filterMonths.
  // This function just returns all records; month-level filtering is applied in processData.
  return records;
}

function isMonthInRange(monthKey, dateRange) {
  // monthKey format: "Mar 2026"
  if (!dateRange || !dateRange.start) return true;
  const [mon, yr] = monthKey.split(" ");
  const monthDate = new Date(`${mon} 01 ${yr}`);
  const start = new Date(
    dateRange.start.getFullYear(),
    dateRange.start.getMonth(),
    1,
  );
  const end = dateRange.end
    ? new Date(dateRange.end.getFullYear(), dateRange.end.getMonth(), 1)
    : start;
  return monthDate >= start && monthDate <= end;
}

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
      setSelecting(d);
      setStart(d);
      setEnd(null);
      onApply(d, d);
    } else {
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
  const COL_COUNT = 7;

  return (
    <div style={cal.wrap}>
      <div style={cal.header}></div>
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
          const col = i % COL_COUNT;
          const isStart = sameDay(d, start);
          const isEnd = sameDay(d, previewEnd);
          const isSelecting = sameDay(d, selecting);
          const isToday = sameDay(d, today);
          const isInRange =
            inRange(d, previewStart, previewEnd) && !isStart && !isEnd;
          const rangeActive = isInRange || isStart || isEnd;
          const isRangeStart = sameDay(d, previewStart);
          const isRangeEnd = sameDay(d, previewEnd);
          const roundLeft = isRangeStart || col === 0;
          const roundRight = isRangeEnd || col === 6;

          return (
            <div
              key={i}
              onClick={() => handleDay(d)}
              onMouseEnter={() => setHovered(d)}
              onMouseLeave={() => setHovered(null)}
              style={{ position: "relative", ...cal.cell }}
            >
              {rangeActive &&
                previewStart &&
                previewEnd &&
                !sameDay(previewStart, previewEnd) && (
                  <div
                    style={{
                      position: "absolute",
                      top: "50%",
                      transform: "translateY(-50%)",
                      left: roundLeft ? "50%" : "0",
                      right: roundRight ? "50%" : "0",
                      height: 30,
                      background: "rgba(59,130,246,0.13)",
                      zIndex: 0,
                      ...(isRangeStart ? { borderRadius: "50% 0 0 50%" } : {}),
                      ...(isRangeEnd ? { borderRadius: "0 50% 50% 0" } : {}),
                      ...(!isRangeStart && !isRangeEnd && col === 0
                        ? { borderRadius: "0" }
                        : {}),
                    }}
                  />
                )}
              <div
                style={{
                  position: "relative",
                  zIndex: 1,
                  width: 30,
                  height: 30,
                  display: "flex",
                  alignItems: "center",
                  justifyContent: "center",
                  borderRadius: "50%",
                  background:
                    isStart || isSelecting || isEnd ? "#3B82F6" : "transparent",
                  color:
                    isStart || isSelecting || isEnd
                      ? "#fff"
                      : isInRange
                        ? "#1d4ed8"
                        : isToday
                          ? "#3B82F6"
                          : "#374151",
                  fontWeight: isStart || isEnd || isToday ? 700 : 400,
                  fontSize: "0.82rem",
                  cursor: "pointer",
                  userSelect: "none",
                  boxShadow:
                    isStart || isEnd
                      ? "0 2px 8px rgba(59,130,246,0.4)"
                      : "none",
                  transition: "background 0.12s",
                }}
              >
                {d.getDate()}
              </div>
            </div>
          );
        })}
      </div>
      <div style={cal.selectedLabel}>
        {selecting && (
          <span style={{ color: "#3B82F6" }}>Now select end date…</span>
        )}
        {start && end && !selecting && (
          <span style={{ color: "#374151" }}>
            <b>{formatDisplay(start)}</b>
            <span style={{ color: "#9ca3af", margin: "0 4px" }}>→</span>
            <b>{formatDisplay(end)}</b>
          </span>
        )}
        {start && !end && !selecting && (
          <span style={{ color: "#374151" }}>
            <b>{formatDisplay(start)}</b>
          </span>
        )}
        {!start && (
          <span style={{ color: "#9ca3af" }}>Click a date to select</span>
        )}
      </div>
    </div>
  );
}

const cal = {
  wrap: {
    background: "#ffffff",
    border: "1px solid #e5e7eb",
    borderRadius: "0.85rem",
    padding: "0.85rem 1rem 0.75rem",
    width: 290,
    boxShadow: "0 8px 32px rgba(0,0,0,0.12)",
    fontFamily: "'DM Sans',sans-serif",
  },
  header: { marginBottom: "0.5rem" },
  nav: {
    display: "flex",
    alignItems: "center",
    justifyContent: "space-between",
    margin: "0.4rem 0 0.4rem",
  },
  navBtn: {
    background: "none",
    border: "none",
    color: "#6b7280",
    cursor: "pointer",
    fontSize: "1.2rem",
    padding: "0 0.4rem",
    lineHeight: 1,
  },
  navTitle: { color: "#374151", fontWeight: 600, fontSize: "0.88rem" },
  grid: {
    display: "grid",
    gridTemplateColumns: "repeat(7,1fr)",
    gap: "0",
    marginBottom: "0.5rem",
  },
  dayLabel: {
    textAlign: "center",
    color: "#9ca3af",
    fontSize: "0.68rem",
    padding: "4px 0",
    userSelect: "none",
  },
  cell: {
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    height: 36,
    cursor: "pointer",
  },
  selectedLabel: {
    fontSize: "0.74rem",
    color: "#6b7280",
    textAlign: "center",
    minHeight: "1.4rem",
    marginTop: "0.3rem",
    borderTop: "1px solid #f3f4f6",
    paddingTop: "0.4rem",
  },
};

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
    : "Select Date Range";

  return (
    <div ref={ref} style={{ position: "relative" }}>
      <button
        onClick={() => setOpen((o) => !o)}
        style={{
          display: "flex",
          alignItems: "center",
          gap: "0.5rem",
          background: "#ffffff",
          border: "1px solid #d1d5db",
          color: dateRange.start ? "#374151" : "#6b7280",
          borderRadius: "8px",
          padding: "0.4rem 0.9rem",
          cursor: "pointer",
          fontFamily: "'DM Sans',sans-serif",
          fontSize: "0.82rem",
          fontWeight: 500,
          boxShadow: "0 1px 4px rgba(0,0,0,0.07)",
        }}
      >
        <span>📅</span>
        <span>{label}</span>
        <span style={{ fontSize: "0.65rem", opacity: 0.5 }}>
          {open ? "▲" : "▼"}
        </span>
      </button>
      {open && (
        <div
          style={{
            position: "absolute",
            top: "calc(100% + 8px)",
            left: 0,
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

const MONTH_ORDER = [
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

function getMonthKey(dateStr) {
  // dateStr format: "12-Mar-2026"
  if (!dateStr) return null;
  const parts = dateStr.split("-");
  if (parts.length !== 3) return null;
  return `${parts[1]} ${parts[2]}`; // "Mar 2026"
}

function processData(records, dateRange = {}) {
  // byCatMonth["Waste_Category"]["Mar 2026"] = total kg
  const byCatMonth = {};
  const byMaterial = {}; // Material_Waste_Category -> kg
  const byRegister = {}; // Register_Type -> kg
  const allMonthsSet = new Set();

  const addTo = (obj, key, val) => {
    obj[key] = (obj[key] || 0) + parseFloat(val || 0);
  };

  records.forEach((r) => {
    const monthKey = getMonthKey(r.Date_field);
    if (!monthKey) return;

    // Filter by exact date range
    if (dateRange?.start) {
      const recDate = parseRecordDate(r.Date_field);
      if (!recDate) return;
      const s = dateRange.start.getTime();
      const e = (dateRange.end || dateRange.start).getTime();
      const t = recDate.getTime();
      if (t < Math.min(s, e) || t > Math.max(s, e) + 86400000 - 1) return;
    }

    // Only Inward Register records for generation charts
    const regType = r.Register_Type?.display_value || r.Register_Type || "";
    if (regType !== "Inward Register") return;

    const kg = parseFloat(r.Total_Quantity_Kg || 0);
    if (kg <= 0) return;

    // Line chart + waste table: per Waste_Category per month
    const cat = r.Waste_Category?.display_value || r.Waste_Category || "Others";

    // Exclude residual/non-generation categories
    const EXCLUDED_CATS = [
      "Direct Residual Waste Gen",
      "C&D Gen",
      "Bio Hazardous Waste Gen",
    ];
    if (EXCLUDED_CATS.includes(cat)) return;

    allMonthsSet.add(monthKey); // only add month if category passes exclusion

    if (!byCatMonth[cat]) byCatMonth[cat] = {};
    byCatMonth[cat][monthKey] = (byCatMonth[cat][monthKey] || 0) + kg;

    // Pie 2: Material_Waste_Category
    const mat =
      r.Material_Waste_Category?.display_value ||
      r.Material_Waste_Category ||
      "Others";
    addTo(byMaterial, mat, kg);
  });

  // Wet Waste Breakdown — Processed Register only
  records.forEach((r) => {
    const monthKey = getMonthKey(r.Date_field);
    if (!monthKey) return;

    // Filter by exact date range
    if (dateRange?.start) {
      const recDate = parseRecordDate(r.Date_field);
      if (!recDate) return;
      const s = dateRange.start.getTime();
      const e = (dateRange.end || dateRange.start).getTime();
      const t = recDate.getTime();
      if (t < Math.min(s, e) || t > Math.max(s, e) + 86400000 - 1) return;
    }

    const regType = r.Register_Type?.display_value || r.Register_Type || "";
    if (regType !== "Processed Register") return;

    const kg = parseFloat(r.Total_Quantity_Kg || 0);
    if (kg <= 0) return;

    const cat = r.Waste_Category?.display_value || r.Waste_Category || "Others";
    addTo(byRegister, cat, kg);
  });

  // Sort months chronologically
  const allSortedMonths = [...allMonthsSet].sort((a, b) => {
    const [ma, ya] = a.split(" "),
      [mb, yb] = b.split(" ");
    return (
      parseInt(ya) - parseInt(yb) ||
      MONTH_ORDER.indexOf(ma) - MONTH_ORDER.indexOf(mb)
    );
  });

  // Always show rolling last 3 months (fill with 0 if no data)
  const today = new Date();
  const rollingMonths = [];
  for (let i = 2; i >= 0; i--) {
    const d = new Date(today.getFullYear(), today.getMonth() - i, 1);
    rollingMonths.push(`${MONTH_ORDER[d.getMonth()]} ${d.getFullYear()}`);
  }
  // Use rolling months as the base; allSortedMonths may have data for them
  const sortedMonths = rollingMonths;

  const categories = Object.keys(byCatMonth);

  // Per-category totals across ALL months (for table)
  const catTotals = categories.map((cat) =>
    Object.values(byCatMonth[cat]).reduce((a, b) => a + b, 0),
  );

  // Total per month (for default line when nothing selected)
  const monthTotals = sortedMonths.map((m) =>
    categories.reduce((sum, cat) => sum + (byCatMonth[cat]?.[m] || 0), 0),
  );

  const filterZeros = (obj) => {
    const labels = [],
      values = [];
    Object.entries(obj).forEach(([k, v]) => {
      if (v > 0) {
        labels.push(k);
        values.push(v);
      }
    });
    return { labels, values };
  };

  // Waste category pie: total per category across all months
  const wastePie = filterZeros(
    Object.fromEntries(categories.map((c, i) => [c, catTotals[i]])),
  );

  // Short labels "Jan '26" for display — data keys stay as "Jan 2026"
  const shortMonths = sortedMonths.map((m) => {
    const [mon, yr] = m.split(" ");
    return `${mon} '${yr.slice(-2)}`;
  });

  return {
    line: {
      months: shortMonths,
      fullMonths: sortedMonths,
      values: monthTotals,
      categories,
      byCatMonth,
    },
    waste: wastePie,
    material: filterZeros(byMaterial),
    register: filterZeros(byRegister),
  };
}

// Single total line — recalculates excluding hidden categories
function buildLineData(lineData, hidden = new Set()) {
  const { months, fullMonths, categories, byCatMonth } = lineData;
  const lookupMonths = fullMonths || months; // use full "Jan 2026" keys for data lookup
  const visibleCats = (categories || []).filter((c) => !hidden.has(c));
  const data = lookupMonths.map((m) =>
    visibleCats.reduce((sum, cat) => sum + (byCatMonth?.[cat]?.[m] || 0), 0),
  );
  return {
    labels: months,
    datasets: [
      {
        label: "Total Waste (kg)",
        data,
        borderColor: "#2DD4BF",
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
        pointBackgroundColor: "#2DD4BF",
        pointBorderColor: "#ffffff",
        pointBorderWidth: 2,
        pointRadius: 5,
        pointHoverRadius: 8,
        tension: 0.02,
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
        datalabels: { display: false },
      },
    ],
  };
}

const piePercentLabelPlugin = {
  id: "piePercentLabels",
  afterDatasetsDraw(chart) {
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
ChartJS.register(piePercentLabelPlugin);

const sharedPieOptions = () => ({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    title: { display: false },
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
    title: { display: false },
    tooltip: {
      callbacks: {
        title: (items) => items[0]?.label || "",
        label: (ctx) =>
          ctx.parsed.y !== null
            ? ` ${ctx.dataset.label}: ${ctx.parsed.y.toFixed(2)} kg`
            : "",
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
  const total = values.reduce((a, b) => a + b, 0);
  const shownLabels = displayLabels || labels;

  return (
    <div
      className="hide-scroll"
      style={{ ...styles.tableWrapper, maxHeight: compact ? "190px" : "100%" }}
    >
      <table style={styles.table}>
        <thead>
          <tr></tr>
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
                  e.currentTarget.style.background = "#77777731";
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
                  style={{ ...styles.td, textAlign: "right", color: "#EA457F" }}
                >
                  {values[i].toFixed(2)}{" "}
                  <span
                    style={{
                      color: "#EA457F",
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

function FullscreenModal({
  title,
  chartType,
  labels,
  values,
  hidden,
  onToggle,
  colorOffset,
  onClose,
  lineData = null,
}) {
  useEffect(() => {
    const handler = (e) => {
      if (e.key === "Escape") onClose();
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
  }, [onClose]);

  useEffect(() => {
    document.body.style.overflow = "hidden";
    return () => {
      document.body.style.overflow = "";
    };
  }, []);

  const chartData = lineData
    ? buildLineData(lineData, hidden)
    : buildPieData(labels, values, hidden, colorOffset);
  const chartOptions = lineData ? lineOptions : sharedPieOptions();
  const ChartComponent = lineData ? Line : Pie;

  return (
    <div
      style={modal.overlay}
      onClick={(e) => {
        if (e.target === e.currentTarget) onClose();
      }}
    >
      <div style={modal.panel}>
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
        <div style={modal.body}>
          <div style={modal.chartSide}>
            <div
              style={{ position: "relative", height: "100%", width: "100%" }}
            >
              <ChartComponent data={chartData} options={chartOptions} />
            </div>
          </div>
          <div style={modal.divider} />
          <div style={modal.tableSide}>
            <DataTable
              labels={labels}
              values={values}
              hidden={hidden}
              onToggle={onToggle}
              colorOffset={colorOffset}
              compact={false}
              isPie={chartType !== "line"}
              displayLabels={null}
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
    background: "#EA457F",
    border: "1px solid rgba(148,163,184,0.15)",
    color: "#ffffff",
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
  body: { display: "flex", flex: 1, overflow: "hidden" },
  chartSide: {
    flex: "0 0 58%",
    padding: "1.25rem 1.5rem",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
  },
  divider: { width: "1px", background: "rgba(148,163,184,0.1)", flexShrink: 0 },
  tableSide: {
    flex: 1,
    display: "flex",
    flexDirection: "column",
    padding: "1rem 1.25rem",
    overflow: "hidden",
  },
};

function QuadrantCard({
  title,
  chartType,
  labels,
  values,
  hidden,
  onToggle,
  colorOffset,
  isEmpty = false,
  lineData = null,
}) {
  const [isFullscreen, setIsFullscreen] = useState(false);

  const chartData = lineData
    ? buildLineData(lineData, hidden)
    : buildPieData(labels, values, hidden, colorOffset);
  const ChartComp = lineData ? Line : Pie;
  const opts = lineData ? lineOptions : sharedPieOptions();

  return (
    <>
      <div style={styles.card}>
        <div style={styles.cardHeader}>
          <span style={styles.cardTitle}>{title}</span>
          <button
            className="fs-btn"
            style={styles.fsBtn}
            onClick={() => setIsFullscreen(true)}
            title="Open fullscreen"
          >
            <FullscreenIcon />
          </button>
        </div>
        <div style={styles.cardDivider} />
        <div style={{ ...styles.chartInner, position: "relative" }}>
          <ChartComp data={chartData} options={opts} />
        </div>
        <DataTable
          labels={labels}
          values={values}
          hidden={hidden}
          onToggle={onToggle}
          colorOffset={colorOffset}
          compact={true}
          isPie={!lineData}
          displayLabels={null}
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
          lineData={lineData}
          onClose={() => setIsFullscreen(false)}
        />
      )}
    </>
  );
}

export default function App() {
  const { apiData, loading } = useContext(DataContext);
  const allRecords = apiData?.RegisterSfData || [];
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

  const handleApply = (start, end) => {
    setDateRange({ start, end });
    resetHidden();
  };
  const handleClear = () => {
    setDateRange({ start: null, end: null });
    resetHidden();
  };

  const raw = processData(allRecords, dateRange);
  const hasData = raw.line.months?.length > 0 || raw.waste.labels.length > 0;

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
        <div
          style={{ display: "flex", alignItems: "center", minWidth: "200px" }}
        >
          <DateRangePicker dateRange={dateRange} onApply={handleApply} />
        </div>
        <h1 style={styles.title}>Waste Management Dashboard</h1>
        <div
          style={{
            display: "flex",
            alignItems: "center",
            minWidth: "200px",
            justifyContent: "flex-end",
          }}
        >
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
              }}
            >
              <span style={{ fontSize: "0.9rem" }}>✕</span>
              <span>Clear filter</span>
            </button>
          )}
        </div>
      </header>

      <div style={styles.grid}>
        <QuadrantCard
          title="Waste Quantity Trend"
          chartType="line"
          labels={raw.waste.labels}
          values={raw.waste.values}
          hidden={hiddenLine}
          onToggle={toggle(setHiddenLine)}
          colorOffset={0}
          isEmpty={!hasData}
          lineData={raw.line}
        />

        <QuadrantCard
          title="Generated Waste Streams"
          chartType="pie"
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
          labels={raw.material.labels}
          values={raw.material.values}
          hidden={hiddenMaterial}
          onToggle={toggle(setHiddenMaterial)}
          colorOffset={0}
          isEmpty={!hasData}
        />

        <QuadrantCard
          title="Wet Waste Breakdown"
          chartType="pie"
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
    margin: 0,
    fontWeight: 600,
    color: "#35335D",
    letterSpacing: "-0.02em",
    fontSize: "calc(1.325rem + .9vw)",
    textAlign: "center",
    flex: 1,
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
    padding: "0.85rem 1.2rem 1rem",
    display: "flex",
    flexDirection: "column",
    backdropFilter: "blur(8px)",
    gap: "0.75rem",
  },
  cardHeader: {
    display: "flex",
    alignItems: "center",
    justifyContent: "space-between",
  },
  cardTitle: {
    color: "#484575",
    fontWeight: 600,
    fontSize: "0.95rem",
    fontFamily: "'DM Sans', sans-serif",
    letterSpacing: "-0.01em",
  },
  cardDivider: {
    height: "1px",
    background: "rgba(234, 69, 127, 0.45)",
    flexShrink: 0,
    marginBottom: "0.25rem",
  },
  fsBtn: {
    background: "#EA457F",
    border: "1px solid rgba(148,163,184,0.15)",
    color: "#ffffff",
    borderRadius: "7px",
    width: 28,
    height: 28,
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    cursor: "pointer",
    padding: 0,
    flexShrink: 0,
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
    fontSize: "13.6px",
    fontWeight: "600",
  },
  trEven: { background: "#ffffff" },
  trOdd: { background: "#ffffff" },
  loadingScreen: {
    display: "flex",
    flexDirection: "column",
    alignItems: "center",
    justifyContent: "center",
    minHeight: "100vh",
    gap: "1rem",
  },
  spinner: {
    width: 40,
    height: 40,
    border: "3px solid rgba(234,69,127,0.2)",
    borderTop: "3px solid #EA457F",
    borderRadius: "50%",
    animation: "spin 0.8s linear infinite",
  },
  loadingText: {
    color: "#64748b",
    fontFamily: "sans-serif",
    fontSize: "0.9rem",
  },
};
