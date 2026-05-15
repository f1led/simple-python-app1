pipeline {
    agent any

    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Иванов Иван', description: 'ФИО студента')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты')
    }

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = "f1led/student-app:${BUILD_NUMBER}"
        CONTAINER_NAME = "student-app-${ENVIRONMENT}"
        // Порты выбраны из свободных (не конфликтуют с заданными)
        PORT_DEV = '5050'
        PORT_STAGING = '5051'
        PORT_PROD = '5052'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Tests') {
            when {
                expression { params.RUN_TESTS }
            }
            steps {
                echo 'Запуск тестов...'
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install -r requirements.txt
                python -m unittest test_app.py -v
                '''
            }
        }

        stage('Build and Push') {
            steps {
                echo 'Сборка и публикация Docker образа...'
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        def customImage = docker.build("${DOCKER_IMAGE}")
                        customImage.push()
                        
                        // Тегируем latest только если это production
                        if (params.ENVIRONMENT == 'production') {
                            customImage.push('latest')
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT == 'dev' }
            }
            steps {
                echo 'Развертывание в DEV окружении...'
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p ${PORT_DEV}:5000 \
                      -e STUDENT_NAME='${params.STUDENT_NAME} (DEV)' \
                      ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                echo 'Развертывание в STAGING окружении...'
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p ${PORT_STAGING}:5000 \
                      -e STUDENT_NAME='${params.STUDENT_NAME} (STAGING)' \
                      ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Approve Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                input message: "Подтвердите развертывание в PRODUCTION?", ok: "Да, развернуть"
                echo "Развертывание в PRODUCTION одобрено"
            }
        }

        stage('Tag Version') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                echo 'Создание Git-тега для релиза...'
                withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins Pipeline"
                    git tag -a v1.0.${BUILD_NUMBER} -m "Release v1.0.${BUILD_NUMBER}"
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/f1led/simple-python-app.git v1.0.${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                echo 'Развертывание в PRODUCTION окружении...'
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p ${PORT_PROD}:5000 \
                      -e STUDENT_NAME='${params.STUDENT_NAME} (PRODUCTION)' \
                      ${DOCKER_IMAGE}
                    """
                }
            }
        }
    }

    post {
        success {
            script {
                echo "Pipeline успешно выполнен для ${params.ENVIRONMENT}"
            }
            emailext(
                to: 'admin@example.com',
                subject: "SUCCESS: Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Развертывание в ${params.ENVIRONMENT} прошло успешно. Доступ по порту (Dev: ${PORT_DEV}, Stg: ${PORT_STAGING}, Prod: ${PORT_PROD})."
            )
        }
        failure {
            emailext(
                to: 'admin@example.com',
                subject: "FAILED: Pipeline ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Pipeline завершился с ошибкой. Проверьте логи: ${env.BUILD_URL}"
            )
        }
        always {
            echo 'Очистка рабочего пространства...'
            cleanWs()
        }
    }
}
