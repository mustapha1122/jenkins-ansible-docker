pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timestamps()
    }

    triggers {
        cron('*/2 * * * *')
    }

    environment {
        APP_NAME = 'jenkins-project'
        VERSION = '1.0.0'
        DEPLOY_ENV = "${params.ENVIRONNEMENT}"
    }

    parameters {
        string(name: 'ENVIRONNEMENT', defaultValue: 'dev', description: 'Environnement cible')
        choice(name: 'BRANCHE', choices: ['main', 'develop', 'staging'], description: 'Branche a deployer')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Lancer les tests ?')
    }

    stages {

        stage('Docker Info') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    sh 'docker --version'
                    sh 'docker ps'
                    sh 'docker images'
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${params.BRANCHE}"]],
                    userRemoteConfigs: [[
                        url: 'https://gitlab.com/Mustapha1122/jenkins-project.git',
                        credentialsId: 'gitlab-token'
                    ]]
                ])
                echo "App : ${env.APP_NAME} v${env.VERSION}"
                sh 'ls -la'
            }
        }

        stage('Build') {
            parallel {
                stage('Frontend') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            echo "Build Frontend — ${env.DEPLOY_ENV}"
                            sh 'yarn --version'
                        }
                    }
                }
                stage('Backend') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            echo "Build Backend — ${env.DEPLOY_ENV}"
                            sh 'mvn clean package -DskipTests'
                        }
                    }
                }
            }
        }

        stage('Tests') {
            when {
                expression { return params.RUN_TESTS == true }
            }
            steps {
                echo "Lancement des tests..."
                sh 'mvn test'
            }
        }

        stage('Info Build') {
            steps {
                script {
                    def buildInfo = """
                        App     : ${env.APP_NAME}
                        Version : ${env.VERSION}
                        Env     : ${env.DEPLOY_ENV}
                        Build   : #${env.BUILD_NUMBER}
                        URL     : ${env.BUILD_URL}
                    """
                    echo buildInfo
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${env.APP_NAME}:${env.VERSION}-${env.BUILD_NUMBER} ."
                sh "docker build -t ${env.APP_NAME}:latest ."
                sh "docker images | grep ${env.APP_NAME}"
            }
        }

        stage('Docker Push Registry') {
            steps {
                sh "docker tag ${env.APP_NAME}:${env.VERSION}-${env.BUILD_NUMBER} localhost:5000/${env.APP_NAME}:${env.VERSION}-${env.BUILD_NUMBER}"
                sh "docker tag ${env.APP_NAME}:latest localhost:5000/${env.APP_NAME}:latest"
                sh "docker push localhost:5000/${env.APP_NAME}:${env.VERSION}-${env.BUILD_NUMBER}"
                sh "docker push localhost:5000/${env.APP_NAME}:latest"
                echo "Images poussées :"
                echo "  - localhost:5000/${env.APP_NAME}:${env.VERSION}-${env.BUILD_NUMBER}"
                echo "  - localhost:5000/${env.APP_NAME}:latest"
            }
        }

        stage('Deploy') {
            when {
                expression { return params.ENVIRONNEMENT == 'prod' }
            }
            steps {
                echo "Déploiement en production !"
            }
        }

    }

    post {
        always {
            echo "Pipeline terminé — Build #${env.BUILD_NUMBER}"
        }
        success {
            echo "SUCCESS — ${env.APP_NAME} v${env.VERSION}"
            withCredentials([usernamePassword(
                credentialsId: 'gitlab-token',
                usernameVariable: 'GIT_USER',
                passwordVariable: 'GIT_TOKEN'
            )]) {
                sh '''
                    git config user.email "mustapha.khabthani@gmail.com"
                    git config user.name "Jenkins"
                    git remote set-url origin https://$GIT_USER:$GIT_TOKEN@gitlab.com/Mustapha1122/jenkins-project.git
                    git add . || true
                    git commit -m "build automatique Jenkins [skip ci]" || true
                    git push origin main || true
                '''
            }
        }
        failure {
            echo "FAILURE — Vérifier les logs du build #${env.BUILD_NUMBER}"
        }
    }
}
