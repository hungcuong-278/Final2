# 🛡️ HƯỚNG DẪN BẢO MẬT GOOGLE APPS SCRIPT

## CÁC LỚP BẢO MẬT ĐÃ TRIỂN KHAI

### ✅ Frontend (demo10.html) - ĐÃ XONG
1. **Honeypot Field** - Bắt bot tự động
2. **Rate Limiting** - Chỉ cho submit 1 lần/phút
3. **Minimum Fill Time** - Phải mất ít nhất 5 giây để điền form
4. **Metadata Tracking** - Ghi lại timestamp, timezone, fillTime
5. **LocalStorage Tracking** - Lưu lịch sử submit

---

## 🔧 CẬP NHẬT APPS SCRIPT (QUAN TRỌNG!)

Vào: https://script.google.com/home
Tìm project: **VAAE Conference Registration**
Thay code cũ bằng code mới bên dưới:

```javascript
// =====================================================
// GOOGLE APPS SCRIPT - VAAE CONFERENCE 2026
// Version: 2.0 with Security
// =====================================================

const SHEET_NAME = 'Registrations';
const MAX_REQUESTS_PER_HOUR = 100; // Giới hạn 100 đăng ký/giờ
const MIN_FILL_TIME = 5000; // 5 giây tối thiểu

// Cache để track rate limiting
const cache = CacheService.getScriptCache();

function doPost(e) {
  try {
    // Parse request
    const data = JSON.parse(e.postData.contents);
    
    // =====================================================
    // BẢO MẬT 1: Rate Limiting per IP
    // =====================================================
    const clientIP = e.parameter.userip || 'unknown';
    const cacheKey = `ip_${clientIP}`;
    const requestCount = parseInt(cache.get(cacheKey) || '0');
    
    if (requestCount >= MAX_REQUESTS_PER_HOUR) {
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: 'Too many requests. Please try again later.'
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    // Increment counter (expires after 1 hour)
    cache.put(cacheKey, (requestCount + 1).toString(), 3600);
    
    // =====================================================
    // BẢO MẬT 2: Validate Timestamp (không nhận request cũ)
    // =====================================================
    const timestamp = data._timestamp || 0;
    const now = Date.now();
    const age = now - timestamp;
    
    if (age < 0 || age > 300000) { // Request phải trong vòng 5 phút
      Logger.log('⚠️ Invalid timestamp:', { timestamp, now, age });
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: 'Invalid request timestamp'
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    // =====================================================
    // BẢO MẬT 3: Validate Fill Time
    // =====================================================
    if (data._fillTime && data._fillTime < MIN_FILL_TIME) {
      Logger.log('⚠️ Form filled too quickly:', data._fillTime);
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: 'Please fill the form carefully'
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    // =====================================================
    // BẢO MẬT 4: Validate Email Format
    // =====================================================
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(data.Email)) {
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: 'Invalid email format'
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    // =====================================================
    // BẢO MẬT 5: Check duplicate email (trong 1 giờ gần nhất)
    // =====================================================
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
    const lastRow = sheet.getLastRow();
    
    if (lastRow > 1) {
      const recentEmails = sheet.getRange(Math.max(2, lastRow - 100), 5, Math.min(100, lastRow - 1), 1).getValues();
      const recentTimestamps = sheet.getRange(Math.max(2, lastRow - 100), 1, Math.min(100, lastRow - 1), 1).getValues();
      
      for (let i = 0; i < recentEmails.length; i++) {
        if (recentEmails[i][0] === data.Email) {
          const timeDiff = now - new Date(recentTimestamps[i][0]).getTime();
          if (timeDiff < 3600000) { // 1 giờ
            Logger.log('⚠️ Duplicate email within 1 hour:', data.Email);
            return ContentService.createTextOutput(JSON.stringify({
              status: 'error',
              message: 'Email đã được đăng ký trong vòng 1 giờ qua'
            })).setMimeType(ContentService.MimeType.JSON);
          }
        }
      }
    }
    
    // =====================================================
    // LƯU DỮ LIỆU VÀO SHEET
    // =====================================================
    const row = [
      new Date(data.Created_At),
      data.Ho_Ten,
      data.Ngay_Sinh,
      data.Chuc_Vu,
      data.Email,
      data.So_Dien_Thoai,
      data.Don_Vi,
      data.Noi_Dung_Tham_Du,
      data.Dia_Chi_CME || '',
      data.Status || 'Pending',
      data.firebaseDocId || '',
      clientIP,
      data._timezone || 'unknown'
    ];
    
    sheet.appendRow(row);
    
    // Upload file nếu có (từ fallback khi Storage không enable)
    if (data.fileData && data.fileName) {
      const blob = Utilities.newBlob(
        Utilities.base64Decode(data.fileData.split(',')[1]),
        data.mimeType,
        data.fileName
      );
      
      const folder = DriveApp.getFolderById('YOUR_FOLDER_ID'); // Thay bằng folder ID thực
      const file = folder.createFile(blob);
      
      // Cập nhật link file vào sheet
      sheet.getRange(sheet.getLastRow(), 14).setValue(file.getUrl());
    }
    
    Logger.log('✅ Registration saved:', data.Email);
    
    return ContentService.createTextOutput(JSON.stringify({
      status: 'success',
      message: 'Đăng ký thành công!'
    })).setMimeType(ContentService.MimeType.JSON);
    
  } catch (error) {
    Logger.log('❌ Error:', error);
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: 'Internal server error'
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

---

## 📊 HEADER SHEET CẦN CÓ

Đảm bảo Sheet "Registrations" có các cột sau:

| A | B | C | D | E | F | G | H | I | J | K | L | M | N |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Timestamp | Họ Tên | Ngày Sinh | Chức Vụ | Email | SĐT | Đơn Vị | Nội Dung | Địa Chỉ CME | Status | Firebase ID | IP | Timezone | File URL |

---

## 🎯 KẾT QUẢ SAU KHI SETUP

- ✅ Bot tự động bị chặn (honeypot)
- ✅ Spam bị giới hạn (1 lần/phút/user, 100 lần/giờ/server)
- ✅ Request cũ/fake bị loại bỏ (timestamp validation)
- ✅ Email trùng trong 1h bị từ chối
- ✅ Form điền quá nhanh bị chặn
- ✅ Metadata đầy đủ để audit

---

## 🔍 MONITORING

Để xem log:
1. Vào Apps Script Editor
2. Menu: View → Logs (hoặc Ctrl+Enter)
3. Xem các cảnh báo ⚠️ để phát hiện tấn công
