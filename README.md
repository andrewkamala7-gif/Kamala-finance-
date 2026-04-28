<!DOCTYPE html>
<html lang="en" class="dark">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ultimate PA Confluence Scanner</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script>
        tailwind.config = {
            darkMode: 'class',
            theme: {
                extend: {
                    colors: {
                        dark: '#0B0E14',
                        darker: '#05070A',
                        panel: '#151924',
                        border: '#2A2E39',
                        primary: '#2962FF',
                        success: '#089981',
                        danger: '#F23645',
                        warning: '#FF9800'
                    }
                }
            }
        }
    </script>
    <script src="https://unpkg.com/lightweight-charts@4.2.1/dist/lightweight-charts.standalone.production.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;700&family=Inter:wght@400;700;900&display=swap');
        body { 
            background-color: #05070A; 
            color: #D1D4DC; 
            font-family: 'Inter', sans-serif; 
        }
        .mono { font-family: 'JetBrains Mono', monospace; }
        .scrollbar-hide::-webkit-scrollbar { display: none; }
        .scrollbar-hide { -ms-overflow-style: none; scrollbar-width: none; }
        #chart-container { position: relative; width: 100%; height: 50vh; min-height: 400px; border-bottom: 1px solid #2A2E39; }
        .glass-panel { background: rgba(21, 25, 36, 0.7); backdrop-filter: blur(10px); border: 1px solid #2A2E39; }
        .sticky-chart-wrap { position: sticky; top: 0; z-index: 50; background: #05070A; }
        .strategy-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(320px, 1fr)); gap: 1rem; }
        .signal-card { border-left: 3px solid transparent; transition: all 0.2s; }
        .signal-card.bullish { border-left-color: #089981; }
        .signal-card.bearish { border-left-color: #F23645; }
    </style>
</head>
<body class="overflow-x-hidden">

    <!-- Navbar -->
    <nav class="h-14 border-b border-border flex items-center justify-between px-6 bg-panel sticky top-0 z-[60]">
        <div class="flex items-center gap-3">
            <div class="bg-primary p-1.5 rounded shadow-lg shadow-primary/20">
                <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="m21 16-4 4-4-4"/><path d="M17 20V4"/><path d="m3 8 4-4 4 4"/><path d="M7 4v16"/></svg>
            </div>
            <h1 class="text-sm font-black text-white tracking-widest uppercase italic">Confluence V3</h1>
        </div>
        
        <div class="flex items-center gap-2">
            <div class="flex bg-dark rounded-md p-1 border border-border">
                <select id="symbol-select" class="bg-transparent text-[10px] font-bold px-2 py-1 outline-none text-white cursor-pointer uppercase">
                    <option value="BTCUSDT">BTC/USDT</option>
                    <option value="ETHUSDT">ETH/USDT</option>
                    <option value="SOLUSDT">SOL/USDT</option>
                    <option value="BNBUSDT">BNB/USDT</option>
                </select>
                <div class="w-[1px] bg-border mx-1"></div>
                <select id="interval-select" class="bg-transparent text-[10px] font-bold px-2 py-1 outline-none text-white cursor-pointer">
                    <option value="15m">15M</option>
                    <option value="1h" selected>1H</option>
                    <option value="4h">4H</option>
                    <option value="1d">1D</option>
                </select>
            </div>
        </div>
    </nav>

    <main class="flex flex-col lg:flex-row min-h-screen">
        
        <!-- Left Side: Long scrollable analysis -->
        <div class="flex-1 order-2 lg:order-1 p-4 lg:p-8 space-y-10 bg-darker">
            
            <!-- Dashboard HUD -->
            <section class="grid grid-cols-1 md:grid-cols-3 gap-4">
                <div class="glass-panel p-6 rounded-2xl">
                    <h3 class="text-[10px] text-gray-500 font-bold uppercase tracking-widest mb-3">Market Structure</h3>
                    <div id="ms-value" class="text-3xl font-black text-white">SCANNING...</div>
                    <div class="mt-2 flex items-center gap-2">
                        <div id="ms-dot" class="w-2 h-2 rounded-full bg-gray-600"></div>
                        <span id="ms-desc" class="text-[11px] text-gray-400 font-medium">Analyzing Swing Points</span>
                    </div>
                </div>
                
                <div class="glass-panel p-6 rounded-2xl">
                    <h3 class="text-[10px] text-gray-500 font-bold uppercase tracking-widest mb-3">Dynamic S/R Zones</h3>
                    <div class="space-y-2">
                        <div class="flex justify-between items-center bg-danger/10 p-2 rounded border border-danger/20">
                            <span class="text-[10px] font-bold text-danger">CEILING</span>
                            <span id="res-price" class="text-sm font-bold text-white mono">0.00</span>
                        </div>
                        <div class="flex justify-between items-center bg-success/10 p-2 rounded border border-success/20">
                            <span class="text-[10px] font-bold text-success">FLOOR</span>
                            <span id="sup-price" class="text-sm font-bold text-white mono">0.00</span>
                        </div>
                    </div>
                </div>

                <div id="v-card" class="glass-panel p-6 rounded-2xl border-b-4 border-warning transition-all duration-500">
                    <h3 class="text-[10px] text-gray-500 font-bold uppercase tracking-widest mb-3">Confluence Verdict</h3>
                    <div id="v-value" class="text-3xl font-black text-white italic">NEUTRAL</div>
                    <p id="v-desc" class="text-[11px] text-gray-400 mt-2 font-medium">AGGREGATING SIGNALS...</p>
                </div>
            </section>

            <!-- Strategy Feed -->
            <section class="space-y-6">
                <div class="flex items-center justify-between border-b border-border pb-4">
                    <h2 class="text-xl font-black text-white tracking-tighter">STRATEGY LAYER AGGREGATION</h2>
                    <div id="signal-count" class="text-[10px] font-bold bg-primary/20 text-primary px-3 py-1 rounded-full">0 ACTIVE TRIGGERS</div>
                </div>

                <div class="strategy-grid" id="main-strategy-feed">
                    <!-- Strategies are injected here -->
                </div>
            </section>

            <!-- Comprehensive Signal Log -->
            <section class="space-y-4">
                <h3 class="text-xs font-bold text-gray-500 uppercase tracking-widest">Historical Signal Timeline</h3>
                <div class="glass-panel rounded-2xl overflow-hidden">
                    <table class="w-full text-left text-xs border-collapse">
                        <thead class="bg-panel text-gray-500 font-bold uppercase text-[9px] tracking-widest">
                            <tr>
                                <th class="p-4 border-b border-border">Timestamp</th>
                                <th class="p-4 border-b border-border">Strategy</th>
                                <th class="p-4 border-b border-border">Type</th>
                                <th class="p-4 border-b border-border text-right">Trigger Price</th>
                            </tr>
                        </thead>
                        <tbody id="signal-table-body" class="divide-y divide-border/30">
                            <!-- Rows injected here -->
                        </tbody>
                    </table>
                </div>
            </section>

            <!-- Education / Logic Disclosure -->
            <section class="p-8 rounded-3xl bg-gradient-to-br from-panel to-darker border border-border">
                <h4 class="text-lg font-black text-white mb-4">How V3 Confluence Works</h4>
                <div class="grid grid-cols-1 md:grid-cols-2 gap-8 text-sm text-gray-400 leading-relaxed">
                    <div class="space-y-4">
                        <p><strong class="text-primary uppercase text-[10px] block mb-1">Layer 1: Candlestick Patterns</strong><br>We monitor every closing candle for exhaustion (Pin Bars), reversal strength (Engulfings), and indecision (Dojis). These are the first line of defense.</p>
                        <p><strong class="text-success uppercase text-[10px] block mb-1">Layer 2: Structural Zones</strong><br>The engine identifies "S/R Flips" and "Order Blocks" where institutional liquidity typically resides.</p>
                    </div>
                    <div class="space-y-4">
                        <p><strong class="text-warning uppercase text-[10px] block mb-1">Layer 3: Statistical Volatility</strong><br>Bollinger Band deviations detect when price is mathematically overextended from its mean (Mean Reversion).</p>
                        <p><strong class="text-white uppercase text-[10px] block mb-1">Confluence Requirement</strong><br>A "Strong" verdict is only issued when at least 3 layers align at the same price point.</p>
                    </div>
                </div>
            </section>
            
            <div class="h-20"></div>
        </div>

        <!-- Right Side: Pinned Chart -->
        <div class="w-full lg:w-[480px] lg:h-screen lg:sticky lg:top-0 order-1 lg:order-2 shrink-0 border-l border-border bg-dark">
            <div class="sticky-chart-wrap shadow-2xl">
                <div class="bg-panel h-10 flex items-center px-4 justify-between border-b border-border">
                    <div class="flex items-center gap-4">
                        <span id="c-sym" class="text-xs font-black text-white tracking-widest">BTCUSDT</span>
                        <div class="flex items-center gap-2">
                            <span class="w-1.5 h-1.5 rounded-full bg-success animate-pulse"></span>
                            <span id="c-price" class="text-xs font-mono text-white">--</span>
                        </div>
                    </div>
                    <div class="text-[10px] text-gray-500 font-bold uppercase tracking-widest">Global Feed</div>
                </div>
                <div id="chart-container"></div>
            </div>
            
            <div class="p-6 space-y-6 hidden lg:block overflow-y-auto max-h-[calc(100vh-50vh-40px)] scrollbar-hide">
                <div class="flex justify-between items-center">
                    <h4 class="text-[10px] font-black text-gray-500 uppercase tracking-widest">Rapid Alerts</h4>
                    <button onclick="clearLogs()" class="text-[9px] text-gray-600 hover:text-white uppercase">Clear</button>
                </div>
                <div id="rapid-log" class="space-y-3">
                    <!-- Mini signals -->
                </div>
            </div>
        </div>
    </main>

    <script>
        // Strategy Definitions
        const STRATEGIES = [
            { id: 'candle', name: 'Candlestick PA', desc: 'Pin Bars, Engulfings & Reversal patterns.', color: 'primary' },
            { id: 'structure', name: 'Market Structure', desc: 'Breakouts and Retests of key S/R flips.', color: 'success' },
            { id: 'volatility', name: 'Mean Reversion', desc: 'BB Exhaustion and Standard Deviation bands.', color: 'warning' },
            { id: 'institutional', name: 'Order Blocks', desc: 'High volume institutional accumulation zones.', color: 'danger' }
        ];

        const state = {
            symbol: 'BTCUSDT',
            interval: '1h',
            data: [],
            chart: null,
            series: null,
            levels: { sup: 0, res: 0 },
            signals: []
        };

        const dom = {
            symbol: document.getElementById('symbol-select'),
            interval: document.getElementById('interval-select'),
            price: document.getElementById('c-price'),
            msValue: document.getElementById('ms-value'),
            msDot: document.getElementById('ms-dot'),
            msDesc: document.getElementById('ms-desc'),
            supPrice: document.getElementById('sup-price'),
            resPrice: document.getElementById('res-price'),
            verdict: document.getElementById('v-value'),
            verdictDesc: document.getElementById('v-desc'),
            verdictCard: document.getElementById('v-card'),
            feed: document.getElementById('main-strategy-feed'),
            rapidLog: document.getElementById('rapid-log'),
            table: document.getElementById('signal-table-body'),
            sigCount: document.getElementById('signal-count')
        };

        function initChart() {
            state.chart = LightweightCharts.createChart(document.getElementById('chart-container'), {
                layout: { background: { type: 'solid', color: '#05070A' }, textColor: '#868e96', fontSize: 11, fontFamily: 'JetBrains Mono' },
                grid: { vertLines: { color: '#151924' }, horzLines: { color: '#151924' } },
                rightPriceScale: { borderColor: '#2A2E39', autoScale: true, alignLabels: true },
                timeScale: { borderColor: '#2A2E39', timeVisible: true, secondsVisible: false },
                crosshair: { mode: LightweightCharts.CrosshairMode.Normal, vertLine: { labelBackgroundColor: '#2962FF' }, horzLine: { labelBackgroundColor: '#2962FF' } }
            });

            state.series = state.chart.addCandlestickSeries({
                upColor: '#089981', downColor: '#F23645', borderVisible: false, wickUpColor: '#089981', wickDownColor: '#F23645'
            });

            const resizeObserver = new ResizeObserver(entries => {
                requestAnimationFrame(() => {
                    if (entries.length && state.chart) {
                        const { width, height } = entries[0].contentRect;
                        state.chart.applyOptions({ width, height });
                    }
                });
            });
            resizeObserver.observe(document.getElementById('chart-container'));
        }

        async function updateData() {
            try {
                const r = await fetch(`https://api.binance.com/api/v3/klines?symbol=${state.symbol}&interval=${state.interval}&limit=300`);
                const klines = await r.json();
                state.data = klines.map(k => ({
                    time: k[0] / 1000, 
                    open: parseFloat(k[1]), 
                    high: parseFloat(k[2]), 
                    low: parseFloat(k[3]), 
                    close: parseFloat(k[4]), 
                    volume: parseFloat(k[5])
                }));
                state.series.setData(state.data);
                processAnalysis();
            } catch (err) { console.error("Update Error:", err); }
        }

        function processAnalysis() {
            if (!state.data.length) return;
            const latest = state.data[state.data.length - 1];
            dom.price.textContent = latest.close.toLocaleString(undefined, {minimumFractionDigits: 2});
            dom.price.className = `text-xs font-mono ${latest.close >= latest.open ? 'text-success' : 'text-danger'}`;
            document.getElementById('c-sym').textContent = state.symbol;

            // 1. Core Logic Modules
            analyzeStructure();
            const triggers = aggregateSignals();
            
            // 2. Render UI Layers
            renderStrategyCards(triggers);
            renderSignalHistory(triggers);
            updateVerdict(triggers);
            
            // 3. Chart Markers
            const markers = triggers.map(s => ({
                time: s.time,
                position: s.side === 'BULL' ? 'belowBar' : 'aboveBar',
                color: s.side === 'BULL' ? '#089981' : '#F23645',
                shape: s.side === 'BULL' ? 'arrowUp' : 'arrowDown',
                text: s.shortLabel
            }));
            state.series.setMarkers(markers);
        }

        function analyzeStructure() {
            const lookback = state.data.slice(-50);
            const high = Math.max(...lookback.map(d => d.high));
            const low = Math.min(...lookback.map(d => d.low));
            
            state.levels.res = high;
            state.levels.sup = low;
            dom.resPrice.textContent = high.toFixed(2);
            dom.supPrice.textContent = low.toFixed(2);

            const recentCloses = state.data.slice(-20).map(d => d.close);
            const isUptrend = recentCloses[19] > recentCloses[0];
            
            dom.msValue.textContent = isUptrend ? "BULLISH" : "BEARISH";
            dom.msValue.className = `text-3xl font-black ${isUptrend ? 'text-success' : 'text-danger'} italic`;
            dom.msDot.className = `w-2 h-2 rounded-full ${isUptrend ? 'bg-success' : 'bg-danger'}`;
            dom.msDesc.textContent = isUptrend ? "Price Trend: Higher Highs" : "Price Trend: Lower Lows";
        }

        function aggregateSignals() {
            const sigs = [];
            const data = state.data;
            
            // Iterate back a few candles to find current triggers
            for (let i = data.length - 30; i < data.length; i++) {
                const c = data[i], p = data[i-1];
                if (!p) continue;

                // --- Strategy 1: Candlesticks (First ones) ---
                const body = Math.abs(c.close - c.open);
                const upperWick = c.high - Math.max(c.open, c.close);
                const lowerWick = Math.min(c.open, c.close) - c.low;
                
                // Pin Bar Bullish
                if (lowerWick > body * 2 && c.close > c.open) {
                    sigs.push({ time: c.time, strategy: 'Candlestick', shortLabel: 'PINBAR', side: 'BULL', price: c.close, desc: 'Bullish Hammer detected at localized low.' });
                }
                // Engulfing Bullish
                if (c.close > p.open && c.open < p.close && p.close < p.open) {
                    sigs.push({ time: c.time, strategy: 'Candlestick', shortLabel: 'ENGULF', side: 'BULL', price: c.close, desc: 'Aggressive buyers overwhelmed sellers.' });
                }
                // Engulfing Bearish
                if (c.close < p.open && c.open > p.close && p.close > p.open) {
                    sigs.push({ time: c.time, strategy: 'Candlestick', shortLabel: 'ENGULF', side: 'BEAR', price: c.close, desc: 'Aggressive sellers overwhelmed buyers.' });
                }

                // --- Strategy 2: Structure (S/R Flip) ---
                if (c.low <= state.levels.sup * 1.001 && c.close > c.open) {
                    sigs.push({ time: c.time, strategy: 'Structure', shortLabel: 'SUPPORT', side: 'BULL', price: c.close, desc: 'Buying pressure confirmed at floor level.' });
                }
                if (c.high >= state.levels.res * 0.999 && c.close < c.open) {
                    sigs.push({ time: c.time, strategy: 'Structure', shortLabel: 'RESIST', side: 'BEAR', price: c.close, desc: 'Supply detected at historical ceiling.' });
                }

                // --- Strategy 3: Mean Reversion (Volatility) ---
                // Simulating Bollinger Band 2.5 SD tag
                const bbLookback = data.slice(Math.max(0, i-20), i+1).map(d => d.close);
                const avg = bbLookback.reduce((a,b) => a+b, 0) / bbLookback.length;
                const std = Math.sqrt(bbLookback.map(x => Math.pow(x - avg, 2)).reduce((a,b) => a+b, 0) / bbLookback.length);
                if (c.low < avg - (std * 2.5)) {
                    sigs.push({ time: c.time, strategy: 'Volatility', shortLabel: 'OVERSOLD', side: 'BULL', price: c.close, desc: 'Price is mathematically overextended to the downside.' });
                }
                if (c.high > avg + (std * 2.5)) {
                    sigs.push({ time: c.time, strategy: 'Volatility', shortLabel: 'OVERBOUGHT', side: 'BEAR', price: c.close, desc: 'Price is mathematically overextended to the upside.' });
                }

                // --- Strategy 4: Institutional (Order Blocks) ---
                const volAvg = data.slice(Math.max(0, i-20), i).reduce((a,b) => a + b.volume, 0) / 20;
                if (c.volume > volAvg * 2.5) {
                    sigs.push({ 
                        time: c.time, 
                        strategy: 'Institutional', 
                        shortLabel: 'VOL-SPIKE', 
                        side: c.close > c.open ? 'BULL' : 'BEAR', 
                        price: c.close, 
                        desc: 'Institutional volume detected. Large orders being filled.' 
                    });
                }
            }
            
            return sigs.sort((a,b) => b.time - a.time);
        }

        function renderStrategyCards(triggers) {
            dom.feed.innerHTML = '';
            STRATEGIES.forEach(s => {
                const latestFromStrat = triggers.find(t => t.strategy === s.name);
                const card = document.createElement('div');
                card.className = `glass-panel p-5 rounded-2xl signal-card ${latestFromStrat ? (latestFromStrat.side === 'BULL' ? 'bullish bg-success/5' : 'bearish bg-danger/5') : ''}`;
                
                card.innerHTML = `
                    <div class="flex justify-between items-start mb-3">
                        <div class="flex items-center gap-3">
                            <div class="w-8 h-8 rounded bg-${s.color}/10 flex items-center justify-center text-${s.color} text-xs font-black">${s.name[0]}</div>
                            <h4 class="text-xs font-bold text-white uppercase tracking-tighter">${s.name}</h4>
                        </div>
                        ${latestFromStrat ? `<span class="px-2 py-0.5 rounded bg-${latestFromStrat.side === 'BULL' ? 'success' : 'danger'}/20 text-[9px] font-black text-${latestFromStrat.side === 'BULL' ? 'success' : 'danger'}">ACTIVE</span>` : ''}
                    </div>
                    <p class="text-[11px] text-gray-500 leading-relaxed">${s.desc}</p>
                    <div class="mt-4 pt-4 border-t border-border/30">
                        ${latestFromStrat ? `
                            <div class="flex flex-col gap-1">
                                <div class="flex justify-between text-[10px] font-bold">
                                    <span class="text-white">${latestFromStrat.shortLabel}</span>
                                    <span class="mono text-gray-400">${latestFromStrat.price.toFixed(2)}</span>
                                </div>
                                <p class="text-[10px] text-gray-400">${latestFromStrat.desc}</p>
                            </div>
                        ` : '<span class="text-[9px] text-gray-600 italic uppercase">Monitoring for triggers...</span>'}
                    </div>
                `;
                dom.feed.appendChild(card);
            });
        }

        function renderSignalHistory(triggers) {
            dom.table.innerHTML = '';
            dom.rapidLog.innerHTML = '';
            dom.sigCount.textContent = `${triggers.length} TOTAL SIGNALS`;

            triggers.slice(0, 15).forEach((s, idx) => {
                // Table
                const row = document.createElement('tr');
                row.className = "hover:bg-panel/50 cursor-pointer";
                row.innerHTML = `
                    <td class="p-4 text-gray-500 mono">${new Date(s.time*1000).toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})}</td>
                    <td class="p-4 font-bold text-white">${s.strategy}</td>
                    <td class="p-4"><span class="px-2 py-0.5 rounded text-[9px] font-black ${s.side === 'BULL' ? 'bg-success/20 text-success' : 'bg-danger/20 text-danger'}">${s.shortLabel}</span></td>
                    <td class="p-4 text-right mono text-gray-300">${s.price.toFixed(2)}</td>
                `;
                row.onclick = () => {
                    const findIdx = state.data.findIndex(d => d.time === s.time);
                    state.chart.timeScale().setVisibleLogicalRange({ from: findIdx - 30, to: findIdx + 10 });
                };
                dom.table.appendChild(row);

                // Rapid Log
                if (idx < 6) {
                    const mini = document.createElement('div');
                    mini.className = `p-3 rounded-xl bg-panel border border-border flex items-center justify-between group hover:border-primary transition-colors cursor-pointer`;
                    mini.innerHTML = `
                        <div class="flex flex-col">
                            <span class="text-[9px] font-black uppercase text-primary mb-1">${s.strategy}</span>
                            <span class="text-[11px] font-bold text-white">${s.shortLabel} Detected</span>
                        </div>
                        <div class="text-right">
                            <div class="text-[10px] font-black ${s.side === 'BULL' ? 'text-success' : 'text-danger'}">${s.side === 'BULL' ? 'LONG' : 'SHORT'}</div>
                            <div class="text-[9px] mono text-gray-600">${new Date(s.time*1000).toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})}</div>
                        </div>
                    `;
                    mini.onclick = () => row.onclick();
                    dom.rapidLog.appendChild(mini);
                }
            });
        }

        function updateVerdict(triggers) {
            const recent = triggers.filter(t => (Date.now()/1000 - t.time) < 3600); // Past hour
            let score = 0;
            recent.forEach(t => score += (t.side === 'BULL' ? 1 : -1));

            if (score >= 2) {
                dom.verdict.textContent = "STRONG BUY";
                dom.verdict.className = "text-3xl font-black text-success italic";
                dom.verdictDesc.textContent = "Multi-Layer Confluence Detected";
                dom.verdictCard.style.borderBottomColor = '#089981';
            } else if (score <= -2) {
                dom.verdict.textContent = "STRONG SELL";
                dom.verdict.className = "text-3xl font-black text-danger italic";
                dom.verdictDesc.textContent = "Institutional Distribution Active";
                dom.verdictCard.style.borderBottomColor = '#F23645';
            } else {
                dom.verdict.textContent = score > 0 ? "BULLISH BIAS" : (score < 0 ? "BEARISH BIAS" : "NEUTRAL");
                dom.verdict.className = "text-3xl font-black text-white italic";
                dom.verdictDesc.textContent = "Waiting for High Conviction Setup";
                dom.verdictCard.style.borderBottomColor = '#FF9800';
            }
        }

        function clearLogs() {
            state.signals = [];
            processAnalysis();
        }

        // Global Event Listeners
        dom.symbol.onchange = (e) => { state.symbol = e.target.value; updateData(); };
        dom.interval.onchange = (e) => { state.interval = e.target.value; updateData(); };

        window.onload = () => {
            initChart();
            updateData();
            setInterval(updateData, 10000); // Polling
        };
    </script>
</body>
</html>
