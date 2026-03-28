<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>資產管理專家 v3.7.4 - 最終強制定製版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* 最高等級強制背景與文字顏色 */
        html, body { background-color: #f8fafc !important; color: #1e293b !important; }
        .card { background-color: #ffffff !important; color: #1e293b !important; border: 1px solid #e2e8f0; }
        th { color: #ffffff !important; background-color: #1e293b !important; font-weight: 900 !important; }
        td { color: #334155 !important; font-weight: 600 !important; }
        input, select { background-color: #ffffff !important; color: #000000 !important; border: 2px solid #cbd5e1 !important; }
    </style>
</head>
<body class="p-4 md:p-8">

    <div class="max-w-7xl mx-auto">
        <header class="mb-8 border-b-2 border-slate-300 pb-4">
            <h1 class="text-3xl font-black text-slate-900">📊 投資流水管理 v3.7.4</h1>
            <p class="text-slate-600 font-bold">支出(買入/費用)：紅 | 收入(賣出/股息)：綠</p>
        </header>

        <div id="statsGrid" class="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8"></div>

        <div class="card p-6 mb-8 border-t-8 border-slate-900 shadow-xl">
            <form id="mainForm" class="grid grid-cols-1 md:grid-cols-7 gap-4 items-end">
                <div class="md:col-span-1">
                    <label class="block text-xs font-black mb-2">日期</label>
                    <input type="date" id="entryDate" class="w-full">
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-black mb-2">類型</label>
                    <select id="entryType" onchange="toggleFields()" class="w-full font-bold">
                        <option value="買入">買入 (支出)</option>
                        <option value="賣出">賣出 (收入)</option>
                        <option value="費用">費用 (支出)</option>
                        <option value="收入">收入 (股息)</option>
                    </select>
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-black mb-2">代號</label>
                    <input type="text" id="symbol" placeholder="AAPL" class="w-full uppercase font-bold">
                </div>
                <div id="tradeFields" class="md:col-span-2 grid grid-cols-2 gap-2">
                    <input type="number" id="price" step="0.001" placeholder="單價" class="w-full">
                    <input type="number" id="quantity" placeholder="數量" class="w-full">
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-black mb-2">附加金額</label>
                    <input type="number" id="feeOrAmount" step="0.01" value="0" class="w-full font-black text-blue-800 bg-blue-50">
                </div>
                <div class="md:col-span-1">
                    <button type="submit" class="w-full bg-blue-600 text-white py-2 rounded font-black">記錄</button>
                </div>
                <div class="md:col-span-7">
                    <input type="text" id="note" placeholder="備註..." class="w-full text-sm">
                </div>
            </form>
        </div>

        <div class="mb-4">
            <input type="text" id="filterInput" onkeyup="render()" placeholder="🔍 輸入代號查看統計..." 
                   style="border: 3px solid #f59e0b !important; background: white !important; color: black !important;"
                   class="w-full md:w-96 p-3 rounded-xl font-bold">
        </div>

        <div class="card overflow-x-auto">
            <table class="min-w-full">
                <thead style="background-color: #1e293b !important; color: #ffffff !important;">
                    <tr>
                        <th style="padding: 15px; text-align: left; color: white !important;">日期</th>
                        <th style="padding: 15px; text-align: left; color: white !important;">類型</th>
                        <th style="padding: 15px; text-align: left; color: white !important;">代號</th>
                        <th style="padding: 15px; text-align: right; color: white !important;">詳情</th>
                        <th style="padding: 15px; text-align: right; color: white !important;">附加金額</th>
                        <th style="padding: 15px; text-align: right; color: white !important;">現金流</th>
                        <th style="padding: 15px; text-align: center; color: white !important;">操作</th>
                    </tr>
                </thead>
                <tbody id="recordBody" class="divide-y divide-slate-200"></tbody>
            </table>
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
            document.getElementById('tradeFields').style.opacity = isTrade ? "1" : "0.3";
        }

        function render() {
            const filter = document.getElementById('filterInput').value.toUpperCase();
            const tbody = document.getElementById('recordBody');
            const statsGrid = document.getElementById('statsGrid');
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
                const rowHtml = `
                    <tr style="border-left: 6px solid ${isOut ? '#ef4444' : '#10b981'};">
                        <td class="px-6 py-4">${r.date}</td>
                        <td class="px-6 py-4" style="color: ${isOut ? '#dc2626' : '#059669'}; font-weight: 900;">${r.type}</td>
                        <td class="px-6 py-4 font-bold text-slate-900">${r.symbol}</td>
                        <td class="px-6 py-4 text-right">${r.price ? `${r.price}×${r.quantity}` : '--'}</td>
                        <td class="px-6 py-4 text-right" style="color: ${r.feeOrAmount < 0 ? '#dc2626' : '#059669'};">${r.feeOrAmount}</td>
                        <td class="px-6 py-4 text-right" style="color: ${rowCash >= 0 ? '#059669' : '#dc2626'}; font-weight: 900;">${rowCash.toFixed(2)}</td>
                        <td class="px-6 py-4 text-center">
                            <button onclick="startEdit(${r.id})" style="color: #64748b; margin-right: 10px;">✎</button>
                            <button onclick="deleteRecord(${r.id})" style="color: #cbd5e1;">✕</button>
                        </td>
                    </tr>
                `;
                tbody.innerHTML += rowHtml;
            });

            const currentQty = totalBuyQty - totalSellQty;
            const avgCost = totalBuyQty > 0 ? (totalBuyCost / totalBuyQty) : 0;

            if (filter) {
                statsGrid.innerHTML = `
                    <div class="card p-4 border-b-4 border-blue-500"><p class="text-xs font-black text-slate-400">持倉</p><p class="text-xl font-black">${currentQty}</p></div>
                    <div class="card p-4 border-b-4 border-amber-500"><p class="text-xs font-black text-slate-400">成本</p><p class="text-xl font-black">$${avgCost.toFixed(2)}</p></div>
                    <div class="card p-4 border-b-4 border-slate-300"><p class="text-xs font-black text-slate-400">息/費</p><p class="text-xl font-black">$${totalFees}</p></div>
                    <div class="card p-4 bg-slate-900 text-white"><p class="text-xs font-bold text-slate-400">該股票的淨流</p><p class="text-xl font-black">$${totalCashFlow.toFixed(2)}</p></div>
                `;
            } else {
                statsGrid.innerHTML = `<div class="card p-4 border-b-4 border-slate-400 md:col-span-4 font-bold text-center text-slate-600">總帳戶流水淨值: $${totalCashFlow.toFixed(2)}</div>`;
            }
        }

        function startEdit(id) { editingId = id; render(); }
        function deleteRecord(id) { if(confirm('確定刪除？')) { records = records.filter(r => r.id !== id); localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records)); render(); } }
        function exportCSV() { let csv = "\uFEFF日期,類型,代號,單價,數量,附加金額,備註\n"; records.forEach(r => csv += `${r.date},${r.type},${r.symbol},${r.price},${r.quantity},${r.feeOrAmount},${r.note}\n`); const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' }); const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = `Portfolio.csv`; link.click(); }

        document.getElementById('mainForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const record = { id: Date.now(), date: document.getElementById('entryDate').value, type: document.getElementById('entryType').value, symbol: document.getElementById('symbol').value.toUpperCase(), price: parseFloat(document.getElementById('price').value || 0), quantity: parseInt(document.getElementById('quantity').value || 0), feeOrAmount: parseFloat(document.getElementById('feeOrAmount').value || 0), note: document.getElementById('note').value };
            records.push(record);
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            render();
            document.getElementById('mainForm').reset();
            initDate();
        });
        render();
    </script>
</body>
</html>
