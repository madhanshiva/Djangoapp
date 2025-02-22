pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/madhanshiva/Djangoapp.git'
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'mvmadhan/wesalvatore'
        CONTAINER_NAME = 'wesalvatore'
        DOCKER_BUILDKIT = '0'
        TIMESTAMP = new Date().format("yyyyMMddHHmmss")

        // Database connection details (for the app to connect)
        DATABASE_HOST = credentials('DATABASE_HOST')
        DATABASE_USER = credentials('DATABASE_USER')
        DATABASE_PASSWORD = credentials('DATABASE_PASSWORD')
        DATABASE_NAME = credentials('DATABASE_NAME')
        SECRET_KEY = credentials('SECRET_KEY')
    }

    stages {
        stage('Git Clone') {
            steps {
                script {
                    echo "Cloning repository from ${REPO_URL}"
                    git branch: 'main', url: "${REPO_URL}"
                }
            }
        }

        stage('Quality Check') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=django \
                    -Dsonar.projectKey=django \
                    -Dsonar.python.version=3.10'''
                }
            }
        }

        stage('OWASP Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'NVD_KEY', variable: 'NVD_KEY')]) {
                        dependencyCheck additionalArguments: "--scan . --format XML --out dependency-check-report.xml --nvdApiKey=$NVD_KEY",
                                odcInstallation: 'DC',
                                stopBuild: false
                    }
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh '''
                    docker build -t ${DOCKER_IMAGE}:${TIMESTAMP} . 
                    docker tag ${DOCKER_IMAGE}:${TIMESTAMP} ${DOCKER_IMAGE}:latest
                    '''
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        echo "Pushing Docker image to Docker Hub..."
                        sh '''
                        export DOCKER_CONFIG=/tmp/.docker
                        mkdir -p $DOCKER_CONFIG
                        echo "{ \\"auths\\": { \\"https://index.docker.io/v1/\\": { \\"auth\\": \\"$(echo -n ${DOCKER_USER}:${DOCKER_PASSWORD} | base64)\\" } } }" > $DOCKER_CONFIG/config.json
                        docker push ${DOCKER_IMAGE}:latest
                        docker push ${DOCKER_IMAGE}:${TIMESTAMP}
                        '''
                    }
                }
            }
        }

        stage('UAT Deployment') {
            steps {
                script {
                    echo "Deploying application container..."
                    sh '''
                    NETWORK_EXISTS=$(docker network ls --format "{{.Name}}" | grep -w wesalvatore_network || true)
                    if [ -z "$NETWORK_EXISTS" ]; then
                        echo 'Creating Docker network...'
                        docker network create wesalvatore_network
                    else
                        echo 'Network already exists. Skipping creation...'
                    fi

                    if [ "$(docker ps -a -q -f name=${CONTAINER_NAME})" ]; then
                        echo 'Stopping and removing existing app container...'
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    fi

                    echo 'Starting new app container...'
                    docker run -d --restart=always --name ${CONTAINER_NAME} --network wesalvatore_network -p 8000:8000 \
                      -e DATABASE_HOST=${DATABASE_HOST} \
                      -e DATABASE_USER=${DATABASE_USER} \
                      -e DATABASE_PASSWORD=${DATABASE_PASSWORD} \
                      -e DATABASE_NAME=${DATABASE_NAME} \
                      -e SECRET_KEY=${SECRET_KEY} \
                      -v static_volume:/app/staticfiles \
                      -v media_volume:/app/media \
                      ${DOCKER_IMAGE}:latest
                    docker system prune -a -f
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                emailext (
                    subject: "Jenkins Build Notification: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <div style="border: 2px solid #007bff; padding: 15px; background-color: #f8f9fa; font-family: Arial, sans-serif; color: #333;">
                            <h2 style="color: #007bff;">Jenkins Build Notification</h2>
                            <p><strong>Project:</strong> ${env.JOB_NAME}</p>
                            <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                            <p><strong>Build Status:</strong> <span style="color: ${currentBuild.currentResult == 'SUCCESS' ? 'green' : 'red'};"><strong>${currentBuild.currentResult}</strong></span></p>
                            <p><strong>Build URL:</strong> <a href="${BUILD_URL}" style="color: #007bff; text-decoration: none;">Console Output</a></p>
                        </div>
                    """,
                    to: "madhanmv580@gmail.com",
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
