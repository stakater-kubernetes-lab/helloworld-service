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

def envDev = 'dev'
def envStage = 'stage'
def envProd = 'prod'
def stashName = ""

// Hardcoded registry url
//TODO: Should be picked from ENV varaible
def dockerRegistryURL="docker.tools.stackator.com:443"

def branchName = utils.getBranch()
// If branch other then master, append branch name to version
// in order to avoid conflicts in artifact releases
if (! branchName.equalsIgnoreCase("master")){
    canaryVersion = branchName + "-" + canaryVersion
}

mavenNode(mavenImage: 'openjdk:8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm
        }

        stage('Clean') {
            sh 'chmod +x mvnw'
            sh './mvnw clean'
        }

        stage('Canary Release') {
            mavenCanaryRelease2 {
              version = canaryVersion
            }
        }

        stage('Rollout to Dev') {
            kubernetesApply(registry: dockerRegistryURL, environment: envDev)
            //stash deployments
            stashName = label
            stash includes: '**/*.yml', name: stashName
        }
    }
}