#!/usr/bin/env groovy
@Library('github.com/stakater/fabric8-pipeline-library@master')

def versionPrefix = ""
try {
    versionPrefix = VERSION_PREFIX
} catch (Throwable e) {
    versionPrefix = "1.0"
}

def canaryVersion = "${versionPrefix}.${env.BUILD_NUMBER}"
def utils = new io.fabric8.Utils()
def label = "buildpod.${env.JOB_NAME}.${env.BUILD_NUMBER}".replace('-', '_').replace('/', '_')

def branchName = utils.getBranch()

mavenNode(mavenImage: 'maven:3.5-jdk-8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm
        }

        stage('Clean & Package') {
            sh 'mvn clean package'
        }

    }
}