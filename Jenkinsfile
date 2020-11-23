#!groovy

@Library('fedora-pipeline-library@candidate2') _

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

def podYAML = """
spec:
  containers:
  - name: pipeline-agent
    # source: https://github.com/fedora-ci/jenkins-pipeline-library-agent-image
    image: quay.io/fedoraci/pipeline-library-agent:candidate
    tty: true
    alwaysPullImage: true
"""

pipeline {

    agent { label 'rpminspect' }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '45', artifactNumToKeepStr: '100'))
        timeout(time: 12, unit: 'HOURS')
    }

    parameters {
        string(name: 'ARTIFACT_ID', defaultValue: '', trim: true, description: '"koji-build:&lt;taskId&gt;" for Koji builds; Example: koji-build:46436038')
    }

    environment {
        TESTING_FARM_API_KEY = credentials('testing-farm-api-key')
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    artifactId = params.ARTIFACT_ID
                    setBuildNameFromArtifactId(artifactId: artifactId)

                    if (!artifactId) {
                        abort('ARTIFACT_ID is missing')
                    }
                }
                sendMessage(type: 'queued', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Schedule Test') {
            steps {
                script {
                    def requestPayload = """
                        {
                            "api_key": "${env.TESTING_FARM_API_KEY}",
                            "test": {
                                "fmf": {
                                    "url": "${getGitUrl()}",
                                    "ref": "${getGitRef()}"
                                }
                            },
                            "environments": [
                                {
                                    "arch": "x86_64",
                                    "variables": {
                                        "RELEASE_ID": "${getReleaseIdFromBranch()}",
                                        "TASK_ID": "${getIdFromArtifactId(artifactId: artifactId)}",
                                        "DEFAULT_RELEASE_STRING": "fc34"
                                    }
                                }
                            ]
                        }
                    """
                    echo "${requestPayload}"
                    def response = submitTestingFarmRequest(payload: requestPayload)
                    testingFarmRequestId = response['id']
                }
                sendMessage(type: 'running', artifactId: artifactId, pipelineMetadata: pipelineMetadata, dryRun: isPullRequest())
            }
        }

        stage('Wait for Test Results') {
            steps {
                script {
                    testingFarmResult = waitForTestingFarmResults(requestId: testingFarmRequestId, timeout: 60)
                    xunit = testingFarmResult.get('result', [:])?.get('xunit', '') ?: ''
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
