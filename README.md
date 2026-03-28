<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>資產管理專家 v3.7.2 - 視覺加固版</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* 強制固定背景色，防止瀏覽器自動深色模式干擾 */
        body { background-color: #f8fafc !important; color: #1e293b !important; font-family: 'PingFang TC', 'Microsoft JhengHei', sans-serif; }
        .card { background: white !important; border-radius: 12px; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
        .border-outflow { border-left: 5px solid #ef4444 !important; }
        .border-inflow { border-left: 5px solid #10b981 !important; }
        /* 確保輸入框在任何模式下都有白底黑字 */
        input, select { background-color: white !important; color: #1e293b !important; border: 1px solid #cbd5e1 !important; border-radius: 6px; padding: 6px 10px; }
        .stock-tag { background: #e2e8f0; color: #334155; padding: 2px 12px; border-radius: 99px; font-weight: bold; font-family: monospace; }
    </style>
</head>
<body class="p-4 md:p-8">

    <div class="max-w-7xl mx-auto">
        <header class="mb-8 flex justify-between items-end border-b border-slate-200 pb-4">
            <div>
                <h1 class="text-3xl font-extrabold text-slate-900">📊 投資流水管理 <span class="text-sm font-normal text-slate-400">v3.7.2 Stable</span></h1>
                <p class="text-slate-600 text-sm mt-1">支出：買入/費用(紅) | 收入：賣出/收入(綠)</p>
            </div>
            <button onclick="exportCSV()" class="bg-slate-800 text-white px-4 py-2 rounded-lg hover:bg-black transition text-sm shadow-md font-bold">導出 CSV</button>
        </header>

        <div id="statsGrid" class="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8"></div>

        <div class="card p-6 mb-8 border-t-8 border-slate-900 shadow-lg">
            <form id="mainForm" class="grid grid-cols-1 md:grid-cols-7 gap-4 items-end">
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-600 mb-2">日期</label>
                    <input type="date" id="entryDate" class="w-full outline-none focus:ring-2 focus:ring-blue-400">
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-600 mb-2">類型</label>
                    <select id="entryType" onchange="toggleFields()" class="w-full font-medium outline-none">
                        <option value="買入">買入 (支出)</option>
                        <option value="賣出">賣出 (收入)</option>
                        <option value="費用">費用 (支出)</option>
                        <option value="收入">收入 (股息)</option>
                    </select>
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-600 mb-2">代號</label>
                    <input type="text" id="symbol" placeholder="AAPL" class="w-full uppercase outline-none focus:ring-2 focus:ring-blue-400">
                </div>
                <div id="tradeFields" class="md:col-span-2 grid grid-cols-2 gap-2">
                    <div>
                        <label class="block text-xs font-bold text-slate-600 mb-2">單價</label>
                        <input type="number" id="price" step="0.001" class="w-full outline-none">
                    </div>
                    <div>
                        <label class="block text-xs font-bold text-slate-600 mb-2">數量</label>
                        <input type="number" id="quantity" class="w-full outline-none">
                    </div>
                </div>
                <div class="md:col-span-1">
                    <label class="block text-xs font-bold text-slate-600 mb-2">附加金額 (-/+)</label>
                    <input type="number" id="feeOrAmount" step="0.01" value="0" class="w-full bg-blue-50 !border-blue-200 outline-none font-bold text-blue-800">
                </div>
                <div class="md:col-span-1">
                    <button type="submit" class="w-full bg-blue-600 text-white py-2.5 rounded-md hover:bg-blue-700 font-bold transition shadow-md">記錄</button>
                </div>
                <div class="md:col-span-7">
                    <input type="text" id="note" placeholder="在此輸入備註（例如：定期定額、複利再投）" class="w-full italic text-sm outline-none focus:ring-2 focus:ring-blue-400">
                </div>
            </form>
        </div>

        <div class="flex flex-col md:flex-row justify-between items-center mb-4 px-2 gap-4">
            <h2 class="font-bold text-slate-800 text-lg flex items-center gap-2">
                明細清單 <span id="filterLabel" class="hidden stock-tag"></span>
            </h2>
            <div class="relative w-full md:w-80">
                <input type="text" id="filterInput" onkeyup="render()" placeholder="🔍 輸入代號查詢（如：TSLA）" class="w-full p-2.5 !bg-white border-2 border-slate-200 rounded-xl text-sm focus:ring-2 focus:ring-amber-400 focus:border-amber-400 outline-none shadow-sm">
            </div>
        </div>

        <div class="card overflow-x-auto shadow-md border border-slate-200">
            <table class="min-w-full text-sm">
                <thead>
                    <tr class="bg-black text-white uppercase text-[11px] tracking-[0.1em]">
                        <th class="px-6 py-4 text-left font-bold">日期</th>
                        <th class="px-6 py-4 text-left font-bold">類型</th>
                        <th class="px-6 py-4 text-left font-bold">代號</th>
                        <th class="px-6 py-4 text-right font-bold">詳情</th>
                        <th class="px-6 py-4 text-right font-bold">附加金額</th>
                        <th class="px-6 py-4 text-right font-bold">現金流</th>
                        <th class="px-6 py-4 text-center font-bold">操作</th>
                    </tr>
                </thead>
                <tbody id="recordBody" class="divide-y divide-slate-200 bg-white"></tbody>
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
            document.getElementById('tradeFields').style.pointerEvents = isTrade ? "auto" : "none";
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
                const subtotal = r.price * r.quantity;
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

                if (editingId === r.id) {
                    tbody.innerHTML += renderEditRow(r);
                } else {
                    tbody.innerHTML += renderNormalRow(r, rowCash);
                }
            });

            const currentQty = totalBuyQty - totalSellQty;
            const avgCost = totalBuyQty > 0 ? (totalBuyCost / totalBuyQty) : 0;

            if (filter) {
                filterLabel.innerText = filter;
                filterLabel.classList.remove('hidden');
                statsGrid.innerHTML = `
                    <div class="card p-5 border-b-4 border-blue-500">
                        <p class="text-[10px] font-extrabold text-slate-400 uppercase">目前持倉股數</p>
                        <p class="text-2xl font-bold ${currentQty > 0 ? 'text-blue-600' : 'text-slate-400'}">${currentQty.toLocaleString()}</p>
                    </div>
                    <div class="card p-5 border-b-4 border-amber-500">
                        <p class="text-[10px] font-extrabold text-slate-400 uppercase">平均買入成本</p>
                        <p class="text-2xl font-bold text-amber-600">$${avgCost.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                    <div class="card p-5 border-b-4 border-slate-300">
                        <p class="text-[10px] font-extrabold text-slate-400 uppercase">累計附加 (息/費)</p>
                        <p class="text-2xl font-bold ${totalFees >= 0 ? 'text-green-600' : 'text-red-500'}">$${totalFees.toLocaleString()}</p>
                    </div>
                    <div class="card p-5 bg-slate-900 text-white shadow-xl">
                        <p class="text-[10px] font-bold text-slate-400 uppercase">該股票的總損益 (Cash)</p>
                        <p class="text-2xl font-bold ${totalCashFlow >= 0 ? 'text-green-400' : 'text-red-400'}">$${totalCashFlow.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                `;
            } else {
                filterLabel.classList.add('hidden');
                statsGrid.innerHTML = `
                    <div class="card p-5 border-b-4 border-slate-400">
                        <p class="text-[10px] font-extrabold text-slate-400 uppercase">全帳戶流水淨值</p>
                        <p class="text-2xl font-bold text-slate-800">$${totalCashFlow.toLocaleString(undefined, {minimumFractionDigits: 2})}</p>
                    </div>
                    <div class="card p-5 border-b-4 border-red-500">
                        <p class="text-[10px] font-extrabold text-slate-400 uppercase">總計支出費用</p>
                        <p class="text-2xl font-bold text-red-600">$${totalFees.toLocaleString()}</p>
                    </div>
                    <div class="md:col-span-2 card p-5 bg-slate-900 text-white text-center flex flex-col justify-center border-b-4 border-amber-500">
                        <p class="text-sm font-bold text-amber-400 mb-1">💡 查詢小秘訣</p>
                        <p class="text-xs font-medium text-slate-300">輸入股票代號（如 AAPL）即可查看平均成本與持倉數量</p>
                    </div>
                `;
            }
        }

        function renderNormalRow(r, cash) {
            const isOutflow = (r.type === '買入' || r.type === '費用');
            const colorClass = isOutflow ? 'text-red-600' : 'text-green-600';
            const borderClass = isOutflow ? 'border-outflow' : 'border-inflow';

            return `
                <tr class="hover:bg-slate-50 transition-colors ${borderClass}">
                    <td class="px-6 py-4 font-mono text-xs text-slate-500 font-bold">${r.date}</td>
                    <td class="px-6 py-4 font-extrabold ${colorClass}">${r.type}</td>
                    <td class="px-6 py-4 font-mono font-black text-slate-800">${r.symbol}</td>
                    <td class="px-6 py-4 text-right text-slate-600 font-medium">${r.price ? `${r.price.toLocaleString()} × ${r.quantity}` : '--'}</td>
                    <td class="px-6 py-4 text-right font-bold ${r.feeOrAmount < 0 ? 'text-red-500' : 'text-green-600'}">
                        ${r.feeOrAmount > 0 ? '+' : ''}${r.feeOrAmount.toLocaleString()}
                    </td>
                    <td class="px-6 py-4 text-right font-black ${cash >= 0 ? 'text-green-600' : 'text-red-600'}">
                        ${cash >= 0 ? '+' : ''}${cash.toLocaleString(undefined, {minimumFractionDigits: 2})}
                    </td>
                    <td class="px-6 py-4 text-center">
                        <button onclick="startEdit(${r.id})" class="text-slate-400 hover:text-blue-600 mr-4 transition text-lg">✎</button>
                        <button onclick="deleteRecord(${r.id})" class="text-slate-300 hover:text-red-500 transition text-lg">✕</button>
                    </td>
                </tr>
                ${r.note ? `<tr><td colspan="7" class="px-12 py-2 text-[12px] text-slate-500 italic bg-slate-50/50 border-none font-medium text-blue-900/60">└ 備註：${r.note}</td></tr>` : ''}
            `;
        }

        function renderEditRow(r) {
            return `<tr class="bg-amber-50">
                <td class="px-4 py-2"><input type="date" id="editDate" value="${r.date}" class="w-full text-xs"></td>
                <td class="px-4 py-2"><select id="editType" class="w-full text-xs font-bold"><option value="買入" ${r.type==='買入'?'selected':''}>買入</option><option value="賣出" ${r.type==='賣出'?'selected':''}>賣出</option><option value="費用" ${r.type==='費用'?'selected':''}>費用</option><option value="收入" ${r.type==='收入'?'selected':''}>收入</option></select></td>
                <td class="px-4 py-2"><input type="text" id="editSymbol" value="${r.symbol}" class="w-full uppercase text-xs font-bold"></td>
                <td class="px-4 py-2 text-right"><input type="number" id="editPrice" value="${r.price}" class="w-16 text-xs"> x <input type="number" id="editQty" value="${r.quantity}" class="w-16 text-xs"></td>
                <td class="px-4 py-2 text-right"><input type="number" id="editFee" value="${r.feeOrAmount}" class="w-20 text-xs font-bold"></td>
                <td colspan="2" class="px-4 py-2 text-center"><button onclick="saveEdit(${r.id})" class="text-blue-600 font-bold mr-3 text-sm">儲存</button><button onclick="cancelEdit()" class="text-slate-400 text-sm">取消</button></td>
            </tr>`;
        }

        function startEdit(id) { editingId = id; render(); }
        function cancelEdit() { editingId = null; render(); }
        function saveEdit(id) {
            const idx = records.findIndex(r => r.id === id);
            records[idx] = { ...records[idx], date: document.getElementById('editDate').value, type: document.getElementById('editType').value, symbol: document.getElementById('editSymbol').value.toUpperCase(), price: parseFloat(document.getElementById('editPrice').value || 0), quantity: parseInt(document.getElementById('editQty').value || 0), feeOrAmount: parseFloat(document.getElementById('editFee').value || 0) };
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            editingId = null;
            render();
        }

        document.getElementById('mainForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const record = { id: Date.now(), date: document.getElementById('entryDate').value, type: document.getElementById('entryType').value, symbol: document.getElementById('symbol').value.toUpperCase(), price: parseFloat(document.getElementById('price').value || 0), quantity: parseInt(document.getElementById('quantity').value || 0), feeOrAmount: parseFloat(document.getElementById('feeOrAmount').value || 0), note: document.getElementById('note').value };
            records.push(record);
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            render();
            const lastDate = document.getElementById('entryDate').value;
            document.getElementById('mainForm').reset();
            document.getElementById('entryDate').value = lastDate;
            toggleFields();
        });

        function deleteRecord(id) { if(confirm('確定刪除這筆紀錄？')) { records = records.filter(r => r.id !== id); localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records)); render(); } }
        function exportCSV() { let csv = "\uFEFF日期,類型,代號,單價,數量,附加金額,備註\n"; records.forEach(r => csv += `${r.date},${r.type},${r.symbol},${r.price},${r.quantity},${r.feeOrAmount},${r.note}\n`); const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' }); const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = `Portfolio_Final.csv`; link.click(); }

        render();
    </script>
</body>
</html>
