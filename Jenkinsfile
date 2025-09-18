pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonarQube-scanner'
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    echo "Building branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=python-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh """
                            docker build -t mughilan23/masterimage . 
                            docker tag mughilan23/masterimage:latest mughilan23/masterimage:${BUILD_NUMBER}
                        """
                    } else if (env.BRANCH_NAME == "dev") {
                        sh """
                            docker build -t mughilan23/devimage . 
                            docker tag mughilan23/devimage:latest mughilan23/devimage:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL mughilan23/masterimage:latest"
                    } else if (env.BRANCH_NAME == "dev") {
                        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL mughilan23/devimage:latest"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'

                    script {
                        if (env.BRANCH_NAME == "master") {
                            sh """
                                docker push mughilan23/masterimage:latest
                                docker push mughilan23/masterimage:${BUILD_NUMBER}
                            """
                        } else if (env.BRANCH_NAME == "dev") {
                            sh """
                                docker push mughilan23/devimage:latest
                                docker push mughilan23/devimage:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy on Docker') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh "docker rm -f masterapp || true"
                        sh "docker run -itd --name masterapp -p 8010:80 mughilan23/masterimage:latest"
                    } else if (env.BRANCH_NAME == "dev") {
                        sh "docker rm -f devapp || true"
                        sh "docker run -itd --name devapp -p 8020:80 mughilan23/devimage:latest"
                    }
                }
            }
        }

        stage('Cleanup Old Images') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh """
                            docker images "mughilan23/masterimage" --format "{{.Repository}}:{{.Tag}}" \
                            | grep -v "latest" \
                            | grep -v "${BUILD_NUMBER}" \
                            | xargs -r docker rmi -f
                        """
                    } else if (env.BRANCH_NAME == "dev") {
                        sh """
                            docker images "mughilan23/devimage" --format "{{.Repository}}:{{.Tag}}" \
                            | grep -v "latest" \
                            | grep -v "${BUILD_NUMBER}" \
                            | xargs -r docker rmi -f
                        """
                    }

                    // Clean dangling layers also
                    sh "docker image prune -f"
                }
            }
        }
    }
}
