# GitHub Weekly Digest — Routine Instructions

## Mục đích
Mỗi thứ 7 lúc 9h sáng, tự động:
1. Lấy top 5 GitHub repos trending trong tuần (theo stars mới nhận)
2. Tóm tắt từng repo bằng tiếng Việt
3. Gửi thông báo qua LINE push message
4. Ghi log vào Google Sheets

---

## Environment Variables (cần setup trong Routine)
- `GITHUB_TOKEN` — GitHub Personal Access Token (scope: public_repo, read:user)
- `LINE_CHANNEL_ACCESS_TOKEN` — LINE Official Account Channel Access Token (long-lived)
- `LINE_USER_ID` — LINE User ID của người nhận
- `GOOGLE_SHEET_ID` — ID của Google Spreadsheet (lấy từ URL)

---

## Flow thực thi

### Bước 1 — Fetch GitHub trending repos
Gọi GitHub Search API:
```
GET https://api.github.com/search/repositories
  ?q=created:>{DATE_7_DAYS_AGO}
  &sort=stars
  &order=desc
  &per_page=5
```
- `{DATE_7_DAYS_AGO}` = ngày hôm nay trừ 7 ngày, format `YYYY-MM-DD`
- Header: `Authorization: Bearer {GITHUB_TOKEN}`
- Lấy fields: `full_name`, `html_url`, `description`, `stargazers_count`, `language`, `topics`

### Bước 2 — Tóm tắt bằng AI
Với mỗi repo, tạo tóm tắt tiếng Việt ngắn gọn (2-3 câu) dựa trên:
- `description` của repo
- `topics` / `language`
- Tên repo (thường gợi ý chức năng)

Nếu `description` trống → ghi "Không có mô tả" và chỉ nêu ngôn ngữ + topics.

### Bước 3 — Gửi LINE push message
Gọi LINE Messaging API:
```
POST https://api.line.me/v2/bot/message/push
Header:
  Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}
  Content-Type: application/json
Body:
{
  "to": "{LINE_USER_ID}",
  "messages": [{
    "type": "text",
    "text": "{MESSAGE}"
  }]
}
```

Format `{MESSAGE}`:
```
🔥 GitHub Trending — Tuần {DD/MM} - {DD/MM}

1. {repo_name} ⭐ {stars}
{summary_vi}
🔗 {url}

2. ...
```

**Nếu LINE call thất bại (status != 200):**
→ Ghi log lỗi, chuyển sang bước fallback Gmail (xem bên dưới)

### Bước 4 — Ghi log Google Sheets
Ghi vào sheet `history`, mỗi repo 1 row:
| run_date | repo_name | url | stars | language | summary_vi | sent_via |
- `run_date` = ngày chạy (YYYY-MM-DD)
- `sent_via` = `LINE` hoặc `Gmail` hoặc `FAILED`

---

## Fallback — Gmail
Nếu LINE thất bại, gửi email tóm tắt tương tự qua Gmail connector.
Subject: `[GitHub Digest] Tuần {DD/MM - DD/MM}`

---

## Sheet `config` — Cấu trúc (dùng cho scale tương lai)
| key | value | description |
|-----|-------|-------------|
| language_filter | | Lọc theo ngôn ngữ, để trống = tất cả |
| topic_filter | | Lọc theo topic, để trống = tất cả |
| top_n | 5 | Số lượng repo lấy mỗi tuần |
| notify_channel | LINE | LINE hoặc Gmail |

Khi chạy, đọc config từ sheet này trước khi thực thi.

---

## Lưu ý
- Không hardcode token vào file. Luôn đọc từ environment variables.
- Nếu GitHub API trả về lỗi rate limit (403) → log lỗi, dừng, không gửi thông báo.
- Routine này **không cần tạo file hay commit** vào repo. Repo chỉ là workspace.
