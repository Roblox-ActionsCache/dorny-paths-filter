# Paths Changes Filter

This [Github Action](https://github.com/features/actions) enables conditional execution of workflow steps and jobs,
based on the files modified by pull request, feature branch or in pushed commits.

It saves time and resources especially in monorepo setups, where you can run slow tasks (e.g. integration tests or deployments) only for changed components.
Github workflows built-in [path filters](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#onpushpull_requestpaths)
don't allow this because they don't work on a level of individual jobs or steps.

**Real world usage examples:**
- [sentry.io](https://sentry.io/) - [backend-test-py3.6.yml](https://github.com/getsentry/sentry/blob/ca0e43dc5602a9ab2e06d3f6397cc48fb5a78541/.github/workflows/backend-test-py3.6.yml#L32)
- [GoogleChrome/web.dev](https://web.dev/) - [lint-and-test-workflow.yml](https://github.com/GoogleChrome/web.dev/blob/e1f0c28964e99ce6a996c1e3fd3ee1985a7a04f6/.github/workflows/lint-and-test-workflow.yml#L33)


## Supported workflows:
- **Pull requests:**
  - Workflow triggered by **[pull_request](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request)**
    or **[pull_request_target](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request_target)** event
  - Changes are detected against the pull request base branch
  - Uses Github REST API to fetch list of modified files
- **Feature branches:**
  - Workflow triggered by **[push](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push)**
  or any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  - The `base` input parameter must not be the same as the branch that triggered the workflow
  - Changes are detected against the merge-base with configured base branch or default branch
  - Uses git commands to detect changes - repository must be already [checked out](https://github.com/actions/checkout)
- **Master, Release or other long-lived branches:**
  - Workflow triggered by **[push](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#push)** event
  when `base` input parameter is same as the branch that triggered the workflow:
    - Changes are detected against the most recent commit on the same branch before the push
  - Workflow triggered by any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  when `base` input parameter is commit SHA:
    - Changes are detected against the provided `base` commit
  - Workflow triggered by any other **[event](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)**
  when `base` input parameter is same as the branch that triggered the workflow:
    - Changes are detected from last commit
  - Uses git commands to detect changes - repository must be already [checked out](https://github.com/actions/checkout)
- **Local changes**
  - Workflow triggered by any event when `base` input parameter is set to `HEAD`
  - Changes are detected against current HEAD
  - Untracked files are ignored

## Example
```yaml
- uses: dorny/paths-filter@v2
  id: changes
  with:
    filters: |
      src:
        - 'src/**'

  # run only if some file in 'src' folder was changed
  if: steps.changes.outputs.src == 'true'
  run: ...
```
For more scenarios see [examples](#examples) section.

## Notes:
- Paths expressions are evaluated using [picomatch](https://github.com/micromatch/picomatch) library.
  Documentation for path expression format can be found on project github page.
- Picomatch [dot](https://github.com/micromatch/picomatch#options) option is set to true.
  Globbing will match also paths where file or folder name starts with a dot.
- It's recommended to quote your path expressions with `'` or `"`. Otherwise you will get an error if it starts with `*`.
- Local execution with [act](https://github.com/nektos/act) works only with alternative runner image. Default runner doesn't have `git` binary.
  - Use: `act -P ubuntu-latest=nektos/act-environments-ubuntu:18.04`


# What's New
- Add `list-files: csv` format
- Configure matrix job to run for each folder with changes using `changes` output
- Improved listing of matching files with `list-files: shell` and `list-files: escape` options
- Support local changes
- Fixed retrieval of all changes via Github API when there are 100+ changes
- Paths expressions are now evaluated using [picomatch](https://github.com/micromatch/picomatch) library
- Support workflows triggered by any event

For more information see [CHANGELOG](https://github.com/dorny/paths-filter/blob/master/CHANGELOG.md)

# Usage

```yaml
- uses: dorny/paths-filter@v2
  with:
    # Defines filters applied to detected changed files.
    # Each filter has a name and list of rules.
    # Rule is a glob expression - paths of all changed
    # files are matched against it.
    # Rule can optionally specify if the file
    # should be added, modified or deleted.
    # For each filter there will be corresponding output variable to
    # indicate if there's a changed file matching any of the rules.
    # Optionally there can be a second output variable
    # set to list of all files matching the filter.
    # Filters can be provided inline as a string (containing valid YAML document)
    # or as a relative path to separate file (e.g.: .github/filters.yaml).
    # Multiline string is evaluated as embedded filter definition,
    # single line string is evaluated as relative path to separate file.
    # Filters syntax is documented by example - see examples section.
    filters: ''

    # Branch, tag or commit SHA against which the changes will be detected.
    # If it references same branch it was pushed to,
    # changes are detected against the most recent commit before the push.
    # Otherwise it uses git merge-base to find best common ancestor between
    # current branch (HEAD) and base.
    # When merge-base is found, it's used for change detection - only changes
    # introduced by current branch are considered.
    # All files are considered as added if there is no common ancestor with
    # base branch or no previous commit.
    # This option is ignored if action is triggered by pull_request event.
    # Default: repository default branch (e.g. master)
    base: ''

    # How many commits are initially fetched from base branch.
    # If needed, each subsequent fetch doubles the
    # previously requested number of commits until the merge-base
    # is found or there are no more commits in the history.
    # This option takes effect only when changes are detected
    # using git against base branch (feature branch workflow).
    # Default: 100
    initial-fetch-depth: ''

    # Enables listing of files matching the filter:
    #   'none'  - Disables listing of matching files (default).
    #   'csv'   - Coma separated list of filenames.
    #             If needed it uses double quotes to wrap filename with unsafe characters.
    #   'json'  - Matching files paths are formatted as JSON array.
    #   'shell' - Space delimited list usable as command line argument list in Linux shell.
    #             If needed it uses single or double quotes to wrap filename with unsafe characters.
    #   'escape'- Space delimited list usable as command line argument list in Linux shell.
    #             Backslash escapes every potentially unsafe character.
    # Default: none
    list-files: ''

    # Relative path under $GITHUB_WORKSPACE where the repository was checked out.
    working-directory: ''

    # Personal access token used to fetch list of changed files
    # from Github REST API.
    # It's used only if action is triggered by pull request event.
    # Github token from workflow context is used as default value.
    # If empty string is provided, action falls back to detect
    # changes using git commands.
    # Default: ${{ github.token }}
    token: ''
```

## Outputs
- For each filter it sets output variable named by the filter to the text:
   - `'true'` - if **any** of changed files matches any of filter rules
   - `'false'` - if **none** of changed files matches any of filter rules
- For each filter it sets output variable with name `${FILTER_NAME}_count` to the count of matching files.
- If enabled, for each filter it sets output variable with name `${FILTER_NAME}_files`. It will contain list of all files matching the filter.
- `changes` - JSON array with names of all filters matching any of changed files.

# Examples

## Conditional execution

<details>
  <summary>Execute <b>step</b> in a workflow job only if some file in a subfolder is changed</summary>

```yaml
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: |
          backend:
            - 'backend/**'
          frontend:
            - 'frontend/**'

    # run only if 'backend' files were changed
    - name: backend tests
      if: steps.filter.outputs.backend == 'true'
      run: ...

    # run only if 'frontend' files were changed
    - name: frontend tests
      if: steps.filter.outputs.frontend == 'true'
      run: ...

    # run if 'backend' or 'frontend' files were changed
    - name: e2e tests
      if: steps.filter.outputs.backend == 'true' || steps.filter.outputs.frontend == 'true'
      run: ...
```
</details>

<details>
  <summary>Execute <b>job</b> in a workflow only if some file in a subfolder is changed</summary>

```yml
jobs:
  # JOB to run change detection
  changes:
    runs-on: ubuntu-latest
    # Set job outputs to values from filter step
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
    # For pull requests it's not necessary to checkout the code
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: |
          backend:
            - 'backend/**'
          frontend:
            - 'frontend/**'

  # JOB to build and test backend code
  backend:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - ...

  # JOB to build and test frontend code
  frontend:
    needs: changes
    if: ${{ needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - ...
```
</details>

<details>
<summary>Use change detection to configure matrix job</summary>

```yaml
jobs:
  # JOB to run change detection
  changes:
    runs-on: ubuntu-latest
    outputs:
      # Expose matched filters as job 'packages' output variable
      packages: ${{ steps.filter.outputs.changes }}
    steps:
    # For pull requests it's not necessary to checkout the code
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: |
          package1: src/package1
          package2: src/package2

  # JOB to build and test each of modified packages
  build:
    needs: changes
    strategy:
      matrix:
        # Parse JSON array containing names of all filters matching any of changed files
        # e.g. ['package1', 'package2'] if both package folders contains changes
        package: ${{ fromJSON(needs.changes.outputs.packages) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - ...
```
</details>

## Change detection workflows

<details>
  <summary><b>Pull requests:</b> Detect changes against PR base branch</summary>

```yaml
on:
  pull_request:
    branches: # PRs to following branches will trigger the workflow
      - master
      - develop
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        filters: ... # Configure your filters
```
</details>

<details>
  <summary><b>Feature branch:</b> Detect changes against configured base branch</summary>

```yaml
on:
  push:
    branches: # Push to following branches will trigger the workflow
      - feature/**
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      with:
        # This may save additional git fetch roundtrip if
        # merge-base is found within latest 20 commits
        fetch-depth: 20
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        base: develop # Change detection against merge-base with this branch
        filters: ... # Configure your filters
```
</details>

<details>
  <summary><b>Long lived branches:</b> Detect changes against the most recent commit on the same branch before the push</summary>

```yaml
on:
  push:
    branches: # Push to following branches will trigger the workflow
      - master
      - develop
      - release/**
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        # Use context to get branch where commits were pushed.
        # If there is only one long lived branch (e.g. master),
        # you can specify it directly.
        # If it's not configured, the repository default branch is used.
        base: ${{ github.ref }}
        filters: ... # Configure your filters
```
</details>

<details>
  <summary><b>Local changes:</b> Detect staged and unstaged local changes</summary>

```yaml
on:
  push:
    branches: # Push to following branches will trigger the workflow
      - master
      - develop
      - release/**
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

      # Some action which modifies files tracked by git (e.g. code linter)
    - uses: johndoe/some-action@v1

      # Filter to detect which files were modified
      # Changes could be for example automatically committed
    - uses: dorny/paths-filter@v2
      id: filter
      with:
        base: HEAD
        filters: ... # Configure your filters
```
</details>

## Advanced options

<details>
  <summary>Define filter rules in own file</summary>

```yaml
- uses: dorny/paths-filter@v2
      id: filter
      with:
        # Path to file where filters are defined
        filters: .github/filters.yaml
```
</details>

<details>
  <summary>Use YAML anchors to reuse path expression(s) inside another rule</summary>

```yaml
- uses: dorny/paths-filter@v2
      id: filter
      with:
        # &shared is YAML anchor,
        # *shared references previously defined anchor
        # src filter will match any path under common, config and src folders
        filters: |
          shared: &shared
            - common/**
            - config/**
          src:
            - *shared
            - src/**
```
</details>

<details>
  <summary>Consider if file was added, modified or deleted</summary>

```yaml
- uses: dorny/paths-filter@v2
      id: filter
      with:
        # Changed file can be 'added', 'modified', or 'deleted'.
        # By default the type of change is not considered.
        # Optionally it's possible to specify it using nested
        # dictionary, where type(s) of change composes the key.
        # Multiple change types can be specified using `|` as delimiter.
        filters: |
          shared: &shared
            - common/**
            - config/**
          addedOrModified:
            - added|modified: '**'
          allChanges:
            - added|deleted|modified: '**'
          addedOrModifiedAnchors:
            - added|modified: *shared
```
</details>


## Custom processing of changed files

<details>
  <summary>Passing list of modified files as command line args in Linux shell</summary>

```yaml
- uses: dorny/paths-filter@v2
  id: filter
  with:
    # Enable listing of files matching each filter.
    # Paths to files will be available in `${FILTER_NAME}_files` output variable.
    # Paths will be escaped and space-delimited.
    # Output is usable as command line argument list in Linux shell
    list-files: shell

    # In this example changed files will be checked by linter.
    # It doesn't make sense to lint deleted files.
    # Therefore we specify we are only interested in added or modified files.
    filters: |
      markdown:
        - added|modified: '*.md'
- name: Lint Markdown
  if: ${{ steps.filter.outputs.markdown == 'true' }}
  run: npx textlint ${{ steps.filter.outputs.markdown_files }}
```
</details>

<details>
  <summary>Passing list of modified files as JSON array to another action</summary>

```yaml
- uses: dorny/paths-filter@v2
  id: filter
  with:
    # Enable listing of files matching each filter.
    # Paths to files will be available in `${FILTER_NAME}_files` output variable.
    # Paths will be formatted as JSON array
    list-files: json

    # In this example all changed files are passed to following action to do
    # some custom processing.
    filters: |
      changed:
        - '**'
- name: Lint Markdown
  uses: johndoe/some-action@v1
  with:
    files: ${{ steps.filter.outputs.changed_files }}
```
</details>

# See also
- [test-reporter](https://github.com/dorny/test-reporter) - Displays test results from popular testing frameworks directly in GitHub

# License

The scripts and documentation in this project are released under the [MIT License](https://github.com/dorny/paths-filter/blob/master/LICENSE)
