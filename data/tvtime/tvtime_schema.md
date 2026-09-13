# Watch tracking schema (Directus)

Movies and shows I have watched, seeded from a TV Time export. Movies first;
shows will be added as further collections in this folder.

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

## `genres`

Lookup table using IMDb's genre names. Add a genre by inserting a row.

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

## Import

The collections were seeded once from a TV Time export dated 2026-07-02.
Movies came from `tvtime-movies-*.csv`, matched on `title` + `year`; TV Time's
`imdb_id`, `tvdb_id` and `uuid` columns were not stored. Genres came from
IMDb's public dataset (https://datasets.imdbws.com/title.basics.tsv.gz),
looked up by the export's `imdb_id` and linked through `movies_genres`.

Result: 675 watched movies, all with genres, 1831 genre links. New movies are
added by the assistant, which looks up year and genres on the web first.
