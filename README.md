# Veracode Pipeline Scan as Pull Request comment

Veracode [Pipeline scan](https://help.veracode.com/r/c_about_pipeline_scan) allows its customers' to scan binary/code within GitHub workflows.
See: [Pipeline Scan Examples](https://help.veracode.com/r/r_pipeline_scan_examples).

Recently, veracode introduced an Action which allows customers with free GitHub accounts and Enterprise accounts with public repositories to upload the scan results directly to the `Security` tab.
See: [Veracode Static Analysis Pipeline scan and import of results using SARIF](https://github.com/marketplace/actions/veracode-static-analysis-pipeline-scan-and-sarif-import)

For customers who are using Enterprise Account, the `Security` tab is only available on private repositories when the [GitHub Advanced Security](https://docs.github.com/en/github/getting-started-with-github/about-github-advanced-security) is included in the entrprise license.

Another alternative to display the Pipeline Scan result is to use existing Github scripts inside Workflow to send output to Pull request as a comment.

__See example:__

```yaml
  - name: Download the Pipeline Scanner
    uses: wei/curl@master
    with:
      args: -O https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip
  - name: Unzip the Pipeline Scanner
    run: unzip pipeline-scan-LATEST.zip
  - name: Run Pipeline Scanner
    id: pipeline-scan
    continue-on-error: true
    run: java -jar pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_API_ID}}" --veracode_api_key "${{secrets.VERACODE_API_KEY}}" --file "<Archive to Scan>" --fail_on_severity="Very High, High"
```

The above is similar to the documented example for Pipeline scan (with allowing to continue on failure).

In get the scan output, we will output it to a file. We can easily do it with a [Pipeline scan](https://help.veracode.com/r/c_about_pipeline_scan) build-in parameter __`-so`__ or __`--summary_output`__. (Check the documentation if you want to specify the output file name)

Our scan command will look as follow:
```yaml
  - name: Run Pipeline Scanner
    id: pipeline-scan
    continue-on-error: true
    run: java -jar pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_API_ID}}" --veracode_api_key "${{secrets.VERACODE_API_KEY}}" --so true --file "<Archive to Scan>" --fail_on_severity="Very High, High"
```   

The last set of commands will simply read the output file (default name: `results.txt`) and send it to the pull request comments.

We can do it by adding the folllowing to the workflow:
```yaml
  - id: get-comment-body
    run: |
      body=$(cat results.txt)
      body="${body//$'\n'/'<br>'}"
      echo "::set-output name=body1::$body"
  - uses: actions/github-script@v3
    with:
      github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      github.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: "${{ steps.get-comment-body.outputs.body1 }}"
      })
``` 
 
> Note - for those who are not familiar, the __`secrets.GITHUB_TOKEN`__ is automatically generated when the GitHub workflow is running. 
 
The result will be as in the following example:
<p align="center">
  <img src="https://github.com/lerer/veracode-pipeline-PR-comment/blob/main/pull-request-comment.png?raw=true" width="600px" alt="Pipeline scan output in GitHub comment"/>
</p>


-----

The full `Pipeline scan` workflow:
```yaml
  - name: Download the Pipeline Scanner
    uses: wei/curl@master
    with:
      args: -O https://downloads.veracode.com/securityscan/pipeline-scan-LATEST.zip
  - name: Unzip the Pipeline Scanner
    run: unzip pipeline-scan-LATEST.zip
  - name: Run Pipeline Scanner
    id: pipeline-scan
    continue-on-error: true
    run: java -jar pipeline-scan.jar --veracode_api_id "${{secrets.VERACODE_API_ID}}" --veracode_api_key "${{secrets.VERACODE_API_KEY}}" --so true --file "<Archive to Scan>" --fail_on_severity="Very High, High"
  - id: get-comment-body
    run: |
      body=$(cat results.txt)
      body="${body//$'\n'/'<br>'}"
      echo "::set-output name=body1::$body"
  - uses: actions/github-script@v3
    with:
      github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      github.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: "${{ steps.get-comment-body.outputs.body1 }}"
      })
```

