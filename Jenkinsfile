pipeline {
    agent any

    environment {
        APP_NAME     = 'my-jenkins-project'
        REPO_URL     = 'https://github.com/anuri-el/my-jenkins-project.git'
        BRANCH_NAME  = 'main'
        DOCKER_IMAGE = "anuriell/${APP_NAME}"
        DOCKER_TAG   = "${env.BUILD_NUMBER}"
        VENV_DIR     = '.venv'
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        timestamps()
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo "═══ Клонування репозиторію: ${REPO_URL} ═══"
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/${BRANCH_NAME}"]],
                    userRemoteConfigs: [[
                        url: "${REPO_URL}",
                    ]]
                ])
                echo "Код завантажено з гілки: ${BRANCH_NAME}"
                sh 'git log --oneline -5'
            }
        }

        stage('2. Setup Python') {
            steps {
                echo "═══ Налаштування Python середовища ═══"
                sh '''
                    if ! command -v python3 &> /dev/null; then
                        echo "Python3 не знайдено — встановлюємо через apt..."
                        apt-get update -qq
                        apt-get install -y -qq python3 python3-pip python3-venv
                        echo "Python3 встановлено"
                    fi

                    python3 --version
                    
                    python3 -m venv ${VENV_DIR}
                    
                    ${VENV_DIR}/bin/pip install --upgrade pip --quiet
                    ${VENV_DIR}/bin/pip install -r requirements.txt --quiet
                    
                    echo "Python середовище готове"
                    ${VENV_DIR}/bin/pip list
                '''
            }
        }

        stage('3. Lint (flake8)') {
            steps {
                echo "═══ Перевірка стилю коду — flake8 ═══"
                sh '''
                    ${VENV_DIR}/bin/flake8 src/ \
                        --max-line-length=100 \
                        --exclude=__pycache__ \
                        --statistics
                    echo "Lint перевірку пройдено"
                '''
            }
        }

        stage('4. Tests (pytest)') {
            steps {
                echo "═══ Запуск тестів — pytest ═══"
                sh '''
                    ${VENV_DIR}/bin/pytest tests/ \
                        --verbose \
                        --tb=short \
                        --junit-xml=test-results.xml \
                        --cov=src \
                        --cov-report=xml:coverage.xml \
                        --cov-report=term-missing
                    echo "Всі тести пройдено"
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results.xml'
                }
                failure {
                    echo "Тести провалились! Збірка зупинена."
                }
            }
        }

        stage('5. Docker Build') {
            when {
                expression { fileExists('Dockerfile') }
            }
            steps {
                echo "═══ Збірка Docker образу ═══"
                sh """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest
                    echo "Docker образ зібрано: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                """
            }
        }

        stage('6. Deploy') {
            when {
                branch 'main'
            }
            steps {
                echo "═══ Деплой додатку ═══"
                sh '''
                    echo "Розгортання версії: $BUILD_NUMBER"
                    # docker stop app || true
                    # docker rm app || true
                    # docker run -d --name app -p 5000:5000 ${DOCKER_IMAGE}:latest
                    echo "Деплой завершено"
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline завершено. Статус: ${currentBuild.currentResult}"
            echo "Тривалість: ${currentBuild.durationString}"
            sh 'rm -rf ${VENV_DIR} || true'
        }
        success {
            echo "ВСІ КРОКИ УСПІШНІ! Build #${BUILD_NUMBER}"
        }
        failure {
            echo "PIPELINE ПРОВАЛИВСЯ! Build #${BUILD_NUMBER}"
        }
    }
}