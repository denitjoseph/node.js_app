pipeline {

    agent {
        docker {
            image 'node:20'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        DOCKER_IMAGE = "denitjoseph/ultimate-cicd"
        GIT_REPO_NAME = "node.js_app"
        GIT_USER_NAME = "denitjoseph"
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Starting build process...'

                sh '''
                    echo "Workspace:"
                    pwd
                    ls -la
                '''
            }
        }

        stage('Build and Test') {
            steps {
                sh '''
                    cd node-app

                    echo "Installing Node dependencies..."
                    npm ci

                    echo "Running tests..."
                    npm test
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {

                    sh '''
                        echo "Installing Java..."

                        apt-get update
                        apt-get install -y openjdk-17-jre

                        echo "Java version:"
                        java -version

                        cd node-app

                        echo "Running SonarQube analysis..."

                        export SONAR_JAVA_PATH=/usr/bin/java

                        npx sonarqube-scanner \
                          -Dsonar.projectKey=node-express-app \
                          -Dsonar.projectName="Node Express App" \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=node_modules/**,coverage/** \
                          -Dsonar.host.url=$SONAR_HOST_URL
                    '''
                }
            }
        }

        stage('Build and Push Docker Image') {

            steps {

                script {

                    sh '''
                        echo "Installing latest Docker CLI..."

                        apt-get update

                        apt-get install -y \
                            ca-certificates \
                            curl

                        install -m 0755 -d /etc/apt/keyrings

                        curl -fsSL \
                            https://download.docker.com/linux/debian/gpg \
                            -o /etc/apt/keyrings/docker.asc

                        chmod a+r /etc/apt/keyrings/docker.asc

                        echo \
                          "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
                          $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
                          > /etc/apt/sources.list.d/docker.list

                        apt-get update

                        apt-get install -y docker-ce-cli

                        echo "Docker version:"
                        docker --version

                        echo "Docker client/server version:"
                        docker version

                        echo "Building Docker image..."

                        docker build \
                            -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            node-app

                        echo "Docker image built successfully."
                    '''

                    def dockerImage =
                        docker.image("${DOCKER_IMAGE}:${BUILD_NUMBER}")

                    echo "Pushing Docker image to Docker Hub..."

                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'docker-cred'
                    ) {

                        dockerImage.push()

                        dockerImage.push('latest')
                    }

                    echo "Docker image pushed successfully."
                }
            }
        }

        stage('Update Deployment File') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'git-hub',
                        usernameVariable: 'GITHUB_USERNAME',
                        passwordVariable: 'GITHUB_TOKEN'
                    )
                ]) {

                    sh '''
                        echo "Installing Git..."

                        apt-get update
                        apt-get install -y git

                        echo "Cloning repository..."

                        rm -rf repo-temp

                        git clone \
                            https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git \
                            repo-temp

                        cd repo-temp

                        echo "Configuring Git..."

                        git config user.email "denitjoseph25@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        echo "Updating Kubernetes deployment image..."

                        sed -i \
                            "s|image: .*|image: denitjoseph/ultimate-cicd:${BUILD_NUMBER}|g" \
                            node-app-manifests/deployment.yml

                        echo "Updated deployment image:"

                        grep "image:" node-app-manifests/deployment.yml

                        echo "Committing changes..."

                        git add node-app-manifests/deployment.yml

                        git commit \
                            -m "Update Node.js app image to ${BUILD_NUMBER} [skip ci]" \
                            || echo "No changes to commit"

                        echo "Pushing deployment update..."

                        git push origin main

                        echo "Deployment file updated successfully."
                    '''
                }
            }
        }
    }

    post {

        success {
            echo '======================================'
            echo 'PIPELINE COMPLETED SUCCESSFULLY'
            echo '======================================'
        }

        failure {
            echo '======================================'
            echo 'PIPELINE FAILED'
            echo 'Check the stage above for the error.'
            echo '======================================'
        }
    }
}
