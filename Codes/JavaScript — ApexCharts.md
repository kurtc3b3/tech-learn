## What & When

**ApexCharts** is a modern **JavaScript charting library** with a **React wrapper** (`react-apexcharts`). Use it for dashboards, admin panels, and analytics views fed by [[API - FastAPI]] JSON endpoints.

```bash
npm install apexcharts react-apexcharts
```

Hub: [[JavaScript Development]] · UI: [[Codes/JavaScript — React]]

---

## Line Chart (React)

```tsx
import Chart from "react-apexcharts";
import type { ApexOptions } from "apexcharts";

const series = [
  {
    name: "Requests",
    data: [120, 190, 150, 210, 180, 240],
  },
];

const options: ApexOptions = {
  chart: { type: "line", toolbar: { show: false } },
  xaxis: {
    categories: ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat"],
  },
  stroke: { curve: "smooth" },
  colors: ["#3b82f6"],
};

export function RequestsChart() {
  return <Chart options={options} series={series} type="line" height={320} />;
}
```

---

## Fetch Data from API

```tsx
import { useEffect, useState } from "react";
import Chart from "react-apexcharts";
import axios from "axios";

interface Point {
  label: string;
  value: number;
}

export function LiveChart() {
  const [points, setPoints] = useState<Point[]>([]);

  useEffect(() => {
    axios.get<Point[]>("/api/metrics/daily").then((res) => setPoints(res.data));
  }, []);

  return (
    <Chart
      type="bar"
      height={300}
      series={[{ name: "Sales", data: points.map((p) => p.value) }]}
      options={{
        xaxis: { categories: points.map((p) => p.label) },
        chart: { type: "bar" },
      }}
    />
  );
}
```

Pair with [[Codes/JavaScript — axios]] and [[API - FastAPI]].

---

## Common Chart Types

| `type` | Use when |
| --- | --- |
| `line` | Time series, trends |
| `bar` | Category comparison |
| `area` | Cumulative metrics |
| `pie` / `donut` | Part-of-whole |
| `heatmap` | Density grids |
| `candlestick` | OHLC financial |

---

## Styling & Dark Mode

```typescript
const options: ApexOptions = {
  theme: { mode: "dark" },
  chart: { background: "transparent" },
  grid: { borderColor: "#374151" },
  tooltip: { theme: "dark" },
};
```

Works alongside [[Codes/JavaScript — Tailwind CSS]] layout wrappers.

---

## ApexCharts vs Alternatives

| Need | Prefer |
| --- | --- |
| Rich React dashboards | **ApexCharts** |
| Python/static plots | [[ML — matplotlib]], [[ML — seaborn]] |
| Simple sparklines | inline SVG or lightweight lib |
| Notebook exploration | matplotlib in [[ML — pandas]] workflows |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Install | `npm i apexcharts react-apexcharts` |
| Import | `import Chart from "react-apexcharts"` |
| Props | `options`, `series`, `type`, `height` |
| Update data | change `series` state → re-render |
| Export PNG | `chart.exportToSVG()` via ref API |

Docs: [apexcharts.com](https://apexcharts.com/docs/react-charts/)

---

## Related Notes

- [[Codes/JavaScript — React]]
- [[Codes/JavaScript — axios]]
- [[Codes/JavaScript — Tailwind CSS]]
- [[API - FastAPI]]
- [[ML — matplotlib]]
- [[JavaScript Development]]

---

## Tags

#apexcharts #react #charts #dashboard #javascript #typescript #dataviz #frontend
