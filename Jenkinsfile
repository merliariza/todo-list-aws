pipeline {
    agent none
    options {
        timestamps()
    }
    stages {
        stage('Get Code') {
            agent any
            environment {
                PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
            }
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/master']],
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
        stage('Static Test') {
            agent any
            environment {
                PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
            }
            steps {
                unstash 'source-code'
                sh '''
                pip3 install --break-system-packages flake8 bandit
                mkdir -p reports
                flake8 src --tee --output-file reports/flake8-report.txt || true
                bandit -r src -f txt -o reports/bandit-report.txt || true
                '''
                stash name: 'reports', includes: 'reports/*'
            }
        }
        stage('Deploy') {
            agent any
            environment {
                PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
            }
            steps {
                unstash 'source-code'
                script {
                    sh '''
                    sam build
                    sam validate
                    sam deploy \
                        --config-env staging \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset \
                        --resolve-s3 \
                        --s3-bucket ""
                    '''
                    env.BASE_URL = sh(
                        script: '''
                            aws cloudformation describe-stacks \
                                --stack-name todo-list-aws-production \
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
            agent any
            environment {
                PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
            }
            steps {
                unstash 'source-code'
                sh 'pip3 install --break-system-packages pytest requests'
                withEnv(["BASE_URL=${env.BASE_URL}"]) {
                    sh 'pytest test/integration/todoApiTest.py -v'
                }
            }
        }
        stage('Promote') {
            agent any
            environment {
                PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
            }
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
                    git fetch origin
                    git checkout master
                    git merge origin/develop --no-edit -X theirs
                    git push origin master
                    '''
                }
            }
        }
    }
    post {
        always {
            node('built-in') {
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