# Roadmap & Coding Agent Prompt

## Roadmap
- Phase 1: MVP scaffold (FastAPI stubs + React shell)
- Phase 2: Integrate Polygon API (SPY bars, option quotes)
- Phase 3: Add caching, greeks overlays, explicit K1/K2, NYSE holiday calendar
- Phase 4: Extend strategies (iron condor, bull put spread, etc.)
- Phase 5: Refine UX (report downloads, side-by-side day compare)

## Prompt for Coding Agent
You are to build an options trading simulator based on real historical data. The goal is to replay past trading days and show how iron butterfly (and later condor, spreads) strategies would have performed intraday.

Requirements:
- Frontend: React + Plotly/D3 scrubber UI (see scaffold doc)
- Backend: FastAPI service that fetches real SPY 1m bars and option quotes from Polygon API (see integration doc)
- Endpoints: /simulate/init, /simulate/execute, /simulate/frame, /simulate/sell, /simulate/report
- Fill model: mid, nbbo, mid±slippage
- User flow: init → execute → play/scrub → sell → report
- Deliverables: working Dockerized stack, with clear README instructions
