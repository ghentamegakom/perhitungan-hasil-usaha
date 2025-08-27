<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Perhitungan Hasil Usaha</title>
  <style>
    :root{--bg:#f6f8fb;--card:#ffffff;--muted:#6b7280;--accent:#0ea5a4}
    body{font-family:Inter,ui-sans-serif,system-ui,Segoe UI,Roboto,Helvetica,Arial; background:var(--bg); margin:0;padding:20px}
    .container{max-width:1000px;margin:0 auto}
    header{display:flex;justify-content:space-between;align-items:center;margin-bottom:18px}
    h1{margin:0;font-size:20px}
    .card{background:var(--card);padding:16px;border-radius:10px;box-shadow:0 2px 8px rgba(10,10,10,0.04);margin-bottom:12px}
    form{display:grid;grid-template-columns:repeat(4,1fr);gap:10px;align-items:end}
    label{display:block;font-size:12px;color:var(--muted);margin-bottom:6px}
    input,select{width:100%;padding:8px;border:1px solid #e6e9ef;border-radius:6px;font-size:14px}
    button{padding:10px 12px;border-radius:8px;border:0;background:var(--accent);color:white;cursor:pointer}
    table{width:100%;border-collapse:collapse;margin-top:12px}
    th,td{padding:8px;border-bottom:1px solid #eef2f7;text-align:left}
    .right{text-align:right}
    .controls{display:flex;gap:8px;align-items:center}
    .small{font-size:13px}
    .muted{color:var(--muted)}
    .flex{display:flex;gap:8px}
    .tag-income{background:#ecfdf5;color:#065f46;padding:4px 8px;border-radius:999px;font-weight:600}
    .tag-expense{background:#fff7ed;color:#92400e;padding:4px 8px;border-radius:999px;font-weight:600}
    .totals{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin-top:12px}
    .totals .stat{padding:12px;border-radius:8px;background:linear-gradient(180deg,#fff,#fcfcff);box-shadow:0 1px 4px rgba(10,10,10,0.03)}
    .actions{display:flex;gap:6px}
    @media (max-width:800px){form{grid-template-columns:repeat(2,1fr)}.totals{grid-template-columns:repeat(1,1fr)}}
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Perhitungan Hasil Usaha</h1>
      <div class="muted small">Simpan otomatis ke browser • Eksport CSV tersedia</div>
    </header>

    <section class="card">
      <h3>Tambah Transaksi</h3>
      <form id="entryForm">
        <div>
          <label for="date">Tanggal</label>
          <input type="date" id="date" required />
        </div>
        <div>
          <label for="type">Tipe</label>
          <select id="type">
            <option value="income">Pemasukan</option>
            <option value="expense">Pengeluaran</option>
          </select>
        </div>
        <div>
          <label for="account">Rekening / Akun</label>
          <input id="account" placeholder="Contoh: Penjualan, Gaji, Bahan" required />
        </div>
        <div>
          <label for="amount">Nominal (angka)</label>
          <input id="amount" type="number" step="0.01" min="0" required />
        </div>
        <div style="grid-column:1/-1;display:flex;gap:8px;margin-top:6px">
          <button type="submit">Tambah</button>
          <button type="button" id="clearAll" style="background:#ef4444">Hapus Semua</button>
          <div style="flex:1"></div>
          <select id="currency" class="small" title="Ganti simbol mata uang">
            <option value="Rp">Rp</option>
            <option value="$">$</option>
            <option value="€">€</option>
            <option value="IDR">IDR</option>
          </select>
        </div>
      </form>
    </section>

    <section class="card">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div>
          <label for="filterType" class="small muted">Filter periode</label>
          <div class="flex">
            <select id="filterType">
              <option value="all">Semua</option>
              <option value="date">Per Tanggal</option>
              <option value="month">Per Bulan</option>
              <option value="year">Per Tahun</option>
            </select>
            <input type="date" id="filterDate" />
            <select id="filterMonth" style="display:none">
              <!-- will be filled by JS -->
            </select>
            <select id="filterYear" style="display:none"></select>
            <button id="applyFilter" class="small">Terapkan</button>
            <button id="resetFilter" class="small">Reset</button>
          </div>
        </div>
        <div class="actions">
          <button id="exportCsv" class="small">Eksport CSV</button>
          <button id="importSample" class="small">Isi Contoh</button>
        </div>
      </div>

      <div class="totals" id="summary">
        <!-- totals injected by JS -->
      </div>

      <h3 style="margin-top:12px">Daftar Transaksi</h3>
      <div id="tableWrap"></div>
    </section>

    <footer style="margin-top:18px;color:var(--muted);font-size:13px">Built with ❤️ — buka file <strong>index.html</strong> di VS Code lalu jalankan Live Server atau buka di browser.</footer>
  </div>

  <script>
    // Data model: array of {id, date(YYYY-MM-DD), type: 'income'|'expense', account, amount}
    const STORAGE_KEY = 'hasilUsaha.entries.v1';
    let entries = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
    const form = document.getElementById('entryForm');
    const tableWrap = document.getElementById('tableWrap');
    const summary = document.getElementById('summary');
    const currencySel = document.getElementById('currency');

    // helpers
    const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2,8);
    const fmt = (n) => {
      const sym = currencySel.value || 'Rp';
      // format with thousand separators
      const parts = Number(n).toFixed(2).split('.');
      parts[0] = parts[0].replace(/\B(?=(\d{3})+(?!\d))/g, '.');
      return sym + ' ' + parts.join(',');
    }

    function save(){ localStorage.setItem(STORAGE_KEY, JSON.stringify(entries)); }

    // render table
    function renderTable(filtered = entries){
      if(filtered.length === 0){ tableWrap.innerHTML = '<div class="muted small">Belum ada transaksi</div>'; return }
      let html = '<table><thead><tr><th>Tanggal</th><th>Tipe</th><th>Rekening</th><th class="right">Nominal</th><th>Aksi</th></tr></thead><tbody>';
      filtered.slice().sort((a,b)=> b.date.localeCompare(a.date)).forEach(e=>{
        html += `<tr data-id="${e.id}"><td>${e.date}</td><td>${e.type==='income'?'<span class="tag-income">Pemasukan</span>':'<span class="tag-expense">Pengeluaran</span>'}</td><td>${e.account}</td><td class="right">${fmt(e.amount)}</td><td><button data-action="edit">Edit</button> <button data-action="delete">Hapus</button></td></tr>`;
      });
      html += '</tbody></table>';
      tableWrap.innerHTML = html;

      // attach actions
      tableWrap.querySelectorAll('button').forEach(btn=>{
        const tr = btn.closest('tr');
        const id = tr.dataset.id;
        btn.addEventListener('click', ()=>{
          const action = btn.dataset.action;
          if(action==='delete'){
            if(confirm('Hapus transaksi ini?')){ entries = entries.filter(x=>x.id!==id); save(); applyCurrentFilter(); }
          } else if(action==='edit'){
            const e = entries.find(x=>x.id===id);
            if(!e) return;
            document.getElementById('date').value = e.date;
            document.getElementById('type').value = e.type;
            document.getElementById('account').value = e.account;
            document.getElementById('amount').value = e.amount;
            // remove old entry and allow re-save as update
            entries = entries.filter(x=>x.id!==id);
            save(); applyCurrentFilter();
          }
        });
      });
    }

    function computeSummary(filtered = entries){
      const totalIncome = filtered.filter(e=>e.type==='income').reduce((s,e)=>s+Number(e.amount),0);
      const totalExpense = filtered.filter(e=>e.type==='expense').reduce((s,e)=>s+Number(e.amount),0);
      const profit = totalIncome - totalExpense;

      // per account aggregation
      const perAccount = {};
      filtered.forEach(e=>{
        const key = e.account + '|' + e.type;
        perAccount[key] = (perAccount[key] || 0) + Number(e.amount);
      });

      // render
      summary.innerHTML = `
        <div class="stat"><div class="muted">Total Pemasukan</div><div style="font-size:18px;font-weight:700;margin-top:6px">${fmt(totalIncome)}</div></div>
        <div class="stat"><div class="muted">Total Pengeluaran</div><div style="font-size:18px;font-weight:700;margin-top:6px">${fmt(totalExpense)}</div></div>
        <div class="stat"><div class="muted">Laba / Rugi</div><div style="font-size:18px;font-weight:700;margin-top:6px">${fmt(profit)}</div></div>
      `;

      // below table: per-account breakdown
      let breakdown = '<h4 style="margin-top:12px">Rincian per Rekening</h4>';
      breakdown += '<table><thead><tr><th>Rekening</th><th>Tipe</th><th class="right">Jumlah</th></tr></thead><tbody>';
      Object.keys(perAccount).forEach(k=>{
        const [acct, type] = k.split('|');
        breakdown += `<tr><td>${acct}</td><td>${type}</td><td class="right">${fmt(perAccount[k])}</td></tr>`;
      });
      breakdown += '</tbody></table>';

      summary.innerHTML += breakdown;
    }

    // filter utilities
    const filterType = document.getElementById('filterType');
    const filterDate = document.getElementById('filterDate');
    const filterMonth = document.getElementById('filterMonth');
    const filterYear = document.getElementById('filterYear');

    function populateMonthYearControls(){
      // months
      filterMonth.innerHTML = '';
      for(let m=1;m<=12;m++){
        const mm = String(m).padStart(2,'0');
        const opt = document.createElement('option'); opt.value = mm; opt.textContent = mm;
        filterMonth.appendChild(opt);
      }
      // years (based on entries and current year)
      const years = new Set(entries.map(e=>e.date.slice(0,4)));
      years.add(new Date().getFullYear().toString());
      filterYear.innerHTML = '';
      Array.from(years).sort().forEach(y=>{ const o = document.createElement('option'); o.value=y;o.textContent=y; filterYear.appendChild(o); });
    }

    function applyCurrentFilter(){
      const t = filterType.value;
      let filtered = entries.slice();
      if(t==='date' && filterDate.value){ filtered = filtered.filter(e=>e.date===filterDate.value); }
      if(t==='month' && filterYear.value && filterMonth.value){ filtered = filtered.filter(e=>e.date.startsWith(filterYear.value + '-' + filterMonth.value)); }
      if(t==='year' && filterYear.value){ filtered = filtered.filter(e=>e.date.startsWith(filterYear.value)); }
      renderTable(filtered);
      computeSummary(filtered);
    }

    // form submit
    form.addEventListener('submit', e=>{
      e.preventDefault();
      const data = {
        id: uid(),
        date: document.getElementById('date').value,
        type: document.getElementById('type').value,
        account: document.getElementById('account').value.trim(),
        amount: parseFloat(document.getElementById('amount').value || 0)
      };
      if(!data.date || !data.account) return alert('Lengkapi data.');
      entries.push(data);
      save();
      populateMonthYearControls();
      form.reset();
      applyCurrentFilter();
    });

    // filter UI changes
    filterType.addEventListener('change', ()=>{
      filterDate.style.display = 'none'; filterMonth.style.display = 'none'; filterYear.style.display = 'none';
      if(filterType.value === 'date') filterDate.style.display = '';
      if(filterType.value === 'month'){ filterMonth.style.display = ''; filterYear.style.display = ''; }
      if(filterType.value === 'year') filterYear.style.display = '';
    });

    document.getElementById('applyFilter').addEventListener('click', ()=> applyCurrentFilter());
    document.getElementById('resetFilter').addEventListener('click', ()=>{ filterType.value='all'; filterDate.value=''; applyCurrentFilter(); });

    document.getElementById('clearAll').addEventListener('click', ()=>{
      if(confirm('Hapus semua transaksi?')){ entries = []; save(); applyCurrentFilter(); populateMonthYearControls(); }
    });

    currencySel.addEventListener('change', ()=> applyCurrentFilter());

    document.getElementById('exportCsv').addEventListener('click', ()=>{
      if(entries.length===0) return alert('Tidak ada data untuk diekspor');
      const cols = ['id','date','type','account','amount'];
      const csv = [cols.join(',')].concat(entries.map(r=>cols.map(c=>`"${String(r[c]).replace(/"/g,'""')}"`).join(','))).join('\n');
      const blob = new Blob([csv], {type:'text/csv'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a'); a.href = url; a.download = 'hasil_usaha.csv'; document.body.appendChild(a); a.click(); a.remove(); URL.revokeObjectURL(url);
    });

    document.getElementById('importSample').addEventListener('click', ()=>{
      if(!confirm('Isi contoh akan menimpa/menambah data saat ini. Lanjutkan?')) return;
      const sample = [
        {id:uid(),date:'2025-08-01',type:'income',account:'Penjualan',amount:1250000},
        {id:uid(),date:'2025-08-02',type:'expense',account:'Bahan Baku',amount:350000},
        {id:uid(),date:'2025-08-15',type:'expense',account:'Gaji',amount:400000},
        {id:uid(),date:'2025-08-20',type:'income',account:'Jasa',amount:200000}
      ];
      entries = entries.concat(sample);
      save(); populateMonthYearControls(); applyCurrentFilter();
    });

    // initial
    populateMonthYearControls();
    applyCurrentFilter();
  </script>
</body>
</html>
