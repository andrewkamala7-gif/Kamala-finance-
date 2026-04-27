[index.html](https://github.com/user-attachments/files/27119182/index.html)
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>NovaSignals - Crypto Analysis Engine</title>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com/3.4.1"></script>
    
    <!-- React & ReactDOM -->
    <script crossorigin src="https://unpkg.com/react@18.2.0/umd/react.production.min.js"></script>
    <script crossorigin src="https://unpkg.com/react-dom@18.2.0/umd/react-dom.production.min.js"></script>
    
    <!-- Prop-Types (Required by Recharts UMD) -->
    <script src="https://unpkg.com/prop-types@15.8.1/prop-types.min.js"></script>

    <!-- Recharts (Must come after React and Prop-Types) -->
    <script src="https://unpkg.com/recharts@2.12.7/umd/Recharts.js"></script>
    
    <!-- Babel -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&family=JetBrains+Mono:wght@400;700&display=swap');
        
        body {
            font-family: 'Inter', sans-serif;
            background-color: #020617;
            color: #f1f5f9;
        }

        .mono { font-family: 'JetBrains+Mono', monospace; }

        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: #020617; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #1e293b; border-radius: 10px; }

        .signal-pulse {
            animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
        }

        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: .5; }
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef, useMemo } = React;
        
        // Use window.Recharts to ensure we access the UMD global correctly
        const { AreaChart, Area, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } = window.Recharts;

        // --- Constants ---
        const PAIRS = [
            { id: 'btcusdt', symbol: 'BTC/USDT', name: 'Bitcoin' },
            { id: 'ethusdt', symbol: 'ETH/USDT', name: 'Ethereum' },
            { id: 'solusdt', symbol: 'SOL/USDT', name: 'Solana' },
            { id: 'bnbusdt', symbol: 'BNB/USDT', name: 'Binance Coin' },
            { id: 'xrpusdt', symbol: 'XRP/USDT', name: 'Ripple' },
            { id: 'linkusdt', symbol: 'LINK/USDT', name: 'Chainlink' },
            { id: 'adausdt', symbol: 'ADA/USDT', name: 'Cardano' }
        ];

        // --- Components ---
        const Icon = ({ name, className }) => {
            const icons = {
                trend: <path d="M22 7L13.5 15.5L8.5 10.5L2 17M16 7H22V13" />,
                bolt: <path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z" />,
                bell: <path d="M18 8A6 6 0 0 0 6 8c0 7-3 9-3 9h18s-3-2-3-9M13.73 21a2 2 0 0 1-3.46 0" />,
                shield: <path d="M12 22s8-4 8-10V5l-8-3-8 3v7c0 6 8 10 8 10z" />
            };
            return (
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" className={className}>
                    {icons[name]}
                </svg>
            );
        };

        function App() {
            const [activeId, setActiveId] = useState(PAIRS[0].id);
            const [marketData, setMarketData] = useState({});
            const [history, setHistory] = useState({});
            const [signals, setSignals] = useState({});
            
            const ws = useRef(null);

            // Logic to determine signal based on price action
            const getSignal = (currentPrice, hist) => {
                if (!hist || hist.length < 10) return { type: 'NEUTRAL', strength: 0, reason: 'Accumulating Data' };
                
                const prices = hist.map(d => d.price);
                const avg = prices.reduce((a, b) => a + b, 0) / prices.length;
                const lastPrice = prices[prices.length - 1];
                const prevPrice = prices[prices.length - 2];
                
                const momentum = lastPrice > prevPrice;
                const isAboveAvg = lastPrice > avg;

                if (momentum && isAboveAvg) return { type: 'STRONG BUY', strength: 85, color: 'text-green-400', bg: 'bg-green-500/10' };
                if (!momentum && !isAboveAvg) return { type: 'STRONG SELL', strength: 92, color: 'text-red-400', bg: 'bg-red-500/10' };
                if (momentum && !isAboveAvg) return { type: 'WEAK BUY', strength: 45, color: 'text-emerald-500', bg: 'bg-emerald-500/10' };
                return { type: 'NEUTRAL', strength: 10, color: 'text-slate-400', bg: 'bg-slate-500/10' };
            };

            useEffect(() => {
                const streams = PAIRS.map(p => `${p.id}@trade`).join('/');
                ws.current = new WebSocket(`wss://stream.binance.com:9443/stream?streams=${streams}`);

                ws.current.onmessage = (e) => {
                    const { data } = JSON.parse(e.data);
                    const symbol = data.s.toLowerCase();
                    const price = parseFloat(data.p);

                    setMarketData(prev => ({ ...prev, [symbol]: price }));
                    
                    setHistory(prev => {
                        const symHist = prev[symbol] || [];
                        const newPoint = { time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' }), price };
                        const updated = [...symHist, newPoint].slice(-40);
                        
                        // Update signal
                        const signal = getSignal(price, updated);
                        setSignals(s => ({ ...s, [symbol]: signal }));
                        
                        return { ...prev, [symbol]: updated };
                    });
                };

                return () => {
                    if (ws.current) ws.current.close();
                };
            }, []);

            const activePair = PAIRS.find(p => p.id === activeId);
            const currentSignal = signals[activeId] || { type: 'ANALYZING', strength: 0, color: 'text-slate-500' };

            return (
                <div className="flex h-screen overflow-hidden">
                    {/* Sidebar */}
                    <aside className="w-80 bg-slate-950 border-r border-slate-800 flex flex-col">
                        <div className="p-6 border-b border-slate-800 flex items-center gap-3">
                            <div className="bg-indigo-600 p-2 rounded-lg shadow-lg shadow-indigo-500/20">
                                <Icon name="bolt" className="w-5 h-5 text-white" />
                            </div>
                            <h1 className="font-bold text-xl tracking-tight">Nova<span className="text-indigo-400">Signals</span></h1>
                        </div>

                        <div className="flex-1 overflow-y-auto custom-scrollbar p-3 space-y-2">
                            <p className="text-[10px] uppercase font-bold text-slate-500 px-3 py-2 tracking-widest">Market Monitoring</p>
                            {PAIRS.map(pair => {
                                const price = marketData[pair.id];
                                const signal = signals[pair.id];
                                return (
                                    <button 
                                        key={pair.id}
                                        onClick={() => setActiveId(pair.id)}
                                        className={`w-full p-4 rounded-xl flex items-center justify-between transition-all border ${activeId === pair.id ? 'bg-indigo-600/10 border-indigo-500/50' : 'bg-transparent border-transparent hover:bg-slate-900'}`}
                                    >
                                        <div className="text-left">
                                            <div className="font-bold text-sm">{pair.symbol}</div>
                                            <div className="text-[10px] font-mono text-slate-400">${price?.toLocaleString()}</div>
                                        </div>
                                        {signal && (
                                            <div className={`text-[10px] font-bold px-2 py-1 rounded ${signal.bg} ${signal.color}`}>
                                                {signal.type}
                                            </div>
                                        )}
                                    </button>
                                );
                            })}
                        </div>
                    </aside>

                    {/* Main Content */}
                    <main className="flex-1 flex flex-col bg-[#020617]">
                        <header className="h-20 border-b border-slate-800 flex items-center justify-between px-8 bg-slate-950/50 backdrop-blur-xl sticky top-0 z-10">
                            <div className="flex items-center gap-6">
                                <div>
                                    <h2 className="text-2xl font-bold">{activePair.symbol}</h2>
                                    <p className="text-xs text-slate-500">{activePair.name} / US Dollar</p>
                                </div>
                                <div className="h-8 w-[1px] bg-slate-800"></div>
                                <div>
                                    <p className="text-xs text-slate-500 mb-1">Live Price</p>
                                    <p className="text-xl font-mono font-bold text-indigo-400">${marketData[activeId]?.toLocaleString() || '---'}</p>
                                </div>
                            </div>
                            <div className="flex items-center gap-4">
                                <div className="flex items-center gap-2 bg-slate-900 rounded-full px-4 py-2 border border-slate-800">
                                    <div className="w-2 h-2 rounded-full bg-green-500 signal-pulse"></div>
                                    <span className="text-xs font-bold text-slate-300">WebSocket Connected</span>
                                </div>
                            </div>
                        </header>

                        <div className="p-8 space-y-8 overflow-y-auto flex-1 custom-scrollbar">
                            {/* Signal Hero Card */}
                            <div className={`rounded-3xl p-8 border border-slate-800 shadow-2xl relative overflow-hidden bg-gradient-to-br from-slate-900 to-black`}>
                                <div className="absolute top-0 right-0 p-8 opacity-10">
                                    <Icon name="trend" className="w-32 h-32" />
                                </div>
                                <div className="relative z-10 grid grid-cols-1 md:grid-cols-3 gap-8">
                                    <div className="md:col-span-2">
                                        <div className="flex items-center gap-3 mb-4">
                                            <span className="text-xs font-black tracking-widest uppercase text-indigo-400">Current AI Signal</span>
                                            <div className="h-[1px] flex-1 bg-slate-800"></div>
                                        </div>
                                        <h3 className={`text-6xl font-black mb-4 tracking-tighter ${currentSignal.color}`}>
                                            {currentSignal.type}
                                        </h3>
                                        <p className="text-slate-400 max-w-md">
                                            Our technical engine has detected a {currentSignal.type.toLowerCase()} opportunity based on real-time volume and price momentum analysis.
                                        </p>
                                    </div>
                                    <div className="flex flex-col justify-center items-center md:items-end">
                                        <div className="text-center md:text-right">
                                            <p className="text-xs text-slate-500 uppercase font-bold tracking-widest mb-1">Signal Strength</p>
                                            <div className="text-5xl font-mono font-black text-white">{currentSignal.strength}%</div>
                                            <div className="w-full h-2 bg-slate-800 rounded-full mt-4 overflow-hidden w-40">
                                                <div 
                                                    className="h-full bg-indigo-500 transition-all duration-1000" 
                                                    style={{ width: `${currentSignal.strength}%` }}
                                                ></div>
                                            </div>
                                        </div>
                                    </div>
                                </div>
                            </div>

                            {/* Chart Area */}
                            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
                                <div className="lg:col-span-2 bg-slate-950 border border-slate-800 rounded-3xl p-6 h-[450px] flex flex-col">
                                    <div className="flex items-center justify-between mb-6">
                                        <h4 className="font-bold flex items-center gap-2"><Icon name="trend" className="w-4 h-4 text-indigo-400" /> Technical Chart</h4>
                                        <div className="flex gap-2">
                                            {['Live', '1H', '4H', '1D'].map(t => (
                                                <button key={t} className={`px-3 py-1 text-[10px] font-bold rounded ${t === 'Live' ? 'bg-indigo-600' : 'bg-slate-900 hover:bg-slate-800'}`}>
                                                    {t}
                                                </button>
                                            ))}
                                        </div>
                                    </div>
                                    <div className="flex-1 w-full min-h-0">
                                        <ResponsiveContainer width="100%" height="100%">
                                            <AreaChart data={history[activeId] || []}>
                                                <defs>
                                                    <linearGradient id="chartGradient" x1="0" y1="0" x2="0" y2="1">
                                                        <stop offset="5%" stopColor="#6366f1" stopOpacity={0.3}/>
                                                        <stop offset="95%" stopColor="#6366f1" stopOpacity={0}/>
                                                    </linearGradient>
                                                </defs>
                                                <CartesianGrid strokeDasharray="3 3" stroke="#1e293b" vertical={false} />
                                                <XAxis dataKey="time" hide />
                                                <YAxis domain={['auto', 'auto']} stroke="#475569" tick={{fontSize: 10}} width={60} />
                                                <Tooltip 
                                                    contentStyle={{ backgroundColor: '#020617', border: '1px solid #1e293b', borderRadius: '12px' }}
                                                    itemStyle={{ color: '#818cf8' }}
                                                />
                                                <Area 
                                                    type="monotone" 
                                                    dataKey="price" 
                                                    stroke="#6366f1" 
                                                    strokeWidth={3} 
                                                    fillOpacity={1} 
                                                    fill="url(#chartGradient)" 
                                                    isAnimationActive={false} 
                                                />
                                            </AreaChart>
                                        </ResponsiveContainer>
                                    </div>
                                </div>

                                {/* Checklist/Strategy */}
                                <div className="bg-slate-950 border border-slate-800 rounded-3xl p-6 space-y-6">
                                    <h4 className="font-bold flex items-center gap-2"><Icon name="shield" className="w-4 h-4 text-indigo-400" /> Analysis Report</h4>
                                    
                                    <div className="space-y-4">
                                        <div className="flex items-start gap-3 p-3 bg-slate-900/50 rounded-xl border border-slate-800">
                                            <div className="w-5 h-5 rounded bg-green-500/20 flex items-center justify-center text-green-500 text-[10px] font-bold mt-0.5">?</div>
                                            <div>
                                                <p className="text-xs font-bold text-slate-200">Volume Profile</p>
                                                <p className="text-[10px] text-slate-500">Positive inflow detected in last 5m.</p>
                                            </div>
                                        </div>
                                        <div className="flex items-start gap-3 p-3 bg-slate-900/50 rounded-xl border border-slate-800">
                                            <div className="w-5 h-5 rounded bg-indigo-500/20 flex items-center justify-center text-indigo-400 text-[10px] font-bold mt-0.5">i</div>
                                            <div>
                                                <p className="text-xs font-bold text-slate-200">RSI Indicator</p>
                                                <p className="text-[10px] text-slate-500">Currently neutral. No overbought risk.</p>
                                            </div>
                                        </div>
                                        <div className="flex items-start gap-3 p-3 bg-slate-900/50 rounded-xl border border-slate-800">
                                            <div className="w-5 h-5 rounded bg-red-500/20 flex items-center justify-center text-red-500 text-[10px] font-bold mt-0.5">!</div>
                                            <div>
                                                <p className="text-xs font-bold text-slate-200">Support Level</p>
                                                <p className="text-[10px] text-slate-500">Major support at -2.4% from current.</p>
                                            </div>
                                        </div>
                                    </div>

                                    <div className="pt-4">
                                        <button className="w-full bg-indigo-600 hover:bg-indigo-500 text-white font-bold py-4 rounded-2xl transition-all shadow-lg shadow-indigo-600/20 flex items-center justify-center gap-2">
                                            <Icon name="bell" className="w-4 h-4" /> Enable Signal Alerts
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </main>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
