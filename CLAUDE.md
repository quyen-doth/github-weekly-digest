# GitHub Weekly Digest — Routine Instructions

## Mục đích
Mỗi ngay lúc 9h sáng, tự động:
1. Lấy top 5 GitHub repos trending trong tuần (theo stars mới nhận)
2. Tóm tắt từng repo bằng tiếng Việt
3. Gửi thông báo qua Gmail
4. Ghi log vào Google Sheets

---

## Cấu hình cố định
- `GOOGLE_SHEET_ID` = 1ygmm-A3L6sw-vQE6cCu3MNY6yeEJPy9b9i79zPqmyl8

---

## Flow thực thi

### Bước 1 — Đọc config
Mở Google Sheet ID: 1ygmm-A3L6sw-vQE6cCu3MNY6yeEJPy9b9i79zPqmyl8
- Sheet "config": đọc top_n (mặc định 5), language_filter, topic_filter

### Bước 2 — Fetch GitHub trending repos
Gọi API:
```
GET https://api.github.com/search/repositories
  ?q=created:>{NGÀY_HÔM_NAY_TRỪ_7_NGÀY}
  &sort=stars
  &order=desc
  &per_page={top_n}
```
- Header: `Authorization: Bearer {GITHUB_TOKEN từ Instructions}`
- Lấy fields: `full_name`, `html_url`, `description`, `stargazers_count`, `language`, `topics`
- Nếu API trả về 403 (rate limit) → log lỗi, dừng, không gửi email

### Bước 3 — Tóm tắt tiếng Việt
Với mỗi repo, viết 2-3 câu tiếng Việt về chức năng và điểm nổi bật dựa trên description, language, topics.
Nếu description trống → dựa vào tên repo và topics.

### Bước 4 — Gửi Gmail
Dùng Gmail connector gửi email tới {GMAIL từ Instructions} với:
- Subject: `[GitHub Digest] Tuần {DD/MM} - {DD/MM}`
- Body format:

```
🔥 GitHub Trending — Tuần {DD/MM} - {DD/MM}

1. {repo_name} ⭐ {stars}
{tóm tắt tiếng Việt}
🔗 {url}

2. ...
```

### Bước 5 — Ghi log Google Sheets
Dùng Google Drive connector append các rows vào sheet "history" của spreadsheet ID: 1ygmm-A3L6sw-vQE6cCu3MNY6yeEJPy9b9i79zPqmyl8
Mỗi repo 1 row theo thứ tự cột:
run_date | week_range | rank | repo_name | url | stars_gained | language | topics | summary_vi | sent_via | notes

- `sent_via` = "Gmail" nếu thành công, "FAILED" nếu lỗi
- `run_date` = ngày chạy format YYYY-MM-DD

---

## Lưu ý
- Không tạo file mới trong repo
- Không hỏi xác nhận
- Thực thi thẳng từng bước và báo cáo kết quả cuối
