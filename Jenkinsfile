pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        BASE_URL = ''
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
                python3 -m venv venv
                . venv/bin/activate
                pip install flake8 bandit pytest
                '''
            }
        }

        stage('Static Test') {
            steps {
                sh '''
                mkdir -p reports
                . venv/bin/activate

                flake8 src --tee --output-file reports/flake8-report.txt || true

                bandit -r src -f txt -o reports/bandit-report.txt || true
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                }
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
                echo $BASE_URL
                . venv/bin/activate
                pytest test/integration/todoApiTest.py
                '''
            }
        }

        stage('Promote') {
            steps {
                sh '''
                git config user.email "jenkins@local"
                git config user.name "jenkins"

                git checkout master
                git pull origin master
                git merge origin/develop
                git push origin master
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline ejecutado correctamente'
        }

        failure {
            echo 'Pipeline fallido'
        }

        always {
            archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
        }
    }
}