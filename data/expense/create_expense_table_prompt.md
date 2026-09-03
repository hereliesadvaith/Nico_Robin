# Prompt: create the expense collections in Directus

Copy everything below the line into the Expense project chat (Sonnet, with the
Directus MCP connected).

---

Use the Directus MCP tools to create two collections for tracking my personal
expenses: a small `expense_categories` lookup and the main `expenses` log.
Categories live in their own collection so I can add new ones later by
inserting a row, without touching the schema. Do the steps in order and confirm
each step before moving to the next.

## 1. Check first

List existing collections. If `expense_categories` or `expenses` already
exists, stop and show me its fields instead of creating anything.

## 2. Create `expense_categories`

Settings:

- Primary key: `id`, auto-increment integer
- Icon: `label`
- Note: "Categories used by the expenses collection"
- Display template: `{{name}}`

Fields (exactly these, nothing else):

| Field  | Type   | Interface | Required | Rules                   |
| ------ | ------ | --------- | -------- | ----------------------- |
| `name` | string | input     | yes      | unique, max length 100  |

Insert these four rows and no others:

- Food
- Rent
- Fuel
- Entertainment

## 3. Create `expenses`

Settings:

- Primary key: `id`, auto-increment integer
- Icon: `payments`
- Note: "Personal expense log"
- Enable the system fields `date_created` and `date_updated` (hidden in the form)
- Display template: `{{name}} - {{amount}}`

Fields (exactly these, nothing else):

| Field      | Type                       | Interface            | Required | Rules                                                          |
| ---------- | -------------------------- | -------------------- | -------- | -------------------------------------------------------------- |
| `name`     | string                     | input                | yes      | max length 255                                                 |
| `amount`   | decimal                    | input                | yes      | precision 10, scale 2, must be greater than 0                   |
| `date`     | date                       | datetime (date only) | yes      | default value: current date                                    |
| `category` | many-to-one -> `expense_categories` | select-dropdown-m2o | yes | display the related `name`; on delete of a category set null |

## 4. Verify

Read both collections back and show me the final field list for each with
type, required flag and default value. Confirm `expenses.category` is a
many-to-one relation to `expense_categories`.

Then insert one test expense:

- name: "Test lunch"
- amount: 250.00
- date: today
- category: the Food row

Read it back with the category name expanded, show it to me, then delete it.
Leave the four category rows in place.

## Rules

- Do not create any other collections, relations or roles.
- Do not change permissions.
- If a tool call fails, show me the exact error and stop instead of retrying
  with a different approach.

---

# Later: adding a category

When I ask to add a category, insert a row into `expense_categories` with the
given name. Do not change the schema. Check first that a row with that name
(case-insensitive) does not already exist.
