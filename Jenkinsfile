pipeline {
    agent {
        kubernetes {
            label 'node-carbon'
        }
    }
    environment {
        REPOSITORY = 'molgenis/molgenis-frontend'
    }
    stages {
        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                }
                container('vault') {
                    script {
                        env.GITHUB_TOKEN = sh(script: 'vault read -field=value secret/ops/token/github', returnStdout: true)
                        env.CODECOV_TOKEN = sh(script: 'vault read -field=molgenis-frontend secret/ops/token/codecov', returnStdout: true)
                        env.NEXUS_AUTH = sh(script: 'vault read -field=base64 secret/ops/account/nexus', returnStdout: true)
                        sh "echo '_auth=${NEXUS_AUTH}' > ~/.npmrc"
                    }
                }
                sh "git remote set-url origin https://${GITHUB_TOKEN}@github.com/${REPOSITORY}.git"
                sh "git fetch --tags"
            }
        }
        stage('Install and test: [ pull request ]') {
            when {
                changeRequest()
            }
            steps {
                container('node') {
                    sh "yarn install"
                    sh "yarn lerna bootstrap"
                    sh "yarn lerna run unit"
                    sh "yarn lerna run build"
                }
            }
            post {
                always {
                    container('node') {
                        sh "curl -s https://codecov.io/bash | bash -s - -c -F unit -K -C ${GIT_COMMIT}"
                    }
                }
            }
        }
        stage('Install, test and build: [ master ]') {
            when {
                branch 'master'
            }
            steps {
                container('node') {
                    sh "yarn install"
                    sh "yarn lerna bootstrap"
                    sh "yarn lerna run unit"
                    sh "yarn lerna run build"
                }
            }
            post {
                always {
                    container('node') {
                        sh "curl -s https://codecov.io/bash | bash -s - -c -F unit -K -C ${GIT_COMMIT}"
                    }
                }
            }
        }
        stage('Release canary: [ master ]'){
            when {
                branch 'master'
            }
            steps {
                lock("Tags"){
                    sh "git fetch --tags"
                    container('node') {
                        sh "yarn lerna publish --canary"
                    }
                }
            }
        }
        stage('Release: [ master ]') {
            when {
                branch 'master'
            }
            steps {
                milestone 1
                timeout(time: 30, unit: 'MINUTES') {
                    script {
                        env.RELEASE_SCOPE = input(
                                message: 'Do you want to release?',
                                ok: 'Release'
                        )
                    }
                }
                container('node') {
                    sh "yarn lerna publish"
                    hubotSend(message: "${env.REPOSITORY} has been successfully deployed.", status:'SUCCESS')
                }
            }
        }
    }
    post{
        success {
            hubotSend(message: 'Build success', status:'INFO', site: 'slack-pr-app-team')
        }
        failure {
            hubotSend(message: 'Build failed', status:'ERROR', site: 'slack-pr-app-team')
        }
    }
}
