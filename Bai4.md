# Bài 4: Mô phỏng quy trình Hotfix & Gitflow thực tế

## Mục tiêu

Thực hành mô hình phân nhánh **Gitflow** để xử lý tình huống khẩn cấp: sửa lỗi bảo mật nghiêm trọng trên production (`main`) trong khi `develop` đang phát triển dở dang, sau đó đồng bộ bản vá về cả hai nhánh.

---

## Bối cảnh

- Hệ thống đang chạy trên nhánh `main` phiên bản **v1.0.0** (stable).
- Phát hiện lỗi **lộ dữ liệu người dùng** trong hàm `get_user_data()` — trả về toàn bộ dữ liệu thay vì chỉ dữ liệu của user cụ thể.
- Nhánh `develop` đang phát triển tính năng thanh toán cho v2.0.0, **chưa thể deploy**.
- Yêu cầu: vá lỗi ngay lập tức mà không ảnh hưởng đến tiến độ develop.

---

## Sơ đồ Gitflow áp dụng

```
main     ──●──────────────────────────────●── (tag: v1.0.1)
           │ v1.0.0                      /|
           │                   [merge] / |
           │                          /  |
hotfix  ───┼──────────────────────●──╯   │
           │              fix bug  \     │
           │                        \   │ [merge]
develop ───┼────────●────────────────●──╯
                   WIP payment     (hotfix backport)
```

---

## Các bước thực hiện

### Bước 1: Khởi tạo repo và tạo nhánh main (v1.0.0)

Tạo repository, thêm code production `app.py` có chứa lỗi bảo mật:

```python
# app.py - Version 1.0.0 (có bug)
def get_user_data(user_id):
    # BUG: trả về toàn bộ dữ liệu người dùng
    return {"all_users": "sensitive_data_exposed"}
```

```bash
git init
git add .
git commit -m "Initial release v1.0.0"
git branch -m master main
git tag -a v1.0.0 -m "Release v1.0.0"
```

**Kết quả:**
```
[main (root-commit) c83663d] Initial release v1.0.0
 1 file changed, 10 insertions(+)
```

---

### Bước 2: Tạo nhánh develop và thêm tính năng đang phát triển

```bash
git checkout -b develop
```

Thêm file `feature_payment.py` (tính năng đang làm dở):

```python
# feature_payment.py - WIP
def process_payment(amount, user_id):
    # TODO: integrate payment gateway
    pass
```

```bash
git add .
git commit -m "WIP: Add payment feature (v2.0.0 dev)"
```

**Kết quả:**
```
[develop c480f6d] WIP: Add payment feature (v2.0.0 dev)
```

> Tại thời điểm này `develop` có code chưa hoàn thiện, **không thể deploy ngay lên production**.

---

### Bước 3: Tạo nhánh hotfix/v1.0.1 từ main

Phải tách nhánh hotfix **trực tiếp từ `main`** (không từ `develop`) để đảm bảo chỉ vá đúng lỗi, không kéo theo code phát triển dở dang:

```bash
git checkout main
git checkout -b hotfix/v1.0.1
```

**Kết quả:**
```
Switched to branch 'main'
Switched to a new branch 'hotfix/v1.0.1'
```

---

### Bước 4: Sửa lỗi trên nhánh hotfix

Sửa hàm `get_user_data()` trong `app.py` — chỉ trả về dữ liệu của đúng user được yêu cầu:

```python
# app.py - Version 1.0.1 (đã vá lỗi)
def get_user_data(user_id):
    # FIXED: chỉ trả về dữ liệu của user cụ thể
    db = {"u1": {"name": "Alice"}, "u2": {"name": "Bob"}}
    return db.get(user_id, {})
```

```bash
git add .
git commit -m "Hotfix: fix user data exposure security bug"
```

**Kết quả:**
```
[hotfix/v1.0.1 9953fe5] Hotfix: fix user data exposure security bug
 1 file changed, 5 insertions(+), 4 deletions(-)
```

---

### Bước 5: Merge hotfix vào main và tạo tag v1.0.1

```bash
git checkout main
git merge --no-ff hotfix/v1.0.1 -m "Merge hotfix/v1.0.1 into main"
git tag -a v1.0.1 -m "Release Hotfix 1.0.1"
```

