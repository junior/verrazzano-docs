 // Copyright (c) 2020, Oracle Corporation and/or its affiliates. 

pipeline {
    options {
      disableConcurrentBuilds()
    }

    agent {
    
        docker {
            image "${RUNNER_DOCKER_IMAGE}"
            args "${RUNNER_DOCKER_ARGS}"
            registryUrl "${RUNNER_DOCKER_REGISTRY_URL}"
            registryCredentialsId 'ocir-pull-and-push-account'
        }
    }

    parameters {
        booleanParam (name: 'PUBLISH_TO_GH_PAGES',
                defaultValue: false,
                description: 'When true, builds the production website and pushes to the gh-pages branch')
    }

    environment {
        GIT_AUTH = credentials('github-packages-credentials-rw')
    }

    stages {
        stage('Setup Dependencies') {
            steps {
                sh """
                    npm install
                """
            }
        }

        stage('Build staging documentation') {
            steps {
                sh """
                    mkdir -p staging
                    hugo --source . --destination staging --environment staging
                """
            }
        }

        stage('Build production documentation') {
            steps {
                sh """
                    mkdir -p public
                    hugo --source . --destination production --environment production
                """
            }
        }

        stage('Archive artifacts ') {
            steps {
               archiveArtifacts artifacts: 'staging/**'
               archiveArtifacts artifacts: 'production/**'
            }
        }

        stage('Publish documentation to gh-pages') {
            when { equals expected: true, actual: params.PUBLISH_TO_GH_PAGES }
            steps {
                sh """
                    npm -g install gh-pages@3.0.0
                    git config --local credential.helper "!f() { echo username=\\$GIT_AUTH_USR; echo password=\\$GIT_AUTH_PSW; }; f"
                    git config user.name $GIT_AUTH_USR 
                    git config user.email "70212020+verrazzanobot@users.noreply.github.com"
                    /home/opc/graalvm-ce-java8-20.1.0/jre/languages/js/bin/gh-pages -d production -b gh-pages
                """
            }
        }
    }
}

