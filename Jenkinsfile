pipeline {
    agent any
    stages {
        stage('Get Code') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: '*/master']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/CristinaSanzPosadas/todo-list-aws.git',
                        credentialsId: 'git-token'
                    ]]
                ]
            }
        }

        stage('Deploy') {
            steps {
                echo 'Construyendo...'
                sh "sam build"

                echo 'Validando...'
                sh "sam validate --region us-east-1"

                echo 'Desplegando...'
                sh "sam deploy --config-env production --no-confirm-changeset --no-fail-on-empty-changeset"
            }
        }

        stage('Rest Test') {
            steps {
                withCredentials([string(credentialsId: 'base-url-production', variable: 'BASE_URL')]) {
                    echo "Ejecutando pruebas de integración"
                    sh """
                    export BASE_URL=${BASE_URL}
                    python -m pytest -m read_only --junitxml=integration-tests-readonly.xml test/integration
                    """

                    junit '**/integration-tests-readonly.xml'

                }
            }
        }

        stage('Cleanup') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh 'pkill -f python || echo "No Flask process running"'
                }
                script {
                    echo "Limpiando workspace..."
                    cleanWs()
                }
            }
        }
    }
}
