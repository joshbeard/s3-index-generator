# web-indexer

Quick and simple program to generate basic directory index pages from a local
file directory or S3 bucket, such as a file listing page.

<p align="center">
  <img width="420" height="236" alt="screenshot" src=".github/readme/screenshot.png" />
</p>

This isn't a good solution for a dynamic listing of an S3 bucket (maybe try
Lambda), but it's simple and works for content that remains static between
deployments. This is particularly helpful for hosting static artifacts on S3,
since S3 doesn't natively offer content indexes.

If you're using Nginx, [fancy_index](https://www.nginx.com/resources/wiki/modules/fancy_index/)
is the right tool for the job and does this dynamically with full customization.
If you're using Apache, [mod_autoindex](https://httpd.apache.org/docs/2.4/mod/mod_autoindex.html) is
what you're looking for, maybe with something like [Apaxy](https://oupala.github.io/apaxy/).

## Usage

Download packages from the [releases](https://github.com/joshbeard/web-indexer/releases),
use the Docker image from
[`ghcr.io/joshbeard/web-indexer/web-indexer`](https://github.com/joshbeard/web-indexer/pkgs/container/web-indexer%2Fweb-indexer)
or [`joshbeard/web-indexer`](https://hub.docker.com/r/joshbeard/web-indexer) on
the Docker Hub, use the [GitHub Action](#github-action) or a [GitLab job](#gitlab-ci).

```plain
Usage:
  web-indexer --source <source> --target <target> [flags]

Flags:
  -u, --base-url string      A URL to prepend to the links
  -c, --config string        config file
      --date-format string   The date format to use in the index page (default "2006-01-02 15:04:05 MST")
      --dirs-first           List directories first (default true)
  -h, --help                 help for web-indexer
  -i, --index-file string    The name of the index file (default "index.html")
  -l, --link-to-index        Link to the index file or just the path
  -F, --log-file string      The log file
  -L, --log-level string     The log level (default "info")
  -m, --minify               Minify the index page
      --order string         The order for the items. One of: asc, desc (default "asc")
  -q, --quiet                Suppress log output
  -r, --recursive            List files recursively
  -S, --skip strings         A list of files or directories to skip. Comma separated or specified multiple times
      --sort-by string       The order for the index page. One of: last_modified, name, natural_name (default "natural_name")
  -s, --source string        REQUIRED. The source directory or S3 URI to list
  -t, --target string        REQUIRED. The target directory or S3 URI to write to
  -f, --template string      A custom template file to use for the index page
  -T, --title string         The title of the index page
  -v, --version              version for web-indexer
```

### Examples

The `source` and `target` arguments can be specified using their respective
flags, or provided as two unflagged arguments:

```shell
web-indexer SOURCE TARGET
web-indexer --source SOURCE --target TARGET
```

Index a local directory and write the index file to the same directory:

```shell
web-indexer --source /path/to/directory --target /path/to/directory
```

Index a local directory and write the index file to a different directory:

```shell
web-indexer --source /path/to/directory --target /foo/bar
```

Index a local directory and upload the index file to an S3 bucket:

```shell
web-indexer --source /path/to/directory --target s3://bucket/path
```

Index an S3 bucket and write the index file to a local directory:

```shell
web-indexer --source s3://bucket/path --target /path/to/directory
```

Index an S3 bucket and upload the index file to the same bucket and path:

```shell
web-indexer --source s3://bucket/path --target s3://bucket/path
```

Set a title for the index pages:

```shell
web-indexer --source /path/to/directory --target /path/to/directory --title 'Index of {relativePath}'
```

Load a config:

```shell
web-indexer -config /path/to/config.yaml
```

## GitHub Action

It's also available as a GitHub action.

For example:

```yaml
jobs:
  build:
    steps:
      # ... Other config ...
      - name: S3 Index Generator
        uses: joshbeard/web-indexer@0.4.1
        with:
          config: .web-indexer.yml
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: 'us-east-1'
```

Example of using this more manually, such as with a private runner with a
volume mounted from outside the workspace. This example expects a
`.web-indexer.yml` config to exist in the repository's root and is passing in
the AWS variables for an S3 target:

```yaml
jobs:
  build:
    runs-on: self-hosted
    steps:
      - name: Web Index Generator
        run: |
          docker run --rm \
            -v /mnt/repos:/mnt/repos \
            -v ${PWD}:/workspace \
            -w /workspace \
            -e AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }} \
            -e AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }} \
            -e AWS_REGION='us-east-1' \
            -e CONFIG=.web-indexer.yml \
            ghcr.io/joshbeard/web-indexer/web-indexer:latest
```

Refer to the [`action.yml`](action.yml) for all available inputs, which
correspond to the CLI arguments and configuration parameters.

## GitLab CI

```yaml
web-index:
  image:
    name: joshbeard/web-indexer
    entrypoint: ['']
  variables:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: 'us-east-1'
  script:
    web-indexer
```

## Configuration

You can configure the behavior of `web-indexer` using command-line arguments
and/or a YAML config file. Both are evaluated with the command-line arguments
taking precedence.

Configuration files named `.web-indexer.yml` or `.web-indexer.yaml` in the
current working directory will be automatically loaded.

The full configuration with default values for each key are provided below:

```yaml
# base_url is an optional URL to prefix to links. If unset, links are relative.
base_url: ""

# date_format is the date format to use for indexed files modified time.
# This is provided in Go's `time` package format.
# See https://pkg.go.dev/time#pkg-examples
date_format: "2006-01-02 15:04:05 UTC"

# dirs_first toggles if directories should be ordered before files in the
# list.
dirs_first: true

# index_file is the name of the file to generate.
index_file: "index.html"

# link_to_index toggles linking to the index_file for sub-paths or just the
# root of the subpath (foo/ vs foo/index.html).
link_to_index: false

# log_file is an optional path to a file to log to
# if log_level is set and this is not, messages are logged to stderr
log_file: ""

# log_level configures the verbosity of logging.
# Acceptable values: info, error, warn, debug
# Set this to an empty string or use the 'quiet' option  to suppress all log
# output.
log_level: "info"

# minify toggles minifying the generated HTML.
minify: false

# quiet suppresses all log output
quiet: false

# order the items (asc)ending or (desc)ending (by sort).
order: "asc"

# recursive enables indexing the source recursively.
recursive: false

# skips is a list of filenames to skip.
skips: []

# sort_by determines how the items are sorted.
# Valid values: last_modified, name, name_natural
# name_natural sorts by name in a human friendly way (e.g. 1,2,10 not 1,10,2).
sort_by: "name_natural"

# source is the path to a local directory or an S3 URI.
source: "blah/"

# target is the path to a local directory or an S3 URI.
target: "blah/"

# template is the path to a local Go template file to use for generating the
# indexes. The built-in template is used by default.
template: ""

# title customizes the title field available in the template.
# Certain tokens can be used to be dynamically replaced.
#   {source}       - the base source path
#   {target}       - the target path or URI
#   {path}         - the full path including the source
#   {relativePath} - the path relative to the source
title: ""
```

### Example Configuration

```yaml
source: /mnt/repos
target: s3://my-cool-repos/
title: "Index of {relativePath}"
minify: true
recursive: true
link_to_index: true
order: desc
```
