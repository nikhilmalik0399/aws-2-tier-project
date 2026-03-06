pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ACCOUNT_ID     = '917791789598'
        IMAGE_TAG          = "1.0.${BUILD_NUMBER}"
        SCANNER_HOME       = tool 'sonar-scanner'
        FRONTEND_ECR_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/frontend-repo"
        BACKEND_ECR_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/backend-repo"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source Code') {
            steps {
                git branch: 'main',
                credentialsId: 'github-cred',
                url: 'https://github.com/nikhilmalik0399/AWs-Eks.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=app \
                        -Dsonar.projectKey=app
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('OWASP Dependency Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                odcInstallation: 'DP-Check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Authenticate to AWS ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials-id']
                ]) {
                    sh '''
                        aws sts get-caller-identity

                        aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
                        docker login --username AWS --password-stdin \
                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Build and Push Docker Images') {

            parallel {

                stage('Frontend Build & Push') {
                    steps {
                        dir('frontend') {
                            sh '''
                                echo "Building Frontend Image..."
                                docker build -t ${FRONTEND_ECR_URI}:${IMAGE_TAG} .

                                docker push ${FRONTEND_ECR_URI}:${IMAGE_TAG}

                                trivy image --severity HIGH,CRITICAL --exit-code 1 ${FRONTEND_ECR_URI}:${IMAGE_TAG}
                            '''
                        }
                    }
                }

                stage('Backend Build & Push') {
                    steps {
                        dir('backend') {
                            sh '''
                                echo "Building Backend Image..."
                                docker build -t ${BACKEND_ECR_URI}:${IMAGE_TAG} .

                                docker push ${BACKEND_ECR_URI}:${IMAGE_TAG}

                                trivy image --severity HIGH,CRITICAL --exit-code 1 ${BACKEND_ECR_URI}:${IMAGE_TAG}
                            '''
                        }
                    }
                }
            }
        }

        stage('Prepare CD Repo') {
            steps {
                cleanWs()

                git branch: 'main',
                credentialsId: 'github-cred',
                url: 'https://github.com/nikhilmalik0399/Helm-Charts-AWs-Teir-Appln-.git'
            }
        }

        stage('Update Helm Values') {

            environment {
                GIT_REPO_NAME = "Helm-Charts-AWs-Teir-Appln"
                GIT_USER_NAME = "nikhilmalik0399"
            }

            steps {

                withCredentials([usernamePassword(
                        credentialsId: 'github-cred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                )]) {

                    sh '''
                        git config user.email "nikhilmalik0399@gmail.com"
                        git config user.name "${GIT_USER_NAME}"

                        echo "Updating Helm values.yaml"

                        sed -i "s|image: .*frontend-repo:.*|image: ${FRONTEND_ECR_URI}:${IMAGE_TAG}|" helm-chart/values.yaml
                        sed -i "s|image: .*backend-repo:.*|image: ${BACKEND_ECR_URI}:${IMAGE_TAG}|" helm-chart/values.yaml

                        git add helm-chart/values.yaml

                        if git diff --cached --quiet; then
                            echo "No changes detected"
                        else
                            git commit -m "Update image version to ${IMAGE_TAG}"
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
                        fi
                    '''
                }
            }
        }

        stage('Cleanup Docker Images') {
            steps {
                sh '''
                    docker system prune -af
                '''
            }
        }
    }
}
