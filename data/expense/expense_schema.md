# Expense tracking schema (Directus)

## `expense_categories`

Lookup table. Add a category by inserting a row.

| Field  | Type    | Notes                          |
| ------ | ------- | ------------------------------ |
| `id`   | integer | primary key, auto-increment    |
| `name` | string  | required, unique, max 100 chars |

Seeded rows: Food, Rent, Fuel, Entertainment.

## `expenses`

Main log. One row per expense.

| Field          | Type      | Notes                                                   |
| -------------- | --------- | ------------------------------------------------------- |
| `id`           | integer   | primary key, auto-increment                             |
| `name`         | string    | required, short description, max 255 chars              |
| `amount`       | decimal   | required, precision 10 scale 2, greater than 0           |
| `date`         | date      | required, date only, defaults to today                  |
| `category`     | integer   | required, many-to-one -> `expense_categories.id`       |
| `date_created` | timestamp | system, hidden                                          |
| `date_updated` | timestamp | system, hidden                                          |

Default sort: `date` descending. Expand `category.name` to show the category
label in results.
