# DEMO12 - Popup bac si va Admin Mode (Tong ket)

## 1) Muc tieu da trien khai trong demo12

- Nguoi dung click vao ten/anh bac si de xem popup thong tin ngan gon.
- Khong mo tab/link moi de xem ho so.
- Khong keo dai noi dung the bac si o phan danh sach.
- Admin co luong rieng de cap nhat ho so tu CV PDF.

## 2) Kien truc popup hien tai

### A. Popup cho nguoi dung (viewer)

- ID: `doctor-info-modal`
- Muc dich: chi xem thong tin bac si.
- Truong hien thi:
  - Ho ten
  - Ngay sinh
  - Don vi cong tac
  - Qua trinh cong tac
- Neu chua co du lieu:
  - Hien thi: `Bac si chua cap nhat thong tin. Ban vui long cho thong tin tu Ban To Chuc.`

### B. Popup cho quan tri (editor)

- ID: `doctor-admin-modal`
- Muc dich: cap nhat ho so tu CV PDF hoac nhap tay.
- Truong quan tri:
  - Ho ten
  - Ngay sinh
  - Don vi cong tac
  - Qua trinh cong tac

## 3) Admin Mode

### Cach mo

- Phim tat: `Ctrl + Shift + A`
- Hoac bam nut noi `Admin` o goc phai duoi.
- Yeu cau mat khau quan tri.

### Bao ve co ban

- Sai mat khau 5 lan -> khoa 10 phut.
- Trang thai mo admin luu theo session trinh duyet.

## 4) Xu ly CV PDF

- Dung `pdf.js` de doc text tu file PDF ngay tren trinh duyet.
- Chi nhat cac truong can thiet:
  - Ho ten
  - Ngay sinh
  - Don vi cong tac
  - Qua trinh cong tac
- Loc bo du lieu nhay cam neu phat hien dong chua:
  - CCCD/CMND
  - que quan/nguyen quan
  - ho khau/thuong tru
  - passport
  - ma so thue

## 5) Du lieu CV hien di ve dau?

- File PDF **khong upload len server** trong demo12 hien tai.
- Du lieu sau khi nhat duoc luu o **localStorage cua trinh duyet**:
  - Key chinh: `demo12_speaker_profiles_v1`
- Dieu nay co nghia:
  - Du lieu chi ton tai tren thiet bi/trinh duyet dang dung.
  - May khac hoac trinh duyet khac se khong thay du lieu do.
  - Xoa du lieu trinh duyet co the mat du lieu da luu cuc bo.

## 6) Cac key local/session dang dung

- localStorage:
  - `demo12_speaker_profiles_v1`
  - `demo12_admin_attempts`
  - `demo12_admin_lock_until`
- sessionStorage:
  - `demo12_admin_unlocked`

## 7) Gioi han hien tai va khuyen nghi

- Day la co che frontend-only, phu hop giai doan demo/trien khai nhanh.
- De production an toan hon:
  - Chuyen luu ho so sang backend (DB).
  - Co tai khoan admin that (JWT/session server).
  - Log lich su chinh sua.
  - Phan quyen ro ai duoc sua.

## 8) File da chinh

- `demo12.html`
- `DEMO12_POPUP_ADMIN_NOTES.md`
