# Infracost Jenkins

This project provides the Jenkins integration for Infracost, so you can see cloud cost estimates and FinOps best practices for Terraform in pull requests 💰

![Example GitHub screenshot](.github/assets/pr-comment.png?raw=true)

## Quick start

This quick start guide assumes that Jenkins is being used with GitHub, however, it can be updated to work with Azure DevOps Repos, Bitbucket or GitLab. We recommend you email [hello@infracost.io](mailto:hello@infracost.io) if you would like to do a proof-of-concept with Jenkins so we can discuss your requirements and support you.

1. If you haven't done so already, [download Infracost](https://www.infracost.io/docs/#quick-start) and run `infracost auth login` to get a free API key.

2. Retrieve your Infracost API key by running `infracost configure get api_key`.

3. Create a new credential in Jenkins' management panel, called `infracost-api-key`, and enter your Infracost API key.

4. Create a GitHub token (such as [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)) that can be used by the pipeline to post comments. The token needs to have `repo` scope so it can post comments. If you are using SAML single sign-on, you must first [authorize the token](https://docs.github.com/en/enterprise-cloud@latest/authentication/authenticating-with-saml-single-sign-on/authorizing-a-personal-access-token-for-use-with-saml-single-sign-on).
    If you are using, Azure DevOps Repos, you will need a token (such as [Personal Access Token](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows)) that can post pull request comments. For Bitbucket, see [this page](https://bitbucket.org/infracost/infracost-bitbucket-pipeline) for details of how to create the token. For GitLab, see [this page](https://gitlab.com/infracost/infracost-gitlab-ci) for details of how to create the token.

5. Create a new credential in Jenkins' management panel, called `github-token`, and enter your token.

6. The Infracost Jenkins integrations requires two jobs:
    - The first job (`infracost-pr.Jenkinsfile`) runs on pull requests to check them and post the pull request comment. Ideally you should configure this job to be automatically triggered when new pull requests are opened. The following example enables you to enter the pull request ULR and run the job manually.
    - The second job (`infracost-default-branch-cron.Jenkinsfile`) runs on the main/master branch so the Infracost Cloud dashboard is updated. Ideally you should configure this job to be automatically triggered when the main/master branch is updated. The following example uses a cron schedule to run this job every hour, running this job once or twice a day should also be sufficient.

```Jenkinsfile
// This is the infracost-pr.Jenkinsfile
// Requires Pipeline Utility Steps plugin (https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/)

pipeline {
    agent any

    parameters {
        string(name: 'PR_URL', defaultValue: '', description: 'Pull request URL')
    }

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
                INFRACOST_API_KEY = credentials('infracost-api-key')
                GITHUB_TOKEN = credentials('github-token')
                INFRACOST_VCS_PROVIDER = 'github'
            }

            steps {
                // Extract the GitHub repository name and the PR number from the PR URL
                script {
                    def prUrl = params.PR_URL

                    def matcher = prUrl =~ /https:\/\/github\.com\/([^\/]+\/[^\/]+)\/pull\/(\d+)/
                    if (!matcher) {
                        error "Failed to parse PR URL: ${prUrl}"
                    }
                    env.REPO_NAME = matcher[0][1]
                    env.PR_NUMBER = matcher[0][2]
                    
                    // This is needed because Jenkins can't serialize the matcher object
                    matcher = null

                    // Set PR and Repository URL environment variables
                    env.INFRACOST_VCS_REPOSITORY_URL = "https://github.com/${env.REPO_NAME}"
                    env.INFRACOST_VCS_PULL_REQUEST_URL = prUrl

                    // Call GitHub API to get PR details
                    def apiUrl = "https://api.github.com/repos/${env.REPO_NAME}/pulls/${env.PR_NUMBER}"
                    echo "Calling GitHub API: ${apiUrl}"
                    def output = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + apiUrl + '\"', returnStdout: true).trim()
                    def prDetails = readJSON text: output

                    // Set environment variables based on PR details
                    env.INFRACOST_VCS_PULL_REQUEST_TITLE = prDetails.title
                    env.INFRACOST_VCS_BRANCH = prDetails.head.ref
                    env.INFRACOST_VCS_PULL_REQUEST_AUTHOR = prDetails.user.login
                    env.INFRACOST_VCS_PULL_REQUEST_LABELS = prDetails.labels.collect { it.name }.join(',')
                    env.INFRACOST_VCS_BASE_BRANCH = prDetails.base.ref
                    env.INFRACOST_VCS_COMMIT_SHA = prDetails.head.sha


                    // Additional API call for commit details
                    def commitUrl = prDetails.head.repo.commits_url.replace('{/sha}', "/${env.INFRACOST_VCS_COMMIT_SHA}")
                    echo "Calling GitHub API: ${commitUrl}"
                    def commitOutput = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + commitUrl + '\"', returnStdout: true).trim()
                    def commitDetails = readJSON text: commitOutput

                    // Set more environment variables based on commit details
                    env.INFRACOST_VCS_COMMIT_MESSAGE = commitDetails.commit.message
                    env.INFRACOST_VCS_COMMIT_AUTHOR_EMAIL = commitDetails.commit.author.email
                    env.INFRACOST_VCS_COMMIT_AUTHOR_NAME = commitDetails.commit.author.name
                    env.INFRACOST_VCS_COMMIT_TIMESTAMP = commitDetails.commit.author.date

                    // Get the merge base SHA using GitHub API
                    def mergeBaseApiUrl = "https://api.github.com/repos/${env.REPO_NAME}/compare/${env.INFRACOST_VCS_BASE_BRANCH}...${env.INFRACOST_VCS_COMMIT_SHA}"
                    echo "Calling GitHub API: ${mergeBaseApiUrl}"
                    def mergeBaseOutput = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + mergeBaseApiUrl + '\"', returnStdout: true).trim()
                    def mergeBaseDetails = readJSON text: mergeBaseOutput
                    def mergeBaseSha = mergeBaseDetails.merge_base_commit.sha

                    env.MERGE_BASE_SHA = mergeBaseSha

                    echo "Found merge base SHA: ${mergeBaseSha}"

                    echo 'Finished setting environment variables'
                }

                // Find the merge base SHA using head SHA and base ref, then clone and checkout the base branch into
                // a temporary directory.
                script {
                    echo "Running Infracost on base branch"

                    // Clone the repository to a temporary directory
                    sh "git clone https://github.com/${env.REPO_NAME}.git /tmp/repo"

                    // Checkout the merge base commit
                    sh "cd /tmp/repo && git checkout -q ${env.MERGE_BASE_SHA}"

                    sh 'infracost breakdown --path=/tmp/repo \
                        --format=json \
                        --out-file=/tmp/infracost-base.json'
                }

                script {
                    echo "Running Infracost on PR branch"

                    sh "cd /tmp/repo && git checkout ${env.INFRACOST_VCS_COMMIT_SHA}"

                    // Generate an Infracost diff and save it to a JSON file.
                    sh 'infracost diff --path=/tmp/repo \
                                    --format=json \
                                    --compare-to=/tmp/infracost-base.json \
                                    --out-file=/tmp/infracost.json'
                }

                script {
                    echo "Generating Infracost comment"

                    // Post PR comment and upload the infracost.json file to Infracost Cloud
                    sh 'infracost comment github --path=/tmp/infracost.json \
                                                --repo=$REPO_NAME \
                                                --pull-request=$PR_NUMBER \
                                                --github-token=$GITHUB_TOKEN \
                                                --behavior=update'
                }
            }
        }
    }
}
```

```Jenkinsfile
// This is the infracost-default-branch-cron.Jenkinsfile
// Requires Pipeline Utility Steps plugin (https://www.jenkins.io/doc/pipeline/steps/pipeline-utility-steps/)

// The following need to be added to 'In-process Script Approval' > 'Signatures already approved'
// staticMethod java.time.ZonedDateTime now
// staticMethod java.time.ZonedDateTime parse java.lang.CharSequence

import java.time.ZonedDateTime
import java.time.format.DateTimeFormatter
import java.time.temporal.ChronoUnit

pipeline {
    agent any

    triggers {
        cron('H * * * *') // Runs every hour
    }

    parameters {
        text(name: 'REPO_NAMES', defaultValue: '', description: 'List of GitHub repository names, one per line (e.g., "user/repo1\nuser/repo2")')
        string(name: 'AGE_THRESHOLD', defaultValue: '3600', description: 'Age of the most recent commits and PRs to check (default 3600 seconds = 1 hour)')
    }

    environment {
        GITHUB_TOKEN = credentials('github-token')
        INFRACOST_API_KEY = credentials('alistair-test-infracost-api-key')
    }

    stages {
        stage('Infracost default branches and PR status update') {
            agent {
                docker {
                    image 'infracost/infracost:ci-0.10'
                    args "--user=root --entrypoint=''"
                }
            }

            steps {
                script {
                    def repoNames = params.REPO_NAMES.readLines()
                    def ageThresholdSeconds = params.AGE_THRESHOLD.toInteger()

                    def prStatusUpdates = []

                    def parallelTasks = [:]

                    repoNames.each { repoName ->
                        parallelTasks["Infracost-${repoName}"] = {
                            echo "Processing repository: ${repoName}"

                            // Fetch merged PRs within the time frame
                            // This fetches the latest 100 closed PRs for the repo
                            def prsApiUrl = "https://api.github.com/repos/${repoName}/pulls?per_page=100&state=closed&sort=updated&direction=desc"
                            def prsOutput = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + prsApiUrl +'\"', returnStdout: true).trim()
                            def prsDetails = readJSON text: prsOutput

                            prsDetails.each { pr ->
                                def closedAt = ZonedDateTime.parse(pr.closed_at)
                                def currentDateTime = ZonedDateTime.now()
                                def secondsSinceClosed = ChronoUnit.SECONDS.between(closedAt, currentDateTime)

                                if (secondsSinceClosed <= ageThresholdSeconds) {
                                    def status = pr.merged_at ? "MERGED" : "CLOSED"
                                    prStatusUpdates << [url: pr.html_url, status: status]
                                }
                            }

                            // Get the default branch of the repository
                            def repoApiUrl = "https://api.github.com/repos/${repoName}"
                            def repoOutput = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + repoApiUrl + '\"', returnStdout: true).trim()
                            def repoDetails = readJSON text: repoOutput
                            def defaultBranch = repoDetails.default_branch

                            // Fetch the latest commit on the default branch
                            def commitsApiUrl = "https://api.github.com/repos/${repoName}/commits?sha=${defaultBranch}"
                            def latestCommitOutput = sh(script: 'curl -s -H \"Authorization: token $GITHUB_TOKEN\" \"' + commitsApiUrl + '\"', returnStdout: true).trim()
                            def latestCommitDetails = readJSON text: latestCommitOutput

                            if (latestCommitDetails.size() == 0) {
                                echo "No commits found on the default branch for repository ${repoName}. Skipping."
                                return
                            }

                            def latestCommit = latestCommitDetails[0]
                            def latestCommitDate = ZonedDateTime.parse(latestCommit.commit.author.date)
                            def currentDateTime = ZonedDateTime.now()
                            def secondsSinceLastCommit = ChronoUnit.SECONDS.between(latestCommitDate, currentDateTime)

                            if (secondsSinceLastCommit > ageThresholdSeconds) {
                                echo "Skipping repository ${repoName} as the most recent commit is older than ${ageThresholdSeconds} seconds."
                                return
                            }

                            // Set the environment variables
                            // We can't set them directly on the env since that will override other parallel tasks
                            def localEnvVars = """
                                INFRACOST_VCS_REPOSITORY_URL=${repoDetails.html_url}
                                INFRACOST_VCS_BRANCH=${defaultBranch}
                                INFRACOST_VCS_COMMIT_SHA=${latestCommit.sha}
                                INFRACOST_VCS_COMMIT_MESSAGE='${latestCommit.commit.message}'
                                INFRACOST_VCS_COMMIT_AUTHOR_EMAIL=${latestCommit.commit.author.email}
                                INFRACOST_VCS_COMMIT_AUTHOR_NAME='${latestCommit.commit.author.name}'
                                INFRACOST_VCS_COMMIT_TIMESTAMP=${latestCommit.commit.author.date}
                            """.trim()

                            echo "Default branch for ${repoName} is ${defaultBranch}"

                            // Create a repo slug so we can use the repo name in paths
                            def repoSlug = repoName.trim().replace('/', '_')

                            // Clone the repository to a temporary directory
                            sh "git clone https://github.com/${repoName}.git /tmp/${repoSlug}"

                            // Checkout the default branch
                            sh "cd /tmp/${repoSlug} && git checkout ${defaultBranch}"

                            // Run Infracost breakdown
                            sh """
                                export ${localEnvVars}
                                infracost breakdown --path=/tmp/${repoSlug} \
                                                    --format=json \
                                                    --out-file=/tmp/infracost-${repoSlug}.json
                            """

                            // Run Infracost upload
                            sh "infracost upload --path=/tmp/infracost-${repoSlug}.json  || echo 'Always pass main branch runs even if there are policy failures'"

                            // Cleanup
                            sh "rm -rf /tmp/${repoSlug}"
                        }
                    }

                    parallel parallelTasks

                    if (prStatusUpdates.size() > 0) {
                        def updatesString = prStatusUpdates.collect { "PR: ${it.url}, Status: ${it.status}" }.join("\n")

                        echo "Updating PR status for ${prStatusUpdates.size()} PRs:\n${updatesString}"

                        def graphqlQuery = [
                            query: "mutation {\n" + prStatusUpdates.collect { update ->
                                "updatePullRequestStatus(url: \"${update.url}\", status: ${update.status})"
                            }.join("\n") + "\n}"
                        ]

                        def graphqlData = groovy.json.JsonOutput.toJson(graphqlQuery)
                        def graphqlApiUrl = 'https://dashboard.api.infracost.io/graphql'

                        def response = sh(script: """
                            curl -s -X POST -H "Content-Type: application/json" -H "X-Api-Key: \$INFRACOST_API_KEY" \
                            -d '${graphqlData}' '${graphqlApiUrl}'
                        """, returnStdout: true).trim()

                        def jsonResponse = readJSON text: response

                        // Check for errors in the response
                        if (jsonResponse.errors != null && jsonResponse.errors.size() > 0) {
                            jsonResponse.errors.each { error ->
                                echo "Error: ${error.message}"
                            }
                            error "GraphQL API returned errors. See above messages for details."
                        }

                        echo "All PR statuses updated successfully."
                    }
                }
            }
        }
    }
}
```

7. 🎉 That's it! Send a new pull request using [these code snippets](https://www.infracost.io/docs/infracost_cloud/get_started/#4-send-a-pull-request) to test Infracost. You should see a pull request comment that gets updated, e.g. the 📉 and 📈 emojis will update as changes are pushed! Check the build Console Output and [this page](https://www.infracost.io/docs/troubleshooting/) if there are issues.

    <img src=".github/assets/pr-comment.png" alt="Example pull request" width="70%" />

8. In [Infracost Cloud](https://dashboard.infracost.io), go to Org Settings and enable the dashboard, then trigger your CI/CD pipeline again. This causes the CLI to send its JSON output to your dashboard; the JSON does not contain any cloud credentials or secrets, see the [FAQ](https://infracost.io/docs/faq/) for more information.

    This is our SaaS product that builds on top of Infracost open source. It enables you to check for best practices such as using latest generation instance types or block storage, e.g. consider switching AWS gp2 volumes to gp3 as they are more performant and cheaper. Team leads, managers and FinOps practitioners can also setup [tagging policies](https://www.infracost.io/docs/infracost_cloud/tagging_policies/) and [guardrails](https://www.infracost.io/docs/infracost_cloud/guardrails/) to help guide the team.

    <img src=".github/assets/infracost-cloud-dashboard.png" alt="Infracost Cloud enables you to check for best practices." width="80%" />

## Private Terraform modules

If you use private Terraform modules in your project you'll need to correctly configure the Jenkins pipeline to fetch these. You can find more information about private modules [on our docs](https://www.infracost.io/docs/guides/terraform_modules/).

## Comment options

The Infracost CLI can post cost estimates to pull request or commits on GitHub, GitLab, Azure Repos and Bitbucket. Run `infracost comment --help` to see the the list of options or [see our docs](https://www.infracost.io/docs/features/cli_commands/#comment-on-pull-requests).

## Contributing

Issues and pull requests are welcome. Please create issues in [this repo](https://github.com/infracost/infracost) or [join our community Slack](https://www.infracost.io/community-chat), we are a friendly bunch and happy to help you get started :)

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
