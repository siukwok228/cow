<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>資產管理專家 v3.7.5</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body style="background-color: #f8fafc !important; color: #1e293b !important; padding: 20px;">

    <div style="max-width: 1200px; margin: 0 auto;">
        <header style="border-bottom: 2px solid #cbd5e1; margin-bottom: 30px; padding-bottom: 10px;">
            <h1 style="font-size: 28px; font-weight: 900; color: #0f172a;">📊 投資流水管理 v3.7.5</h1>
            <p style="color: #64748b; font-weight: bold;">支出(紅) / 收入(綠)</p>
        </header>

        <div id="statsGrid" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; margin-bottom: 30px;"></div>

        <div style="background: white; padding: 20px; border-radius: 12px; border-top: 8px solid #0f172a; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.1); margin-bottom: 30px;">
            <form id="mainForm" style="display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 10px;">
                <input type="date" id="entryDate" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px;">
                <select id="entryType" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px; font-weight: bold;">
                    <option value="買入">買入 (支出)</option>
                    <option value="賣出">賣出 (收入)</option>
                    <option value="費用">費用 (支出)</option>
                    <option value="收入">收入 (股息)</option>
                </select>
                <input type="text" id="symbol" placeholder="代號" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px; text-transform: uppercase;">
                <input type="number" id="price" step="0.001" placeholder="單價" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px;">
                <input type="number" id="quantity" placeholder="數量" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px;">
                <input type="number" id="feeOrAmount" step="0.01" value="0" style="border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px; background: #eff6ff; font-weight: bold; color: #1d4ed8;">
                <button type="submit" style="background: #2563eb; color: white; border: none; padding: 10px; border-radius: 6px; font-weight: 900; cursor: pointer;">記錄</button>
                <input type="text" id="note" placeholder="備註..." style="grid-column: 1 / -1; border: 2px solid #cbd5e1; padding: 8px; border-radius: 6px;">
            </form>
        </div>

        <input type="text" id="filterInput" onkeyup="render()" placeholder="🔍 輸入代號搜尋..." 
               style="width: 100%; max-width: 400px; padding: 12px; border: 3px solid #fbbf24; border-radius: 10px; margin-bottom: 20px; background: white; color: black; font-weight: bold;">

        <div style="background: white; border-radius: 12px; overflow-x: auto; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1);">
            <table style="width: 100%; border-collapse: collapse; min-width: 700px;">
                <thead>
                    <tr style="background-color: #1e293b !important;">
                        <th style="padding: 15px; color: white !important; text-align: left; font-weight: 900;">日期</th>
                        <th style="padding: 15px; color: white !important; text-align: left; font-weight: 900;">類型</th>
                        <th style="padding: 15px; color: white !important; text-align: left; font-weight: 900;">代號</th>
                        <th style="padding: 15px; color: white !important; text-align: right; font-weight: 900;">詳情</th>
                        <th style="padding: 15px; color: white !important; text-align: right; font-weight: 900;">附加金額</th>
                        <th style="padding: 15px; color: white !important; text-align: right; font-weight: 900;">現金流</th>
                        <th style="padding: 15px; color: white !important; text-align: center; font-weight: 900;">操作</th>
                    </tr>
                </thead>
                <tbody id="recordBody"></tbody>
            </table>
        </div>
    </div>

    <script>
        const initDate = () => { document.getElementById('entryDate').value = new Date().toISOString().split('T')[0]; };
        initDate();
        let records = JSON.parse(localStorage.getItem('v3_7_portfolio_data')) || [];
        let editingId = null;

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
                tbody.innerHTML += `
                    <tr style="border-bottom: 1px solid #e2e8f0; border-left: 6px solid ${isOut ? '#ef4444' : '#10b981'};">
                        <td style="padding: 15px; font-family: monospace;">${r.date}</td>
                        <td style="padding: 15px; color: ${isOut ? '#ef4444' : '#10b981'}; font-weight: 900;">${r.type}</td>
                        <td style="padding: 15px; font-weight: 900;">${r.symbol}</td>
                        <td style="padding: 15px; text-align: right;">${r.price ? `${r.price}×${r.quantity}` : '--'}</td>
                        <td style="padding: 15px; text-align: right; color: ${r.feeOrAmount < 0 ? '#ef4444' : '#10b981'};">${r.feeOrAmount}</td>
                        <td style="padding: 15px; text-align: right; font-weight: 900; color: ${rowCash >= 0 ? '#10b981' : '#ef4444'};">${rowCash.toFixed(2)}</td>
                        <td style="padding: 15px; text-align: center;">
                            <button onclick="deleteRecord(${r.id})" style="color: #cbd5e1; cursor: pointer; border: none; background: none; font-size: 18px;">✕</button>
                        </td>
                    </tr>
                `;
            });

            const currentQty = totalBuyQty - totalSellQty;
            const avgCost = totalBuyQty > 0 ? (totalBuyCost / totalBuyQty) : 0;

            if (filter) {
                statsGrid.innerHTML = `
                    <div style="background: white; padding: 15px; border-radius: 8px; border-bottom: 4px solid #3b82f6;"><p style="font-size: 10px; color: #94a3b8; font-weight: 900;">持倉股數</p><p style="font-size: 20px; font-weight: 900;">${currentQty}</p></div>
                    <div style="background: white; padding: 15px; border-radius: 8px; border-bottom: 4px solid #f59e0b;"><p style="font-size: 10px; color: #94a3b8; font-weight: 900;">平均成本</p><p style="font-size: 20px; font-weight: 900;">$${avgCost.toFixed(2)}</p></div>
                    <div style="background: white; padding: 15px; border-radius: 8px; border-bottom: 4px solid #94a3b8;"><p style="font-size: 10px; color: #94a3b8; font-weight: 900;">累計費用</p><p style="font-size: 20px; font-weight: 900;">$${totalFees}</p></div>
                    <div style="background: #1e293b; color: white; padding: 15px; border-radius: 8px;"><p style="font-size: 10px; color: #94a3b8; font-weight: 900;">該股票的損益</p><p style="font-size: 20px; font-weight: 900; color: ${totalCashFlow >= 0 ? '#4ade80' : '#f87171'};">$${totalCashFlow.toFixed(2)}</p></div>
                `;
            } else {
                statsGrid.innerHTML = `<div style="background: white; padding: 15px; border-radius: 8px; border-bottom: 4px solid #94a3b8; grid-column: 1 / -1; text-align: center; font-weight: 900;">全帳戶流水淨值: $${totalCashFlow.toFixed(2)}</div>`;
            }
        }

        document.getElementById('mainForm').addEventListener('submit', (e) => {
            e.preventDefault();
            const record = { id: Date.now(), date: document.getElementById('entryDate').value, type: document.getElementById('entryType').value, symbol: document.getElementById('symbol').value.toUpperCase(), price: parseFloat(document.getElementById('price').value || 0), quantity: parseInt(document.getElementById('quantity').value || 0), feeOrAmount: parseFloat(document.getElementById('feeOrAmount').value || 0), note: document.getElementById('note').value };
            records.push(record);
            localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records));
            render();
            document.getElementById('mainForm').reset();
            initDate();
        });

        function deleteRecord(id) { if(confirm('確定刪除？')) { records = records.filter(r => r.id !== id); localStorage.setItem('v3_7_portfolio_data', JSON.stringify(records)); render(); } }
        function exportCSV() { let csv = "\uFEFF日期,類型,代號,單價,數量,附加金額,備註\n"; records.forEach(r => csv += `${r.date},${r.type},${r.symbol},${r.price},${r.quantity},${r.feeOrAmount},${r.note}\n`); const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' }); const link = document.createElement("a"); link.href = URL.createObjectURL(blob); link.download = `Portfolio.csv`; link.click(); }

        render();
    </script>
</body>
</html>
