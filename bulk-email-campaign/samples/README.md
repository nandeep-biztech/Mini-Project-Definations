# Sample CSV Files

Sample CSV files for testing the Bulk Email Campaign System. All files conform to spec:

- Column names in snake_case (no spaces)
- Required `email` column present
- Valid email format
- Valid data types (dates as YYYY-MM-DD, amounts as numeric)

## Files

| File | Purpose | Placeholders |
|------|---------|--------------|
| `contacts_basic.csv` | Simple contact list | `{first_name}`, `{last_name}` |
| `meeting_reminder.csv` | Meeting reminders | `{first_name}`, `{meeting_date}`, `{meeting_time}` |
| `campaign_newsletter.csv` | Newsletter with company/amount | `{first_name}`, `{last_name}`, `{company}`, `{amount}` |

## Example Template

For `contacts_basic.csv`:

```
Hello {first_name},

Your campaign is ready. We hope this finds you well.

Best regards,
Team
```
