pipeline {
    agent any
    options {
        timestamps()
    }
    environment {
        PATH = "/var/lib/jenkins/.local/bin:${env.PATH}"
    }
    stages {
        stage('Get Code') {
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
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                sam build
                sam validate
                sam deploy \
                    --config-env production \
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
                    def baseUrl = sh(
                        script: '''
                            aws cloudformation describe-stacks \
                                --stack-name todo-list-aws-production \
                                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                                --output text
                        ''',
                        returnStdout: true
                    ).trim()
                    echo "BASE_URL capturada: ${baseUrl}"
                    withEnv(["BASE_URL=${baseUrl}"]) {
                        sh 'pytest test/integration/todoApiTest.py -v -k "listtodos"'
                    }
                }
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