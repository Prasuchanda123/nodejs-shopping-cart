pipeline {
    agent any
    tools {
        nodejs 'nodejs'
    }
    environment {
        ACR_NAME = 'luckyregistry11'
        ACR_LOGIN_SERVER = 'luckyregistry11.azurecr.io'
        IMAGE_NAME = 'nodejs-shpping-cart'
        RESOURCE_GROUP = 'lucky'
        AKS_CLUSTER = 'lucky-aks-cluster11'
        HELM_RELEASE = 'nodejs-shopping-cart'
        HELM_CHART_PATH = 'helm/nodejs-shopping-cart'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/luckyncpl/nodejs-shopping-cart'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube SAST Scan') {
                steps {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=nodejs-shopping-cart_12345 \
                        -Dsonar.projectName=nodejs-shopping-cart \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,helm/**,data/** \
                        -Dsonar.token=$SONAR_AUTH_TOKEN
                        '''
                    }
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

        stage('Snyk SCA Scan') {
            steps {
                withCredentials([string(credentialsId: 'snyk-token', variable: 'SNYK_TOKEN')]) {
                    sh '''
                    snyk test --severity-threshold=high
                    synk monitor
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                docker tag $IMAGE_NAME:$IMAGE_TAG $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                docker tag $IMAGE_NAME:$IMAGE_TAG $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 0 $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Login to ACR') {
            steps {
                sh '''
                az acr login --name $ACR_NAME
                '''
            }
        }

        stage('Push Image to ACR') {
            steps {
                sh '''
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Connect to AKS') {
            steps {
                sh '''
                az aks get-credentials \
                --resource-group $RESOURCE_GROUP \
                --name $AKS_CLUSTER \
                --overwrite-existing
                '''
            }
        }

        stage('Helm Deploy to AKS') {
            steps {
                sh '''
                helm upgrade --install $HELM_RELEASE $HELM_CHART_PATH \
                --set image.repository=$ACR_LOGIN_SERVER/$IMAGE_NAME \
                --set image.tag=$IMAGE_TAG
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get pods
                kubectl get svc
                kubectl get hpa
                '''
            }
        }
    }
}