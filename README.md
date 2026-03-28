<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>資產管理專家 v3.7.6 - 旗艦美化版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* 全局強制背景，防止深色模式干擾 */
        body { background-color: #f1f5f9 !important; color: #1e293b !important; font-family: 'PingFang TC', sans-serif; }
        .card { background-color: white !important; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.05); }
        
        /* 強制表頭樣式：解決文字看不見的問題 */
        .table-header { background-color: #1e293b !important; }
        .table-header th { color: #ffffff !important; font-weight: 700 !important; padding: 16px 12px !important; }

        /* 輸入框美化 */
        input, select { 
            border: 1px solid #cbd5e1 !important; 
            border-radius: 8px !important; 
            padding: 10px !important;
            background-color: white !important;
            color: #1e293b !important;
        }
        input:focus { border-color: #3b82f6 !important; ring: 2px #bfdbfe; outline: none; }

        .border-outflow { border-left: 6px solid #ef4444 !important; }
        .border-inflow { border-left: 6px solid #10b981 !important; }
    </style>
</head>
<body class="p-4 md:p-10">

    <div class="max-w-6xl mx-auto">
        <header class="mb-10 flex flex-col md:flex-row justify-between items-center gap-4">
            <div>
                <h1 class="text-3xl font-black text-slate-900 tracking-tight">📊 投資流水管理 <span class="text-blue-600">v3.7.6</span></h1>
                <p class="text-slate-500 font-medium mt-1">支出統一標紅 | 收入統一標綠</p>
            </div>
            <button onclick="exportCSV()" class="bg-slate-800 hover:bg-black text-white px-6 py-2.5 rounded-xl font-bold transition shadow-lg text-sm">
                匯出 CSV 備份
            </button>
        </header>

        <div id="statsGrid" class="grid grid-cols-1 md:grid-cols-4 gap-5 mb-10"></div>

        <div class="card p-8 mb-10 border-t-8 border-slate-800">
            <h3 class="text-sm font-black text-slate-400 uppercase tracking-widest mb-6">新增交易紀錄</h3>
            <form id="mainForm" class="grid grid-cols-1 md:grid-cols-4 lg:grid-cols-7 gap-4 items-end">
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-500 mb-2">日期</label>
                    <input type="date" id="entryDate" class="w-full">
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-500 mb-2">類型</label>
                    <select id="entryType" onchange="toggleFields()" class="w-full font-bold">
                        <option value="買入">買入 (支出)</option>
                        <option value="賣出">賣出 (收入)</option>
                        <option value="費用">費用 (支出)</option>
                        <option value="收入">收入 (股息)</option>
                    </select>
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-500 mb-2">股票代號</label>
                    <input type="text" id="symbol" placeholder="如: AAPL" class="w-full uppercase font-bold">
                </div>
                <div id="tradeFields" class="md:col-span-2 grid grid-cols-2 gap-2">
                    <div>
                        <label class="block text-xs font-bold text-slate-500 mb-2">單價</label>
                        <input type="number" id="price" step="0.001" placeholder="0.00" class="w-full">
                    </div>
                    <div>
                        <label class="block text-xs font-bold text-slate-500 mb-2">數量</label>
                        <input type="number" id="quantity" placeholder="0" class="w-full">
                    </div>
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-500 mb-2">附加金額</label>
                    <input type="number" id="feeOrAmount" step="0.01" value="0" class="w-full font-black text-blue-700 bg-blue-50">
                </div>
                <div class="md:col-span-1">
                    <button type="submit" class="w-full bg-blue-600 hover:bg-blue-700 text-white py-3 rounded-lg font-black transition shadow-md">
                        確認記錄
                    </button>
                </div>
                <div class="lg:col-span-7">
                    <input type="text" id="note" placeholder="在此輸入備註（例如：複利投資、手續費說明...）" class="w-full text-sm italic">
                </div>
            </form>
        </div>

        <div class="flex flex-col md:flex-row justify-between items-center mb-6 gap-4">
            <h2 class="text-xl font-bold text-slate-800 flex items-center gap-2">
                📜 明細清單 <span id="filterLabel" class="hidden bg-blue-100 text-blue-700 px-3 py-1 rounded-full text-xs font-mono font-bold"></span>
            </h2>
            <div class="relative w-full md:w-80">
                <input type="text" id="filterInput" onkeyup="render()" placeholder="🔍 搜尋代號查看統計..." class="w-full !border-2 !border-amber-400 !rounded-2xl shadow-sm font-bold">
            </div>
        </div>

        <div class="card overflow-hidden border border-slate-200">
            <div class="overflow-x-auto">
                <table class="w-full text-sm text-left">
                    <thead class="table-header">
                        <tr>
                            <th class="px-6">日期</th>
                            <th class="px-6">類型</th>
                            <th class="px-6 text-center">代號</th>
                            <th class="px-6 text-right">交易詳情</th>
                            <th class="px-6 text-right">附加金額</th>
                            <th class="px-6 text-right">現金流</th>
                            <th class="px-6 text-center">操作</th>
                        </tr>
                    </thead>
                    <tbody id="recordBody" class="divide-y divide-slate-100">
                        </tbody>
                </table>
            </div>
        </div>
    </div>

    <script>
        const initDate = () => { document.getElementById('entryDate').value = new Date().toISOString().split('T')[0]; };
        initDate();

        let records = JSON.parse(localStorage.getItem('v3_7_portfolio_data')) || [];
        let editingId = null;

        function toggleFields() {
            const type = document.getElementById('entryType').value;
            const isTrade = (type === '買入' || type === '賣出');
            const fields = document.getElementById('tradeFields');
            fields.style.opacity = isTrade ? "1" : "0.3";
            fields.style.pointerEvents = isTrade ? "auto" : "none";
        }

        function render() {
            const filter = document.getElementById('filterInput').value.toUpperCase();
            const tbody = document.getElementById('recordBody');
            const statsGrid = document.getElementById('statsGrid');
            const filterLabel = document.getElementById('filterLabel');
            tbody.innerHTML = '';

            records.sort((a, b) => new Date(b.date) - new Date(a.date));
            const filtered = records.filter(r => r.symbol.includes(filter));
            
            let totalBuyQty = 0, totalBuyCost = 0, totalSellQty = 0, totalCashFlow = 0, totalFees = 0;

            filtered.forEach(r => {
                const subtotal = (r.price || 0) * (r.quantity || 0);
                let rowCash = 0;
                if (r.type === '買入') { 
                    totalBuyQty += r.quantity; 
                    totalBuyCost += (subtotal + Math.abs(r.feeOrAmount)); 
                    rowCash = -subtotal + r.feeOrAmount; 
                } else if (r.type === '賣出') { 
                    totalSellQty += r.quantity; 
                    rowCash = subtotal + r.feeOrAmount; 
                } else { 
                    rowCash = r.feeOrAmount; 
                }
                totalCashFlow += rowCash; 
                totalFees += r.feeOrAmount;

                const isOut = (r.type === '買入' || r.type === '費用');
                tbody.innerHTML += `
                    <tr class="hover:bg-slate-50 transition-colors ${isOut ? 'border-outflow' : 'border-inflow'}">
                        <td class="px-6 py-4 font-mono text-slate-500 font-bold">${r.date}</td>
                        <td class="px-6 py-4 font-black ${isOut ? 'text-red-600' : 'text-green-600'}">${r.type}</td>
                        <td class="px-6 py-4 font-black text-slate-800 text-center">${r.symbol}</td>
                        <td class="px-6 py-4 text-right font-medium text-slate-600">${r.price ? `${r.price.toLocaleString()} × ${r.quantity.toLocaleString()}` : '--'}</td>
                        <td class="px-6 py-4 text-right font-bold ${r.feeOrAmount < 0 ? 'text-red-500' : 'text-green-600'}">${r.feeOrAmount >= 0 ? '+' : ''}${r.feeOrAmount.toLocaleString()}</td>
                        <td class="px-6 py-4 text-right font-black ${rowCash >= 0 ? 'text-green-600' : 'text-red-600'}">${rowCash >= 0 ? '+' : ''}${rowCash.toLocaleString(undefined, {minimumFractionDigits: 2})}</td>
                        <td class="px-6 py-4 text-center">
                            <button onclick="deleteRecord(${r.id})" class="text-slate-300 hover:text-red-500 transition-colors text-lg">✕</button>
                        </td>
                    </tr>
                    ${r.note ? `<tr><td colspan="7" class="px-12 py-2 text-[12px] text-slate-400 italic bg-slate-50/50 border-none font-medium">備註：${r.note}</td></tr>` : ''}
                `;
            });

            const currentQty = totalBuyQty - totalSellQty;
            const avgCost = totalBuyQty > 0 ? (totalBuyCost / totalBuyQty) : 0;

            if (filter) {
                filterLabel.innerText = filter;
                filterLabel.classList.remove('hidden');
                statsGrid.innerHTML = `
                    <div class="card p-6 border-b-4 border-blue-500">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">目前持倉</p>
                        <p class="text-2xl font-black text-slate-800">${currentQty.toLocaleString()}</p>
                    </div>
                    <div class="card p-6 border-b-4 border-amber-500">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">平均成本</p>
                        <p class="text-2xl font-black text-amber-600">$${avgCost.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                    <div class="card p-6 border-b-4 border-slate-300">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">累積息費</p>
                        <p class="text-2xl font-black ${totalFees >= 0 ? 'text-green-600' : 'text-red-500'}">$${totalFees.toLocaleString()}</p>
                    </div>
                    <div class="card p-6 bg-slate-800 text-white shadow-xl">
                        <p class="text-[10px] font-bold text-slate-400 uppercase tracking-widest mb-1">該股票的總淨流</p>
                        <p class="text-2xl font-black ${totalCashFlow >= 0 ? 'text-green-400' : 'text-red-400'}">$${totalCashFlow.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                `;
            } else {
                filterLabel.classList.add('hidden');
                statsGrid.innerHTML = `
                    <div class="card p-6 border-b-4 border-slate-400 md:col-span-2">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">帳戶流水總計</p>
                        <p class="text-2xl font-black text-slate-800">$${totalCashFlow.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                    <div class="card p-6 border-b-4 border-red-500 md:col-span-1">
                        <p class="text-[10px] font-black text-slate-400 uppercase tracking-widest mb-1">總支出費用</p>
                        <p class="text-2xl font-black text-red-600">$${totalFees.toLocaleString()}</p>
                    </div>
                    <div class="card p-6 bg-amber-50 md:col-span-1 border border-amber-200 flex flex-col justify-center">
                        <p class="text-xs font-bold text-amber-700 leading-tight italic">💡 請輸入「代號」查詢持倉與損益</p>
                    </div>
                `;
            }
        }

        document.getElementById('mainForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const symbol = document.getElementById('symbol').value.toUpperCase();
            if(!symbol) { alert('請輸入股票代號'); return; }
            const record = { id: Date.now(), date: document.getElementById('entryDate').value, type: document.getElementById('entryType').value, symbol: symbol, price: parseFloat(document.getElementById('price').value || 0), quantity: parseInt(document.getElementById('quantity').value || 0), feeOrAmount: parseFloat(document.getElementById('feeOrAmount').value || 0), note: document.getElementById('note').value };
            records.push(record);
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            render();
            const lastDate = document.getElementById('entryDate').value;
            document.getElementById('mainForm').reset();
            document.getElementById('entryDate').value = lastDate;
            toggleFields();
        });

        function deleteRecord(id) { if(confirm('確定要刪除這筆紀錄嗎？')) { records = records.filter(r => r.id !== id); localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records)); render(); } }
        function exportCSV() { let csv = "\uFEFF日期,類型,代號,單價,數量,附加金額,備註\n"; records.forEach(r => csv += `${r.date},${r.type},${r.symbol},${r.price},${r.quantity},${r.feeOrAmount},${r.note}\n`); const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' }); const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = `Portfolio_v3.7.6.csv`; link.click(); }

        render();
    </script>
</body>
</html>
