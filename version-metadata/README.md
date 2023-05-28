# Version Metadata

This GitHub action checks what the current version number in the package.json file (location can be specified) is, if it changed since the last check and which files were updated in the process.

Using this you can easily automate publishing to a package registry such as NPM or [GPR](https://github.com/features/packages).

This action only computes metadata and doesn't push git tags, publishes a package, creates github releases, etc.


## Usage

### GitHub Workflow

You have to set up a step like this in your workflow (this assumes you've already [checked out](https://github.com/actions/checkout) your repo):

```yml
- id: version # This will be the reference for getting the outputs.
  uses: Quantco/ui-actions/version-metadata@v1 # You can choose the version/branch you prefer.

  with:
    # You can use this to indicate a custom path to your `package.json`. If you keep
    # your package file in the root directory (which is the usual approach) you can
    # omit this. For monorepos something like `./packages/lib/package.json` might be
    # what you want.
    # Default: package.json
    file: ./lib/package.json

    # Needed for both private and public repos as the GitHub api client needs a token
    # to be instantiated.
    token: ${{ secrets.GITHUB_TOKEN }}
```

Let's assume you just merged a pull request into main in which you did the following things:

- commit A
  - increment version from `1.2.2` to `1.2.3`
  - change 3 files in `lib/src/`
- commit B
  - change 2 files in `lib/src`
- commit C
  - increment version from `1.2.3` to `1.2.4`
  - change 3 files in `lib/src/`

The action will output the following:

```yml
changed: true
oldVersion: '1.2.2'
newVersion: '1.2.4'
type: 'patch'
changedFiles: (stringified JSON object)
  all: [ ... ]
  added: [ ... ]
  modifed: [ ... ]
  renamed: [ ... ]
  removed: [ ... ]
changes: (stringified JSON object)
  - { oldVersion: '1.2.2', newVersion: '1.2.3', type: 'patch', commit: 'A' }
  - { oldVersion: '1.2.3', newVersion: '1.2.4', type: 'patch', commit: 'C' }
commitResponsible: 'C'
commitBase: '~A' # SHA of A's **parent**, i.e. the commit before A (TODO: what does this mean for merge commits with 2 parents?)
commitHead: 'C'
json: "{ ... }" # stringified JSON object with all the above properties
```


### Outputs

- `changed`: either "true" or "false", indicates whether the version has changed.
- `oldVersion`: version before changes, current version if nothing changed
- `newVersion`: version after changes, current version if nothing changed
- `type`: type of change (major, minor, patch, pre-release)
- `changes`: array of changes (see below)
- `changedFiles`: categorized list of changed files (see below)
- `commitBase`: commit SHA of the base commit (previous head before pushing / merging new commits)
- `commitHead`: commit SHA of the head commit
- `json`: stringified JSON object with all the above properties

> `type` is only available if `changed` is "true".

`changedFiles` is an object with the following properties:

- all
- added
- modified
- renamed
- removed

each being an array of strings with the relative path of the changed files.

`changes` is an array of objects with the following properties:

- `oldVersion`: version before changes
- `newVersion`: version after changes
- `type`: type of change (major, minor, patch, pre-release)
- `commit`: commit SHA

It contains an entry for each time the version number changed for the commits considered (base and head, base being the previous head before pushing the new commits).

Note that the output might be a bit confusing if multiple merge commits of intertwined pull requests (time-wise) are involved at once.
This is due to how checking the file contents for each commit works, merge commits are fully ignored.

To access these outputs, you need to access the context of the step you previously set up: you can find more info about steps contexts [here](https://help.github.com/en/articles/contexts-and-expression-syntax-for-github-actions#steps-context).

With step id `version` you'll find the outputs at `steps.version.outputs.OUTPUT_NAME`.

```yml
- name: Check if version has been updated
  id: version
  uses: Quantco/ui-actions/version-metadata@v1

- if: steps.version.outputs.changed == 'true'
  run: |
    echo "New version is ${{ steps.version.outputs.newVersion }}"
    echo "Previous version was ${{ steps.version.outputs.oldVersion }}"

- if: steps.version.outputs.changed == 'false'
  run: 'echo "Version has not changed"'
```

## Support for repos without `package.json`

If you want to use this action in a non-`node` project, you can use the `file` input in combination with the `extractor` or `regex` input to specify a different file to check for version changes.
The `extractor` should point to an executable which outputs the version number to stdout given the file contents as input.
Alternatively, you can use the `regex` input to specify a regex which will be used to extract the version number from the file contents.

### `regex` example

Let's assume you have a file `version.yml` in the root directory of your repo which contains the version number in the following format:

```yml
package:
  version: 1.2.3
```

Then you could use the following configuration:

```yml
- name: Check if version has been updated
  id: version
  uses: Quantco/ui-actions/version-metadata@v1
  with:
    file: ./version.yml
    regex: 'version: (.*)'
```

### `extractor` example

If you have a more complex versioning scheme, you can write a small script which outputs the version number to stdout.

Let's assume you have a file `version.py` which contains the version number in the following format:

```py
...
version_info = (0, 2, 1)
...
```

Then you could write the following script `read_version` that extracts `0.2.1` from the file contents:

```bash
#!/usr/bin/env bash

# find the correct line, then get everything between the parentheses and replace ", " with "."
echo $(cat $1 | grep "version_info =" | sed "s/^.*(\([^()]*\)).*$/\1/" | sed "s/, /./g")
```

You can then use this script in the action:

```yml
- name: Check if version has been updated
  id: version
  uses: Quantco/ui-actions/version-metadata@v1
  with:
    file: ./version.py
    extractor: ./read_version
```

## Examples

```yml
# checkout, setup-node, etc. omitted

- name: Check if version has been updated
  id: version
  uses: Quantco/ui-actions/version-metadata@v1

# if version was manually incremented publish it
- name: Publish to NPM
  if: steps.version.outputs.changed == 'true'
  run: |
    npm publish
  with:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }} # needed for GitHub Package Registry, can omit otherwise

# You can use this to determine if auto-incrementing the version and publishing is useful
- name: Output changed files
  run: |
    echo "Changed files: ${{ fromJSON(steps.version.outputs.changedFiles).all }}"
```


## Local testing

In order to test this locally you can use the `test.sh` script.
It sets a few environment variables which are used by `@actions/core` to mock the GitHub API.
Change these variables to your liking.

```sh
INPUT_TOKEN="<TOKEN>" ./test.sh
```

The `MOCKING` environment variable is checked by `src/index.ts` to determine whether to use the mocked API or the real one.

> **Hint**: if you just want to see the json output you can use
> ```sh
> INPUT_TOKEN="<TOKEN>" ./test.sh | grep -o '^::set-output name=json::.*$' | sed 's/::set-output name=json:://g' | jq
> ```


## License

This action is distributed under the MIT license, check the [license](../LICENSE) for more info.
