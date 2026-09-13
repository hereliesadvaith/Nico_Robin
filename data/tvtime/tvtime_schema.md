# Watch tracking schema (Directus)

Movies and TV series I have watched, seeded from a TV Time export. Four
collections: `movies`, `series`, a shared `genres` lookup, and a hidden
junction table for each many-to-many.

## `movies`

One row per movie. Both the watched log and the watchlist live here,
separated by `status`.

| Field           | Type      | Notes                                                                              |
| --------------- | --------- | ---------------------------------------------------------------------------------- |
| `id`            | integer   | primary key, auto-increment                                                        |
| `title`         | string    | required, max 255 chars, IMDb primary title                                        |
| `year`          | integer   | release year, as on IMDb                                                           |
| `imdb_id`       | string    | IMDb title id, e.g. `tt0107290`. Unique in practice; the canonical key for a movie |
| `status`        | string    | dropdown: `watched`, `watchlist`; defaults to `watched`                            |
| `watched_at`    | date      | date only, last time watched. Empty for `watchlist` rows                           |
| `rewatch_count` | integer   | defaults to 0, times watched beyond the first                                      |
| `genres`        | m2m       | many-to-many -> `genres` through `movies_genres`                                   |
| `date_created`  | timestamp | system, hidden                                                                     |
| `date_updated`  | timestamp | system, hidden                                                                     |

Display template: `{{title}} ({{year}})`. Default sort when listing:
`watched_at` descending. `https://www.imdb.com/title/<imdb_id>/` is the IMDb
page. Expand `genres.genres_id.name` to show genre names.
Filter by genre with `{"genres": {"genres_id": {"name": {"_eq": "Horror"}}}}`.

## `series`

One row per TV series. Episodes are not tracked individually; only a running
count of episodes watched.

| Field              | Type      | Notes                                                                                                    |
| ------------------ | --------- | -------------------------------------------------------------------------------------------------------- |
| `id`               | integer   | primary key, auto-increment                                                                              |
| `title`            | string    | required, max 255 chars, unique in practice                                                              |
| `year`             | integer   | year the first episode aired                                                                             |
| `imdb_id`          | string    | IMDb title id, e.g. `tt0944947`. Unique in practice; the canonical key for a series                      |
| `status`           | string    | dropdown: `watching`, `up_to_date`, `paused`, `finished`, `stopped`, `watchlist`; defaults to `watching` |
| `episodes`         | integer   | episodes watched so far, defaults to 0                                                                   |
| `genres`           | m2m       | many-to-many -> `genres` through `series_genres`                                                         |
| `date_created`     | timestamp | system, hidden                                                                                           |
| `date_updated`     | timestamp | system, hidden                                                                                           |

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
The episode count was taken from `tvtime-series-episodes-*.csv`, which was
otherwise discarded. TV Time statuses were mapped: continuing -> `watching`,
up_to_date -> `up_to_date`, watch_later -> `paused`, not_started_yet ->
`watchlist`.

Result: 141 series (97 up_to_date, 20 watching, 3 paused, 21 watchlist), all
with a year and genres, 390 genre links.

`imdb_id` was added on 2026-09-13 and backfilled from the IMDb dataset
(`title.basics.tsv.gz` + `title.ratings.tsv.gz`), matching each row on title
and year and breaking ties by overlap with the row's genres, then by vote
count. Series reused the match from the original import. Twenty movies whose
TV Time title or year differed from IMDb were resolved by hand. Every row in
both collections has an `imdb_id`.

On the same day `title` and `year` of every row were rewritten to IMDb's
primary title and start year: 102 movies and 20 series changed, mostly
capitalisation, punctuation, "(US)"/"(2021)" suffixes, and release years that
TV Time had one year late. Notable renames: "Harry Potter and the
Philosopher's Stone" -> "Sorcerer's Stone", "Norsemen" -> "Vikingane", "The
Odd Family: Zombie On Sale" -> "Zombie for Sale". The one-off scripts were
not kept.

On 2026-09-13 the series count field was replaced by `episodes` and the TV
Time counts were discarded. For the 75 `finished` and 25 `up_to_date` series
it was refilled from IMDb's `title.episode.tsv.gz` joined with
`title.basics.tsv.gz`: episodes of the show's `imdb_id`, excluding specials
(no season number), episodes dated after 2026 and undated episodes, which
are unaired. Result: 4345 episodes across finished shows, 662 across
up_to_date shows. `watching` and `paused` rows are still 0 and need counts
entered by hand.

New movies and series are added by the assistant, which looks up the IMDb
id, year and genres on the web first.
