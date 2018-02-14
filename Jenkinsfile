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

mavenNode(mavenImage: 'maven:3.5-jdk-8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm
            git_branch = scm.getBranches()[0].toString()
        }

        stage('Clean') {
            sh './mvn clean'
        }

        stage('Canary Release') {
            // If branch other then master, append branch name to version
            // in order to avoid conflicts in artifact releases
            if (! git_branch.equalsIgnoreCase("master")){
                canaryVersion = git_branch + "-" + canaryVersion
            }
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