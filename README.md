<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>月間勤務時間集計ツール（一括指定 + 有給対応）</title>
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

/* --- 曜日・祝日・有給の配色 --- */

/* 日曜日 (赤色系) */
.sunday-row {
    background-color: #fcebeb; /* 非常に薄い赤 */
    color: #cc3333; /* 濃い赤 */
    font-weight: bold;
}

/* 土曜日 (青色系) */
.saturday-row {
    background-color: #ebf0fc; /* 非常に薄い青 */
    color: #3333cc; /* 濃い青 */
    font-weight: bold;
}

/* 祝日 (いい感じの色 - 薄い緑系) */
/* 土日と祝日が重なった場合、この色が優先されます */
.national-holiday-row {
    background-color: #ebfceb; /* 非常に薄い緑 */
    color: #1a7e1a; /* 濃い緑 */
    font-weight: bold;
}

/* 有給 (別のいい感じの色 - 薄い黄色系) */
/* !important を使い、他の色分けを上書き */
.paid-leave-row {
    background-color: #fffbe5 !important; 
    color: #997300 !important; /* 濃い茶色 */
    font-weight: bold;
}

/* --- その他 --- */
#result { font-weight: bold; margin-top: 20px; font-size: 1.1em; }
#paidCount { font-weight: bold; margin-top: 10px; font-size: 1.1em; }
.label { font-size: 0.9em; color: #333; }
</style>
</head>
<body>

<header>
  <label class="label">年月:
    <input type="month" id="monthSelect">
  </label>
  <button id="loadBtn">日付を表示</button>

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
<div id="paidCount"></div>

<script>
// APIから取得した祝日データを格納する変数
let currentHolidays = [];

/**
 * Holidays JP APIから指定された年の祝日を取得する
 * @param {number} year - 取得する年
 * @returns {Promise<string[]>} YYYY-MM-DD 形式の祝日日付の配列
 */
async function fetchHolidays(year) {
    try {
        const url = `https://holidays-jp.github.io/api/v1/${year}/date.json`;
        const response = await fetch(url);

        if (!response.ok) {
            // APIが200以外のコードを返した場合（例：まだデータのない遠い年など）
            console.warn(`祝日データ取得HTTPエラー (${response.status})。祝日なしとして続行します。`);
            return [];
        }

        const data = await response.json();
        // APIのレスポンスは { "YYYY-MM-DD": "祝日名", ... } の形式なので、日付のキーだけを取得
        return Object.keys(data);
    } catch (error) {
        // ネットワークエラーなどの場合
        console.error('祝日データの取得に失敗しました:', error);
        alert('祝日データの取得に失敗しました。ネットワーク接続を確認してください。');
        return [];
    }
}

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
  if (diff < 0) diff += 24 * 60; // 翌日またぎ
  return diff > 0 ? diff : 0;
}

