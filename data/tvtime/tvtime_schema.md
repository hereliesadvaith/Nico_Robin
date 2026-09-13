# Watch tracking schema (Directus)

Movies and TV series I have watched, seeded from a TV Time export. Four
collections: `movies`, `series`, a shared `genres` lookup, and a hidden
junction table for each many-to-many.

## `movies`

One row per movie. Both the watched log and the watchlist live here,
separated by `status`.

| Field           | Type      | Notes                                                        |
| --------------- | --------- | ------------------------------------------------------------ |
| `id`            | integer   | primary key, auto-increment                                  |
| `title`         | string    | required, max 255 chars                                      |
| `year`          | integer   | release year. `title` + `year` identifies a movie            |
| `status`        | string    | dropdown: `watched`, `watchlist`; defaults to `watched`      |
| `watched_at`    | date      | date only, last time watched. Empty for `watchlist` rows     |
| `rewatch_count` | integer   | defaults to 0, times watched beyond the first                |
| `genres`        | m2m       | many-to-many -> `genres` through `movies_genres`             |
| `date_created`  | timestamp | system, hidden                                               |
| `date_updated`  | timestamp | system, hidden                                               |

Display template: `{{title}} ({{year}})`. Default sort when listing:
`watched_at` descending. Expand `genres.genres_id.name` to show genre names.
Filter by genre with `{"genres": {"genres_id": {"name": {"_eq": "Horror"}}}}`.

## `series`

One row per TV series. Episodes are not tracked individually; only a running
count of episodes watched.

| Field              | Type      | Notes                                                                                        |
| ------------------ | --------- | -------------------------------------------------------------------------------------------- |
| `id`               | integer   | primary key, auto-increment                                                                  |
| `title`            | string    | required, max 255 chars, unique in practice                                                  |
| `year`             | integer   | year the first episode aired                                                                 |
| `status`           | string    | dropdown: `watching`, `up_to_date`, `paused`, `finished`, `stopped`, `watchlist`; defaults to `watching` |
| `episodes_watched` | integer   | defaults to 0                                                                                |
| `genres`           | m2m       | many-to-many -> `genres` through `series_genres`                                             |
| `date_created`     | timestamp | system, hidden                                                                               |
| `date_updated`     | timestamp | system, hidden                                                                               |

Status meanings: `watching` = mid-way with episodes left; `up_to_date` =
caught up on everything aired so far; `paused` = started but set aside;
`finished` = show ended and all watched; `stopped` = dropped for good;
`watchlist` = not started.

Display template: `{{title}} ({{year}})`. Default sort when listing: `title`.
Genres are read and filtered the same way as on `movies`.

## `genres`

Lookup table using IMDb's genre names, shared by `movies` and `series`. Add
a genre by inserting a row.

| Field  | Type    | Notes                           |
| ------ | ------- | ------------------------------- |
| `id`   | integer | primary key, auto-increment     |
| `name` | string  | required, unique, max 100 chars |

Seeded with the 21 genres that occur in the import: Action, Adventure,
Animation, Biography, Comedy, Crime, Documentary, Drama, Family, Fantasy,
History, Horror, Music, Musical, Mystery, Romance, Sci-Fi, Sport, Thriller,
War, Western.

## `movies_genres`

Hidden junction table for the many-to-many. Deleting a movie or a genre
removes its rows.

| Field       | Type    | Notes                          |
| ----------- | ------- | ------------------------------ |
| `id`        | integer | primary key, auto-increment    |
| `movies_id` | integer | -> `movies.id`, cascade delete |
| `genres_id` | integer | -> `genres.id`, cascade delete |

## `series_genres`

Same shape as `movies_genres`, for series.

| Field       | Type    | Notes                          |
| ----------- | ------- | ------------------------------ |
| `id`        | integer | primary key, auto-increment    |
| `series_id` | integer | -> `series.id`, cascade delete |
| `genres_id` | integer | -> `genres.id`, cascade delete |

## Import

The collections were seeded once from a TV Time export dated 2026-07-02.
Movies came from `tvtime-movies-*.csv`, matched on `title` + `year`; TV Time's
`imdb_id`, `tvdb_id` and `uuid` columns were not stored. Genres came from
IMDb's public dataset (https://datasets.imdbws.com/title.basics.tsv.gz),
looked up by the export's `imdb_id` and linked through `movies_genres`.

Result: 675 watched movies, all with genres, 1831 genre links.

Series came from `tvtime-series-*.csv`. The export had no IMDb ids for
series, so each title was matched against the IMDb dataset's TV rows, taking
the candidate with the most votes; two were fixed by hand (WHAT / IF is the
2019 Netflix drama, PLUR1BUS is Pluribus 2025). `year` is IMDb's start year.
`episodes_watched` was counted from `tvtime-series-episodes-*.csv`, which was
otherwise discarded. TV Time statuses were mapped: continuing -> `watching`,
up_to_date -> `up_to_date`, watch_later -> `paused`, not_started_yet ->
`watchlist`.

Result: 141 series (97 up_to_date, 20 watching, 3 paused, 21 watchlist), all
with a year and genres, 390 genre links.

New movies and series are added by the assistant, which looks up year and
genres on the web first.
