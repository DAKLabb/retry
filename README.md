# Overview
While it has been possible to automatically retry workflows/jobs/steps in other CI frameworks for years (I have clear memories of doing this in GitLab in 2016), it is still not so easy in GitHub. While there are a number of potential approach to this (see [my notes](https://github.com/DAKLabb/gh-actions?tab=readme-ov-file#retrying-on-failure) on gh-actions), this repo is focused on a methodology for re-running failed workflows.

Note: This work is based on [a comment](https://github.com/orgs/community/discussions/67654#discussioncomment-8038649) from [nkraetzschmar](https://github.com/nkraetzschmar) in the GH community.

## Rerunning a workflow on failure
Generally speaking, if a workflow is failing intermitently, it is best to try to fix the source of the failure. That said, if your action is consuming external services that may experience availability issues, you may not have that option. In this case, you can have a workflow step that exectutes conditionally to re-run the workflow.

My particular use-case for retying was due to flake from the GH API. I had a step, `check`, that would use the GH CLI to get information about a PR which sometimes failed (especially on larger PRs). If this step failed, the job would fail, and all subsequent jobs would be skipped as this information was needed as an input to them.

I added a new step (shown below) that would trigger a re-run of the workflow if this step failed.
```yaml
      - name: retry-on-failure
         if: failure() && steps.check.outcome == 'failure' && fromJSON(github.run_attempt) < 3
         env:
           GH_REPO: ${{ github.repository }}
           GH_TOKEN: ${{ github.token }}
           GH_DEBUG: api
         run: gh workflow run rerun.yml -F run_id=${{ github.run_id }}
```

This step will only run if we are in a failure state, the prior job (`id`: `check`) failed, and we haven't re-run more than 2 tiems.

If the condition above is satisfied, this step will use the GH CLI to call the "Rerun Workflow" (shown below) to trigger the job. A sepearete workflow is used for this b/c a workflow cannot be rerun from itself (as discussed [here](https://github.com/orgs/community/discussions/67654#discussioncomment-7052837))

```yaml
name: Rerun Workflow

 on:
   workflow_dispatch:
     inputs:
       run_id:
         required: true

 jobs:
   rerun:
     runs-on: ubuntu-latest
     steps:
       - name: Rerun workflow with run_id ${{ inputs.run_id }}
         env:
           GH_REPO: ${{ github.repository }}
           GH_TOKEN: ${{ github.token }}
           GH_DEBUG: api
         run: |
           gh run watch ${{ inputs.run_id }} > /dev/null 2>&1
           gh run rerun ${{ inputs.run_id }} --failed
```

See it in action [here](https://github.com/DAKLabb/retry/actions).