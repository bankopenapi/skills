# BankOpenAPI Agent Skills

Skill cho AI agent (Claude Code, Cursor, Codex, GitHub Copilot…) để tích hợp [BankOpenAPI](https://bankopenapi.com): nhận tiền chuyển khoản vào tài khoản ngân hàng Việt Nam, hiện mã VietQR, nhận webhook khi tiền về, đơn tự sang đã thanh toán.

Agent skill for integrating BankOpenAPI: bank-transfer payments, VietQR checkout and payment webhooks in Vietnam.

## Cài đặt

```bash
npx skills add bankopenapi/skills
```

Hoặc lấy thẳng từ website (Agent Skills Discovery):

```bash
npx skills add https://bankopenapi.com
```

Cài tay cho Claude Code:

```bash
mkdir -p .claude/skills/bankopenapi
curl -fsSL https://bankopenapi.com/skills/bankopenapi/SKILL.md -o .claude/skills/bankopenapi/SKILL.md
```

## Skill

| Skill | Dùng khi |
| --- | --- |
| [`bankopenapi`](skills/bankopenapi/SKILL.md) | Thu tiền chuyển khoản cho đơn hàng: tạo yêu cầu thanh toán, hiện VietQR, nhận và verify webhook `payment_request.paid`, chống xử lý trùng, thử bằng Sandbox |

Ví dụ nhờ agent sau khi cài:

- "Tích hợp thanh toán chuyển khoản BankOpenAPI vào trang checkout: tạo yêu cầu thanh toán khi đặt đơn, hiện mã VietQR, nhận webhook để chuyển đơn sang đã thanh toán."
- "Viết endpoint /webhooks/bankopenapi cho app Next.js này, verify chữ ký và chống xử lý trùng bằng Prisma."

## MCP server

Cho agent gọi thẳng BankOpenAPI bằng API key của bạn (tạo yêu cầu thanh toán, xem giao dịch, số dư):

```bash
claude mcp add --transport http bankopenapi https://app.bankopenapi.com/api/mcp \
  --header "Authorization: Bearer $BANKOPENAPI_API_KEY"
```

Cấu hình cho Cursor và client khác: https://bankopenapi.com/tai-lieu/ai-agent

## Tài liệu

- Tài liệu API: https://bankopenapi.com/tai-lieu
- Sandbox (API key `boa_test_…`, không cần tài khoản ngân hàng thật): https://bankopenapi.com/tai-lieu/sandbox
- Đăng ký: https://app.bankopenapi.com/dang-ky

Nội dung skill được đồng bộ từ bankopenapi.com. Góp ý, báo lỗi: mở issue hoặc gửi email tới teams@bankopenapi.com.

## License

MIT
