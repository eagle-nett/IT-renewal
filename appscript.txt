/**
 * TRẠM GIA HẠN — Apps Script backend
 * ------------------------------------------------------------
 * Gắn script này vào Google Sheet chứa tab "Renewals" (import từ
 * file renewals-sheet-template.xlsx). Không cần sửa gì trong file
 * này để chạy — chỉ cần thiết lập Script Properties bên dưới rồi
 * Deploy > New deployment > Web app.
 *
 * SCRIPT PROPERTIES cần thiết lập (menu Project Settings > Script
 * Properties trong Apps Script editor):
 *   NOTIFY_EMAIL   -> email nhận thông báo, ví dụ: lamhiephung88@gmail.com
 *   LEAD_DAYS      -> số ngày muốn được nhắc trước (mặc định 7 nếu bỏ trống)
 *
 * Sau khi deploy Web app, chạy hàm installDailyTrigger() MỘT LẦN
 * (chọn hàm trong thanh công cụ Apps Script rồi bấm Run) để tự
 * động kiểm tra + gửi email mỗi ngày lúc 7 giờ sáng.
 */

var SHEET_NAME = 'Renewals';
var HEADERS = ['ID', 'Group', 'Provider', 'Category', 'ServiceName', 'Target',
  'ExpiryDate', 'Cost', 'Currency', 'Note', 'Status', 'LastNotifiedDate'];

// ---------------------------------------------------------------
// Web app entry points
// ---------------------------------------------------------------

function doGet(e) {
  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var items = [];
  for (var i = 1; i < data.length; i++) {
    var row = data[i];
    var obj = {};
    for (var c = 0; c < headers.length; c++) {
      obj[headers[c]] = row[c];
    }
    if (!obj.ID) continue; // bỏ qua dòng trống
    obj.ExpiryDate = fmtDate_(obj.ExpiryDate);
    obj.LastNotifiedDate = fmtDate_(obj.LastNotifiedDate);
    items.push(obj);
  }
  return jsonOut_({ ok: true, items: items, generatedAt: new Date().toISOString() });
}

// Front-end gửi POST với Content-Type: text/plain để tránh CORS
// preflight — nội dung vẫn là JSON hợp lệ, chỉ khai báo header khác đi.
function doPost(e) {
  try {
    var body = JSON.parse(e.postData.contents);
    switch (body.action) {
      case 'renew': return renewItem_(body.id, body.newExpiry);
      case 'create': return createItem_(body.item || {});
      case 'update': return updateItem_(body.id, body.item || {});
      case 'delete': return deleteItem_(body.id);
      default: return jsonOut_({ ok: false, error: 'action không hợp lệ: ' + body.action });
    }
  } catch (err) {
    return jsonOut_({ ok: false, error: String(err) });
  }
}

function renewItem_(id, newExpiryIso) {
  if (!id || !newExpiryIso) {
    return jsonOut_({ ok: false, error: 'Thiếu id hoặc newExpiry' });
  }
  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var idCol = headers.indexOf('ID');
  var expiryCol = headers.indexOf('ExpiryDate');
  var lastNotifCol = headers.indexOf('LastNotifiedDate');
  for (var i = 1; i < data.length; i++) {
    if (String(data[i][idCol]) === String(id)) {
      sheet.getRange(i + 1, expiryCol + 1).setValue(toDate_(newExpiryIso));
      if (lastNotifCol > -1) sheet.getRange(i + 1, lastNotifCol + 1).setValue('');
      return jsonOut_({ ok: true, id: id, newExpiry: newExpiryIso });
    }
  }
  return jsonOut_({ ok: false, error: 'Không tìm thấy ID: ' + id });
}

// Thêm dòng mới. `item` là object các cột (ID và LastNotifiedDate bị bỏ qua nếu có gửi lên).
function createItem_(item) {
  if (!item.ServiceName) return jsonOut_({ ok: false, error: 'Thiếu ServiceName' });
  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var id = nextId_(data, headers);
  var row = headers.map(function (h) {
    if (h === 'ID') return id;
    if (h === 'LastNotifiedDate') return '';
    if (h === 'Status') return item.Status || 'active';
    if (h === 'Currency') return item.Currency || 'VND';
    if (h === 'ExpiryDate') return item.ExpiryDate ? toDate_(item.ExpiryDate) : '';
    return (item[h] !== undefined && item[h] !== null) ? item[h] : '';
  });
  sheet.appendRow(row);
  return jsonOut_({ ok: true, id: id });
}

// Cập nhật một hoặc nhiều cột của dòng có ID = id. Chỉ những cột có mặt
// trong `item` mới bị ghi đè — cột không gửi lên giữ nguyên giá trị cũ.
function updateItem_(id, item) {
  if (!id) return jsonOut_({ ok: false, error: 'Thiếu id' });
  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var idCol = headers.indexOf('ID');
  var lastNotifCol = headers.indexOf('LastNotifiedDate');
  for (var i = 1; i < data.length; i++) {
    if (String(data[i][idCol]) === String(id)) {
      headers.forEach(function (h, c) {
        if (h === 'ID' || h === 'LastNotifiedDate') return;
        if (!Object.prototype.hasOwnProperty.call(item, h)) return;
        var v = item[h];
        if (h === 'ExpiryDate') v = v ? toDate_(v) : '';
        sheet.getRange(i + 1, c + 1).setValue(v);
      });
      if (lastNotifCol > -1 && Object.prototype.hasOwnProperty.call(item, 'ExpiryDate')) {
        sheet.getRange(i + 1, lastNotifCol + 1).setValue('');
      }
      return jsonOut_({ ok: true, id: id });
    }
  }
  return jsonOut_({ ok: false, error: 'Không tìm thấy ID: ' + id });
}

