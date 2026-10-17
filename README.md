# TurnSignal for Playwright

GitHub Action for [TurnSignal for Playwright](https://turnsignal.ai/product?ref=marketplace): on every CI run, see **which Playwright tests your change broke**, which failures were **already on main**, which tests are **flaky**, and which **never reported** because a runner crashed or ran out of memory. On pull requests it keeps one comment up to date with that verdict.

It never changes your exit code. If TurnSignal is unreachable, your build is unaffected.

It can also review the **test changes in a pull request** without any account: it flags weakened assertions, changed expected values, disabled tests and added waits (often an AI "fix"), and says who or which coding agent made them. See [Review test changes](#review-test-changes-no-token).

Free during the public beta ([pricing](https://turnsignal.ai/pricing?ref=marketplace), [security](https://turnsignal.ai/security?ref=marketplace), [terms](https://turnsignal.ai/terms), [privacy](https://turnsignal.ai/privacy)). No repository access needed.

## Setup

1. Create a project at [app.turnsignal.ai](https://app.turnsignal.ai) and save its token as the repository secret `TURNSIGNAL_TOKEN`.
2. Add the reporter to your Playwright config:

```bash
npm i -D turnsignal
```

```ts
// playwright.config.ts
export default defineConfig({
  reporter: [['list'], ['turnsignal']],
});
```

3. Use the Action in your workflow:

```yaml
jobs:
  tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2]
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 22
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
        env:
          TURNSIGNAL_TOKEN: ${{ secrets.TURNSIGNAL_TOKEN }}
      # Sends anything the reporter could not deliver (crashed runner, network blip).
      - if: always()
        uses: sekharsdet/turnsignal-action@v1
        with:
          token: ${{ secrets.TURNSIGNAL_TOKEN }}

  report:
    needs: tests
    if: always()
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: sekharsdet/turnsignal-action@v1
        with:
          command: comment
          token: ${{ secrets.TURNSIGNAL_TOKEN }}
```

## Review test changes (no token)

`governance` compares the tests changed in a pull request with the base branch on the CI machine and keeps one PR comment up to date: what needs a human review, what is worth a look, and what is safe (locator or rename only). Nothing is uploaded and no TurnSignal account is needed.

```yaml
jobs:
  test-changes:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # needs the base branch history
      - uses: sekharsdet/turnsignal-action@v1
        with:
          command: governance
```

## Inputs

| Input | Default | What it does |
| --- | --- | --- |
| `token` | | TurnSignal project token (required for `upload` and `comment`) |
| `command` | `upload` | `upload` after the test step, `comment` in a final job, or `governance` on pull requests |
| `working-directory` | `.` | Folder with your `playwright.config` |
| `wait` | `60` | `comment`: seconds to wait for the run to finish |
| `github-token` | `github.token` | Used to post the PR comments |
| `api-url` | | Only for a self-hosted server |
| `version` | `0.6` | `turnsignal` npm version or range |

On fork pull requests (no write permission) the verdict goes to the job summary instead of a comment.

Docs: [turnsignal.ai/docs](https://turnsignal.ai/docs?ref=marketplace) · Questions: hello@turnsignal.ai

## License

MIT
