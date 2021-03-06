#!/usr/bin/env groovy
@Library('github.com/stakater/fabric8-pipeline-library@master')

def versionPrefix = ""
try {
    versionPrefix = VERSION_PREFIX
} catch (Throwable e) {
    versionPrefix = "1.0"
}

def canaryVersion = "${versionPrefix}.${env.BUILD_NUMBER}"
def label = "buildpod.${env.JOB_NAME}.${env.BUILD_NUMBER}".replace('-', '_').replace('/', '_')

// TODO: Change to 'lab'
def envLab = 'test2'
def stashName = ""

// Hardcoded registry url
//TODO: Should be picked from ENV varaible
def dockerRegistryURL="docker.tools.stackator.com:443"

mavenNode(mavenImage: 'openjdk:8') {
    container(name: 'maven') {

        stage('Checkout') {
            checkout scm
            git_branch = scm.getBranches()[0].toString()
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
            kubernetesApply(registry: dockerRegistryURL, environment: envLab)
            //stash deployments
            stashName = label
            stash includes: '**/*.yml', name: stashName
        }
    }
}