pipeline {
    agent any
    options {
        timestamps()
    }
    environment {
        BASE_URL = ''
        PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
    }
    stages {
        stage('Get Code') {
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
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                pip3 install --break-system-packages flake8 bandit pytest requests
                '''
            }
        }
        stage('Static Test') {
            steps {
                sh '''
                mkdir -p reports
                flake8 src --tee --output-file reports/flake8-report.txt || true
                bandit -r src -f txt -o reports/bandit-report.txt || true
                '''
            }
        }
        stage('Deploy') {
            steps {
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
            }
        }
        stage('Rest Test') {
            steps {
                script {
                    env.BASE_URL = sh(
                        script: '''
                        aws cloudformation describe-stacks \
                            --stack-name staging-todo-list-aws \
                            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                            --output text
                        ''',
                        returnStdout: true
                    ).trim()
                }
                sh '''
                export BASE_URL=${BASE_URL}
                pytest test/integration/todoApiTest.py -v
                '''
            }
        }
        stage('Promote') {
            steps {
                sh '''
                git config user.email "jenkins@local"
                git config user.name "Jenkins"
                git fetch origin master
                git checkout master
                git merge origin/develop --no-edit
                git push origin master
                '''
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline ejecutado correctamente'
        }
        failure {
            echo 'Pipeline fallido'
        }
    }
}