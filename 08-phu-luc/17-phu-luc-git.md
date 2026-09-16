# Phụ Lục C — Git cho Người Dùng Python

## C.1 Cấu hình lần đầu

```bash
git config --global user.name "Tên Bạn"
git config --global user.email "ban@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
```

## C.2 Tạo repo

```bash
git init
git add .
git commit -m "Khởi tạo dự án"
```

### .gitignore cho Python

```gitignore
__pycache__/
*.py[cod]
*.egg-info/
.eggs/
build/
dist/
.venv/
venv/
.env
.pytest_cache/
.mypy_cache/
.ruff_cache/
htmlcov/
.coverage
*.db
*.sqlite3
.DS_Store
.idea/
.vscode/
```

## C.3 Quy trình cơ bản

```bash
git status
git diff
git add file.py
git add -p            # thêm từng phần
git commit -m "Thêm tính năng X"
git log --oneline
```

### Commit tốt

- Một commit một thay đổi logic.
- Thông điệp rõ: `Thêm validation cho form đăng ký`.
- Không commit file build, `.env`, `__pycache__`.

### Sửa commit cuối

```bash
git commit --amend -m "Thông điệp mới"
git add file_quen.py
git commit --amend --no-edit
```

## C.4 Branch

```bash
git branch                    # liệt kê
git branch feature-x          # tạo
git checkout feature-x        # chuyển
git switch feature-x          # 3.23+ hiện đại
git checkout -b feature-y     # tạo + chuyển
git merge feature-x           # gộp vào branch hiện tại
git branch -d feature-x       # xóa
```

### Merge conflict

```text
<<<<<<< HEAD
code branch hiện tại
=======
code branch được merge
>>>>>>> feature-x
```

Sửa thủ công, `git add`, rồi `git commit`.

## C.5 Remote

```bash
git remote add origin https://github.com/user/repo.git
git remote -v
git push -u origin main
git pull
git fetch
```

### Clone

```bash
git clone https://github.com/user/repo.git
git clone --depth 1 https://github.com/user/repo.git   # nông
```

## C.6 Hoàn tác

```bash
git restore file.py           # bỏ thay đổi chưa staged
git restore --staged file.py  # bỏ khỏi staging
git reset HEAD~1              # bỏ commit cuối, giữ thay đổi
git reset --hard HEAD~1       # bỏ commit + thay đổi (NGUY HIỂM)
git revert <hash>             # tạo commit đảo ngược
git clean -fd                 # xóa file untracked (NGUY HIỂM)
```

`reset --hard` và `clean -fd` mất dữ liệu vĩnh viễn — cẩn thận.

## C.7 Stash

```bash
git stash                     # cất thay đổi
git stash list
git stash pop                 # lấy lại + xóa stash
git stash apply               # lấy lại, giữ stash
git stash drop stash@{0}
```

Hữu ích khi cần chuyển branch gấp.

## C.8 Rebase

```bash
git switch feature
git rebase main
```

Áp commit của feature lên đầu main, lịch sử tuyến tính.

### Interactive rebase

```bash
git rebase -i HEAD~3
```

```
pick abc123 Commit 1
squash def456 Commit 2
reword ghi789 Commit 3
```

- `pick` — giữ.
- `reword` — sửa thông điệp.
- `squash` — gộp vào commit trên.
- `drop` — xóa.
- `edit` — dừng để sửa.

**Không rebase commit đã push** lên branch chung — viết lại lịch sử gây rối cho người khác.

## C.9 Tag và release

```bash
git tag v1.0.0
git tag -a v1.0.0 -m "Release 1.0"
git push origin v1.0.0
git push --tags
```

## C.10 Log và tìm kiếm

```bash
git log --oneline --graph --all
git log --author="Nam"
git log -p file.py
git log -S "hàm_bí_ẩn"        # tìm commit thêm/xóa chuỗi
git blame file.py             # ai sửa dòng nào
git show <hash>
```

## C.11 Bisect — tìm commit gây lỗi

```bash
git bisect start
git bisect bad                # commit hiện tại lỗi
git bisect good v1.0.0        # commit cũ OK
# git checkout tự động giữa khoảng
# test, rồi:
git bisect good   # hoặc
git bisect bad
# lặp tới khi tìm ra commit đầu tiên gây lỗi
git bisect reset
```

## C.12 Submodule và subtree

```bash
git submodule add https://github.com/user/lib.git libs/lib
git submodule update --init --recursive

git subtree add --prefix=libs/lib https://github.com/user/lib.git main --squash
```

Submodule giữ repo riêng; subtree nhúng vào lịch sử.

## C.13 Hook

`.git/hooks/pre-commit`:

```bash
#!/bin/sh
ruff check . || exit 1
pytest -q || exit 1
```

```bash
chmod +x .git/hooks/pre-commit
```

Hoặc dùng `pre-commit` framework (khuyến nghị).

## C.14 Quy trình làm việc nhóm

### Feature branch

```bash
git switch main
git pull
git switch -c feature/login
# code, commit
git push -u origin feature/login
# mở Pull Request
```

### Pull Request tốt

- Nhỏ, tập trung một vấn đề.
- Có mô tả, test, screenshot nếu UI.
- CI xanh trước khi review.
- Tự review trước khi gửi.

### Code review

- Đọc kỹ diff.
- Comment cụ thể, không chung chung.
- Đề xuất, không ra lệnh.
- Approve khi đủ tốt, không cần hoàn hảo.

## C.15 Git với Python project

```bash
# Tạo cấu trúc
mkdir myproject && cd myproject
git init
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

Commit đầu:

```bash
cat > .gitignore << 'EOF'
__pycache__/
.venv/
*.egg-info/
.pytest_cache/
.mypy_cache/
.ruff_cache/
EOF

git add .gitignore pyproject.toml src/ tests/ README.md
git commit -m "Khởi tạo dự án"
```

## C.16 Mẹo

- `git config --global alias.st status`
- `git config --global alias.lg "log --oneline --graph --all"`
- `git config --global pull.rebase true` — pull dùng rebase.
- `.gitattributes` chuẩn hóa line ending:

```text
* text=auto eol=lf
*.png binary
```

## C.17 Bài tập

1. Tạo repo, commit đầu tiên với `.gitignore` chuẩn Python.
2. Tạo branch, thêm tính năng, merge vào main.
3. Tạo conflict cố ý, giải quyết.
4. Dùng `git bisect` tìm commit gây lỗi trong một repo mẫu.
5. Viết pre-commit hook chạy ruff.
6. Tạo PR trên GitHub, tự review.

Stashed.