#!/usr/bin/env groovy
@Library('github.com/stakater/fabric8-pipeline-library@master')

import groovy.json.JsonOutput
import java.util.Optional
import hudson.tasks.test.AbstractTestResultAction
import hudson.model.Actionable
import hudson.tasks.junit.CaseResult

def versionPrefix = ""
try {
    versionPrefix = VERSION_PREFIX
} catch (Throwable e) {
    versionPrefix = "1.0"
}

def canaryVersion   = "${versionPrefix}.${env.BUILD_NUMBER}"
def utils           = new io.fabric8.Utils()
def label           = "buildpod.${env.JOB_NAME}.${env.BUILD_NUMBER}".replace('-', '_').replace('/', '_')

def branchName      = utils.getBranch()

def publish_branch  = "master"
def registry        = "docker.io"
def registry_user   = "stakater-user"
def robot_secret    = "quay-robot-zabra-container-rw"
def image_name      = "zabra-container"
def image_tag       = "${env.RELEASE_VERSION}" != "null" ? "${env.RELEASE_VERSION}" : "latest"

def slackNotificationChannel = "${env.CHANNEL_NAME}"
def author = ""
def message = ""
def testSummary = ""
def total = 0
def failed = 0
def skipped = 0

def isPublishingBranch = { ->
    return env.GIT_BRANCH == 'origin/master' || env.GIT_BRANCH =~ /release.+/
}

def isResultGoodForPublishing = { ->
    return currentBuild.result == null
}

def notifySlack(text, channel, attachments) {
    def slackURL = "${env.SLACK_WEBHOOK_URL}"
    def jenkinsIcon = 'https://wiki.jenkins-ci.org/download/attachments/2916393/logo.png'

    def payload = JsonOutput.toJson([text: text,
        channel: channel,
        username: "Jenkins",
        icon_url: jenkinsIcon,
        attachments: attachments
    ])

    sh "curl -X POST --data-urlencode \'payload=${payload}\' ${slackURL}"
}

def getGitAuthor = {
    def commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
    author = sh(returnStdout: true, script: "git --no-pager show -s --format='%an' ${commit}").trim()
}

def getLastCommitMessage = {
    message = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim()
}

@NonCPS
def getTestSummary = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def summary = ""

    if (testResultAction != null) {
        total = testResultAction.getTotalCount()
        failed = testResultAction.getFailCount()
        skipped = testResultAction.getSkipCount()

        summary = "Passed: " + (total - failed - skipped)
        summary = summary + (", Failed: " + failed)
        summary = summary + (", Skipped: " + skipped)
    } else {
        summary = "No tests found"
    }
    return summary
}

@NonCPS
def getFailedTests = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def failedTestsString = "```"

    if (testResultAction != null) {
        def failedTests = testResultAction.getFailedTests()

        if (failedTests.size() > 9) {
            failedTests = failedTests.subList(0, 8)
        }

        for(CaseResult cr : failedTests) {
            failedTestsString = failedTestsString + "${cr.getFullDisplayName()}:\n${cr.getErrorDetails()}\n\n"
        }
        failedTestsString = failedTestsString + "```"
    }
    return failedTestsString
}

def populateGlobalVariables = {
    getLastCommitMessage()
    getGitAuthor()
    testSummary = getTestSummary()
}

mavenNode(mavenImage: 'maven:3.5-jdk-8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm

            // retrieve the URI used for checking out the source
            // this assumes one branch with one uri
            git_uri = scm.getRepositories()[0].getURIs()[0].toString()
            git_branch = scm.getBranches()[0].toString()
        }

        try {
            stage('Clean & Package') {
                sh 'mvn clean package'
            }
        } catch (hudson.AbortException ae) {
            // I ignore aborted builds, but you're welcome to notify Slack here
        } catch (e) {
            def buildStatus = "Failed"

            if (isPublishingBranch()) {
              buildStatus = "MasterFailed"
            }

            notifySlack("", slackNotificationChannel, [
              [
                  title: "${env.JOB_NAME}, build #${env.BUILD_NUMBER}",
                  title_link: "${env.BUILD_URL}",
                  color: "danger",
                  author_name: "${author}",
                  text: "${buildStatus}",
                  fields: [
                      [
                          title: "Branch",
                          value: "${env.GIT_BRANCH}",
                          short: true
                      ],
                      [
                          title: "Test Results",
                          value: "${testSummary}",
                          short: true
                      ],
                      [
                          title: "Last Commit",
                          value: "${message}",
                          short: false
                      ],
                      [
                          title: "Error",
                          value: "${e}",
                          short: false
                      ]
                  ]
              ]
            ])

            throw e
        }

        try {
            stage('Build Docker Image') {
                sh "docker build -t ${image_name}:${env.JOB_BASE_NAME}.${env.BUILD_ID} ."
            }
        } catch (hudson.AbortException ae) {
            // I ignore aborted builds, but you're welcome to notify Slack here
        } catch (e) {
            def buildStatus = "Failed"

            if (isPublishingBranch()) {
              buildStatus = "MasterFailed"
            }

            notifySlack("", slackNotificationChannel, [
              [
                  title: "${env.JOB_NAME}, build #${env.BUILD_NUMBER}",
                  title_link: "${env.BUILD_URL}",
                  color: "danger",
                  author_name: "${author}",
                  text: "${buildStatus}",
                  fields: [
                      [
                          title: "Branch",
                          value: "${env.GIT_BRANCH}",
                          short: true
                      ],
                      [
                          title: "Last Commit",
                          value: "${message}",
                          short: false
                      ],
                      [
                          title: "Error",
                          value: "${e}",
                          short: false
                      ]
                  ]
              ]
            ])

            throw e
        }

        // only push from master
        stage('Publish Docker Image') {
            if (git_branch.contains(publish_branch)) {
                sh "docker login ${registry} -u ${USERNAME} -p ${PASSWORD}"
                sh "docker tag ${image_name}:${env.JOB_BASE_NAME}.${env.BUILD_ID} ${registry}/${registry_user}/${image_name}:${image_tag}"
                sh "docker push ${registry}/${registry_user}/${image_name}:${image_tag}"
            } else {
                echo "Not pushing to docker repo:\n    BRANCH_NAME='${env.BRANCH_NAME}'\n    GIT_BRANCH='${git_branch}'\n    git_uri='${git_uri}'"
            }
        }
    }
}