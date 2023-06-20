---
title: "Hunting vulnerabilies in pipelines"
date: 2023-06-20
draft: false
---

CI/CD pipelines are a crucial part of software development lifecycle. They allow you to automate building, testing and deploying your projects. And now in today's left shifting landscape, security has become an intergral part of these pipelines. We utilize various security tools within these pipelines to ensure that our applications are secure. 

But what about the security of the pipelines themselves? Can these pipelines even be vulnerable?

Unfortunately, the answer is yes. In this write-up I will discuss how I discovered a vulnerability within the pipelines used by my team.

## Understanding the vulnerable pipeline

My organization uses GitHub Actions to test all our projects and automate numerous manual processes. One such process is backporting pull requests (PRs) to different versions of a project.

```yaml
name: Pull Request Backporting

on:
  pull_request_target:
    types: [closed, labeled]

env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

jobs:
  compute-targets:
    if: ${{ github.event.pull_request.state == 'closed' && github.event.pull_request.merged }}
    runs-on: ubuntu-latest
    outputs:
      target-branches: ${{ steps.set-targets.outputs.targets }}
    env:
      LABELS: ${{ toJSON(github.event.pull_request.labels) }}
    steps:
      - name: Set target branches
        id: set-targets
        uses: kiegroup/kie-ci/.ci/actions/parse-labels@main
        with:
          labels: ${LABELS}
  
  backporting:
    if: ${{ github.event.pull_request.state == 'closed' && github.event.pull_request.merged && needs.compute-targets.outputs.target-branches != '[]' }}
    name: "[${{ matrix.target-branch }}] - Backporting"
    runs-on: ubuntu-latest
    needs: compute-targets
    strategy:
      matrix: 
        target-branch: ${{ fromJSON(needs.compute-targets.outputs.target-branches) }}
      fail-fast: false
    steps:
      - name: Backporting
        uses: kiegroup/kie-ci/.ci/actions/backporting@main
        with:
          target-branch: ${{ matrix.target-branch }}
```
&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; *The workflow configuration used for backporting PRs*  
&nbsp;  

After merging a pull request, we apply labels with a specific prefix to indicate which branches require the same changes. This triggers the workflow. First, the `compute-target` job extracts these branches using the `parse-label` action. Then, the extracted branches are passed to the `backporting` job which utilizes the `backporting` action to create pull requests across all the extracted branches.

## The vulnerability

Can you spot the problem here? *Hint:* It is not related to the use of `pull_request_target` workflow trigger.

Surprised? I was too. The immediate red flag that I noticed was the use of `pull_request_target` as it can be risky if not used properly (see [GitHub's advisory on it](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)). However, we were in fact using it safely in this case since we didn't check out the code and run any scripts. 

The problem lies within the `parse-label` action. It is vulnerable to a [**script injection attack**](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#understanding-the-risk-of-script-injections) when constructing a label that the action recognizes. This allows attackers to execute arbitrary code during the workflow run.

The action can be exploited by creating labels named in the following pattern:  

```
LABEL_PREFIX-$(COMMAND_TO_EXECUTE)
```

So an example label for this exploit would be `backport-$(curl www.google.com)` which would result in making a call to google.com (assuming that the action is using the default label prefix which is `backport-`).

```yaml
name: 'Parse Labels'
description: 'Extract target branches or refs from labels'

inputs:
  labels:
    description: "List of label objects to be parsed, e.g., github.event.pull_request.labels"
    required: true
  label-prefix:
    description: "Extract targets from labels matching this provided prefix"
    default: "backport-"
    required: false

outputs:
  targets:
    description: "Extracted targets"
    value: ${{ steps.extract-targets.outputs.targets }}

runs:
  using: 'composite'
  steps:
    - name: Fetch labels
      id: fetch-labels
      shell: bash
      run: |
        echo "Labels retrieved below"
        echo "${{ inputs.labels }}"
        filtered_labels="$(echo ${{ inputs.labels }} | jq -c 'map(select(.name | startswith("${{ inputs.label-prefix }}")))')"
        echo "filtered_labels = ${filtered_labels}"
        echo "FILTERED_LABELS=${filtered_labels}" >> $GITHUB_ENV

    - name: Extract targets
      id: extract-targets
      shell: bash
      run: |
        targets="$(echo $FILTERED_LABELS | jq -c '[.[] | .name | sub("${{ inputs.label-prefix }}"; "")]')"
        echo "targets=$(echo $targets)" >> $GITHUB_OUTPUT
    
    # The vulnerabile step
    - name: Printing extracted targets
      shell: bash
      run: echo "Extracted target branches ${{ steps.extract-targets.outputs.targets }}"
```
&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; *The workflow configuration for the parse-label action*  
&nbsp;  

### A step by step breakdown

The `Fetch labels` step iterates through all the labels taken from `github.event.pull_request.labels` and extracts any label with a name starting with the specified prefix (in this case, it was the default value - `backport-`). Each label is represented by an object with the following fields:

```json
{
    "color": string,
    "default": boolean,
    "description": string,
    "id": number,
    "node_id": string,
    "url": string,
    "name": string,
}
```

Next, the `Extract targets` step extracts out the `name` field from the filtered labels and removes the prefix from it. It then stores these extracted targets as a GitHub step output.

Finally, our injected script is executed in the `Printing extracted targets` step. Here the workflow simply echoes the extracted targets, which in our example would be `$(curl www.google.com)`. Thus, what actually ends up getting executed in this step is:
```bash
echo "Extracted target branches $(curl www.google.com)"
```
Bash then interprets this as exexcuting `curl www.google.com` and then printing "Extracted target branches" along with result of the `curl` command.

## Impact

Arbitrary code execution can lead to significant damage. However, the impact of this vulnerability was relatively limited becuase not all users can create new labels. To create labels for any repository within our organization, a certain level of privilege is required.

## Fix

The fix was rather simple - use an intermediate environment variable. This approach is the [recommeded solution by GitHub](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#using-an-intermediate-environment-variable). With this method, the value of the expression `${{ steps.extract-targets.outputs.targets }}` is stored in memory as a variable, and doesn't interact with the script generation process.

```yaml
    - name: Printing extracted targets
      env:
        # set untrusted input to intermediate env variable
        TARGETS: ${{ steps.extract-targets.outputs.targets }}
      shell: bash
      run: echo "Extracted target branches $TARGETS"
```
&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp;  *The faulty step after it was fixed*  
&nbsp;  

## Conclusion

In conclusion, securing your application alone is not sufficient. It is equally crucial to secure your CI/CD platforms. The vulnerability discovered in this pipeline highlights the need for implementing robust security measures for these platforms, just as we do for all our applications.