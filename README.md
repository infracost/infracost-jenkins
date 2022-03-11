# Infracost Jenkins

This project provides instructions for using Infracost in a Jenkins pipeline. This enables you to see cloud cost estimates for Terraform in pull requests. 💰

![Example GitHub screenshot](https://github.com/infracost/actions/blob/master/.github/assets/screenshot.png?raw=true)

## Quick start


1. Create a new credential in Jenkins' management panel, called `jenkins-infracost-api-key`, and enter your Infracost API key. To get an API key [download Infracost](https://www.infracost.io/docs/#quick-start) and run `infracost register`.

2. Create a new file at `Jenkinsfile` in your repo with the following content and update it to match your environment.

```
pipeline {
    agent any
    stages {
        stage('terraform') {
            agent {
                docker {
                    image 'hashicorp/terraform:latest'
                    args "--user=root --entrypoint=''"
                }
            }

            // IMPORTANT: add any required cloud credentials
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
            }

            steps {
                sh 'cd path/to/terraform'
                sh 'terraform init'
                sh 'terraform plan -out=tfplan.binary'
                sh 'terraform show -json tfplan.binary > plan.json'
                stash includes: 'plan.json', name: 'plan_json'
            }
        }

        stage('infracost') {
            agent {
                docker {
                    // Always use the latest 0.9.x version to pick up bug fixes and new resources.
                    // See https://www.infracost.io/docs/integrations/cicd/#docker-images for other options
                    image 'infracost/infracost:ci-0.9'
                    args "--user=root --entrypoint=''"
                }
            }

            // Set up any required credentials for posting the comment, e.g. GitHub token, GitLab token
            environment {
                INFRACOST_API_KEY = credentials('jenkins-infracost-api-key')
            }

            steps {
                unstash 'plan_json'

                // Generate Infracost JSON output, the following docs might be useful:
                // Multi-project/workspaces: https://www.infracost.io/docs/features/config_file
                // Combine Infracost JSON files: https://www.infracost.io/docs/features/cli_commands/#combined-output-formats
                sh 'infracost breakdown --path plan.json --format json --out-file infracost.json'

                // IMPORTANT: update this depending on which VCS provider you use and which plugin you are using.
                // Infracost comment supports GitHub, GitLab, Azure Repos and Bitbucket
                // You will need to update the environment variables below to match your environment.
                // For the full list of options, see: https://www.infracost.io/docs/features/cli_commands/#comment-on-pull-requests
                sh 'infracost comment github --path infracost.json --repo $GITHUB_REPO --pull-request $GITHUB_PULL_REQUEST_NUMBER --github-token $GITHUB_TOKEN'
            }
        }
    }
}
```

3. 🎉 That's it! Send a new pull request to change something in Terraform that costs money. You should see a merge request comment that gets updated, e.g. the 📉 and 📈 emojis will update as changes are pushed! Check the build Console Output and [this page](https://www.infracost.io/docs/troubleshooting/) if there are issues.

## Comment options

For different commenting options `infracost comment` command supports the following flags:

- `--behavior <value>`: Optional, defaults to `update`. The behavior to use when posting cost estimate comments. Must be one of the following:
  - `update`: Create a single comment and update it on changes. This is the "quietest" option. Pull request followers will only be notified on the comment create (not updates), and the comment will stay at the same location in the comment history.
  - `delete-and-new`: Delete previous cost estimate comments and create a new one. Pull request followers will be notified on each comment.
  - `hide-and-new`: Minimize previous cost estimate comments and create a new one. Pull request followers will be notified on each comment. Only supported by GitHub.
  - `new`: Create a new cost estimate comment. Pull request followers will be notified on each comment.
- `--pull-request <pull-request-number>`: Required when posting a comment on a pull request. Mutually exclusive with `--commit` flag.
- `--commit <commit-sha>`: Required when posting a comment on a commit. Mutually exclusive with `--pull-request` flag.
- `--tag <tag>`:  Optional. Customize hidden markdown tag used to detect comments posted by Infracost. This is useful if you have multiple workflows that post comments to the same pull request or commit and you want to avoid them over-writing each other.

Run `infracost comment --help` to see the the list of options or [see our docs](https://www.infracost.io/docs/features/cli_commands/#comment-on-pull-requests).

## Examples

We don't yet have examples for different use cases with Infracost for Jenkins, but we do have a selection of [examples for GitLab](https://gitlab.com/infracost/infracost-gitlab-ci/-/tree/master/examples#examples) that can be modified to work with Jenkins.

## Contributing

Issues and pull requests are welcome. Please create issues in [this repo](https://github.com/infracost/infracost) or [join our community Slack slack](https://www.infracost.io/community-chat), we are a friendly bunch and happy to help you get started :)

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
