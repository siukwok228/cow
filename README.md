<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>資產管理專家 v3.7.8 - 終極修復</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body { background-color: #f1f5f9 !important; color: #1e293b !important; font-family: 'PingFang TC', sans-serif; }
        .card { background-color: white !important; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.05); }
        
        /* 錄入區樣式 */
        input, select { 
            border: 1px solid #cbd5e1 !important; 
            border-radius: 8px !important; 
            padding: 8px 10px !important;
            background-color: white !important;
            color: #1e293b !important;
            font-size: 14px !important;
        }
        input:focus { border-color: #3b82f6 !important; outline: none; }

        /* 左側狀態條 */
        .border-outflow { border-left: 6px solid #ef4444 !important; }
        .border-inflow { border-left: 6px solid #10b981 !important; }
        
        /* 強制讓日期和類型不換行 */
        .nowrap-cell { white-space: nowrap !important; }
    </style>
</head>
<body class="p-4 md:p-10">

    <div class="max-w-7xl mx-auto">
        <header class="mb-10 flex flex-col md:flex-row justify-between items-center gap-4 border-b border-slate-200 pb-6">
            <div>
                <h1 class="text-3xl font-black text-slate-900 tracking-tight">📊 投資流水管理 v3.7.8</h1>
                <p class="text-slate-500 font-medium mt-1">支出(紅) | 收入(綠)</p>
            </div>
            <button onclick="exportCSV()" class="bg-slate-800 hover:bg-black text-white px-6 py-2.5 rounded-xl font-bold transition shadow-lg text-sm">匯出 CSV</button>
        </header>

        <div id="statsGrid" class="grid grid-cols-1 md:grid-cols-4 gap-5 mb-10"></div>

        <div class="card p-5 mb-8 border border-slate-200 shadow-xl">
            <form id="mainForm" class="flex flex-col md:flex-row items-center gap-3">
                <input type="date" id="entryDate" class="w-full md:w-auto">
                <select id="entryType" onchange="toggleFields()" class="w-full md:w-auto font-bold min-w-[130px]">
                    <option value="買入">買入 (支出)</option>
                    <option value="賣出">賣出 (收入)</option>
                    <option value="費用">費用 (支出)</option>
                    <option value="收入">收入 (股息)</option>
                </select>
                <input type="text" id="symbol" placeholder="代號" class="w-full md:w-32 uppercase font-bold flex-grow">
                <div id="tradeFields" class="w-full md:w-auto flex items-center gap-2 flex-grow">
                    <input type="number" id="price" step="0.001" placeholder="單價" class="w-1/2 md:w-28">
                    <input type="number" id="quantity" placeholder="數量" class="w-1/2 md:w-24">
                </div>
                <input type="number" id="feeOrAmount" step="0.01" value="0" class="w-full md:w-28 font-black text-blue-700 bg-blue-50">
                <button type="submit" class="w-full md:w-auto bg-blue-600 text-white px-6 py-2.5 rounded-lg font-black shadow-md">記錄</button>
            </form>
            <div class="mt-3">
                <input type="text" id="note" placeholder="備註事項..." class="w-full text-sm italic !bg-slate-50 border-none rounded-md p-2">
            </div>
        </div>

        <div class="flex flex-col md:flex-row justify-between items-center mb-6 gap-4">
            <h2 class="text-xl font-bold text-slate-800">📜 明細清單</h2>
            <div class="w-full md:w-80">
                <input type="text" id="filterInput" onkeyup="render()" placeholder="🔍 輸入代號搜尋..." 
                       class="w-full !border-2 !border-slate-800 !rounded-2xl shadow-xl font-bold p-3">
            </div>
        </div>

        <div class="card overflow-hidden border border-slate-200 shadow-2xl">
            <div class="overflow-x-auto">
                <table class="w-full text-sm text-left table-fixed min-w-[900px]">
                    <thead>
                        <tr style="background-color: #1e293b !important;">
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 140px;">日期</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 100px;">類型</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 120px; text-center">代號</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 180px; text-align: right;">交易詳情</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 120px; text-align: right;">附加金額</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 150px; text-align: right;">現金流</th>
                            <th style="padding: 16px 12px; color: #ffffff !important; font-weight: 800; width: 80px; text-center">操作</th>
                        </tr>
                    </thead>
                    <tbody id="recordBody" class="divide-y divide-slate-100 bg-white"></tbody>
                </table>
            </div>
        </div>
        <div class="h-20"></div>
    </div>

    <script>
        const initDate = () => { document.getElementById('entryDate').value = new Date().toISOString().split('T')[0]; };
        initDate();
        let records = JSON.parse(localStorage.getItem('v3_7_portfolio_data')) || [];

        function toggleFields() {
            const isTrade = (document.getElementById('entryType').value === '買入' || document.getElementById('entryType').value === '賣出');
            document.getElementById('tradeFields').style.opacity = isTrade ? "1" : "0.3";
        }

        function render() {
            const filter = document.getElementById('filterInput').value.toUpperCase();
            const tbody = document.getElementById('recordBody');
            tbody.innerHTML = '';
            records.sort((a, b) => new Date(b.date) - new Date(a.date));
            const filtered = records.filter(r => r.symbol.includes(filter));
            
            let totalBuyQty = 0, totalBuyCost = 0, totalSellQty = 0, totalCashFlow = 0, totalFees = 0;

            filtered.forEach(r => {
                const subtotal = (r.price || 0) * (r.quantity || 0);
                let rowCash = 0;
                if (r.type === '買入') { totalBuyQty += r.quantity; totalBuyCost += (subtotal + Math.abs(r.feeOrAmount)); rowCash = -subtotal + r.feeOrAmount; }
                else if (r.type === '賣出') { totalSellQty += r.quantity; rowCash = subtotal + r.feeOrAmount; }
                else { rowCash = r.feeOrAmount; }
                totalCashFlow += rowCash; totalFees += r.feeOrAmount;

                const isOut = (r.type === '買入' || r.type === '費用');
                tbody.innerHTML += `
                    <tr class="hover:bg-slate-50 ${isOut ? 'border-outflow' : 'border-inflow'}">
                        <td class="px-6 py-4 font-mono font-bold text-slate-600 nowrap-cell">${r.date}</td>
                        <td class="px-6 py-4 font-black ${isOut ? 'text-red-600' : 'text-green-600'} nowrap-cell">${r.type}</td>
                        <td class="px-6 py-4 font-black text-slate-800 text-center">${r.symbol}</td>
                        <td class="px-6 py-4 text-right font-medium text-slate-600">${r.price ? `${r.price.toLocaleString()} × ${r.quantity.toLocaleString()}` : '--'}</td>
                        <td class="px-6 py-4 text-right font-bold ${r.feeOrAmount < 0 ? 'text-red-500' : 'text-green-600'}">${r.feeOrAmount.toLocaleString()}</td>
                        <td class="px-6 py-4 text-right font-black ${rowCash >= 0 ? 'text-green-600' : 'text-red-600'}">${rowCash.toLocaleString(undefined, {minimumFractionDigits: 2})}</td>
                        <td class="px-6 py-4 text-center">
                            <button onclick="deleteRecord(${r.id})" class="text-slate-300 hover:text-red-500 text-lg">✕</button>
                        </td>
                    </tr>
                `;
            });
            updateStats(filter, totalBuyQty - totalSellQty, totalBuyQty > 0 ? (totalBuyCost / totalBuyQty) : 0, totalFees, totalCashFlow);
        }

        function updateStats(filter, q, c, f, t) {
            const grid = document.getElementById('statsGrid');
            if (filter) {
                grid.innerHTML = `
                    <div class="card p-6 border-b-4 border-blue-500"><p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">目前持倉</p><p class="text-2xl font-black">${q.toLocaleString()}</p></div>
                    <div class="card p-6 border-b-4 border-amber-500"><p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">平均成本</p><p class="text-2xl font-black">$${c.toLocaleString(undefined,{minimumFractionDigits: 2})}</p></div>
                    <div class="card p-6 border-b-4 border-slate-300"><p class="text-[10px] font-black text-slate-400 uppercase tracking-widest">累積息費</p><p class="text-2xl font-black">${f.toLocaleString()}</p></div>
                    <div class="card p-6 bg-slate-800 text-white shadow-xl"><p class="text-[10px] font-bold text-slate-400 uppercase tracking-widest">標的淨流</p><p class="text-2xl font-black ${t>=0?'text-green-400':'text-red-400'}">$${t.toLocaleString(undefined,{minimumFractionDigits: 2})}</p></div>
                `;
            } else {
                grid.innerHTML = `<div class="card p-6 border-b-4 border-slate-400 md:col-span-2"><p class="text-xs font-black text-slate-400 uppercase">帳戶流水總計</p><p class="text-2xl font-black">$${t.toLocaleString(undefined,{minimumFractionDigits: 2})}</p></div>
                <div class="card p-6 border-b-4 border-red-500 md:col-span-1"><p class="text-xs font-black text-slate-400 uppercase">總支出費用</p><p class="text-2xl font-black text-red-600">$${f.toLocaleString()}</p></div>
                <div class="card p-6 bg-amber-50 md:col-span-1 border border-amber-200 flex items-center justify-center font-bold text-amber-700 italic text-sm">💡 輸入代號查詢損益</div>`;
            }
        }

        document.getElementById('mainForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const symbol = document.getElementById('symbol').value.toUpperCase();
            if(!symbol) return alert('請輸入股票代號');
            records.push({ id: Date.now(), date: document.getElementById('entryDate').value, type: document.getElementById('entryType').value, symbol, price: parseFloat(document.getElementById('price').value || 0), quantity: parseInt(document.getElementById('quantity').value || 0), feeOrAmount: parseFloat(document.getElementById('feeOrAmount').value || 0), note: document.getElementById('note').value });
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            render(); document.getElementById('mainForm').reset(); initDate();
        });

        function deleteRecord(id) { if(confirm('確定刪除？')) { records = records.filter(r => r.id !== id); localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records)); render(); } }
        function exportCSV() { let csv = "\uFEFF日期,類型,代號,單價,數量,附加金額\n"; records.forEach(r => csv += `${r.date},${r.type},${r.symbol},${r.price},${r.quantity},${r.feeOrAmount}\n`); const link = document.createElement("a"); link.href = URL.createObjectURL(new Blob([csv], { type: 'text/csv;charset=utf-8;' })); link.download = `Portfolio_v3.7.8.csv`; link.click(); }
        render();
    </script>
</body>
</html>
