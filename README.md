# -_-<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>月間勤務時間集計ツール（一括指定入力UI）</title>
<style>
body { font-family: sans-serif; margin: 20px; background: #f9f9f9; }
header { 
  position: sticky; 
  top: 0; 
  background: white; 
  padding: 10px; 
  box-shadow: 0 2px 6px rgba(0,0,0,0.1);
  display: flex; 
  gap: 12px; 
  align-items: center;
  z-index: 10;
  flex-wrap: wrap;
}
button { 
  padding: 8px 12px; 
  border: none; 
  background: #0078d7; 
  color: white; 
  border-radius: 6px; 
  cursor: pointer; 
}
button:hover { background: #005fa3; }
table { border-collapse: collapse; width: 100%; margin-top: 10px; background: white; }
th, td { border: 1px solid #ccc; padding: 6px; text-align: center; }
th { background: #eee; }
input[type="number"] { width: 80px; }
.holiday-row { background-color: #ffe5e5; color: red; font-weight: bold; }
#result { font-weight: bold; margin-top: 20px; font-size: 1.1em; }
.label { font-size: 0.9em; color: #333; }
</style>
</head>
<body>

<header>
  <label class="label">年月:
    <input type="month" id="monthSelect">
  </label>
  <button id="loadBtn">日付を表示</button>

  <!-- 一括指定用のUI（ここで選択／入力） -->
  <label class="label">一括開始:
    <input type="time" id="bulkStart" value="09:00">
  </label>
  <label class="label">一括終了:
    <input type="time" id="bulkEnd" value="18:00">
  </label>
  <label class="label">一括休憩(分):
    <input type="number" id="bulkBreak" value="60" min="0" max="600" style="width:70px">
  </label>
  <button id="applyBulkBtn">全日一括適用</button>
</header>

<div id="tableArea"></div>
<div id="result"></div>

<script>
// --- 2025年の祝日（ベタ書き・振替は簡易的に追加） ---
let holidays2025 = [
  '2025-01-01','2025-01-13','2025-02-11','2025-02-23',
  '2025-03-20','2025-04-29','2025-05-03','2025-05-04','2025-05-05',
  '2025-05-06','2025-07-21','2025-08-11','2025-09-15','2025-09-23',
  '2025-10-13','2025-11-03','2025-11-23','2025-12-23'
];
// 簡易振替処理（祝日が日曜なら翌日を休みに追加）
(function addSubstitute() {
  const add = [];
  holidays2025.forEach(s => {
    const d = new Date(s);
    if (d.getDay() === 0) {
      const n = new Date(d);
      n.setDate(n.getDate() + 1);
      add.push(n.toISOString().slice(0,10));
    }
  });
  holidays2025 = holidays2025.concat(add);
})();

function getDaysInMonth(year, month) {
  return new Date(year, month, 0).getDate();
}
function getWeekdayName(date) {
  return ['日','月','火','水','木','金','土'][date.getDay()];
}

// 実働（分）を計算
function calcWorkMinutes(start, end, breakMin) {
  if (!start || !end) return 0;
  const [sh, sm] = start.split(':').map(Number);
  const [eh, em] = end.split(':').map(Number);
  let diff = (eh * 60 + em) - (sh * 60 + sm) - (parseInt(breakMin) || 0);
  if (diff < 0) diff += 24 * 60; // 翌日にまたぐ場合
  return diff > 0 ? diff : 0;
}

// 日付テーブル生成
document.getElementById('loadBtn').addEventListener('click', () => {
  const monthValue = document.getElementById('monthSelect').value;
  if (!monthValue) {
    alert('年月を選択してください');
    return;
  }

  const [year, month] = monthValue.split('-').map(Number);
  const days = getDaysInMonth(year, month);

  // テーブル作成（id を付けて tbody を扱いやすくする）
  let html = '<table id="workTable"><thead><tr><th>日付</th><th>曜日</th><th>開始</th><th>終了</th><th>休憩(分)</th><th>実働時間</th></tr></thead><tbody>';
  for (let d = 1; d <= days; d++) {
    const date = new Date(year, month - 1, d);
    const dateStr = `${year}-${String(month).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
    const dayName = getWeekdayName(date);
    const isWeekend = (date.getDay() === 0 || date.getDay() === 6);
    const isHoliday = holidays2025.includes(dateStr);
    const rowClass = (isWeekend || isHoliday) ? 'holiday-row' : '';

    html += `<tr class="${rowClass}">
      <td>${dateStr}</td>
      <td>${dayName}${isHoliday ? '（祝）' : ''}</td>
      <td><input type="time" class="start"></td>
      <td><input type="time" class="end"></td>
      <td><input type="number" class="break" min="0" max="600" value="60"></td>
      <td class="work">0.00</td>
    </tr>`;
  }
  html += '</tbody></table>';
  document.getElementById('tableArea').innerHTML = html;
  document.getElementById('result').textContent = '合計実働時間：0.00 時間';

  // 休日行は入力を無効にして赤ハイライト（視覚的に分かる）
  document.querySelectorAll('#workTable tbody tr').forEach(row => {
    if (row.classList.contains('holiday-row')) {
      row.querySelectorAll('input').forEach(i => i.disabled = true);
    }
  });

  // 各入力にイベントを追加（自動計算）
  document.querySelectorAll('#workTable tbody input').forEach(input => {
    input.addEventListener('input', autoCalc);
  });

  // 初回集計
  autoCalc();
});

// ヘッダーの一括入力UIから適用
document.getElementById('applyBulkBtn').addEventListener('click', () => {
  const bulkStart = document.getElementById('bulkStart').value;
  const bulkEnd = document.getElementById('bulkEnd').value;
  const bulkBreak = document.getElementById('bulkBreak').value;

  if (!bulkStart || !bulkEnd) {
    alert('一括開始・一括終了は両方指定してください');
    return;
  }

  // 全行に適用（休日はスキップ）
  document.querySelectorAll('#workTable tbody tr').forEach(row => {
    if (row.classList.contains('holiday-row')) return;
    const startInput = row.querySelector('.start');
    const endInput = row.querySelector('.end');
    const breakInput = row.querySelector('.break');
    if (startInput && endInput && breakInput) {
      startInput.value = bulkStart;
      endInput.value = bulkEnd;
      breakInput.value = bulkBreak;
    }
  });

  // 一括適用後は自動で再計算
  autoCalc();
});

// 自動集計（行ごとの実働を出して合計を表示）
function autoCalc() {
  let totalMinutes = 0;
  const rows = document.querySelectorAll('#workTable tbody tr');
  rows.forEach(row => {
    const start = row.querySelector('.start')?.value;
    const end = row.querySelector('.end')?.value;
    const breakMin = row.querySelector('.break')?.value;
    const workCell = row.querySelector('.work');

    if (!workCell) return;

    if (row.classList.contains('holiday-row')) {
      workCell.textContent = '-';
      return;
    }

    const minutes = calcWorkMinutes(start, end, breakMin);
    workCell.textContent = (minutes / 60).toFixed(2);
    totalMinutes += minutes;
  });

  const totalHours = (totalMinutes / 60).toFixed(2);
  document.getElementById('result').textContent = `合計実働時間：${totalHours} 時間`;
}
</script>

</body>
</html>
