pipeline {

    agent {
        docker {
            image 'node:20'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {

        stage('Checkout') {
            steps {
                sh '''
                    echo "Starting build process..."
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

                    echo "Installing dependencies..."
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
                        apt-get install -y openjdk17-jre

                        echo "Java version:"
                        java -version

                        cd node-app

                        echo "Running SonarQube analysis..."

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

            environment {
                DOCKER_IMAGE = "denitjoseph/ultimate-cicd:${BUILD_NUMBER}"
            }

            steps {
                script {

                    echo "Installing Docker CLI..."

                    sh '''
                        apt-get update
                        apt-get install -y docker.io
                    '''

                    echo "Building Docker image..."

                    sh '''
                        docker build \
                          -t ${DOCKER_IMAGE} \
                          node-app
                    '''

                    echo "Pushing Docker image to Docker Hub..."

                    def dockerImage =
                        docker.image("${DOCKER_IMAGE}")

                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        'docker-cred'
                    ) {

                        dockerImage.push()

                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Update Deployment File') {

            environment {
                GIT_REPO_NAME = 'node.js_app'
                GIT_USER_NAME = 'denitjoseph'
            }

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

                        echo "Cloning GitHub repository..."

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

                        grep "image:" \
                          node-app-manifests/deployment.yml

                        echo "Committing deployment change..."

                        git add node-app-manifests/deployment.yml

                        git commit \
                          -m "Update Node.js app image tag to ${BUILD_NUMBER} [skip ci]" \
                          || echo "No changes to commit"

                        echo "Pushing deployment change to GitHub..."

                        git push origin main
                    '''
                }
            }
        }
    }
}
