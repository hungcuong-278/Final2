# 🛡️ GOOGLE APPS SCRIPT V2 - BẢO MẬT CHỐNG SPAM

## HƯỚNG DẪN CẬP NHẬT

1. Vào: https://script.google.com/home
2. Mở project **VAAE Conference Registration**
3. **THAY THẾ TOÀN BỘ** code cũ bằng code mới bên dưới
4. Save (Ctrl+S)
5. Deploy → Manage deployments → Edit → Version: New → Deploy

---

## 📋 CODE MỚI (COPY TOÀN BỘ)

```javascript
// =====================================================
// VAAE CONFERENCE 2026 - APPS SCRIPT V2 WITH SECURITY
// Last Updated: 03/02/2026
// =====================================================

var sheetName = 'Trang tính1';
var scriptProp = PropertiesService.getScriptProperties();

// 🛡️ SECURITY SETTINGS
const MAX_REQUESTS_PER_HOUR = 100;  // Giới hạn tổng số request/giờ
const MIN_FILL_TIME = 5000;          // Form phải điền ít nhất 5 giây
const DUPLICATE_CHECK_HOURS = 1;     // Chặn email trùng trong 1 giờ

function setup() {
  var doc = SpreadsheetApp.getActiveSpreadsheet();
  scriptProp.setProperty('key', doc.getId());
}

function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.tryLock(10000);

  try {
    var doc = SpreadsheetApp.openById(scriptProp.getProperty('key'));
    var sheet = doc.getSheetByName(sheetName);
    var data = JSON.parse(e.postData.contents);
    var now = Date.now();

    // =====================================================
    // 🛡️ BẢO MẬT 1: Rate Limiting (Cache-based)
    // =====================================================
    var cache = CacheService.getScriptCache();
    var requestCount = parseInt(cache.get('total_requests') || '0');
    
    if (requestCount >= MAX_REQUESTS_PER_HOUR) {
      Logger.log('⚠️ RATE LIMIT: Too many requests');
      return createResponse('error', 'Hệ thống đang bận, vui lòng thử lại sau!');
    }
    cache.put('total_requests', (requestCount + 1).toString(), 3600); // Reset sau 1h

    // =====================================================
    // 🛡️ BẢO MẬT 2: Validate Timestamp (chống replay attack)
    // =====================================================
    var timestamp = data._timestamp || 0;
    var age = now - timestamp;
    
    if (age < 0 || age > 300000) { // Request phải trong vòng 5 phút
      Logger.log('⚠️ INVALID TIMESTAMP: age=' + age + 'ms');
      return createResponse('error', 'Yêu cầu không hợp lệ, vui lòng tải lại trang!');
    }

    // =====================================================
    // 🛡️ BẢO MẬT 3: Validate Fill Time (chống bot)
    // =====================================================
    var fillTime = data._fillTime || 0;
    if (fillTime > 0 && fillTime < MIN_FILL_TIME) {
      Logger.log('⚠️ BOT DETECTED: fillTime=' + fillTime + 'ms');
      return createResponse('error', 'Vui lòng kiểm tra lại thông tin!');
    }

    // =====================================================
    // 🛡️ BẢO MẬT 4: Validate Email Format
    // =====================================================
    var emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!data.Email || !emailRegex.test(data.Email)) {
      Logger.log('⚠️ INVALID EMAIL: ' + data.Email);
      return createResponse('error', 'Email không hợp lệ!');
    }

    // =====================================================
    // 🛡️ BẢO MẬT 5: Check duplicate email (trong 1 giờ)
    // =====================================================
    var lastRow = sheet.getLastRow();
    if (lastRow > 1) {
      var checkRows = Math.min(200, lastRow - 1); // Kiểm tra 200 dòng gần nhất
      var dataRange = sheet.getRange(lastRow - checkRows + 1, 1, checkRows, 4).getValues();
      
      for (var i = dataRange.length - 1; i >= 0; i--) {
        var rowTimestamp = new Date(dataRange[i][0]).getTime();
        var rowEmail = dataRange[i][3]; // Cột Email
        
        // Nếu email trùng và trong vòng 1 giờ
        if (rowEmail === data.Email && (now - rowTimestamp) < DUPLICATE_CHECK_HOURS * 3600000) {
          Logger.log('⚠️ DUPLICATE EMAIL: ' + data.Email + ' (registered ' + Math.round((now - rowTimestamp)/60000) + ' mins ago)');
          return createResponse('error', 'Email này đã đăng ký trong vòng 1 giờ qua!');
        }
      }
    }

    // =====================================================
    // ✅ TẤT CẢ VALIDATION PASSED - XỬ LÝ DỮ LIỆU
    // =====================================================

    // 1. Xử lý ảnh (Chỉ lưu nếu có ảnh gửi lên)
    var fileUrl = "Không có ảnh";
    if (data.fileData && data.fileName) {
      try {
        var folderName = "Anh_Chuyen_Khoan_HNKH";
        var folders = DriveApp.getFoldersByName(folderName);
        var folder = folders.hasNext() ? folders.next() : DriveApp.createFolder(folderName);
        var decodedBlob = Utilities.base64Decode(data.fileData.split(',')[1]);
        var blob = Utilities.newBlob(decodedBlob, data.mimeType, data.fileName);
        var file = folder.createFile(blob);
        file.setSharing(DriveApp.Access.ANYONE_WITH_LINK, DriveApp.Permission.VIEW);
        fileUrl = file.getUrl();
        Logger.log('✅ File uploaded: ' + fileUrl);
      } catch (fileError) {
        Logger.log('⚠️ File upload error: ' + fileError.toString());
        fileUrl = "Lỗi upload: " + fileError.toString();
      }
    }

    // 2. Ghi vào Sheet (thêm cột metadata)
    var nextRow = sheet.getLastRow() + 1;
    var newRow = [
      new Date(),                    // A: Timestamp
      data.Ho_Ten,                   // B: Họ Tên
      data.Ngay_Sinh,                // C: Ngày Sinh
      data.Email,                    // D: Email
      data.So_Dien_Thoai,            // E: SĐT
      data.Don_Vi,                   // F: Đơn Vị
      data.Chuc_Vu,                  // G: Chức Vụ
      data.Noi_Dung_Tham_Du,         // H: Nội Dung
      data.Dia_Chi_CME || '',        // I: Địa Chỉ CME
      fileUrl,                       // J: Link File
      data.firebaseDocId || '',      // K: Firebase ID
      data._timezone || 'N/A',       // L: Timezone
      fillTime                       // M: Fill Time (ms)
    ];
    sheet.getRange(nextRow, 1, 1, newRow.length).setValues([newRow]);
    Logger.log('✅ Data saved to row: ' + nextRow);

    // 3. PHÂN LOẠI VÀ GỬI EMAIL
    if (data.Noi_Dung_Tham_Du && data.Noi_Dung_Tham_Du.includes("CME")) {
      sendEmailCME(data);
    } else {
      sendEmailRegular(data);
    }

    return createResponse('success', 'Đăng ký thành công!', nextRow);
  }
  catch (error) {
    Logger.log('❌ ERROR: ' + error.toString());
    return createResponse('error', 'Lỗi hệ thống: ' + error.toString());
  }
  finally {
    lock.releaseLock();
  }
}

// Helper function để tạo response
function createResponse(status, message, row) {
  return ContentService
    .createTextOutput(JSON.stringify({ 
      'result': status, 
      'message': message,
      'row': row || null 
    }))
    .setMimeType(ContentService.MimeType.JSON);
}

// === MẪU 1: DÀNH CHO KHÁCH CME (CÓ TRẢ PHÍ) ===
function sendEmailCME(data) {
  var subject = "Xác nhận đăng ký & Biên lai: HNKH VAAE 2026";
  var htmlBody = `
    <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; border: 1px solid #e0e0e0; border-radius: 8px; overflow: hidden;">
      <div style="background-color: #0f172a; padding: 20px; text-align: center;">
        <h2 style="color: #ffffff; margin: 0;">ĐÃ TIẾP NHẬN THÔNG TIN</h2>
        <p style="color: #b45309; margin: 5px 0 0; font-weight: bold;">HỘI NGHỊ KHOA HỌC THƯỜNG NIÊN VAAE 2026</p>
      </div>
      <div style="padding: 30px; background-color: #ffffff;">
        <p>Kính gửi Quý đại biểu <strong>${data.Chuc_Vu} ${data.Ho_Ten}</strong>,</p>
        <p>Ban Tổ chức (BTC) xin trân trọng thông báo đã nhận được đơn đăng ký tham dự HNKH kèm yêu cầu cấp chứng nhận CME của Quý vị.</p>
        
        <div style="background-color: #f0fdf4; padding: 15px; border-radius: 6px; margin: 20px 0; border-left: 4px solid #15803d;">
          <h3 style="margin-top: 0; color: #15803d; font-size: 16px;">TRẠNG THÁI: ĐANG ĐỐI SOÁT THANH TOÁN</h3>
          <p style="margin-bottom: 0;">BTC đã nhận được ảnh biên lai chuyển khoản. Chúng tôi sẽ tiến hành đối soát với ngân hàng và gửi <strong>Thư xác nhận chính thức + Mã tham dự</strong> trong vòng 3-5 ngày làm việc.</p>
        </div>

        <p><strong>Thông tin đăng ký:</strong></p>
        <ul style="color: #333;">
          <li>Đơn vị: ${data.Don_Vi}</li>
          <li>Nội dung: ${data.Noi_Dung_Tham_Du}</li>
          <li>Địa chỉ nhận CME: ${data.Dia_Chi_CME}</li>
        </ul>
        
        <div style="background-color: #eff6ff; padding: 15px; border-radius: 6px; margin: 20px 0;">
          <p style="margin: 0; font-size: 14px;"><strong>📍 Địa điểm:</strong> Khách sạn Mường Thanh Cửa Đông</p>
          <p style="margin: 5px 0 0; font-size: 13px; color: #666;">Số 167, Nguyễn Phong Sắc, Trường Vinh, TP Vinh, Nghệ An</p>
        </div>
        
        <hr style="border: 0; border-top: 1px solid #eee; margin: 30px 0;">
        <p style="text-align: center; font-style: italic; color: #666;">Hẹn gặp Quý đại biểu tại Nghệ An!</p>
      </div>
    </div>
  `;
  
  try {
    MailApp.sendEmail({to: data.Email, subject: subject, htmlBody: htmlBody});
    Logger.log('✅ Email CME sent to: ' + data.Email);
  } catch (mailError) {
    Logger.log('⚠️ Email error: ' + mailError.toString());
  }
}

// === MẪU 2: DÀNH CHO KHÁCH THƯỜNG (MIỄN PHÍ) ===
function sendEmailRegular(data) {
  var subject = "Đăng ký thành công: HNKH VAAE 2026";
  var htmlBody = `
    <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; border: 1px solid #e0e0e0; border-radius: 8px; overflow: hidden;">
      <div style="background-color: #1e3a8a; padding: 20px; text-align: center;">
        <h2 style="color: #ffffff; margin: 0;">ĐĂNG KÝ THÀNH CÔNG</h2>
        <p style="color: #ffffff; margin: 5px 0 0; opacity: 0.9;">HỘI NGHỊ KHOA HỌC THƯỜNG NIÊN VAAE 2026</p>
      </div>
      <div style="padding: 30px; background-color: #ffffff;">
        <p>Kính gửi Quý đại biểu <strong>${data.Chuc_Vu} ${data.Ho_Ten}</strong>,</p>
        <p>Chúc mừng Quý vị đã đăng ký thành công tham dự Hội nghị Khoa học Thường niên VAAE 2026.</p>
        
        <div style="background-color: #eff6ff; padding: 15px; border-radius: 6px; margin: 20px 0; border-left: 4px solid #1d4ed8;">
          <h3 style="margin-top: 0; color: #1e3a8a; font-size: 16px;">Thông tin tham dự:</h3>
          <ul style="padding-left: 20px; color: #333;">
            <li><strong>Thời gian:</strong> 11 - 12/04/2026</li>
            <li><strong>Địa điểm:</strong> Khách sạn Mường Thanh Cửa Đông, Nghệ An</li>
            <li><strong>Nội dung đăng ký:</strong> ${data.Noi_Dung_Tham_Du}</li>
          </ul>
        </div>
        
        <div style="background-color: #fef3c7; padding: 15px; border-radius: 6px; margin: 20px 0;">
          <p style="margin: 0; font-size: 14px;"><strong>📍 Địa chỉ:</strong> Số 167, Nguyễn Phong Sắc, Trường Vinh, TP Vinh, Nghệ An</p>
          <p style="margin: 5px 0 0; font-size: 13px;"><a href="https://maps.google.com/?q=18.685953982366772,105.69708621039022" style="color: #1d4ed8;">Xem bản đồ →</a></p>
        </div>

        <p>Mã QR Check-in sẽ được gửi tới email này gần ngày diễn ra sự kiện. Quý vị vui lòng theo dõi email thường xuyên.</p>
        <hr style="border: 0; border-top: 1px solid #eee; margin: 30px 0;">
        <p style="text-align: center; font-style: italic; color: #666;">Trân trọng đón tiếp!</p>
      </div>
    </div>
  `;
  
  try {
    MailApp.sendEmail({to: data.Email, subject: subject, htmlBody: htmlBody});
    Logger.log('✅ Email Regular sent to: ' + data.Email);
  } catch (mailError) {
    Logger.log('⚠️ Email error: ' + mailError.toString());
  }
}
```

