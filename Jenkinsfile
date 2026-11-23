pipeline {
    agent any

    options {
        timeout(time: 45, unit: 'MINUTES')
    }

    environment {
        DOCKER_IMAGE = 'abintony/project4-1'
        GITHUB_REPO = 'https://github.com/Abintony24/project4-1.git'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    echo "Checking project files..."

                    test -f Dockerfile
                    test -f requirements.txt
                    test -f manage.py
                    test -f project4/settings.py
                    test -f k8s/deploy.yml
                    test -f k8s/service.yml

                    echo "Project file checks passed."
                '''
            }
        }

        stage('Semgrep - SAST') {
            steps {
                sh '''
                    semgrep scan --config p/security-audit .
                '''
            }
        }

        stage('Trivy - Filesystem Scan') {
            steps {
                sh '''
                    trivy fs --scanners vuln,misconfig,secret .
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                      -t ${DOCKER_IMAGE}:latest \
                      .
                '''
            }
        }

        stage('Trivy - Docker Image Scan') {
            steps {
                sh '''
                    trivy image ${DOCKER_IMAGE}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'dockerhub-credentials'
                    ) {
                        sh '''
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }

        stage('Update Deployment Files') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'github-token',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        git config user.name "abintony24"
                        git config user.email "abintony24@gmail.com"

                        sed -i \
                          "s|image: abintony/project4-1:.*|image: abintony/project4-1:${BUILD_NUMBER}|" \
                          k8s/deploy.yml

                        git add k8s/deploy.yml

                        if git diff --cached --quiet; then
                            echo "No deployment file changes."
                            exit 0
                        fi

                        git commit -m "Update Project4 image to build ${BUILD_NUMBER}"

                        AUTH=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 -w 0)

                        git -c http.extraheader="AUTHORIZATION: basic ${AUTH}" \
                            push "$GITHUB_REPO" HEAD:main

                        unset AUTH
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Project4 CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            cleanWs()
        }
    }
}
