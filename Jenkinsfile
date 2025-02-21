pipeline {
    agent any
    stages {
        stage('Get Code') {
            steps {
                checkout scm: [
                    $class: 'GitSCM',
                    branches: [[name: '*/develop']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/CristinaSanzPosadas/todo-list-aws.git',
                        credentialsId: 'git-token'
                    ]]
                ]
            }
        }

        stage('Static Test') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh script: """
                        python -m pip install flake8 bandit --user
                        whoami
                        hostname
                        echo "Workspace: ${env.WORKSPACE}"
                        echo 'Ejecutando análisis de código estático...'
                        python -m flake8 --format=pylint src --exit-zero > flake8.out
                        echo 'Ejecutando pruebas de seguridad...'
                        python -m bandit -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: {severity}: {test_id}: {msg}"
                    """
                    script {
                        echo "Resultados de Flake8:"
                        if (fileExists('flake8.out')) {
                            echo "El archivo flake8.out existe"
                            recordIssues tools: [
                                flake8(name: 'Flake8', pattern: 'flake8.out')
                            ]
                        } else {
                            echo "No se encontró flake8.out"
                        }

                        echo "Resultados de Bandit:"
                        if (fileExists('bandit.out')) {
                            echo "El archivo bandit.out existe"
                            recordIssues tools: [
                                pyLint(name: 'Bandit', pattern: 'bandit.out')
                            ]
                        } else {
                            echo "No se encontró bandit.out"
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Construyendo...'
                sh "sam build"

                echo 'Validando...'
                sh "sam validate --region us-east-1"

                echo 'Desplegando...'
                sh "sam deploy --config-env staging --no-confirm-changeset --no-fail-on-empty-changeset"
            }
        }

        stage('Rest Test') {
            steps {
                withCredentials([string(credentialsId: 'base-url-staging', variable: 'BASE_URL')]) {
                    echo "Ejecutando pruebas de integración"
                    sh """
                    export BASE_URL=${BASE_URL}
                    python -m pytest --junitxml=integration-tests.xml test/integration
                    """
                    junit '**/integration-tests.xml'
                }
            }
        }

        stage('Promote') {
            steps {
                withCredentials([
                    string(credentialsId: 'git-email', variable: 'GIT_EMAIL'),
                    string(credentialsId: 'git-user', variable: 'GIT_NAME'),
                    string(credentialsId: 'git-token', variable: 'GIT_TOKEN')
                ]) {
                    script {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: '*/develop']],
                            userRemoteConfigs: [[
                                url: 'https://github.com/CristinaSanzPosadas/todo-list-aws.git',
                                credentialsId: 'git-token'
                            ]]
                        ])
                        sh """
                        git config --global user.email "${GIT_EMAIL}"
                        git config --global user.name "${GIT_NAME}"

                        git fetch --all
                        git checkout master
                        git pull origin master
                        git merge origin/develop --no-ff -m "Merge develop into master"

                        git push https://${GIT_TOKEN}@github.com/CristinaSanzPosadas/todo-list-aws.git HEAD:master
                        """
                    }
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
