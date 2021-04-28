# Infracost Jenkins

You can add infracost to your Jenkins' pipelines by adding an Infracost stage to your project's `Jenkinsfile`.

As mentioned in the [FAQ](https://www.infracost.io/docs/faq), **no** cloud credentials, secrets, tags or resource identifiers are sent to the Cloud Pricing API. That API does not become aware of your cloud spend; it simply returns cloud prices to the CLI so calculations can be done on your machine. Infracost does not make any changes to your Terraform state or cloud resources.

## Requirements:
* Setting the `jenkins-infracost-api-key` credential in the Jenkins' management panel.
* HTML publisher plugin (optional): For outputting the diff result in a HTML file.

## Example
Sample stage config:
```
stage('infracost-diff') {
    agent {
        docker {
            image 'infracost'
            args '--user=root --entrypoint='
        }
    }
    environment {
        INFRACOST_API_KEY = credentials('jenkins-infracost-api-key')
        IAC_PATH = 'terraform'
    }

    steps {
        sh '/scripts/ci/jenkins_diff.sh'

        publishHTML (target: [
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: true,
            reportDir: './',
            reportFiles: 'infracost_diff_output.html',
            reportName: "Infracost Diff Output"
        ])
    }
}
```

After a successful run you can find the diff output in the `Infracost Diff Output` sub-menu:

<img src="screenshot.png" width=557 alt="Example screenshot" />


## Environment variables

This section describes the most common environment variables. Other supported environment variables are described in the [this page](https://www.infracost.io/docs/integrations/environment_variables).

#### `INFRACOST_API_KEY`

**Required** To get an API key [download Infracost](https://www.infracost.io/docs/#installation) and run `infracost register`.

#### `IAC_PATH`
**Optional** Path to the Terraform directory or JSON/plan file. Either `IAC_PATH` or `CONFIG_FILE` is required.

#### `INFRACOST_TERRAFORM_BINARY`

**Optional** Used to change the path to the `terraform` binary or version, should be set to the path of the Terraform or Terragrunt binary being used in Atlantis. If you're using the `infracost/infracost-atlantis` image (which is based on the [`runatlantis/atlantis`](https://github.com/runatlantis/atlantis/blob/master/Dockerfile) image), you can set this to any of the supported paths, e.g. `/usr/local/bin/terraform0.12.30` or `/usr/local/bin/terraform0.13.6`.

#### `USAGE_FILE`

**Optional** Path to Infracost [usage file](https://www.infracost.io/docs/usage_based_resources#infracost-usage-file) that specifies values for usage-based resources, see [this example file](https://github.com/infracost/infracost/blob/master/infracost-usage-example.yml) for the available options.

#### `CONFIG_FILE`

**Optional** If you need to set the Terraform version on a per-repo basis, you can define that in a [config file](https://www.infracost.io/docs/config_file/) and set this input to its path. In such cases, the `usage_file` input cannot be used and must be defined in the config-file too.

#### `FAIL_CONDITION`

**Optional** A JSON string describing the condition that triggers the pipeline to fail, it's `percentage_threshold` field shoud be set. Example:
```
// The build will fail if the price change is above 20 percent.

{"percentage_threshold": 20}
```

## Contributing

Merge requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
