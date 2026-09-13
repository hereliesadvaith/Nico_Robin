# Todo schema (Directus)

## `todos`

One row per task. Created and completed by the assistant over the Directus MCP.

| Field          | Type      | Notes                                                        |
| -------------- | --------- | ------------------------------------------------------------ |
| `id`           | integer   | primary key, auto-increment                                  |
| `title`        | string    | required, max 255 chars                                      |
| `status`       | string    | dropdown: `open`, `done`; defaults to `open`                 |
| `priority`     | string    | dropdown: `low`, `normal`, `high`; defaults to `normal`      |
| `due_date`     | date      | optional, date only. The 6 AM digest uses this.              |
| `completed_at` | timestamp | optional, set when `status` becomes `done`                   |
| `notes`        | text      | optional                                                     |
| `date_created` | timestamp | system, hidden                                               |
| `date_updated` | timestamp | system, hidden                                               |

`status` is the archive field: Data Studio hides `done` rows by default.
Default sort when listing: `due_date` ascending, then `priority`.

## `daily_digests`

One row per day, written by the n8n workflow **Todo** at 06:00 Asia/Kolkata.
The assistant reads it; it should not write to it.

| Field          | Type      | Notes                                                        |
| -------------- | --------- | ------------------------------------------------------------ |
| `id`           | integer   | primary key, auto-increment                                  |
| `date`         | date      | required, unique. The day the digest is for.                 |
| `todo_count`   | integer   | number of open todos due on or before `date`                 |
| `body`         | text      | plain-text list, ready to show as-is                         |
| `items`        | json      | array of `{id, title, priority, due_date, overdue}`          |
| `date_created` | timestamp | system, hidden                                               |
| `date_updated` | timestamp | system, hidden                                               |

## n8n workflow `Todo`

Schedule: cron `0 6 * * *` in Asia/Kolkata. Steps:

1. `GET /items/todos` filtered to `status = open` and `due_date <= today`,
   using the internal address `http://directus:8055` and the n8n credential
   **Directus account**.
2. Sort high > normal > low, then by `due_date`; mark rows with
   `due_date < today` as overdue; build `body` and `items`.
3. Look up `daily_digests` for today's `date`. Update the row if it exists,
   otherwise create it. Running the workflow twice in a day never creates a
   second row.

A second trigger, a webhook at `POST /webhook/todo-digest`, runs the same steps
on demand so the digest can be regenerated without waiting for 6 AM. It takes
no body and returns the digest row.
