- 👋 Hi, I’m @n13l420
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
n13l420/n13l420 is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
name: User
on:
  push: "man"  
    paths-ignore:
      - 'doc/**'
      - '**/man/*'
      - '**.md'
      - '**.rdoc'
      - '**/.document'
      - '.*.yml'
  pull_request:
    # Do not use paths-ignore for required status checks
    # https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/troubleshooting-required-status-checks#handling-skipped-but-required-checks
  merge_group:
  
concurrency:
  group: ${{ github.workflow }} / ${{ startsWith(github.event_name, 'pull') && github.ref_name || github.sha }}
  cancel-in-progress: ${{ startsWith(github.event_name, 'pull') }}

permissions:
  contents: read

jobs:
  make:
    strategy:
      matrix:
        include:
          - test_task: check
          - test_task: test-all
            test_opts: --repeat-count=2
          - test_task: test-bundler-parallel
          - test_task: test-bundled-gems
          - test_task: check
            os: macos-12
          - test_task: check
            os: macos-13
      fail-fast: false

    env:
      GITPULLOPTIONS: --no-tags origin ${{ github.ref }}

    runs-on: ${{ matrix.os || (github.repository == 'ruby/ruby' && 'macos-arm-oss' || 'macos-14') }}

    if: >-
      ${{!(false
      || contains(github.event.head_commit.message, '[DOC]')
      || contains(github.event.head_commit.message, 'Document')
      || contains(github.event.pull_request.title, '[DOC]')
      || contains(github.event.pull_request.title, 'Document')
      || contains(github.event.pull_request.labels.*.name, 'Document')
      || (github.event_name == 'push' && github.actor == 'dependabot[bot]')
      )}}

    steps:
      - uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 # v4.1.7
        with:
          sparse-checkout-cone-mode: false
          sparse-checkout: /.github

      - name: Install libraries
        uses: ./.github/actions/setup/macos

      - uses: ./.github/actions/setup/directories
        with:
          srcdir: src
          builddir: build
          makeup: true
          clean: true
          dummy-files: ${{ matrix.test_task == 'check' }}
          # Set fetch-depth: 0 so that Launchable can receive commits information.
          fetch-depth: 10

      - name: make sure that kern.coredump=1
        run: |
          sysctl -n kern.coredump
          sudo sysctl -w kern.coredump=1
          sudo chmod -R +rwx /cores/

      - name: Run configure
        run: ../src/configure -C --disable-install-doc ${ruby_configure_args}

      - run: make prepare-gems
        if: ${{ matrix.test_task == 'test-bundled-gems' }}

      - run: make

      - name: Set test options for skipped tests
        run: |
          set -x
          TESTS="$(echo "${{ matrix.skipped_tests }}" | sed 's| |$$/ -n!/|g;s|^|-n!/|;s|$|$$/|')"
          echo "TESTS=${TESTS}" >> $GITHUB_ENV
        if: ${{ matrix.test_task == 'check' && matrix.skipped_tests }}

      - name: Set up Launchable
        uses: ./.github/actions/launchable/setup
        with:
          os: ${{ matrix.os || (github.repository == 'ruby/ruby' && 'macos-arm-oss' || 'macos-14') }}
          test-opts: ${{ matrix.test_opts }}
          launchable-token: ${{ secrets.LAUNCHABLE_TOKEN }}
          builddir: build
          srcdir: src
        continue-on-error: true

      - name: Set extra test options
        run: |
          echo "TESTS=$TESTS ${{ matrix.test_opts }}" >> $GITHUB_ENV
          echo "RUBY_TEST_TIMEOUT_SCALE=3" >> $GITHUB_ENV # With --repeat-count=2, flaky test by timeout occurs frequently for some reason
        if: matrix.test_opts

      - name: make ${{ matrix.test_task }}
        run: |
          ulimit -c unlimited
          make -s ${{ matrix.test_task }} ${TESTS:+TESTS="$TESTS"}
        timeout-minutes: 60
        env:
          RUBY_TESTOPTS: '-q --tty=no'
          TEST_BUNDLED_GEMS_ALLOW_FAILURES: ''
          PRECHECK_BUNDLED_GEMS: 'no'

      - name: make skipped tests
        run: |
          make -s test-all TESTS="${TESTS//-n!\//-n/}"
        env:
          GNUMAKEFLAGS: ''
          RUBY_TESTOPTS: '-v --tty=no'
          PRECHECK_BUNDLED_GEMS: 'no'
        if: ${{ matrix.test_task == 'check' && matrix.skipped_tests }}
        continue-on-error: ${{ matrix.continue-on-skipped_tests || false }}

      - uses: ./.github/actions/slack
        with:
          label: ${{ matrix.os || (github.repository == 'ruby/ruby' && 'macos-arm-oss' || 'macos-14') }} / ${{ matrix.test_task }}
          SLACK_WEBHOOK_URL: ${{ secrets.SIMPLER_ALERTS_URL }} # ruby-lang slack: ruby/simpler-alerts-bot
        if: ${{ failure() }}

      - name: Resolve job ID
        id: job_id
        uses: actions/github-script@main
        env:
          matrix: ${{ toJson(matrix) }}
        with:
          script: |
            const { data: workflow_run } = await github.rest.actions.listJobsForWorkflowRun({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: context.runId
            });
            const matrix = JSON.parse(process.env.matrix);
            const job_name = `${context.job}${matrix ? ` (${Object.values(matrix).join(", ")})` : ""}`;
            return workflow_run.jobs.find((job) => job.name === job_name).id;

  result:
    if: ${{ always() }}
    name: ${{ github.workflow }} result
    runs-on: macos-latest
    needs: [make]
    steps:
      - run: exit 1
        working-directory:
        if: ${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}

defaults:
  run:
    working-directory: build
