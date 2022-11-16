# cache

This action allows caching dependencies and build outputs to improve workflow execution time.

[![Tests](https://github.com/actions/cache/actions/workflows/workflow.yml/badge.svg)](https://github.com/actions/cache/actions/workflows/workflow.yml)

## Documentation

See ["Caching dependencies to speed up workflows"](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows).

## What's New
### v3
* Added support for caching from GHES 3.5.
* Fixed download issue for files > 2GB during restore.
* Updated the minimum runner version support from node 12 -> node 16.
* Fixed avoiding empty cache save when no files are available for caching.
* Fixed tar creation error while trying to create tar with path as `~/` home folder on `ubuntu-latest`.
* Fixed zstd failing on amazon linux 2.0 runners.
* Fixed cache not working with github workspace directory or current directory.
* Fixed the download stuck problem by introducing a timeout of 1 hour for cache downloads.
* Fix zstd not working for windows on gnu tar in issues.
* Allowing users to provide a custom timeout as input for aborting download of a cache segment using an environment variable `SEGMENT_DOWNLOAD_TIMEOUT_MINS`. Default is 60 minutes.

Refer [here](https://github.com/actions/cache/blob/v2/README.md) for previous versions

## Usage

### Pre-requisites
Create a workflow `.yml` file in your repositories `.github/workflows` directory. An [example workflow](#example-workflow) is available below. For more information, reference the GitHub Help Documentation for [Creating a workflow file](https://help.github.com/en/articles/configuring-a-workflow#creating-a-workflow-file).

If you are using this inside a container, a POSIX-compliant `tar` needs to be included and accessible in the execution path.

### Inputs

* `path` - A list of files, directories, and wildcard patterns to cache and restore. See [`@actions/glob`](https://github.com/actions/toolkit/tree/main/packages/glob) for supported patterns.
* `key` - An explicit key for restoring and saving the cache
* `restore-keys` - An ordered list of prefix-matched keys to use for restoring stale cache if no cache hit occurred for key.

#### Environment Variables
* `SEGMENT_DOWNLOAD_TIMEOUT_MINS` - Segment download timeout (in minutes, default `60`) to abort download of the segment if not completed in the defined number of minutes. [Read more](#cache-segment-restore-timeout)

### Outputs

* `cache-hit` - A boolean value to indicate an exact match was found for the key. 

> Note: `cache-hit` will be set to `true` only when cache hit occurs for the exact `key` match. For a partial key match via `restore-keys` or a cache miss, it will be set to `false`.

See [Skipping steps based on cache-hit](#skipping-steps-based-on-cache-hit) for info on using this output

### Cache scopes
The cache is scoped to the key, [version](#cache-version) and branch. The default branch cache is available to other branches.

See [Matching a cache key](https://help.github.com/en/actions/configuring-and-managing-workflows/caching-dependencies-to-speed-up-workflows#matching-a-cache-key) for more info.

### Example workflow

```yaml
name: Caching Primes

on: push

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Cache Primes
      id: cache-primes
      uses: actions/cache@v3
      with:
        path: prime-numbers
        key: ${{ runner.os }}-primes

    - name: Generate Prime Numbers
      if: steps.cache-primes.outputs.cache-hit != 'true'
      run: /generate-primes.sh -d prime-numbers

    - name: Use Prime Numbers
      run: /primes.sh -d prime-numbers
```

> Note: You must use the `cache` action in your workflow before you need to use the files that might be restored from the cache. If the provided `key` matches an existing cache, a new cache is not created and if the provided `key` doesn't match an existing cache, a new cache is automatically created provided the job completes successfully.

## Implementation Examples

Every programming language and framework has its own way of caching.

See [Examples](examples.md) for a list of `actions/cache` implementations for use with:

- [C# - NuGet](./examples.md#c---nuget)
- [Clojure - Lein Deps](./examples.md#clojure---lein-deps)
- [D - DUB](./examples.md#d---dub)
- [Deno](./examples.md#deno)
- [Elixir - Mix](./examples.md#elixir---mix)
- [Go - Modules](./examples.md#go---modules)
- [Haskell - Cabal](./examples.md#haskell---cabal)
- [Haskell - Stack](./examples.md#haskell---stack)
- [Java - Gradle](./examples.md#java---gradle)
- [Java - Maven](./examples.md#java---maven)
- [Node - npm](./examples.md#node---npm)
- [Node - Lerna](./examples.md#node---lerna)
- [Node - Yarn](./examples.md#node---yarn)
- [OCaml/Reason - esy](./examples.md#ocamlreason---esy)
- [PHP - Composer](./examples.md#php---composer)
- [Python - pip](./examples.md#python---pip)
- [Python - pipenv](./examples.md#python---pipenv)
- [R - renv](./examples.md#r---renv)
- [Ruby - Bundler](./examples.md#ruby---bundler)
- [Rust - Cargo](./examples.md#rust---cargo)
- [Scala - SBT](./examples.md#scala---sbt)
- [Swift, Objective-C - Carthage](./examples.md#swift-objective-c---carthage)
- [Swift, Objective-C - CocoaPods](./examples.md#swift-objective-c---cocoapods)
- [Swift - Swift Package Manager](./examples.md#swift---swift-package-manager)

## Creating a cache key

A cache key can include any of the contexts, functions, literals, and operators supported by GitHub Actions.

For example, using the [`hashFiles`](https://docs.github.com/en/actions/learn-github-actions/expressions#hashfiles) function allows you to create a new cache when dependencies change.

```yaml
  - uses: actions/cache@v3
    with:
      path: |
        path/to/dependencies
        some/other/dependencies
      key: ${{ runner.os }}-${{ hashFiles('**/lockfiles') }}
```

Additionally, you can use arbitrary command output in a cache key, such as a date or software version:

```yaml
  # http://man7.org/linux/man-pages/man1/date.1.html
  - name: Get Date
    id: get-date
    run: |
      echo "::set-output name=date::$(/bin/date -u "+%Y%m%d")"
    shell: bash

  - uses: actions/cache@v3
    with:
      path: path/to/dependencies
      key: ${{ runner.os }}-${{ steps.get-date.outputs.date }}-${{ hashFiles('**/lockfiles') }}
```

See [Using contexts to create cache keys](https://help.github.com/en/actions/configuring-and-managing-workflows/caching-dependencies-to-speed-up-workflows#using-contexts-to-create-cache-keys)

## Cache Limits

A repository can have up to 10GB of caches. Once the 10GB limit is reached, older caches will be evicted based on when the cache was last accessed.  Caches that are not accessed within the last week will also be evicted.

## Skipping steps based on cache-hit

Using the `cache-hit` output, subsequent steps (such as install or build) can be skipped when a cache hit occurs on the key.  It is recommended to install the missing/updated dependencies in case of a partial key match when the key is dependent on the `hash` of the package file.

Example:
```yaml
steps:
  - uses: actions/checkout@v3

  - uses: actions/cache@v3
    id: cache
    with:
      path: path/to/dependencies
      key: ${{ runner.os }}-${{ hashFiles('**/lockfiles') }}

  - name: Install Dependencies
    if: steps.cache.outputs.cache-hit != 'true'
    run: /install.sh
```

> Note: The `id` defined in `actions/cache` must match the `id` in the `if` statement (i.e. `steps.[ID].outputs.cache-hit`)


## Cache Version
Cache version is a hash [generated](https://github.com/actions/toolkit/blob/500d0b42fee2552ae9eeb5933091fe2fbf14e72d/packages/cache/src/internal/cacheHttpClient.ts#L73-L90) for a combination of compression tool used (Gzip, Zstd, etc. based on the runner OS) and the `path` of directories being cached. If two caches have different versions, they are identified as unique caches while matching. This for example, means that a cache created on `windows-latest` runner can't be restored on `ubuntu-latest` as cache `Version`s are different. 

> Pro tip: [List caches](https://docs.github.com/en/rest/actions/cache#list-github-actions-caches-for-a-repository) API can be used to get the version of a cache. This can be helpful to troubleshoot cache miss due to version. 

<details>
  <summary>Example</summary>
The workflow will create 3 unique caches with same keys. Ubuntu and windows runners will use different compression technique and hence create two different caches. And `build-linux` will create two different caches as the `paths` are different.

```yaml
jobs:
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Cache Primes
        id: cache-primes
        uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: ./generate-primes.sh -d prime-numbers

      - name: Cache Numbers
        id: cache-numbers
        uses: actions/cache@v3
        with:
          path: numbers
          key: primes

      - name: Generate Numbers
        if: steps.cache-numbers.outputs.cache-hit != 'true'
        run: ./generate-primes.sh -d numbers

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v3

      - name: Cache Primes
        id: cache-primes
        uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: ./generate-primes -d prime-numbers
```
</details>

## Force deletion of caches overriding default cache eviction policy
Caches have [branch scope restriction](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache) in place. This means that if caches for a specific branch are using a lot of storage quota, it may result into more frequently used caches from `default` branch getting thrashed. For example, if there are many pull requests happening on a repo and are creating caches, these cannot be used in default branch scope but will still occupy a lot of space till they get cleaned up by [eviction policy](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#usage-limits-and-eviction-policy). But sometime we want to clean them up on a faster cadence so as to ensure default branch is not thrashing. In order to achieve this, [gh-actions-cache cli](https://github.com/actions/gh-actions-cache/) can be used to delete caches for specific branches.

This workflow uses `gh-actions-cache` to delete all the caches created by a branch. 
<details>
  <summary>Example</summary>

```yaml
name: cleanup caches by a branch
on:
  pull_request:
    types:
      - closed
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v3

      - name: Cleanup
        run: |
          gh extension install actions/gh-actions-cache
          
          REPO=${{ github.repository }}
          BRANCH=${{ github.ref }}

          echo "Fetching list of cache key"
          cacheKeysForPR=$(gh actions-cache list -R $REPO -B $BRANCH | cut -f 1 )
        
          set +e
          echo "Deleting caches..."
          for cacheKey in $cacheKeysForPR
          do
              gh actions-cache delete $cacheKey -R $REPO -B $BRANCH --confirm
          done
          echo "Done"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
</details>

## Known practices and workarounds
Following are some of the known practices/workarounds which community has used to fulfill specific requirements. You may choose to use them if suits your use case. Note these are not necessarily the only or the recommended solution.

- [Cache segment restore timeout](./workarounds.md#cache-segment-restore-timeout)
- [Update a cache](./workarounds.md#update-a-cache)
- [Use cache across feature branches](./workarounds.md#use-cache-across-feature-branches)
- [Improving cache restore performance on Windows/Using cross-os caching](./workarounds.md#improving-cache-restore-performance-on-windows-using-cross-os-caching)

#### Windows environment variables
Please note that Windows environment variables (like `%LocalAppData%`) will NOT be expanded by this action. Instead, prefer using `~` in your paths which will expand to HOME directory. For example, instead of `%LocalAppData%`, use `~\AppData\Local`. For a list of supported default environment variables, see [this](https://docs.github.com/en/actions/learn-github-actions/environment-variables) page. 

## Contributing
We would love for you to contribute to `actions/cache`, pull requests are welcome! Please see the [CONTRIBUTING.md](CONTRIBUTING.md) for more information.

## License
The scripts and documentation in this project are released under the [MIT License](LICENSE)
