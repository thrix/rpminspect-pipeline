#!groovy

retry (10) {
    // load pipeline configuration into the environment
    httpRequest("${FEDORA_CI_PIPELINES_CONFIG_URL}/environment").content.split('\n').each { l ->
        l = l.trim(); if (l && !l.startsWith('#')) { env["${l.split('=')[0].trim()}"] = "${l.split('=')[1].trim()}" }
    }
}

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

    libraries {
        lib("fedora-pipeline-library@${env.PIPELINE_LIBRARY_VERSION}")
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: env.DEFAULT_DAYS_TO_KEEP_LOGS, artifactNumToKeepStr: env.DEFAULT_ARTIFACTS_TO_KEEP))
        timeout(time: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
        string(name: 'TEST_PROFILE', defaultValue: env.DEFAULT_TEST_PROFILE, trim: true, description: "A name of the test profile to use; Example: ${env.DEFAULT_TEST_PROFILE}")
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
                                    RPMINSPECT_PROFILE_NAME: "${config.profile_name}",
                                    ARCHES: "${config.arches}",
                                    DEBUG: "off"
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
        aborted {
            script {
                if (isTimeoutAborted(timeout: env.DEFAULT_PIPELINE_TIMEOUT_MINUTES, unit: 'MINUTES')) {
                    sendMessage(type: 'error', artifactId: artifactId, errorReason: 'Timeout has been exceeded, pipeline aborted.', pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
                }
            }
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
