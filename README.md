# playwright-loadbalancer
A GitHub action to provide a dynamic matrix based on the amount of test files changed in a pull request.

## Why this action?

Playwright provides an option to run tests on multiple runners using its sharding option as shown [here](https://playwright.dev/docs/test-sharding#github-actions-example). It also provides an option to run only tests that are changed in a pull request as shown [here](https://playwright.dev/docs/release-notes#--only-changed-cli-option). 

This creates an interesting use case where we may not be using our GitHub runners optimally when adding/updating test cases and running them using `npx playwright test --only-changed=origin/main` option. 

Lets look at this use case with an example. Lets assume that we have 60 test files and we need 6 maximum runners to finish all our tests within 5 minutes. However, for our pull requests, we use this command `npx playwright test --only-changed=origin/main` to run only changed tests. Imagine a situation where we only have one test file touched, with just a few test cases and with 'fullyParallel: false", with the setup shown in playwright example, we would still end up spinning 6 runners. Essentially wasting resources setting up all the dependent steps only to run tests in one runner and no tests in other runners.

It would be nice if we can calculate the ratio of changed files vs total files and based on that ratio provide a dynamic matrix that can be used to spin up runners. This action, does just that!

## Inputs

```yaml {"id":"01J2XFHJFST5N0A1651KZ5JCAT"}
inputs:
  max-runners:  
    description: 'maximum number of runners to be used'
    required: true
```

## Outputs

```yaml {"id":"01J2XFHJFST5N0A1651MMCD9FR"}
outputs:
  dynamic-matrix:
    description: "dynamic matrix to use"
    value: ${{ steps.set-matrix.outputs.dynamic_matrix }}

```

## Example usage

Below is an example that shows how to use this action.

```yaml {"id":"01J2NSXS32KV8TSMM4W64D9WMT"}
on:
  workflow_dispatch:

name: Get BrowserStack Result
jobs:
  browserstack_result:
    name: Get BrowserStack Result
    runs-on: ubuntu-latest
    outputs:
      build-id: ${{ steps.get_browserstack_result.outputs.build-id }}
      quality-gate-result: ${{ steps.get_browserstack_result.outputs.quality-gate-result }}
    steps:
    - uses: actions/checkout@v4

    - name: Get BrowserStack Result
      uses: PramodKumarYadav/playwright-loadbalancer@v10
      id: get_browserstack_result
      with:
        project-name: 'your-test-project-name'
        build-name: 'your-test-build-name'
        build-tags: 'your-custom-build-tag-name' 
        browserstack-username: ${{ secrets.BROWSERSTACK_USERNAME }}
        browserstack-access-key: ${{ secrets.BROWSERSTACK_ACCESS_KEY }}
        timeout-in-seconds: 60

    - name: Print BrowserStack Result in same job (and fail if quality gate failed)
      # Possible quality-gate-result values: 'passed', 'failed', 'Quality Gate Analysis in process. Please try again in a few minutes.
      run: |
          echo "Quality Gate Result: ${{ steps.get_browserstack_result.outputs.quality-gate-result }}"
          if [[ "${{ steps.get_browserstack_result.outputs.quality-gate-result }}" != "passed" ]]; then
            exit 1
          fi
  
  use_browserstack_result:
    name: Print BrowserStack Result in another job
    runs-on: ubuntu-latest
    needs: browserstack_result
    steps:
    - name: Use BrowserStack Result
      run: |
        echo "Build ID: ${{ needs.browserstack_result.outputs.build-id }}"
        echo "Quality Gate Result: ${{ needs.browserstack_result.outputs.quality-gate-result }}"
```

## Reference

To create and push new tags:

```sh {"id":"01J2XFHJFT1K765K3D5J6BDSSC"}
pramodyadav@Pramods-Laptop playwright-loadbalancer % git tag -a -m "add your message here" v1                   
pramodyadav@Pramods-Laptop playwright-loadbalancer % git push --follow-tags   
```