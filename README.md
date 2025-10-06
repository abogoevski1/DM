# DM
import React, { useState, useRef, useEffect } from "react";

// VSM Codespaces-ready single-file React component
// Usage: put this file in a React app (Vite or CRA) and import into App.jsx
// Styling uses TailwindCSS utility classes (assumes Tailwind is configured in the project)

export default function VSMApp() {
  const [items, setItems] = useState(() => {
    // sample default map
    return [
      makeItem("Process A", 80, 60, 1, 100, 50, 50, 80, 60),
      makeItem("Process B", 60, 50, 1, 120, 150, 250, 260, 150),
    ];
  });
  const [selectedId, setSelectedId] = useState(null);
  const [mode, setMode] = useState("select"); // select, connect
  const [connectFrom, setConnectFrom] = useState(null);
  const [connections, setConnections] = useState(() => []);
  const [canvasSize, setCanvasSize] = useState({ width: 1200, height: 800 });
  const canvasRef = useRef(null);

  // helpers
  function makeItem(title = "Process", ct = 60, avail = 480, shift = 1, demand = 1000, x = 50, y = 80) {
    return {
      id: `i_${Math.random().toString(36).slice(2, 9)}`,
      title,
      cycleTime: Number(ct),
      availabilityMinPerShift: Number(avail),
      shiftsPerDay: Number(shift),
      dailyDemand: Number(demand),
      x: Number(x),
      y: Number(y),
      w: 180,
      h: 70,
      inventory: 0,
      note: "",
    };
  }

  // persist to localStorage
  useEffect(() => {
    const saved = localStorage.getItem("vsm_items");
    const savedConn = localStorage.getItem("vsm_conns");
    if (saved) setItems(JSON.parse(saved));
    if (savedConn) setConnections(JSON.parse(savedConn));
  }, []);
  useEffect(() => {
    localStorage.setItem("vsm_items", JSON.stringify(items));
  }, [items]);
  useEffect(() => {
    localStorage.setItem("vsm_conns", JSON.stringify(connections));
  }, [connections]);

  // dragging
  const dragState = useRef({ draggingId: null, offsetX: 0, offsetY: 0 });

  function onMouseDownItem(e, id) {
    e.stopPropagation();
    const el = items.find((it) => it.id === id);
    if (!el) return;
    dragState.current = {
      draggingId: id,
      offsetX: e.clientX - el.x,
      offsetY: e.clientY - el.y,
    };
    setSelectedId(id);
  }

  function onMouseMoveCanvas(e) {
    if (!dragState.current.draggingId) return;
    const { draggingId, offsetX, offsetY } = dragState.current;
    setItems((prev) =>
      prev.map((it) => (it.id === draggingId ? { ...it, x: e.clientX - offsetX, y: e.clientY - offsetY } : it))
    );
  }

  function onMouseUpCanvas() {
    dragState.current = { draggingId: null, offsetX: 0, offsetY: 0 };
  }

  // add elements
  function addProcess() {
    setItems((s) => [...s, makeItem(`Process ${s.length + 1}`, 60, 480, 1, 1000, 60 + s.length * 220, 80)]);
  }
  function addInventory() {
    const it = makeItem(`Inventory ${items.length + 1}`, 0, 0, 0, 0, 80 + items.length * 220, 200);
    it.type = "inventory";
    setItems((s) => [...s, it]);
  }

  function deleteSelected() {
    if (!selectedId) return;
    setConnections((c) => c.filter((conn) => conn.from !== selectedId && conn.to !== selectedId));
    setItems((s) => s.filter((it) => it.id !== selectedId));
    setSelectedId(null);
  }

  // connection mode
  function startConnect(id) {
    setMode("connect");
    setConnectFrom(id);
  }
  function makeConnection(toId) {
    if (!connectFrom || connectFrom === toId) return;
    setConnections((s) => [...s, { id: `c_${Math.random().toString(36).slice(2, 9)}`, from: connectFrom, to: toId }]);
    setMode("select");
    setConnectFrom(null);
  }

  // calculations
  function computeTaktTime(totalAvailableMinPerDay, dailyDemand) {
    if (!dailyDemand) return 0;
    return totalAvailableMinPerDay / dailyDemand;
  }

  function totals() {
    const totalAvailable = items.reduce((acc, it) => acc + (it.availabilityMinPerShift || 0) * (it.shiftsPerDay || 0), 0);
    const totalDemand = items.reduce((acc, it) => acc + (it.dailyDemand || 0), 0);
    const totalCycle = items.reduce((acc, it) => acc + (it.cycleTime || 0), 0);
    return { totalAvailable, totalDemand, totalCycle };
  }

  const { totalAvailable, totalDemand, totalCycle } = totals();
  const taktTime = computeTaktTime(totalAvailable, Math.max(1, totalDemand));

  // exports
  function exportJSON() {
    const payload = { items, connections, meta: { exportedAt: new Date().toISOString() } };
    const blob = new Blob([JSON.stringify(payload, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "vsm-map.json";
    a.click();
    URL.revokeObjectURL(url);
  }
  function exportCSV() {
    // basic CSV with items
    const rows = [
      ["id", "title", "cycleTime", "availabilityMinPerShift", "shiftsPerDay", "dailyDemand", "x", "y", "inventory", "note"],
    ];
    items.forEach((it) => {
      rows.push([it.id, it.title, it.cycleTime, it.availabilityMinPerShift, it.shiftsPerDay, it.dailyDemand, it.x, it.y, it.inventory || 0, `"${(it.note || "").replace(/\"/g, '"')}`]);
    });
    const txt = rows.map((r) => r.join(",")).join("\n");
    const blob = new Blob([txt], { type: "text/csv" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "vsm-items.csv";
    a.click();
    URL.revokeObjectURL(url);
  }

  function clearAll() {
    if (!confirm("Clear all items and connections?")) return;
    setItems([]);
    setConnections([]);
    setSelectedId(null);
  }

  // simple connector rendering
  function renderConnections() {
    return connections.map((conn) => {
      const from = items.find((i) => i.id === conn.from);
      const to = items.find((i) => i.id === conn.to);
      if (!from || !to) return null;
      const x1 = from.x + from.w / 2;
      const y1 = from.y + from.h;
      const x2 = to.x + to.w / 2;
      const y2 = to.y;
      const midY = (y1 + y2) / 2;
      const path = `M ${x1} ${y1} C ${x1} ${midY} ${x2} ${midY} ${x2} ${y2}`;
      return (
        <g key={conn.id}>
          <path d={path} strokeWidth={2} stroke="#334155" fill="none" markerEnd="url(#arrow)" />
        </g>
      );
    });
  }

  // UI
  return (
    <div className="min-h-screen bg-slate-50 p-4">
      <div className="max-w-full mx-auto">
        <header className="flex items-center justify-between mb-4">
          <h1 className="text-2xl font-semibold">VSM Tool — Codespaces Ready</h1>
          <div className="flex gap-2">
            <button className="btn" onClick={addProcess} title="Add process">
              + Process
            </button>
            <button className="btn" onClick={addInventory} title="Add inventory box">
              + Inventory
            </button>
            <button className="btn" onClick={exportJSON} title="Export JSON">
              Export JSON
            </button>
            <button className="btn" onClick={exportCSV} title="Export CSV">
              Export CSV
            </button>
            <button className="btn-ghost text-red-600" onClick={clearAll}>
              Clear
            </button>
          </div>
        </header>

        <div className="flex gap-4">
          <aside className="w-80 p-3 bg-white rounded shadow-sm">
            <div className="mb-3">
              <h2 className="font-medium">Map controls</h2>
              <p className="text-sm text-slate-500">Mode: {mode}</p>
            </div>

            <div className="mb-3">
              <h3 className="font-medium">Totals & quick calcs</h3>
              <div className="text-sm text-slate-700 mt-2">
                <div>Available minutes per day: <strong>{totalAvailable}</strong></div>
                <div>Sum daily demand: <strong>{totalDemand}</strong></div>
                <div>Sum cycle times: <strong>{totalCycle} min</strong></div>
                <div className="mt-2">Takt time: <strong>{isFinite(taktTime) ? taktTime.toFixed(2) : "—"} min</strong></div>
              </div>
            </div>

            <div className="mb-3">
              <h3 className="font-medium">Selected</h3>
              {!selectedId && <div className="text-sm text-slate-500">No item selected</div>}
              {selectedId && (
                <SelectedEditor
                  key={selectedId}
                  item={items.find((it) => it.id === selectedId)}
                  onChange={(patched) => setItems((s) => s.map((it) => (it.id === selectedId ? { ...it, ...patched } : it)))}
                  onDelete={deleteSelected}
                  onStartConnect={() => startConnect(selectedId)}
                />
              )}
            </div>

            <div>
              <h3 className="font-medium">Map actions</h3>
              <div className="mt-2 flex gap-2">
                <button
                  className={`px-3 py-1 rounded ${mode === "select" ? "bg-slate-900 text-white" : "bg-slate-100"}`}
                  onClick={() => {
                    setMode("select");
                    setConnectFrom(null);
                  }}
                >
                  Select
                </button>
                <button
                  className={`px-3 py-1 rounded ${mode === "connect" ? "bg-slate-900 text-white" : "bg-slate-100"}`}
                  onClick={() => setMode("connect")}
                >
                  Connect
                </button>
              </div>
            </div>

            <div className="mt-4 text-xs text-slate-500">Tip: Drag process boxes to rearrange. Select a process and click "Connect" to make flows.</div>
          </aside>

          <main className="flex-1 bg-white rounded shadow-sm p-2">
            <div
              ref={canvasRef}
              onMouseMove={onMouseMoveCanvas}
              onMouseUp={onMouseUpCanvas}
              onMouseLeave={onMouseUpCanvas}
              className="relative overflow-auto"
              style={{ height: 720 }}
            >
              <svg width={canvasSize.width} height={canvasSize.height} className="absolute left-0 top-0 -z-10">
                <defs>
                  <marker id="arrow" markerWidth="8" markerHeight="8" refX="6" refY="4" orient="auto">
                    <path d="M0,0 L8,4 L0,8 z" fill="#1f2937" />
                  </marker>
                </defs>
              </svg>

              <svg width={canvasSize.width} height={canvasSize.height} className="absolute left-0 top-0">
                {renderConnections()}
              </svg>

              {/* items */}
              {items.map((it) => (
                <div
                  key={it.id}
                  onMouseDown={(e) => onMouseDownItem(e, it.id)}
                  onDoubleClick={() => setSelectedId(it.id)}
                  className={`absolute rounded-lg shadow cursor-move user-select-none p-3 ${selectedId === it.id ? "ring-2 ring-indigo-500" : "bg-white"}`}
                  style={{ left: it.x, top: it.y, width: it.w, height: it.h }}
                >
                  <div className="flex justify-between items-start">
                    <div className="font-semibold text-sm truncate">{it.title}</div>
                    <div className="flex gap-1">
                      <button
                        className="text-xs px-2 py-1 rounded bg-slate-100"
                        onClick={(e) => {
                          e.stopPropagation();
                          if (mode === "connect") {
                            if (connectFrom) makeConnection(it.id);
                            else startConnect(it.id);
                          }
                        }}
                      >
                        ➤
                      </button>
                    </div>
                  </div>
                  <div className="text-xs text-slate-500 mt-2">
                    <div>CT: {it.cycleTime} min</div>
                    <div>Avail/min per shift: {it.availabilityMinPerShift}</div>
                    <div>Shifts/day: {it.shiftsPerDay}</div>
                    <div>Demand/day: {it.dailyDemand}</div>
                  </div>
                </div>
              ))}
            </div>
          </main>
        </div>
      </div>
    </div>
  );
}

function SelectedEditor({ item, onChange, onDelete, onStartConnect }) {
  if (!item) return null;
  return (
    <div>
      <div className="mb-2 text-sm font-medium">{item.title}</div>
      <label className="block text-xs">Title</label>
      <input className="input" value={item.title} onChange={(e) => onChange({ title: e.target.value })} />

      <label className="block text-xs mt-2">Cycle time (min)</label>
      <input className="input" type="number" value={item.cycleTime} onChange={(e) => onChange({ cycleTime: Number(e.target.value) })} />

      <label className="block text-xs mt-2">Availability (min/shift)</label>
      <input className="input" type="number" value={item.availabilityMinPerShift} onChange={(e) => onChange({ availabilityMinPerShift: Number(e.target.value) })} />

      <label className="block text-xs mt-2">Shifts / day</label>
      <input className="input" type="number" value={item.shiftsPerDay} onChange={(e) => onChange({ shiftsPerDay: Number(e.target.value) })} />

      <label className="block text-xs mt-2">Daily demand</label>
      <input className="input" type="number" value={item.dailyDemand} onChange={(e) => onChange({ dailyDemand: Number(e.target.value) })} />

      <label className="block text-xs mt-2">Inventory</label>
      <input className="input" type="number" value={item.inventory || 0} onChange={(e) => onChange({ inventory: Number(e.target.value) })} />

      <label className="block text-xs mt-2">Note</label>
      <textarea className="input h-16" value={item.note || ""} onChange={(e) => onChange({ note: e.target.value })} />

      <div className="flex gap-2 mt-3">
        <button className="btn" onClick={onStartConnect}>Start Connect</button>
        <button className="btn-ghost text-red-600" onClick={onDelete}>Delete</button>
      </div>
    </div>
  );
}

// Tailwind helper classnames as simple style snippets for self-contained use when Tailwind not available
// In a real repo, remove the fallback block and rely on actual Tailwind.

const style = document.createElement("style");
style.innerHTML = `
.btn{ background:#0f172a;color:white;padding:6px 10px;border-radius:8px;font-size:13px }
.btn-ghost{ padding:6px 10px;border-radius:8px;font-size:13px }
.input{ width:100%; padding:6px 8px;border-radius:6px;border:1px solid #e6e7eb;margin-top:4px }
`;
document.head.appendChild(style);
