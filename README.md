<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<title>BackRock Loan Calculators</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<style>
  body{font-family:Arial, sans-serif;background:#f4f7fb;margin:0;padding:20px;display:flex;justify-content:center;}
  .container{
    width:1000px;
    background:#fff;
    padding:22px;
    border-radius:12px;
    box-shadow:0 8px 20px rgba(0,0,0,.12);
    position: relative; /* Ensure the container is a positioning context for the pseudo-element */
  }
  .container::before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-image: BR Logo.jpg; /* Replace with your logo URL or path */
    background-size: contain;
    background-position: center;
    background-repeat: no-repeat;
    opacity: 0.3; /* Adjust for desired transparency (0.0 fully transparent, 1.0 fully opaque) */
    z-index: -1; /* Place behind content */
  }
  h1{color:#003366;margin-top:0}
  .btn{background:#007bff;color:#fff;border:none;padding:8px 12px;border-radius:6px;cursor:pointer}
  .btn:hover{filter:brightness(.95)}
  .calculator{border:1px solid #e0e6ef;border-radius:10px;padding:14px;margin-bottom:18px;background:#fafbff}
  .calc-head{display:flex;justify-content:space-between;align-items:center}
  .inputs{margin-top:12px}
  label{display:block;margin:8px 0 4px;font-size:13px}
  input[type="text"], select {width:100%;padding:8px;border-radius:6px;border:1px solid #cdd6e6;box-sizing:border-box;font-size:14px}
  .row{display:grid;grid-template-columns:1fr 1fr;gap:10px}
  .results{margin-top:12px;padding:12px;border-radius:8px;background:#eef4fb;border:1px solid #d7e6fb}
  .results strong{color:#003366}
  .remark{font-size:12px;color:#666;margin-top:6px}
  .summary{margin-top:18px;padding:12px;border-radius:8px;background:#f6fbff;border:1px solid #dbeafc}
  .summary ul{padding-left:16px;margin:8px 0}
  .summary li{margin-bottom:10px;padding-bottom:8px;border-bottom:1px dashed #dbeafc}
  .small-btn{padding:6px 8px;font-size:13px;border-radius:6px;cursor:pointer}
  .legal-list{font-size:13px;margin-top:8px;padding-left:14px}
  .muted{color:#666;font-size:12px}
  .floating-btn{position:fixed;bottom:20px;right:20px;background:#007bff;color:#fff;border:none;width:60px;height:60px;border-radius:50%;font-size:24px;display:flex;align-items:center;justify-content:center;box-shadow:0 4px 10px rgba(0,0,0,0.2);cursor:pointer;z-index:1000;}
  .floating-btn:hover{filter:brightness(.95)}
  .summary a{text-decoration:none;color:#003366;}
  .summary a:hover{text-decoration:underline;}
  @media(max-width:880px){.row{grid-template-columns:1fr}}
</style>
</head>
<body>
  <div class="container">
    <h1>BackRock Loan Calculators</h1>

    <div id="calculators"></div>

    <div class="summary" id="summaryBox">
      <h3>Summary</h3>
      <div id="summaryContent">No calculators included yet.</div>
      <div id="summaryTotals" style="display:none;margin-top:10px;padding:10px;background:#eef4fb;border-radius:6px">
        <div><strong>Total Cash on Hand:</strong> RM <span id="sumCash">0.00</span></div>
        <div><strong>Total Monthly Payments:</strong> RM <span id="sumMonthly">0.00</span></div>
        <div><strong>Total Service Fees:</strong> RM <span id="sumService">0.00</span></div>
      </div>
      <div style="margin-top:10px">
        <button class="btn" style="background:#6f42c1" onclick="downloadPDF()">Download Summary as PDF</button>
      </div>
    </div>
  </div>

  <button class="floating-btn" onclick="addCalculator()">➕</button>

<script>
/* =========================
   Config & helpers
   ========================= */
let calcCount = 0;
let includedCalcs = {}; // which calculators included in summary
let modes = {}; // 'loan' or 'monthly'

// Mortality table (approx) - used for takaful calc
const mortalityRates = {
  20:0.0005,25:0.0006,30:0.0007,35:0.0008,40:0.0010,45:0.0012,
  50:0.0015,55:0.0020,60:0.0025,65:0.0030,70:0.0040
};

// number formatting helpers
function fmt(n){
  if(n === null || n === undefined || isNaN(Number(n))) return "0.00";
  return Number(n).toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2});
}
function parseNumStr(s){
  if(!s && s !== 0) return 0;
  return Number(String(s).replace(/,/g,'') || 0) || 0;
}

/* Attach focus/blur behavior: remove commas on focus, format on blur.
   Also support Enter key to trigger calculate for that calculator. */
function attachSmartFormatting(el){
  if(!el) return;
  el.addEventListener('focus', (e)=>{
    const v = e.target.value ? String(e.target.value).replace(/,/g,'') : '';
    e.target.value = (v === "0.00" ? "" : v);
  });
  el.addEventListener('blur', (e)=>{
    const raw = e.target.value ? e.target.value.toString().replace(/,/g,'') : '';
    const num = parseFloat(raw) || 0;
    e.target.value = fmt(num);
  });
  el.addEventListener('keydown', (e)=>{
    if(e.key === 'Enter'){
      e.preventDefault();
      const calc = el.closest('.calculator');
      const id = calc && calc.dataset && calc.dataset.id ? Number(calc.dataset.id) : null;
      if(id) calculate(id);
    }
  });
}

/* -------------------------
   Stamp duty / legal fee helpers (approx)
   ------------------------- */
function calcStampDutyMOT(price){
  let duty = 0;
  let p = price;
  if(p > 1000000){ duty += (p - 1000000) * 0.04; p = 1000000; }
  if(p > 500000){ duty += (p - 500000) * 0.03; p = 500000; }
  if(p > 100000){ duty += (p - 100000) * 0.02; p = 100000; }
  if(p > 0) duty += p * 0.01;
  return duty;
}
function calcStampDutyLoan(loan){ return loan * 0.005; }
function calcLegalFees(amount){
  let fee = 0;
  let a = amount;
  if(a > 5000000){ fee += (a-5000000)*0.005; a = 5000000; }
  if(a > 3000000){ fee += (a-3000000)*0.006; a = 3000000; }
  if(a > 1000000){ fee += (a-1000000)*0.007; a = 1000000; }
  if(a > 500000){ fee += (a-500000)*0.008; a = 500000; }
  if(a > 0) fee += a * 0.01;
  return fee;
}

/* =========================
   Render + UI
   ========================= */

function addCalculator(){
  calcCount++;
  modes[calcCount] = 'loan';
  const container = document.getElementById('calculators');

  const wrapper = document.createElement('div');
  wrapper.className = 'calculator';
  wrapper.id = 'calc' + calcCount;
  wrapper.dataset.id = String(calcCount);

  wrapper.innerHTML = `
    <div class="calc-head">
      <div style="display:flex;gap:12px;align-items:center">
        <h3 style="margin:0">Calculator ${calcCount}</h3>
        <input id="title${calcCount}" type="text" placeholder="Type a title (e.g. P2P Loan)" style="padding:6px;border-radius:6px;border:1px solid #cdd6e6">
      </div>
      <div style="display:flex;gap:8px;align-items:center">
        <select id="type${calcCount}" onchange="renderInputs(${calcCount})">
          <option value="Mortgage">Mortgage</option>
          <option value="P2P">P2P</option>
          <option value="Bank SME Loan">Bank SME Loan</option>
          <option value="SME IMS">SME Bank Loan (IMS)</option>
          <option value="Personal Loan">Personal Loan</option>
          <option value="Hire Purchase">Hire Purchase</option>
        </select>
        <button class="small-btn" onclick="toggleMode(${calcCount})">Switch to Monthly Mode</button>
        <button class="small-btn" style="background:#dc3545;color:#fff" onclick="removeCalculator(${calcCount})">Remove</button>
      </div>
    </div>

    <div class="inputs" id="inputs${calcCount}"></div>

    <div style="margin-top:10px; display:flex; gap:8px;">
      <button class="btn" onclick="calculate(${calcCount})">Calculate</button>
      <button id="summaryBtn${calcCount}" class="small-btn" onclick="toggleSummary(${calcCount})">Add to Summary</button>
    </div>

    <div class="results" id="results${calcCount}"></div>
  `;

  container.appendChild(wrapper);
  renderInputs(calcCount);

  const titleEl = document.getElementById('title'+calcCount);
  titleEl.addEventListener('keydown', (e)=>{
    if(e.key === 'Enter'){ e.preventDefault(); calculate(calcCount); }
  });
}

function removeCalculator(id){
  const el = document.getElementById('calc'+id);
  if(el) el.remove();
  delete includedCalcs[id];
  delete modes[id];
  updateSummaryUI();
}

function toggleMode(id){
  modes[id] = modes[id] === 'loan' ? 'monthly' : 'loan';
  renderInputs(id);
}

/* Render inputs based on calculator type & mode */
function renderInputs(id){
  const type = document.getElementById('type'+id).value;
  const mode = modes[id] || 'loan';
  const container = document.getElementById('inputs'+id);
  let html = '';

  if(type === 'Mortgage'){
    if(mode === 'loan'){
      html += `<label>SNP Price (follow the purchase price, included cash back)</label><input id="snp${id}" type="text" placeholder="1,000,000.00">`;
      html += `<label>Loan Size (RM): (base on the loan amount with bank)</label><input id="loan${id}" type="text" placeholder="1,000,000.00">`;
    } else {
      html += `<label>Monthly Payment (RM)</label><input id="monthly${id}" type="text" placeholder="5,000.00">`;
    }
    html += `<div class="row">`;
    html += `<div><label>Interest Rate (%): effective rate 3.5-4.5%pa</label><input id="rate${id}" type="text" placeholder="4.00"></div>`;
    html += `<div><label>Tenure (Years): max 35 years or age 70</label><input id="tenure${id}" type="text" placeholder="35"></div>`;
    html += `</div>`;
    html += `<label>Service Fees (%): If take BackRock stock then 0% on service fees. If not Base 5%, Max 7%</label><input id="fees${id}" type="text" placeholder="5">`;
    html += `<label>Cash Back Mode</label>
             <select id="cashbackMode${id}" onchange="onCashbackModeChange(${id})">
               <option value="none">None</option>
               <option value="percent">By % of SNP</option>
               <option value="manual">Manual RM</option>
             </select>
             <input id="cashback${id}" type="text" placeholder="Enter % or RM" style="margin-top:6px">`;
    html += `<label>Legal Fees Discount (%)</label><input id="legalDiscount${id}" type="text" placeholder="0">`;
    html += `<label><input id="includeLegal${id}" type="checkbox" checked> Include legal fees in bank deduction</label>`;
    html += `<div class="remark">Remark: Legal fees list below is estimated. Final price based on law firm quotation.</div>`;
  }

  else if(type === 'Personal Loan'){
    if(mode === 'loan'){
      html += `<label>Loan Size (RM): Max 200k, if more than 200k please refer to Keyman.</label><input id="loan${id}" type="text" placeholder="200,000.00">`;
    } else {
      html += `<label>Monthly Payment (RM)</label><input id="monthly${id}" type="text" placeholder="2,500.00">`;
    }
    html += `<div class="row">`;
    html += `<div><label>Interest Rate (%): 2.89%-18%pa</label><input id="rate${id}" type="text" placeholder="10.00"></div>`;
    html += `<div><label>Tenure (Years): 5-10</label><input id="tenure${id}" type="text" placeholder="5"></div>`;
    html += `</div>`;
    html += `<label>Service Fees (%): Base 14%, Max 20%</label><input id="fees${id}" type="text" placeholder="14">`;
    html += `<div class="row">`;
    html += `<div><label>Age for Takaful</label><input id="age${id}" type="text" placeholder="35"></div>`;
    html += `<div><label>Include Takaful?</label><select id="takaful${id}"><option value="no">No</option><option value="yes">Yes</option></select></div>`;
    html += `</div>`;
    html += `<label>Advance Payment (Months) - this will multiply by monthly and be part of bank deduction</label><input id="advance${id}" type="text" placeholder="0">`;
    html += `<div id="takafulRemark${id}" class="remark"></div>`;
  }

  else { // P2P, Bank SME, SME IMS, Hire Purchase
    if(mode === 'loan'){
      html += `<label>Loan Size (RM)</label><input id="loan${id}" type="text" placeholder="${type==='Bank SME Loan'?'1,000,000.00':'100,000.00'}">`;
    } else {
      html += `<label>Monthly Payment (RM)</label><input id="monthly${id}" type="text" placeholder="${type==='Bank SME Loan'?'10,000.00':'5,000.00'}">`;
    }
    html += `<div class="row">`;
    if(type === 'P2P'){
      html += `<div><label>Interest Rate (%): 8%-14%pa</label><input id="rate${id}" type="text" placeholder="10.00"></div>`;
      html += `<div><label>Tenure (Years): Max 3</label><input id="tenure${id}" type="text" placeholder="3"></div>`;
      html += `</div><label>Service Fees (%) - Direct base 12%, Max 18%; IMS base 16%-22%</label><input id="fees${id}" type="text" placeholder="15">`;
    } else if(type === 'Hire Purchase'){
      html += `<div><label>Interest Rate (%): 2.5-7%pa</label><input id="rate${id}" type="text" placeholder="5.00"></div>`;
      html += `<div><label>Tenure (Years): 3-9</label><input id="tenure${id}" type="text" placeholder="5"></div>`;
      html += `</div><div class="remark">Service fees not applied to Hire Purchase</div><input id="fees${id}" type="hidden">`;
    } else {
      html += `<div><label>Interest Rate (%): 5.5%-9%pa</label><input id="rate${id}" type="text" placeholder="6.50"></div>`;
      html += `<div><label>Tenure (Years): 5-7 years</label><input id="tenure${id}" type="text" placeholder="6"></div>`;
      html += `</div><label>Service Fees (%)</label><input id="fees${id}" type="text" placeholder="${type==='SME IMS'?'14':'6'}">`;
      if(type === 'SME IMS'){
        html += `<div class="remark">Note (IMS): does not include 1.5% Trx Fees @ Credit-in and Commitment Fees RM15k</div>`;
      }
    }
  }

  container.innerHTML = html;

  // attach formatting to inputs
  const inputs = container.querySelectorAll('input[type="text"], input[type="number"]');
  inputs.forEach(el => attachSmartFormatting(el));
}

/* when user changes cashback mode show nothing special (we parse value on calculate)
   but keep this function to allow future UI changes */
function onCashbackModeChange(id){
  // nothing to do here now; value read in calculate()
}

/* Scroll to calculator when clicking summary title */
function scrollToCalculator(id){
  const calc = document.getElementById('calc'+id);
  if(calc){
    calc.scrollIntoView({ behavior: 'smooth' });
  }
}

/* =========================
   Calculation logic (with mortgage cashback override)
   ========================= */

function calculate(id){
  const type = document.getElementById('type'+id).value;
  const mode = modes[id] || 'loan';

  function v(key){ const el = document.getElementById(key+id); return el ? parseNumStr(el.value) : 0; }

  let loanInput = v('loan');
  let monthlyInput = v('monthly');
  let ratePerc = (v('rate') || 0) / 100;
  let tenureYears = v('tenure') || 0;
  let feesPerc = (v('fees') || 0) / 100;
  let loan = 0, monthly = 0;
  const nMonths = Math.max(1, Math.round(tenureYears * 12));

  // Compute based on mode
  if(mode === 'loan'){
    loan = loanInput;
    if(type === 'Mortgage'){
      const monthlyRate = ratePerc / 12;
      if(monthlyRate === 0) monthly = loan / nMonths;
      else monthly = loan * monthlyRate / (1 - Math.pow(1 + monthlyRate, -nMonths));
    } else {
      monthly = (loan * (1 + (ratePerc * tenureYears))) / (nMonths);
    }
  } else {
    monthly = monthlyInput;
    if(type === 'Mortgage'){
      const monthlyRate = ratePerc / 12;
      if(monthlyRate === 0) loan = monthly * nMonths;
      else loan = monthly * (1 - Math.pow(1 + monthlyRate, -nMonths)) / monthlyRate;
    } else {
      const denom = (ratePerc * tenureYears + 1) || 1;
      loan = (monthly * (tenureYears * 12)) / denom;
    }
  }

  // service fee (not for hire purchase)
  let serviceFee = (type === 'Hire Purchase') ? 0 : loan * feesPerc;

  // bank deduction, cash on hand, legal items, takaful, advance
  let bankDeduction = 0, cashOnHand = 0;
  let legalItems = null, takafulAmount = 0, advanceAmount = 0;

  if(type === 'Mortgage'){
    const snpVal = v('snp') || loan;
    const mot = calcStampDutyMOT(snpVal);
    const loanStamp = calcStampDutyLoan(loan);
    const legalSNP = calcLegalFees(snpVal);
    const legalLoan = calcLegalFees(loan);
    legalItems = { mot, loanStamp, legalSNP, legalLoan };

    const discountPerc = (v('legalDiscount') || 0) / 100;
    let totalLegal = (legalSNP + legalLoan) * (1 - discountPerc);

    // bankDeduction base
    bankDeduction = loan * 0.005 + (document.getElementById('includeLegal'+id)?.checked ? totalLegal : 0);

    // Cashback override:
    // If cashback mode percent -> cashbackValue = percent * SNP price (as RM)
    // If cashback mode manual -> cashbackValue = manual RM input
    // If cashback specified (non-zero) => Cash on Hand = cashbackValue
    // Else fallback to loan - bankDeduction - serviceFee
    const cbMode = document.getElementById('cashbackMode'+id)?.value || 'none';
    const cbRaw = parseNumStr(document.getElementById('cashback'+id)?.value || 0);
    let cashbackValue = 0;
    if(cbMode === 'percent' && cbRaw > 0){
      cashbackValue = (snpVal || loan) * (cbRaw / 100);
    } else if(cbMode === 'manual' && cbRaw > 0){
      cashbackValue = cbRaw;
    }

    if(cashbackValue > 0){
      cashOnHand = cashbackValue; // override per your instruction
    } else {
      cashOnHand = loan - bankDeduction - serviceFee;
    }

  } else if(type === 'P2P'){
    bankDeduction = ((0.015 * loan) * tenureYears) + monthly;
    cashOnHand = loan - bankDeduction - serviceFee;
  } else if(type === 'Bank SME Loan' || type === 'SME IMS'){
    bankDeduction = 0.015 * loan;
    cashOnHand = loan - bankDeduction - serviceFee;
  } else if(type === 'Personal Loan'){
    const includeTakaful = (document.getElementById('takaful'+id)?.value === 'yes');
    const age = v('age') || 30;
    const advMonths = v('advance') || 0;
    advanceAmount = monthly * advMonths;
    let ageKeys = Object.keys(mortalityRates).map(Number).sort((a,b)=>a-b);
    let chosen = ageKeys[0];
    for(const k of ageKeys) if(k <= age) chosen = k;
    const mortRate = mortalityRates[chosen] || 0.004;
    // Takaful = (base takaful) * tenure, where base takaful = loan * mortality rate
    if(includeTakaful){
      const baseTakaful = loan * mortRate;
      takafulAmount = baseTakaful * tenureYears;
    } else {
      takafulAmount = 0;
    }
    bankDeduction = (0.005 * loan) + takafulAmount + advanceAmount;
    cashOnHand = loan - bankDeduction - serviceFee;
    if(includeTakaful){
      document.getElementById('takafulRemark'+id).innerText = `Takaful calculated by (loan × mortality rate) × tenure (estimated). Amount may vary.`;
    } else {
      document.getElementById('takafulRemark'+id).innerText = '';
    }
  } else if(type === 'Hire Purchase'){
    bankDeduction = 0.005 * loan;
    serviceFee = 0;
    cashOnHand = loan - bankDeduction - serviceFee;
  } else {
    bankDeduction = 0.015 * loan;
    cashOnHand = loan - bankDeduction - serviceFee;
  }

  // Render results
  const resultsEl = document.getElementById('results'+id);
  let html = '';
  html += `<div><strong>Loan Size:</strong> RM ${fmt(loan)}</div>`;
  html += `<div><strong>Monthly Payment:</strong> RM ${fmt(monthly)}</div>`;
  html += `<div><strong>Service Fees:</strong> RM ${fmt(serviceFee)}</div>`;

  if(legalItems){
    html += `<div style="margin-top:8px"><strong>Legal & Stamp (estimated):</strong>
              <div class="legal-list">
                MOT Stamp Duty: RM ${fmt(legalItems.mot)}<br>
                Loan Stamp Duty: RM ${fmt(legalItems.loanStamp)}<br>
                Legal Fees (SNP): RM ${fmt(legalItems.legalSNP)}<br>
                Legal Fees (Loan): RM ${fmt(legalItems.legalLoan)}
              </div>
            </div>`;
    const includeFlag = document.getElementById('includeLegal'+id)?.checked ?? true;
    html += `<div style="margin-top:6px"><label><input id="includeLegalList${id}" type="checkbox" ${includeFlag ? 'checked' : ''} onchange="syncIncludeLegal(${id})"> Include listed legal fees in bank deduction</label></div>`;
  }

  // show cashback mode result if applicable
  if(type === 'Mortgage'){
    const cbMode = document.getElementById('cashbackMode'+id)?.value || 'none';
    const cbRaw = parseNumStr(document.getElementById('cashback'+id)?.value || 0);
    if(cbMode === 'percent' && cbRaw > 0){
      const snpVal = v('snp') || loan;
      const cbAmt = (snpVal || loan) * (cbRaw / 100);
      html += `<div style="margin-top:6px"><strong>Cashback (${cbRaw}% of SNP):</strong> RM ${fmt(cbAmt)} <div class="muted">Cash on Hand override</div></div>`;
    } else if(cbMode === 'manual' && cbRaw > 0){
      html += `<div style="margin-top:6px"><strong>Cashback (Manual):</strong> RM ${fmt(cbRaw)} <div class="muted">Cash on Hand override</div></div>`;
    }
  }

  if(type === 'Personal Loan' && (takafulAmount > 0)){
    html += `<div style="margin-top:6px"><strong>Takaful:</strong> RM ${fmt(takafulAmount)}</div>`;
  }
  if(type === 'Personal Loan' && advanceAmount > 0){
    html += `<div style="margin-top:6px"><strong>Advance Payment:</strong> RM ${fmt(advanceAmount)}</div>`;
  }

  html += `<div style="margin-top:8px"><strong>Bank Deduction (estimated):</strong> RM ${fmt(bankDeduction)}</div>`;
  html += `<div style="margin-top:10px"><strong>Cash on Hand:</strong> <strong>RM ${fmt(cashOnHand)}</strong></div>`;

  resultsEl.innerHTML = html;

  // persist for summary
  const calcNode = document.getElementById('calc'+id);
  calcNode.dataset.loan = String(loan || 0);
  calcNode.dataset.monthly = String(monthly || 0);
  calcNode.dataset.service = String(serviceFee || 0);
  calcNode.dataset.cash = String(cashOnHand || 0);

  if(includedCalcs[id]) updateSummaryUI();
}

/* sync includeLegal checkbox click inside results to the input checkbox */
function syncIncludeLegal(id){
  const listCheckbox = document.getElementById('includeLegalList'+id);
  const inputCheckbox = document.getElementById('includeLegal'+id);
  if(inputCheckbox && listCheckbox){
    inputCheckbox.checked = listCheckbox.checked;
    calculate(id);
  }
}

/* =========================
   Summary logic
   ========================= */

function toggleSummary(id){
  if(includedCalcs[id]){
    delete includedCalcs[id];
    document.getElementById('summaryBtn'+id).innerText = 'Add to Summary';
  } else {
    includedCalcs[id] = true;
    document.getElementById('summaryBtn'+id).innerText = 'Remove from Summary';
  }
  updateSummaryUI();
}

function updateSummaryUI(){
  const summaryContent = document.getElementById('summaryContent');
  const totalsEl = document.getElementById('summaryTotals');
  let html = '';
  let totalCash = 0, totalMonthly = 0, totalService = 0;

  for(const idStr of Object.keys(includedCalcs)){
    const id = Number(idStr);
    const node = document.getElementById('calc'+id);
    if(!node) continue;
    const title = document.getElementById('title'+id)?.value || document.getElementById('type'+id)?.value || ('Calculator '+id);
    const resultsHtml = document.getElementById('results'+id)?.innerHTML || '';
    const cash = Number(node.dataset.cash || 0);
    const monthly = Number(node.dataset.monthly || 0);
    const service = Number(node.dataset.service || 0);
    totalCash += cash;
    totalMonthly += monthly;
    totalService = service;

    html += `<li><a href="#calc${id}" onclick="scrollToCalculator(${id})"><strong>${title}</strong></a><div style="margin-top:6px">${resultsHtml}</div>
             <div style="margin-top:6px"><button style="background:#dc3545;color:#fff;padding:6px;border:none;border-radius:6px" onclick="removeFromSummary(${id})">Delete from Summary</button></div></li>`;
  }

  if(html === '') summaryContent.innerHTML = 'No calculators included yet.';
  else summaryContent.innerHTML = `<ul style="list-style:none;padding-left:0;margin:0">${html}</ul>`;

  if(Object.keys(includedCalcs).length > 0){
    totalsEl.style.display = 'block';
    document.getElementById('sumCash').innerText = fmt(totalCash);
    document.getElementById('sumMonthly').innerText = fmt(totalMonthly);
    document.getElementById('sumService').innerText = fmt(totalService);
  } else {
    totalsEl.style.display = 'none';
    document.getElementById('sumCash').innerText = '0.00';
    document.getElementById('sumMonthly').innerText = '0.00';
    document.getElementById('sumService').innerText = '0.00';
  }
}

function removeFromSummary(id){
  delete includedCalcs[id];
  const btn = document.getElementById('summaryBtn'+id);
  if(btn) btn.innerText = 'Add to Summary';
  updateSummaryUI();
}

/* =========================
   PDF (only summary)
   ========================= */
function downloadPDF(){
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF({unit:'pt', format:'a4'});
  let y = 40;
  const left = 40;
  doc.setFontSize(16);
  doc.text('BackRock Loan Calculators - Summary', left, y);
  y += 24;
  doc.setFontSize(11);

  for(const idStr of Object.keys(includedCalcs)){
    const id = Number(idStr);
    const title = document.getElementById('title'+id)?.value || document.getElementById('type'+id)?.value || ('Calculator '+id);
    doc.setFont(undefined, 'bold');
    doc.text(title, left, y); y += 14;
    doc.setFont(undefined, 'normal');
    const resultsText = (document.getElementById('results'+id)?.innerText || '').split('\n').map(s=>s.trim()).filter(Boolean);
    resultsText.forEach(line=>{
      const wrapped = doc.splitTextToSize(line, 500);
      doc.text(wrapped, left + 8, y);
      y += 14 * wrapped.length;
    });
    y += 8;
    if(y > 740){ doc.addPage(); y = 40; }
  }

  // Totals
  const sumCash = document.getElementById('sumCash').innerText || '0.00';
  const sumMonthly = document.getElementById('sumMonthly').innerText || '0.00';
  const sumService = document.getElementById('sumService').innerText || '0.00';
  doc.setFont(undefined, 'bold');
  doc.text('Totals', left, y); y += 14;
  doc.setFont(undefined, 'normal');
  doc.text(`Total Cash on Hand: RM ${sumCash}`, left + 6, y); y += 14;
  doc.text(`Total Monthly Payments: RM ${sumMonthly}`, left + 6, y); y += 14;
  doc.text(`Total Service Fees: RM ${sumService}`, left + 6, y); y += 14;

  doc.save('BackRock_Summary.pdf');
}

/* =========================
   Init - add one calculator by default
   ========================= */
addCalculator();
</script>
</body>
</html>
