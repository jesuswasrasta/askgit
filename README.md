[![GoDev](https://img.shields.io/static/v1?label=godev&message=reference&color=00add8)](https://pkg.go.dev/github.com/augmentable-dev/askgit)
[![BuildStatus](https://github.com/augmentable-dev/askgit/workflows/tests/badge.svg)](https://github.com/augmentable-dev/askgit/actions?workflow=tests)
[![Go Report Card](https://goreportcard.com/badge/github.com/augmentable-dev/askgit)](https://goreportcard.com/report/github.com/augmentable-dev/askgit)
[![TODOs](https://badgen.net/https/api.tickgit.com/badgen/github.com/augmentable-dev/askgit)](https://www.tickgit.com/browse?repo=github.com/augmentable-dev/askgit)
[![codecov](https://codecov.io/gh/augmentable-dev/askgit/branch/master/graph/badge.svg)](https://codecov.io/gh/augmentable-dev/askgit)


# askgit

`askgit` is a command-line tool for running SQL queries on git repositories.
It's meant for ad-hoc querying of git repositories on disk through a common interface (SQL), as an alternative to patching together various shell commands.
It can execute queries that look like:
```sql
-- how many commits have been authored by user@email.com?
SELECT count(*) FROM commits WHERE author_email = 'user@email.com'
```
More in-depth examples and documentation can be found below.

## Installation

### Homebrew

```
brew tap augmentable-dev/askgit
brew install askgit
```

### Go

```
go get -v -tags=sqlite_vtable github.com/augmentable-dev/askgit
```

Will use the go tool chain to install a binary to `$GOBIN`.

```
GOBIN=$(pwd) go get -v -tags=sqlite_vtable github.com/augmentable-dev/askgit
```

Will produce a binary in your current directory.


### Using Docker

Build an image locally using docker

```
docker build -t askgit:latest .
```

Or use an official image from [docker hub](https://hub.docker.com/repository/docker/augmentable/askgit)

```
docker pull augmentable/askgit:latest
```

#### Running commands

`askgit` operates on a git repository. This repository needs to be attached as a volume. This example uses the (bash) built-in command `pwd` for the current working directory

> [**pwd**] Print the absolute pathname of the current working directory.

```
docker run -v `pwd`:/repo:ro askgit "SELECT * FROM commits"
```

#### Running commands from STDIN

For piping commands via STDIN, the docker command needs to be told to run non-interactively, as well as attaching the repository at `/repo`.

```
cat query.sql | docker run -i -v `pwd`:/repo:ro askgit
```

## Usage

```
askgit -h
```

Will output the most up to date usage instructions for your version of the CLI.
Typically the first argument is a SQL query string:

```
askgit "SELECT * FROM commits"
```

Your current working directory will be used as the path to the git repository to query by default.
Use the `--repo` flag to specify an alternate path, or even a remote repository reference (http(s) or ssh).
`askgit` will clone the remote repository to a temporary directory before executing a query.

You can also pass a query in via `stdin`:

```
cat query.sql | askgit
```

By default, output will be an ASCII table.
Use `--format json` or `--format csv` for alternatives.
See `-h` for all the options.

### Tables

#### `commits`

Similar to `git log`, the `commits` table includes all commits in the history of the currently checked out commit.

| Column          | Type     |
|-----------------|----------|
| id              | TEXT     |
| message         | TEXT     |
| summary         | TEXT     |
| author_name     | TEXT     |
| author_email    | TEXT     |
| author_when     | DATETIME |
| committer_name  | TEXT     |
| committer_email | TEXT     |
| committer_when  | DATETIME |
| parent_id       | TEXT     |
| parent_count    | INT      |
| tree_id         | TEXT     |
| additions       | INT      |
| deletions       | INT      |

#### `files`

The `files` table iterates over _ALL_ the files in a commit history, by default from what's checked out in the repository.
The full table is every file in every tree of a commit history.
Use the `commit_id` column to filter for files that belong to the work tree of a specific commit.

| Column     | Type |
|------------|------|
| commit_id  | TEXT |
| tree_id    | TEXT |
| file_id    | TEXT |
| name       | TEXT |
| contents   | TEXT |
| executable | INT  |


#### `refs`

| Column | Type |
|--------|------|
| name   | TEXT |
| type   | TEXT |
| hash   | TEXT |

### Example Queries

This will return all commits in the history of the currently checked out branch/commit of the repo.
```sql
SELECT * FROM commits
```

Return the (de-duplicated) email addresses of commit authors:
```sql
SELECT DISTINCT author_email FROM commits
```

Return the commit counts of every author (by email):
```sql
SELECT author_email, count(*) FROM commits GROUP BY author_email ORDER BY count(*) DESC
```

Same as above, but excluding merge commits:
```sql
SELECT author_email, count(*) FROM commits WHERE parent_count < 2 GROUP BY author_email ORDER BY count(*) DESC
```

This is an expensive query.
It will iterate over every file in every tree of every commit in the current history:
```sql
SELECT * FROM files
```


Outputs the set of files in the tree of a certain commit:
```sql
SELECT * FROM files WHERE commit_id='some_commit_id'
```


Same as above if you just have the commit short id:
```sql
SELECT * FROM files WHERE commit_id LIKE 'shortened_commit_id%'
```


Returns author emails with lines added/removed, ordered by total number of commits in the history:
```sql
SELECT count(*) AS commits, SUM(additions) AS additions, SUM(deletions) AS  deletions, author_email FROM commits GROUP BY author_email ORDER BY commits
```



Returns commit counts by author, broken out by day of the week:

```sql
SELECT
    count(*) AS commits,
    count(CASE WHEN strftime('%w',author_when)='0' THEN 1 END) AS sunday,
    count(CASE WHEN strftime('%w',author_when)='1' THEN 1 END) AS monday,
    count(CASE WHEN strftime('%w',author_when)='2' THEN 1 END) AS tuesday,
    count(CASE WHEN strftime('%w',author_when)='3' THEN 1 END) AS wednesday,
    count(CASE WHEN strftime('%w',author_when)='4' THEN 1 END) AS thursday,
    count(CASE WHEN strftime('%w',author_when)='5' THEN 1 END) AS friday,
    count(CASE WHEN strftime('%w',author_when)='6' THEN 1 END) AS saturday,
    author_email
FROM commits GROUP BY author_email ORDER BY commits
```


#### Interactive mode
```
askgit --interactive
```

Will display a basic terminal UI for composing and executing queries, powered by [gocui](https://github.com/jroimartin/gocui).