// Xoá hẳn dòng khỏi Sheet.
function deleteItem_(id) {
  if (!id) return jsonOut_({ ok: false, error: 'Thiếu id' });
  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var idCol = headers.indexOf('ID');
  for (var i = 1; i < data.length; i++) {
    if (String(data[i][idCol]) === String(id)) {
      sheet.deleteRow(i + 1);
      return jsonOut_({ ok: true, id: id });
    }
  }
  return jsonOut_({ ok: false, error: 'Không tìm thấy ID: ' + id });
}

function nextId_(data, headers) {
  var idCol = headers.indexOf('ID');
  var max = 0;
  for (var i = 1; i < data.length; i++) {
    var m = String(data[i][idCol] || '').match(/^R(\d+)$/);
    if (m) max = Math.max(max, parseInt(m[1], 10));
  }
  var n = max + 1;
  return 'R' + (n < 10 ? '0' + n : n);
}

function toDate_(iso) {
  var p = String(iso).split('-').map(Number);
  return new Date(p[0], p[1] - 1, p[2]);
}

// ---------------------------------------------------------------
// Nhắc gia hạn qua email — chạy hàng ngày bằng trigger thời gian
// ---------------------------------------------------------------

function dailyCheckAndNotify() {
  var props = PropertiesService.getScriptProperties();
  var email = props.getProperty('NOTIFY_EMAIL');
  if (!email) {
    Logger.log('Chưa thiết lập NOTIFY_EMAIL trong Script Properties — bỏ qua.');
    return;
  }
  var leadDays = Number(props.getProperty('LEAD_DAYS')) || 7;

  var sheet = getSheet_();
  var data = sheet.getDataRange().getValues();
  var headers = data[0];
  var idx = {};
  headers.forEach(function (h, i) { idx[h] = i; });

  var tz = Session.getScriptTimeZone();
  var today = new Date();
  today.setHours(0, 0, 0, 0);
  var todayStr = Utilities.formatDate(today, tz, 'yyyy-MM-dd');

  var due = [];
  for (var i = 1; i < data.length; i++) {
    var row = data[i];
    if (!row[idx.ID]) continue;
    if (String(row[idx.Status]).toLowerCase() === 'cancelled') continue;
    var expiry = row[idx.ExpiryDate];
    if (!(expiry instanceof Date)) continue; // bỏ qua mục chưa có ngày hết hạn

    var ex = new Date(expiry);
    ex.setHours(0, 0, 0, 0);
    var daysLeft = Math.round((ex - today) / 86400000);
    if (daysLeft > leadDays) continue;

    var lastNotified = row[idx.LastNotifiedDate];
    var notifiedToday = lastNotified instanceof Date &&
      Utilities.formatDate(lastNotified, tz, 'yyyy-MM-dd') === todayStr;
    if (notifiedToday) continue;

    due.push({ sheetRow: i + 1, row: row, daysLeft: daysLeft });
  }

  if (due.length === 0) return;
  due.sort(function (a, b) { return a.daysLeft - b.daysLeft; });

  var lines = due.map(function (d) {
    var r = d.row;
    var statusTxt = d.daysLeft < 0 ? ('QUÁ HẠN ' + Math.abs(d.daysLeft) + ' ngày')
      : (d.daysLeft === 0 ? 'HẾT HẠN HÔM NAY' : ('còn ' + d.daysLeft + ' ngày'));
    var expiryTxt = Utilities.formatDate(new Date(r[idx.ExpiryDate]), tz, 'dd/MM/yyyy');
    var provider = r[idx.Provider] ? (' — NCC: ' + r[idx.Provider]) : '';
    return '- [' + statusTxt + '] ' + r[idx.ServiceName] + ' (' + r[idx.Category] + ') — ' +
      r[idx.Target] + ' — hết hạn ' + expiryTxt + provider;
  });

  var subject = '[Trạm Gia Hạn] ' + due.length + ' mục cần chú ý (trong ' + leadDays + ' ngày tới)';
  var sheetUrl = SpreadsheetApp.getActiveSpreadsheet().getUrl();
  var body = 'Các mục sắp hoặc đã hết hạn:\n\n' + lines.join('\n') +
    '\n\nMở Google Sheet để xem chi tiết hoặc cập nhật:\n' + sheetUrl;

  MailApp.sendEmail(email, subject, body);

  due.forEach(function (d) {
    sheet.getRange(d.sheetRow, idx.LastNotifiedDate + 1).setValue(today);
  });
}

// Chạy hàm này MỘT LẦN từ Apps Script editor để tạo trigger hàng ngày lúc 7h sáng.
function installDailyTrigger() {
  ScriptApp.getProjectTriggers().forEach(function (t) {
    if (t.getHandlerFunction() === 'dailyCheckAndNotify') ScriptApp.deleteTrigger(t);
  });
  ScriptApp.newTrigger('dailyCheckAndNotify')
    .timeBased()
    .everyDays(1)
    .atHour(7)
    .create();
  Logger.log('Đã tạo trigger hàng ngày lúc 7:00 cho dailyCheckAndNotify().');
}

// ---------------------------------------------------------------
// Helpers
// ---------------------------------------------------------------

function getSheet_() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  if (!sheet) throw new Error('Không tìm thấy tab "' + SHEET_NAME + '". Đổi tên tab hoặc sửa SHEET_NAME trong Code.gs.');
  return sheet;
}

function fmtDate_(v) {
  if (v instanceof Date) {
    return Utilities.formatDate(v, Session.getScriptTimeZone(), 'yyyy-MM-dd');
  }
  return v || '';
}

function jsonOut_(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
