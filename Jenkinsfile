#!groovy

@Library('fedora-pipeline-library@fedora-stable') _

def pipelineMetadata = [
    pipelineName: 'rpminspect',
    pipelineDescription: 'Run rpminspect on RPM builds',
    testCategory: 'static-analysis',
    testType: 'rpminspect',
    maintainer: 'Fedora CI',
    docs: 'https://github.com/rpminspect/rpminspect#rpminspect',
    contact: [
        irc: '#fedora-ci',
        email: 'ci@lists.fedoraproject.org'
    ],
]
def artifactId
def testingFarmRequestId
def testingFarmResult
def xunit
def repoUrlAndRef
def pipelineRepoUrlAndRef
def hook

def podYAML = """
spec:
  containers:
  - name: pipeline-agent
    # source: https://github.com/fedora-ci/jenkins-pipeline-library-agent-image
    image: quay.io/fedoraci/pipeline-library-agent:3685422
    tty: true
    alwaysPullImage: true
"""

pipeline {

    agent none

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        timeout(time: 12, unit: 'HOURS')
        skipDefaultCheckout(true)
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
        string(name: 'TEST_PROFILE', defaultValue: 'f35', trim: true, description: 'A name of the test profile to use; Example: f35')
    }

    environment {
        TESTING_FARM_API_KEY = credentials('testing-farm-api-key')
    }

    stages {
        stage('Prepare') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    artifactId = params.ARTIFACT_ID
                    setBuildNameFromArtifactId(artifactId: artifactId, profile: params.TEST_PROFILE)

                    checkout scm
                    config = loadConfig(profile: params.TEST_PROFILE)

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }

                    repoUrlAndRef = getRepoUrlAndRefFromTaskId(getIdFromArtifactId(artifactId: artifactId))
                    pipelineRepoUrlAndRef = [url: "${getGitUrl()}", ref: "${getGitRef()}"]
                }
                sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Schedule Test') {
            agent {
                label pipelineMetadata.pipelineName
            }
            steps {
                script {
                    def requestPayload = [
                        api_key: "${env.TESTING_FARM_API_KEY}",
                        test: [
                            fmf: pipelineRepoUrlAndRef
                        ],
                        environments: [
                            [
                                arch: "x86_64",
                                variables: [
                                    PREVIOUS_TAG: "${config.previous_tag}",
                                    TASK_ID: "${getIdFromArtifactId(artifactId: artifactId)}",
                                    DEFAULT_RELEASE_STRING: "${config.default_release_string}",
                                    REPOSITORY_URL: "${repoUrlAndRef.url}",
                                    CONFIG_BRANCH: "${config.config_branch}",
                                    GIT_COMMIT: "${repoUrlAndRef.ref}",
                                    OUTPUT_FORMAT: "${config.output_format}"
                                ]
                            ]
                        ]
                    ]
                    hook = registerWebhook()
                    requestPayload['notification'] = ['webhook': [url: hook.getURL()]]

                    def response = submitTestingFarmRequest(payloadMap: requestPayload)
                    testingFarmRequestId = response['id']
                }
                sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Wait for Test Results') {
            agent none
            steps {
                script {
                    def response = waitForTestingFarm(requestId: testingFarmRequestId, hook: hook)
                    testingFarmResult = response.apiResponse
                    xunit = response.xunit
                }
            }
        }

        stage('Process Test Results (XUnit)') {
            when {
                expression { xunit }
            }
            agent {
                kubernetes {
                    yaml podYAML
                    defaultContainer 'pipeline-agent'
                }
            }
            steps {
                script {
                    // Convert Testing Farm XUnit into JUnit and store the result in Jenkins
                    writeFile file: 'tfxunit.xml', text: "${xunit}"
                    sh script: "tfxunit2junit --docs-url ${pipelineMetadata['docs']} tfxunit.xml > xunit.xml"
                    junit(allowEmptyResults: true, keepLongStdio: true, testResults: 'xunit.xml')
                }
            }
        }
    }

    post {
        always {
            evaluateTestingFarmResults(testingFarmResult)
        }
        success {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, xunit: xunit, dryRun: isPullRequest())
        }
        failure {
            sendMessage(type: 'error', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
        }
        unstable {
            sendMessage(type: 'complete', artifactId: artifactId, pipelineMetadata: pipelineMetadata, xunit: xunit, dryRun: isPullRequest())
        }
    }
}