---

## 📊 CẬP NHẬT HEADER SHEET

Đảm bảo **Trang tính1** có các cột header sau (dòng 1):

| A | B | C | D | E | F | G | H | I | J | K | L | M |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Timestamp | Họ Tên | Ngày Sinh | Email | SĐT | Đơn Vị | Chức Vụ | Nội Dung | Địa Chỉ CME | Link File | Firebase ID | Timezone | Fill Time |

---

## 🛡️ TÓM TẮT BẢO MẬT ĐÃ THÊM

| # | Tính năng | Mô tả |
|---|-----------|-------|
| 1 | **Rate Limiting** | Max 100 request/giờ toàn hệ thống |
| 2 | **Timestamp Validation** | Request chỉ hợp lệ trong 5 phút |
| 3 | **Fill Time Check** | Form phải điền ≥5 giây |
| 4 | **Email Validation** | Kiểm tra format email |
| 5 | **Duplicate Check** | Email trùng trong 1h bị từ chối |
| 6 | **Error Logging** | Log đầy đủ để monitor |

---

## 🔍 CÁCH XEM LOG

1. Mở Apps Script Editor
2. Menu: **View → Logs** (hoặc **Ctrl+Enter**)
3. Xem các cảnh báo `⚠️` để phát hiện spam/bot

---

## ✅ CHECKLIST SAU KHI UPDATE

- [ ] Copy code mới vào Apps Script
- [ ] Save (Ctrl+S)
- [ ] Deploy lại (New Version)
- [ ] Test submit 1 form
- [ ] Kiểm tra Sheet có nhận data
- [ ] Kiểm tra Email có gửi được
- [ ] Test spam 2 lần liên tục → Lần 2 phải bị chặn
