#!/usr/bin/env groovy
@Library('github.com/stakater/fabric8-pipeline-library@master')

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

mavenNode(mavenImage: 'maven:3.5-jdk-8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm
        }

        stage('Clean & Package') {
            sh 'mvn clean package'
        }

        stage('Build Docker Image') {
          sh "docker build -t ${image_name}:${env.JOB_BASE_NAME}.${env.BUILD_ID} ."
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