[![GitHub license file](https://img.shields.io/github/license/turing85/publish-report)](https://github.com/turing85/publish-report/blob/main/LICENSE)

# Github action to publish reports

This action allows us to publish reports as github-actions check and produce comments on Pull requests with a summary of the report.

By identifying comments through `comment-header` (which will be transformed to an invisible `html` comment, containing the `comment-header` to identify comments controlled by this action), we can update an existing comment, adding more information as new reports get available over the run of a workflow.

## Components
This action is a composite action, it uses the following actions:

<table>
  <tr>
    <th>Action</th>
  </tr>

  <tr>
  <td>

[`actions/checkout@v4`][checkout]

  </td>
  </tr>

  <tr>
  <td>

[`marocchino/sticky-pull-request-comment@v2`][comment]

  </td>
  </tr>

  <tr>
  <td>

[`actions/download-artifact@v4`][download]

  </td>
  </tr>

  <tr>
  <td>

[`phoenix-actions/test-reporting@v15`][report]

  </td>
  </tr>

<tr>
  <td>

[`andymckay/cancel-action@0.5`][cancel]

  </td>
  </tr>
</table>

## Quick reference
### Permission setup (for personal repositories)
```yaml
name: My Workflow
...
permissions:
  actions: write       # Necessary to cancel workflow executions
  checks: write        # Necessary to write reports
  pull-requests: write # Necessary to comment on PRs
...
```

### Post initial comment
This snippet will post a comment with the given `comment-header` to the PR that triggered the workflow execution.
Any existing comments with the same `comment-header` will be hidden as `OUTDATED`:

```yaml
jobs:
  ...
  recreate-comment:
    runs-on: ubuntu-latest

    steps:
      - name: Publish Report
        uses: turing85/publish-report@v2
        with:
          checkout: 'true'
          comment-header: my-comment-header
          comment-message-recreate: Hello
          recreate-comment: true
  ...
```

### Generate test report and append message to existing comment
The report will be generated from the files selected by the glob pattern in `pattern-path`.
The name of the report will be `report-name` + `" Report"`.

If all tests succeeded, the message `comment-message-success` will be appended to the comment with the given `comment-header`.
If tests failed, the message in `comment-message-failure` will be appended to the comment with the given `comment-header`.

We set the execution of the `Publish Report` step to `if: ${{ always() }}` so that it is also executed when tests fail (and we get a report).

```yaml
jobs:
  ...
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Git checkout
        uses: actions/checkout@v4
      ...
      - name: Run Tests
        ...
        continue-on-error: true # the "publish" step will fail, so we get a report when tests failed as well
        ...
      ...
      - name: Publish Report
        uses: turing85/publish-report@v2
        if: ${{ always() }}
        with:
          # cancel-workflow-on-error: 'false' # If we do not want to cancel the whole workflow execution on error
          # checkout: 'true' # not needed; project is already checked out 
          comment-header: my-comment-header
          comment-message-success: |
            YAY! {0} passed!  
            
            {1} tests were successful, {2} tests failed, {3} test were skipped.
            
            The report can be found [here]({4}).

          comment-message-failure: |
            On no! {0} failed!  

            {1} tests were successful, {2} tests failed, {3} test were skipped.

            The report can be found [here]({4}).
          report-fail-on-error: true # to fail when tests failed
          report-name: Tests
          report-path: '**/target/surefire-reports/TEST*.xml'
          report-reporter: java-junit
  ...
```

### Generate test report from a downloaded artifact and append message to existing comment
When we have a scenario where we cannot or do not want the report generation the same job as the test, we can upload the test artifacts via [`actions/upload-artifact`][upload].
We can then execute report generation in a separate step, downloading said test artifacts.

Notice that the `name` and `path` used in action `actions/upload-artifact` correlates with `download-artifact-name` and `report-path` in action `turing/publish-report`.

We should configure the `test` job so that it does not fail when tests fail.
This guarantees  that the `test-report` job is executed, and can fail (as long as `report-fail-on-error` is not set to `'false'`).

```yaml
...
jobs:
  ...
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Git checkout
        uses: actions/checkout@v4
      ...
      - name: Run Tests
        ...
      ...
      - name: Upload test artifacts
        uses: actions/upload-artifact@v4
        if: ${{ always() }}
        with:
          name: test-report
          path: '**/target/*-reports/TEST*.xml'
          if-no-files-found: error
          retention-days: 2

  test-report:
    runs-on: ubuntu-latest
    
    needs:
      ...
      - test
      ...

    steps:
      - name: Publish Report
        uses: turing85/publish-report@v2
        with:
          ...
          checkout: true
          ...
          download-artifact-name: test-report
          report-path: '**/target/surefire-reports/TEST*.xml'
          ...
  ...
```

### Support for fork PRs
When forks provide PRs, the corresponding workflow runs are limited in what they can do. This is done for security reasons. For details, please read ["Keeping your GitHub Actions and workflows secure Part 1: Preventing pwn requests" (`securitylab.github.com`)][pwn]. Two consequences are that we cannot

- comment on PRs, and
- upload test reports.

This is where the `pull_request_target` and `workflow_run` events come into play.

The action is designed to be used with the `pull_request_target` and `workflow_run` events. Often times, the same workflow file is used to build the default branch, as well as PRs. For a good user experience, the extension tries to "figure out" when the jobs that comment on PRs should be run. 
Reports are generated and uploaded whenever `recreate-comment` is not `'true'`. The action tries to update the pull request comment if and only if:

- `inputs.comment-enabled` is `'true'`, and
- `inputs.recreate-comment` is not `'true'`, and
  - either `github.event_name` is `'pull_request'` or `'pull_request_target'`,
  - or `github.event.workflow_run.event` is `'pull_request'`

The recommendation is to create three workflows:

- one workflow for the initial comment,
- the main workflow for the actual build, and
- one workflow to update the comment wit the test results.

The initial workflows may look something like this:
```yaml
name: Comment on PR

on:
  pull_request_target:
    ...

permissions:
  pull-requests: write

jobs:
  comment:
    runs-on: ubuntu-latest

    steps:
      - name: (Re)create comment
        uses: turing85/publish-report@v2
        with:
          github-token: ${{ github.token }}
          comment-message-recreate: |
            ## 🚦Reports 🚦
            Reports will be posted here as they get available.
          comment-message-pr-number: ${{ github.event.number }}
          recreate-comment: true
```

You might notice that we override the `comment-message-recreate`. We will discuss this later.

The workflow to update the comment could look like this:
```yaml
name: Build report

on:
  workflow_run:
    workflows:
      - "build"
    types:
      - completed

permissions:
  actions: write
  checks: write
  pull-requests: write

jobs:
  report:
    if: ${{ github.event.workflow_run.event == 'pull_request' }}
    runs-on: ubuntu-latest

    steps:
      - name: Download PR number
        uses: actions/download-artifact@v4
        with:
          github-token: ${{ github.token }}
          name: pr-number
          run-id: ${{ github.event.workflow_run.id }}

      - name: Set PR number
        id: get-pr-number
        run: |
          echo "pr-number=$(cat pr-number.txt)" | tee "${GITHUB_OUTPUT}"
          rm -rf pr-number.txt

      - name: Publish reports
        uses: turing85/publish-report@feature/run-id-and-pr-number
        with:
          comment-message-pr-number: ${{ steps.get-pr-number.outputs.pr-number }}
          download-artifact-name: test-reports
          download-artifact-run-id: ${{ github.event.workflow_run.id }}
          report-name: My Tests
          report-path: '**/target/**/TEST*.xml'
```

We use an artifact named `pr-number` here. Since we use a `workflow_run`, we do not know anything of the pull request. Thus, we need some support from the actual build pipeline. It must create an artifact `pr-number` that contains a file `pr-number.txt`. the content of this file should be the number of the pull request.

The necessary steps to generate this artifact in the actual build workflow may look like this:
```yaml
name: Build

on:
  pull_request:
    branches:
      - main
    ...
  push:
    branches:
      - main
    ...

permissions:
  actions: write
  checks: write
  pull-requests: write


jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      ...
      (build the application, generate the test artifacts)
      ...
      - name: Get PR number
        id: get-pr-number
        if: ${{ always() }}
        run: |
          echo "${{ github.event.number }}" > "pr-number.txt"

      - name: Upload PR number
        uses: actions/upload-artifact@v4
        if: ${{ always() }}
        with:
          name: ${{ env.PR_NUMBER_ARTIFACT_NAME }}
          path: pr-number.txt
```

Now on the point why we override the `comment-message-recreate`. The default value of this variable is:

```
## 🚦Reports for run [#${{ github.run_number }}](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})🚦
Reports will be posted here as they get available.
```

This text contains a link to the workflow run associated with this comment. While very convenient, we are not able to update the comment from the main workflow. The workflow for the initial comment runs in parallel to the main workflow, so there is no way to "figure out" the run id or run number.

The workflow that updates the comment could of course add the link to the comment. However, at this point in time, the run is already over

### Complex example
For a complex example please take a look at the [workflow of `github.com/turing85/advent-of-code-2022`][advent-workflow]

## Inputs

<table>
  <tr>
  <th>Name</th><th>semantics</th><th>Required?</th><th>default</th>
  </tr>

  <tr>
  <th colspan="4" align="center">General Inputs</th>
  </tr> 

  <tr>
  <td>

`github-token`
  </td>
  <td>
The github-token to use.
  </td>
  <td>✅</td>
  <td>

`${{ github.token }}`
  <tr>
  <td>

`cancel-workflow-on-error`
  </td>
  <td>
Whether the entire current workflow should be cancelled on error (i.e. when tests failed).
  </td>
  <td>✅</td>
  <td>

`'false'`
  </td>
  </tr>

  <tr>
  <td>

`checkout`
  </td>
  <td>Whether a checkout should be performed</td>
  <td>✅</td>
  <td>

`'false'`
  </td>
  </tr>

  <tr>
  <th colspan="4" align="center">Comment-related Inputs</th>
  </tr> 

  <tr>
  <td>

`override-comment`
  </td>
  <td>Overrides the comment on a PR.</td>
  <td>✅</td>
  <td>

`'false'`
  </td>
  </tr>

  <tr>
  <td>

`recreate-comment`
  </td>
  <td>Triggers the (re-)creation of the comment in a PR, that is updated with the reports.</td>
  <td>✅</td>
  <td>

`'false'`
  </td>
  </tr>

  <tr>
  <td>

`comment-enabled`
  </td>
  <td>Whether a comment on the PR should be posted.</td>
  <td>✅</td>
  <td>

`'true'`
  </td>
  </tr>

  <tr>
  <td>

`comment-header`
  </td>
  <td>
The header to identify the PR comment.
This is an invisible tag on the comment.
  </td>
  <td>✅</td>
  <td>

`reports`
  </td>
  </tr>

  <tr>
  <td>

`comment-message-failure`
  </td>
  <td>
Message appended to the comment posted on the PR after the tests failed.

The message can be templated for replacement.
The [format feature of github-expressions][github-expressions] is used to replace placeholders.
The following placeholder-mapping applies:
- `{0}` is `inputs.report-name`
- `{1}` is the number of successful tests
- `{2}` is the number of failed tests
- `{3}` is the number of skipped tests
- `{4}` is the URL to the HTML-Report
  </td>
  <td>✅</td>
  <td>

```markdown
<details>
  <summary><h3>😔 {0} failed</h3></summary>

  | Passed | Failed | Skipped |
  |--------|--------|---------|
  | ✅ {1} | ❌ {2} | ⚠️ {3}   |

  You can see the report [here]({4}).
</details>
```
  </td>
  </tr>

  <tr>
  <td>

`comment-message-override`
  </td>
  <td>
The new comment message. Notice that the old comment will be completely overridden.
  </td>
  <td>✅</td>
  <td>

```markdown
## 🚦Reports for run [#${{ github.run_number }}](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})🚦
Reports will be posted here as they get available.
```
  </td>
  </tr>

  <tr>
  <td>

`comment-message-recreate`
  </td>
  <td>
Initial text for the comment posted on the PR.
Subsequent messages will be appended.
  </td>
  <td>✅</td>
  <td>

```markdown
## 🚦Reports for run [#${{ github.run_number }}](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})🚦
Reports will be posted here as they get available.
```
  </td>
  </tr>

  <tr>
  <td>

`comment-message-success`
  </td>
  <td>
Message appended to the comment posted on the PR after the tests succeed.

The message can be templated for replacement.
The [format feature of github-expressions][github-expressions] is used to replace placeholders.
The following placeholder-mapping applies:
- `{0}` is `inputs.report-name` 
- `{1}` is the number of successful tests
- `{2}` is the number of failed tests
- `{3}` is the number of skipped tests
- `{4}` is the URL to the HTML-Report
  </td>
  <td>✅</td>
  <td>
```markdown
<details>
  <summary><h3>🥳 {0} passed</h3></summary>

  | Passed | Failed | Skipped |
  |--------|--------|---------|
  | ✅ {1} | ❌ {2} | ⚠️ {3}   |

  You can see the report [here]({4}).
</details>
```
  </td>
  </tr>

  <tr>
  <td>

`comment-message-pr-number`
  </td>
  <td>The PR number to which the comment should be written.</td>
  <td>✅</td>
  <td>

`${{ github.event.number }}`
  </td>
  </tr>

  <tr>
  <th colspan="4" align="center">Artifact-related Inputs</th>
  </tr> 
  <tr>

  <tr>
  <td>

`download-artifact-name`
  </td>
  <td>The name of the artifact to download.</td>
  <td>✅</td>
  <td>

`''`
  </td>
  </tr>

  <tr>
  <td>

`download-artifact-pattern`
  </td>
  <td>The pattern of the artifact to download.</td>
  <td>✅</td>
  <td>

`''`
  </td>
  </tr>

  <tr>
  <td>

`download-artifact-merge-multiple`
  </td>
  <td>If artifacts should be merged if multiple artifacts are downloaded.</td>
  <td>✅</td>
  <td>

`'false'`
  </td>
  </tr>

  <tr>
  <td>

`download-artifact-run-id`
  </td>
  <td>The run-id for which the artifact should be downloaded.</td>
  <td>✅</td>
  <td>

`${{ github.run_id }}`
  </td>
  </tr>

  <tr>
  <th colspan="4" align="center">Report-related Inputs</th>
  </tr> 
  <tr>

  <tr>
  <td>

`report-fail-on-error`
  </td>
  <td>Whether an error in a test should fail the step.</td>
  <td>✅</td>
  <td>

`'true'`
  </td>
  </tr>

  <tr>
  <td>

`report-list-suites`
  </td>
  <td>
Limits which test suites are listed.
Supported options:
- all
- failed
  </td>
  <td>✅</td>
  <td>

`all`
  </td>
  </tr>

  <tr>
  <td>

`report-list-tests`
  </td>
  <td>
Limits which test cases are listed.
Supported options:
- all
- failed
- none
  </td>
  <td>✅</td>
  <td>

`all`
  </td>
  </tr>

  <tr>
  <td>

`report-name`
  </td>
  <td>
The name of the report.
The Text "Report" will be appended to form the report name that is attached to the check.
So if we pass "JUnit" as report-name, the corresponding report will be called "JUnit Report".
  </td>
  <td>✅</td>
  <td>

`JUnit`
  </td>
  </tr>

  <tr>
  <td>

`report-only-summary`
  </td>
  <td>
Allows you to generate only the summary.

If enabled, the report will contain a table listing each test results file and the number of passed, failed, and skipped tests.

Detailed listing of test suites and test cases will be skipped.
  </td>
  <td>✅</td>
  <td>

`false`
  </td>
  </tr>

  <tr>
  <td>

`report-path`
  </td>
  <td>A glob path to the report files.</td>
  <td>✅</td>
  <td>

`**/*.xml`
  </td>
  </tr>

  <tr>
  <td>

`report-reporter`
  </td>
  <td>
Format of test results.
Supported options:
- dart-json
- dotnet-trx
- flutter-json
- java-junit
- jest-junit
- mocha-json
- mochawesome-json
  </td>
  <td>✅</td>
  <td>

`java-junit`
  </td>
  </tr>
</table>

## Outputs

<table>
  <tr>
  <th>Name</th><th>Semantics</th><th>type</th>
  </tr>

  <tr>
  <td>

`report-url`
  </td>
  <td>URL to the report in workflow checks.</td>
  <td>

`string`
  </td>
  </tr>

  <tr>
  <td>

`tests-failed`
  </td>
  <td>Number of tests failed.</td>
  <td>

`number`
  </td>
  </tr>

  <tr>
  <td>

`tests-passed`
  </td>
  <td>Number of tests passed.</td>
  <td>

`number`
  </td>
  </tr>

  <tr>
  <td>

`tests-skipped`
  </td>
  <td>Number of tests skipped.</td>
  <td>

`number`
  </td>
  </tr>
</table>

## Showcase

Screenshots are taken from [this comment][pr-comment] and [this workflow run][workflow-run].

<table>
  <tr>
  <td>

![initial comment](images/initial-comment.png)
  </td>
  </tr>

  <tr><td style="text-align: center">Initial comment on a PR</td></tr>
</table>

<table>
  <tr>
  <td>

![first report](images/first-added-comment.png)
  </td>
  </tr>

  <tr><td style="text-align: center">First report added</td></tr>
</table>

<table>
  <tr>
  <td>

![second report](images/second-comment-added.png)
  </td>
  </tr>

  <tr><td style="text-align: center">Second report added</td></tr>
</table>

<table>
  <tr>
  <td>

![junit report](images/junit-report.png)
  </td>
  </tr>

  <tr><td style="text-align: center">JUnit Report</td></tr>
</table>

<table>
  <tr>
  <td>

![owasp report](images/owasp-report.png)
  </td>
  </tr>

  <tr><td style="text-align: center">OWASP report</td></tr>
</table>

## Contributors ✨

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/turing85"><img src="https://avatars.githubusercontent.com/u/32584495?v=4?s=100" width="100px;" alt="Marco Bungart"/><br /><sub><b>Marco Bungart</b></sub></a><br /><a href="https://github.com/turing85/publish-report/commits?author=turing85" title="Code">💻</a> <a href="#maintenance-turing85" title="Maintenance">🚧</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/gregoryduckworth"><img src="https://avatars.githubusercontent.com/u/2647926?v=4?s=100" width="100px;" alt="Greg Duckworth"/><br /><sub><b>Greg Duckworth</b></sub></a><br /><a href="https://github.com/turing85/publish-report/commits?author=gregoryduckworth" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://www.lorenzobettini.it"><img src="https://avatars.githubusercontent.com/u/1202254?v=4?s=100" width="100px;" alt="Lorenzo Bettini"/><br /><sub><b>Lorenzo Bettini</b></sub></a><br /><a href="#question-LorenzoBettini" title="Answering Questions">💬</a> <a href="#ideas-LorenzoBettini" title="Ideas, Planning, & Feedback">🤔</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification.
Contributions of any kind welcome!

## License

This project is licensed under the [Apache License 2.0][apacheLicense].
The license file can be found [here][license].

[checkout]: https://github.com/actions/checkout

[comment]: https://github.com/marocchino/sticky-pull-request-comment

[download]: https://github.com/actions/download-artifact

[report]: https://github.com/phoenix-actions/test-reporting

[cancel]: https://github.com/andymckay/cancel-action

[github-expressions]: https://docs.github.com/en/actions/learn-github-actions/expressions#format

[upload]: https://github.com/actions/upload-artifact

[advent-workflow]: https://github.com/turing85/advent-of-code-2022/blob/main/.github/workflows/build.yml

[pr-comment]: https://github.com/turing85/advent-of-code-2022/pull/46#issuecomment-1441025081

[pwn]: https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/

[workflow-run]: https://github.com/turing85/advent-of-code-2022/runs/11535320068

[apacheLicense]: http://www.apache.org/licenses/LICENSE-2.0

[license]: https://github.com/turing85/publish-report/blob/main/LICENSE
