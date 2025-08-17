# IBfly Simulator — FastAPI + React Shell with Polygon Integration

This version wires the backend stubs to **real Polygon data** for SPY bars and option quotes. It replaces mocks with actual data fetch + minute alignment.

---

## Backend Changes (FastAPI)

### 1. Add Polygon client

```python
import os, httpx

POLYGON_KEY = os.getenv("POLYGON_API_KEY")
BASE_URL = "https://api.polygon.io"

async def fetch_polygon(path: str, params: dict = {}):
    if not POLYGON_KEY:
        raise RuntimeError("Missing POLYGON_API_KEY env var")
    params = {**params, "apiKey": POLYGON_KEY}
    async with httpx.AsyncClient(timeout=30.0) as client:
        r = await client.get(BASE_URL + path, params=params)
        r.raise_for_status()
        return r.json()
```

### 2. Symbol building (OCC style)

```python
def build_occ_symbol(underlying: str, expiry: str, right: str, strike: float):
    # expiry = YYYY-MM-DD → YYMMDD
    e = expiry.replace("-", "").replace("20", "", 1)
    # strike padded: 8 chars, 3 decimals
    s = f"{int(strike*1000):08d}"
    return f"O:{underlying}{e}{right}{s}"
```

### 3. Init → fetch SPY bars + options quotes

```python
@app.post("/api/simulate/init")
async def init_sim(req: InitRequest):
    date = req.date
    kc, width = req.strategy.kc, req.strategy.width
    k1, k2 = kc - width, kc + width

    # SPY 1-min bars for date
    bars = await fetch_polygon(f"/v2/aggs/ticker/SPY/range/1/minute/{date}/{date}", {"adjusted": "true"})
    times = [datetime.fromtimestamp(b["t"]/1000) for b in bars["results"] if 930 <= datetime.fromtimestamp(b["t"]/1000).hour*100+datetime.fromtimestamp(b["t"]/1000).minute <= 1600]
    spy = [b["c"] for b in bars["results"]]

    # Build contract symbols
    expiry_map = {"0DTE": date, "1W": (datetime.fromisoformat(date)+timedelta(days=5)).strftime("%Y-%m-%d"), "1M": (datetime.fromisoformat(date)+timedelta(days=21)).strftime("%Y-%m-%d")}
    expiry = expiry_map[req.expiryMode]
    contracts = {
        "k1Put":  build_occ_symbol("SPY", expiry, "P", k1),
        "kcPut":  build_occ_symbol("SPY", expiry, "P", kc),
        "kcCall": build_occ_symbol("SPY", expiry, "C", kc),
        "k2Call": build_occ_symbol("SPY", expiry, "C", k2)
    }

    # Fetch quotes for each leg
    quotes: dict[str, list] = {}
    for leg, sym in contracts.items():
        q = await fetch_polygon(f"/v3/quotes/{sym}", {"timestamp.gte": f"{date}T09:30:00Z", "timestamp.lte": f"{date}T16:00:00Z"})
        quotes[leg] = q.get("results", [])

    run_id = f"run-{int(datetime.now().timestamp())}"
    RUNS[run_id] = {
        "date": date,
        "expiry": expiry,
        "strategy": req.strategy.model_dump(),
        "timeline": times,
        "spy": spy,
        "contracts": contracts,
        "quotes": quotes,
        "entry": None,
        "fills": [],
        "exits": []
    }

    return {"runId": run_id, "timeline": [t.isoformat()+"Z" for t in times], "strikes": {"k1": k1, "kc": kc, "k2": k2}, "contracts": contracts, "entry": None}
```

### 4. Quote alignment helper

```python
def quote_at_time(leg_quotes, t):
    # find latest quote at or before t
    q = None
    for row in leg_quotes:
        qt = datetime.fromtimestamp(row["sip_timestamp"] / 1e9)
        if qt <= t:
            q = row
        else:
            break
    return q
```

### 5. Frame endpoint → use aligned quotes

```python
@app.post("/api/simulate/frame")
async def frame(req: FrameRequest):
    run = RUNS.get(req.runId)
    if not run: raise HTTPException(404)
    t = datetime.fromisoformat(req.t.replace("Z",""))
    idx = int(np.argmin([abs((tt - t).total_seconds()) for tt in run["timeline"]]))
    spy = run["spy"][idx]

    legs_out = []
    for leg, sym in run["contracts"].items():
        q = quote_at_time(run["quotes"][leg], run["timeline"][idx])
        if not q:
            legs_out.append({"leg": leg, "bid": None, "ask": None, "mid": None})
            continue
        bid, ask = q.get("b",0), q.get("a",0)
        mid = (bid+ask)/2 if bid and ask else q.get("p",0)
        legs_out.append({"leg": leg, "bid": bid, "ask": ask, "mid": mid})

    pnl = 0
    if run.get("entry"):
        entry_prices = {f["leg"]: f["price"] for f in run["entry"]["fills"]}
        mult = 100
        pnl = (
            (+1)*(legs_out[0]["mid"] - entry_prices["k1Put"]) +
            (-1)*(legs_out[1]["mid"] - entry_prices["kcPut"]) +
            (-1)*(legs_out[2]["mid"] - entry_prices["kcCall"]) +
            (+1)*(legs_out[3]["mid"] - entry_prices["k2Call"])
        )*mult

    return {"t": run["timeline"][idx].isoformat()+"Z", "spy": spy, "legs": legs_out, "pnl": round(pnl,2)}
```

---

## Frontend Changes (React)

- **No major changes** needed; React already calls `/init`, `/frame`, `/execute`, `/sell`.
- Now the chart shows **real SPY price and leg mids** as returned from backend.
- Playback scrubber will walk across the actual day.

---

## Usage

1. Set `POLYGON_API_KEY` in backend `.env`.
2. Run backend (`uvicorn main:app`).
3. Run frontend (`npm run dev`).
4. In UI, pick a **date with data** (e.g. `2025-08-13`), set `kc`, `width`, expiry mode.
5. Hit **Go (Init)** → executes Polygon calls, caches SPY + quotes.
6. Hit **Execute** → open trade at chosen time, playback with scrubber.

---

## Next Steps

- Add **slippage models** in Execute/Sell endpoints.
- Add **Greeks overlay** from Polygon `iv, delta, gamma, theta` if present.
- Implement **iron condor / spreads**: just different leg arrays in `strategy`.

This upgrade keeps the MVP architecture intact but makes the simulation **quote-driven, real, and accurate**.

