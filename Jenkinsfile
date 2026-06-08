pipeline {
    agent any
    options {
        timestamps()
    }
    environment {
        BASE_URL = ''
        VENV_DIR = "${WORKSPACE}/venv"
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
                sudo apt-get install -y python3-venv python3-full
                python3 -m venv ${VENV_DIR}
                ${VENV_DIR}/bin/pip install --upgrade pip
                ${VENV_DIR}/bin/pip install flake8 bandit pytest requests
                '''
            }
        }
        stage('Static Test') {
            steps {
                sh '''
                mkdir -p reports
                ${VENV_DIR}/bin/flake8 src --tee --output-file reports/flake8-report.txt || true
                ${VENV_DIR}/bin/bandit -r src -f txt -o reports/bandit-report.txt || true
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
                    --resolve-s3
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
                ${VENV_DIR}/bin/pytest test/integration/todoApiTest.py -v
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
            cleanWs()
        }
        success {
            echo 'Pipeline ejecutado correctamente'
        }
        failure {
            echo 'Pipeline fallido'
        }
    }
}