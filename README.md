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
        stage('infracost') {
            agent {
                docker {
                    // Always use the latest 0.10.x version to pick up bug fixes and new resources.
                    // See https://www.infracost.io/docs/integrations/cicd/#docker-images for other options
                    image 'infracost/infracost:ci-0.10'
                    args "--user=root --entrypoint=''"
                }
            }

            // Set up any required credentials for posting the comment, e.g. GitHub token, GitLab token
            environment {
                INFRACOST_API_KEY = credentials('jenkins-infracost-api-key')
            }

            steps {
                // Clone the base branch of the pull request (e.g. main/master) into a temp directory.
                sh 'git clone $GIT_URL --branch=$CHANGE_TARGET --single-branch /tmp/base'

                // Generate Infracost JSON file as the baseline.
                sh 'infracost breakdown --path=/tmp/base \
                                        --format=json \
                                        --out-file=/tmp/infracost-base.json'

                // Generate an Infracost diff and save it to a JSON file.
                sh 'infracost diff --path=terraform \
                                   --format=json \
                                   --compare-to=/tmp/infracost-base.json \
                                   --out-file=/tmp/infracost.json'

                // Posts a comment to the PR using the 'update' behavior.
                // This creates a single comment and updates it. The "quietest" option.
                // The other valid behaviors are:
                //   delete-and-new - Delete previous comments and create a new one.
                //   hide-and-new - Minimize previous comments and create a new one.
                //   new - Create a new cost estimate comment on every push.
                // See https://www.infracost.io/docs/features/cli_commands/#comment-on-pull-requests for other options.
                sh 'infracost comment github --path=/tmp/infracost.json \
                                             --repo=$GITHUB_REPO \
                                             --pull-request=$GITHUB_PULL_REQUEST_NUMBER \
                                             --github-token=$GITHUB_TOKEN \
                                             --behavior=update'
            }
        }
    }
}
```

3. 🎉 That's it! Send a new pull request to change something in Terraform that costs money. You should see a merge request comment that gets updated, e.g. the 📉 and 📈 emojis will update as changes are pushed! Check the build Console Output and [this page](https://www.infracost.io/docs/troubleshooting/) if there are issues.

4. Follow [the docs](https://www.infracost.io/usage-file) if you'd also like to show cost for of usage-based resources such as AWS Lambda or S3. The usage for these resources are fetched from CloudWatch/cloud APIs and used to calculate an estimate.

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

Issues and pull requests are welcome. Please create issues in [this repo](https://github.com/infracost/infracost) or [join our community Slack](https://www.infracost.io/community-chat), we are a friendly bunch and happy to help you get started :)

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
