pipeline {
    agent none
    options {
        timestamps()
    }
    environment {
        PATH = "/var/lib/jenkins/.local/bin:/usr/local/bin:/usr/bin:/bin"
    }
    stages {
        stage('Get Code') {
            agent any
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/develop']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/merliariza/todo-list-aws.git'
                    ]]
                ])
                sh '''
                git branch
                git status
                '''
                stash name: 'source-code', includes: '**/*'
            }
        }
        stage('Install Dependencies') {
            agent { label 'static-agent' }
            steps {
                sh '''
                pip3 install --break-system-packages flake8 bandit pytest requests
                '''
            }
        }
        stage('Static Test') {
            agent { label 'static-agent' }
            steps {
                unstash 'source-code'
                sh '''
                mkdir -p reports
                flake8 src --tee --output-file reports/flake8-report.txt || true
                bandit -r src -f txt -o reports/bandit-report.txt || true
                '''
                stash name: 'reports', includes: 'reports/*'
            }
        }
        stage('Deploy') {
            agent any
            steps {
                unstash 'source-code'
                script {
                    env.BASE_URL = sh(
                        script: '''
                            sam build
                            sam validate
                            sam deploy \
                                --config-env staging \
                                --no-confirm-changeset \
                                --no-fail-on-empty-changeset \
                                --resolve-s3 \
                                --s3-bucket ""
                            aws cloudformation describe-stacks \
                                --stack-name staging-todo-list-aws \
                                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                                --output text
                        ''',
                        returnStdout: true
                    ).trim()
                    echo "BASE_URL capturada: ${env.BASE_URL}"
                }
            }
        }
        stage('Rest Test') {
            agent { label 'test-agent' }
            steps {
                unstash 'source-code'
                sh '''
                pip3 install --break-system-packages pytest requests
                '''
                withEnv(["BASE_URL=${env.BASE_URL}"]) {
                    sh 'pytest test/integration/todoApiTest.py -v'
                }
            }
        }
        stage('Promote') {
            agent any
            steps {
                unstash 'source-code'
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                    git config user.email "jenkins@local"
                    git config user.name "Jenkins"
                    git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/merliariza/todo-list-aws.git
                    git fetch origin master
                    git checkout master
                    git merge origin/develop --no-edit
                    git push origin master
                    '''
                }
            }
        }
    }
    post {
        always {
            node('any') {
                unstash 'reports'
                archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
            }
        }
        success {
            echo 'Pipeline ejecutado correctamente'
        }
        failure {
            echo 'Pipeline fallido'
        }
    }
}