**Kết quả:**
```
Switched to branch 'main'
Merge made by the 'ort' strategy.
 app.py | 9 +++++----
 1 file changed, 5 insertions(+), 4 deletions(-)
```

> Dùng `--no-ff` để tạo merge commit rõ ràng, giữ nguyên lịch sử nhánh hotfix trong đồ thị commit.

---

### Bước 6: Merge hotfix vào develop (backport)

Bắt buộc phải merge ngược hotfix về `develop` để code tương lai cũng được vá lỗi này:

```bash
git checkout develop
git merge --no-ff hotfix/v1.0.1 -m "Merge hotfix/v1.0.1 into develop"
```

**Kết quả:**
```
Switched to branch 'develop'
Merge made by the 'ort' strategy.
 app.py | 9 +++++----
 1 file changed, 5 insertions(+), 4 deletions(-)
```

---

### Bước 7: Xóa nhánh hotfix sau khi hoàn tất

```bash
git branch -d hotfix/v1.0.1
```

**Kết quả:**
```
Deleted branch hotfix/v1.0.1 (was 9953fe5).
```

---

## Kiểm tra kết quả

### Danh sách nhánh hiện có

```bash
git branch -a
```

```
* develop
  main
```

### Danh sách tag

```bash
git tag
```

```
v1.0.0
v1.0.1
```

### Đồ thị lịch sử commit

```bash
git log --graph --oneline --all
```

```
*   76e34da (HEAD -> develop) Merge hotfix/v1.0.1 into develop
|\  
* | c480f6d WIP: Add payment feature (v2.0.0 dev)
| | * 1c9567f (tag: v1.0.1, main) Merge hotfix/v1.0.1 into main
| |/| 
|/|/  
| * 9953fe5 Hotfix: fix user data exposure security bug
|/  
* c83663d (tag: v1.0.0) Initial release v1.0.0
```

---

## Sơ đồ ASCII đầy đủ

```
commit    nhánh        mô tả
─────────────────────────────────────────────────────────
c83663d   main/develop  Initial release v1.0.0  [tag: v1.0.0]
    │
    ├──── develop ──── c480f6d  WIP: Add payment feature
    │
    └──── hotfix/v1.0.1
               │
               9953fe5  Hotfix: fix user data exposure security bug
               │
    ┌──────────┘ merge --no-ff
    │
1c9567f   main          Merge hotfix/v1.0.1 into main  [tag: v1.0.1]

    ┌──────────────────┘ merge --no-ff (backport)
    │
76e34da   develop       Merge hotfix/v1.0.1 into develop
```

---

## Tổng kết các lệnh

| Bước | Lệnh | Mục đích |
|------|------|----------|
| 1 | `git tag -a v1.0.0 -m "..."` | Đánh dấu phiên bản stable |
| 2 | `git checkout -b develop` | Tạo nhánh phát triển |
| 3 | `git checkout main` + `git checkout -b hotfix/v1.0.1` | Tách hotfix từ main |
| 4 | `git commit -m "Hotfix: ..."` | Commit bản vá |
| 5 | `git merge --no-ff hotfix/v1.0.1` (trên main) | Tích hợp vá lỗi vào production |
| 6 | `git tag -a v1.0.1 -m "..."` | Đánh dấu phiên bản mới |
| 7 | `git merge --no-ff hotfix/v1.0.1` (trên develop) | Backport vá lỗi về develop |
| 8 | `git branch -d hotfix/v1.0.1` | Dọn dẹp nhánh hotfix |

## Bài học rút ra

- **Gitflow** giúp tách biệt rõ ràng code production (`main`) và code phát triển (`develop`).
- Nhánh `hotfix/` **luôn phải tách từ `main`**, không phải từ `develop`, để tránh đưa code chưa hoàn thiện lên production.
- Sau khi merge hotfix vào `main`, **bắt buộc backport về `develop`** để tránh tình trạng lỗi tái xuất hiện ở phiên bản tương lai.
- Dùng `--no-ff` khi merge để giữ lại lịch sử nhánh rõ ràng trong đồ thị commit.
- Tag phiên bản (`v1.0.1`) giúp đánh dấu chính xác điểm release trên nhánh `main`.