// 日付テーブル生成
document.getElementById('loadBtn').addEventListener('click', async () => {
  const monthValue = document.getElementById('monthSelect').value;
  if (!monthValue) {
    alert('年月を選択してください');
    return;
  }

  const [year, month] = monthValue.split('-').map(Number);
  
  // --- 祝日データをAPIから取得 ---
  currentHolidays = await fetchHolidays(year);
  // ----------------------------

  const days = getDaysInMonth(year, month);

  let html = `
  <table id="workTable">
    <thead>
      <tr>
        <th>日付</th><th>曜日</th><th>有給</th>
        <th>開始</th><th>終了</th><th>休憩(分)</th><th>実働時間</th>
      </tr>
    </thead>
    <tbody>
  `;

  for (let d = 1; d <= days; d++) {
    const date = new Date(year, month - 1, d);
    const dateStr = `${year}-${String(month).padStart(2,'0')}-${String(d).padStart(2,'0')}`;
    const dayName = getWeekdayName(date);
    const dayOfWeek = date.getDay(); // 0: 日曜日, 6: 土曜日
    
    // APIから取得した祝日データで判定
    const isHoliday = currentHolidays.includes(dateStr); 
    
    let rowClass = '';
    
    // 1. 曜日・祝日による色分けクラスを決定 (祝日を優先)
    if (isHoliday) {
        rowClass = 'national-holiday-row';
    } else if (dayOfWeek === 0) {
        rowClass = 'sunday-row';
    } else if (dayOfWeek === 6) {
        rowClass = 'saturday-row';
    }
    
    // 2. 入力無効化と計算スキップのためのマーカークラスを追加
    // 土日祝すべてに適用
    const isWeekendOrHoliday = (dayOfWeek === 0 || dayOfWeek === 6 || isHoliday);
    if (isWeekendOrHoliday) {
        rowClass += ' disable-marker'; 
    }

    html += `
    <tr class="${rowClass.trim()}">
      <td>${dateStr}</td>
      <td>${dayName}${isHoliday ? '（祝）' : ''}</td>

      <td><input type="checkbox" class="paid"></td>

      <td><input type="time" class="start"></td>
      <td><input type="time" class="end"></td>
      <td><input type="number" class="break" value="60" min="0" max="600"></td>
      <td class="work">0.00</td>
    </tr>`;
  }

  html += `</tbody></table>`;
  document.getElementById('tableArea').innerHTML = html;

  document.getElementById('result').textContent = '合計実働時間：0.00 時間';
  document.getElementById('paidCount').textContent = '有給：0 日';

  const rows = document.querySelectorAll('#workTable tbody tr');

  // 土日祝(disable-markerを持つ行)は入力と有給チェックを無効化
  rows.forEach(row => {
    if (row.classList.contains('disable-marker')) {
      row.querySelectorAll('input').forEach(i => i.disabled = true);
    }
  });

  // 入力イベント追加
  document.querySelectorAll('#workTable input').forEach(inp => {
    inp.addEventListener('input', autoCalc);
    inp.addEventListener('change', autoCalc);
  });

  autoCalc();
});

// 一括適用
document.getElementById('applyBulkBtn').addEventListener('click', () => {
  const bulkStart = document.getElementById('bulkStart').value;
  const bulkEnd = document.getElementById('bulkEnd').value;
  const bulkBreak = document.getElementById('bulkBreak').value;

  if (!bulkStart || !bulkEnd) {
    alert('一括開始・終了を指定してください');
    return;
  }

  document.querySelectorAll('#workTable tbody tr').forEach(row => {
    // disable-marker（土日祝）の行はスキップ
    if (row.classList.contains('disable-marker')) return;
    
    const paid = row.querySelector('.paid')?.checked;
    if (paid) return; // 有給は上書きしない

    row.querySelector('.start').value = bulkStart;
    row.querySelector('.end').value = bulkEnd;
    row.querySelector('.break').value = bulkBreak;
  });

  autoCalc();
});

// 自動集計
function autoCalc() {
  let totalMinutes = 0;
  let paidDays = 0;

  const rows = document.querySelectorAll('#workTable tbody tr');
  rows.forEach(row => {
    // 土日祝判定 (disable-markerクラスを利用)
    const isWeekendOrHoliday = row.classList.contains('disable-marker'); 
    const paidCheck = row.querySelector('.paid');
    const paid = paidCheck?.checked;

    const start = row.querySelector('.start');
    const end = row.querySelector('.end');
    const brk = row.querySelector('.break');
    const workCell = row.querySelector('.work');

    // 休日（土日祝）は計算スキップ
    if (isWeekendOrHoliday) {
      workCell.textContent = '-';
      return;
    }

    // 有給ONのとき
    if (paid) {
      // 有給の色分けクラスを適用
      row.classList.add('paid-leave-row'); 

      // 入力禁止
      start.disabled = true;
      end.disabled = true;
      brk.disabled = true;

      workCell.textContent = '-';
      paidDays++;
      return;
    }

    // 有給OFFに戻した時 → 平日は解除
    row.classList.remove('paid-leave-row');
    start.disabled = false;
    end.disabled = false;
    brk.disabled = false;

    // 実働計算
    const minutes = calcWorkMinutes(start.value, end.value, brk.value);
    workCell.textContent = (minutes / 60).toFixed(2);
    totalMinutes += minutes;
  });

  const totalHours = (totalMinutes / 60).toFixed(2);

  document.getElementById('result').textContent =
    `合計実働時間：${totalHours} 時間`;

  document.getElementById('paidCount').textContent =
    `有給：${paidDays} 日`;
}
</script>

</body>
</html>
