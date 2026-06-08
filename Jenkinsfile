pipeline {
    agent any

    options {
        timestamps()
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

        stage('Static Test') {
            steps {
                sh '''
                mkdir -p reports

                ~/.local/bin/flake8 src \
                --tee \
                --output-file reports/flake8-report.txt || true

                ~/.local/bin/bandit -r src \
                -f txt \
                -o reports/bandit-report.txt || true
                '''
            }

            post {
                always {
                    archiveArtifacts artifacts: 'reports/*', fingerprint: true
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                sam build
                '''

                sh '''
                sam validate
                '''

                sh '''
                sam deploy \
                --config-env staging \
                --no-confirm-changeset \
                --no-fail-on-empty-changeset
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
                echo "Base URL encontrada:"
                echo $BASE_URL

                export BASE_URL=$BASE_URL

                ~/.local/bin/pytest test/integration/todoApiTest.py
                '''
            }
        }

        stage('Promote') {
            steps {
                sh '''
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
            archiveArtifacts artifacts: 'reports/*', fingerprint: true
        }
    }
}