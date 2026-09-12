pipeline {
    agent {
        docker {
            image 'devops-agent:latest'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        APELLIDO = 'jmeza'
        ACR_NAME = 'acrjmeza'
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP = 'rg-cicd-aks-jmeza'
        AKS_NAME = 'aks-jmeza-westus'
    }

    stages {
        stage('[CI] Instalar dependencias de app (npm install)') {
            steps {
                sh 'npm install'
            }
        }

        stage('[CI] Ejecutar pruebas unitarias') {
            steps {
                sh 'npm run test:unit'
            }
        }

        stage('[CI] Ejecutar pruebas de integración') {
            steps {
                sh 'npm run test:integration'
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId', variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret', variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId', variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                        az login --service-principal \
                          --username "$AZ_CLIENT_ID" \
                          --password "$AZ_CLIENT_SECRET" \
                          --tenant "$AZ_TENANT_ID" \
                          --output none
                        az account set --subscription "$AZ_SUBSCRIPTION_ID"
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                    az aks get-credentials \
                      --resource-group "$RESOURCE_GROUP" \
                      --name "$AKS_NAME" \
                      --overwrite-existing
                '''
            }
        }

        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    echo "IMAGE_TAG generado: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                    az acr login --name "$ACR_NAME"
                    docker build -t "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG" .
                    docker push "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG"
                '''
            }
        }

        stage('[CD-DEV] Deploy a AKS') {
            environment {
                ENV = 'dev'
                API_PROVIDER_URL = 'https://dev.api.jmeza.local'
            }
            steps {
                sh '''
                    kubectl create namespace "$ENV" --dry-run=client -o yaml | kubectl apply -f -
                    envsubst '${APELLIDO} ${ENV} ${ACR_LOGIN_SERVER} ${IMAGE_NAME} ${IMAGE_TAG} ${API_PROVIDER_URL}' < k8s.yml | kubectl apply -f -
                    kubectl rollout status --namespace "$ENV" "deployment/my-nodejs-app-$APELLIDO-$ENV" --timeout=180s
                '''
            }
        }

        stage('[CD-DEV] Imprimir IP del servicio') {
            environment { ENV = 'dev' }
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-$APELLIDO-$ENV"
                    for RETRY in $(seq 1 60); do
                      LB_IP=$(kubectl get service "$SERVICE_NAME" --namespace "$ENV" --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
                      [ -n "$LB_IP" ] && break
                      echo "Esperando IP DEV ($RETRY/60)..."
                      sleep 5
                    done
                    [ -n "$LB_IP" ] || exit 1
                    echo "IP del servicio DEV: $LB_IP"
                '''
            }
        }

        stage('Aprobacion QA') {
            steps {
                input message: 'Aprobar despliegue a QA?', ok: 'Desplegar a QA'
            }
        }

        stage('[CD-QA] Deploy a AKS') {
            environment {
                ENV = 'qa'
                API_PROVIDER_URL = 'https://qa.api.jmeza.local'
            }
            steps {
                sh '''
                    kubectl create namespace "$ENV" --dry-run=client -o yaml | kubectl apply -f -
                    envsubst '${APELLIDO} ${ENV} ${ACR_LOGIN_SERVER} ${IMAGE_NAME} ${IMAGE_TAG} ${API_PROVIDER_URL}' < k8s.yml | kubectl apply -f -
                    kubectl rollout status --namespace "$ENV" "deployment/my-nodejs-app-$APELLIDO-$ENV" --timeout=180s
                '''
            }
        }

        stage('[CD-QA] Imprimir IP del servicio') {
            environment { ENV = 'qa' }
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-$APELLIDO-$ENV"
                    for RETRY in $(seq 1 60); do
                      LB_IP=$(kubectl get service "$SERVICE_NAME" --namespace "$ENV" --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
                      [ -n "$LB_IP" ] && break
                      echo "Esperando IP QA ($RETRY/60)..."
                      sleep 5
                    done
                    [ -n "$LB_IP" ] || exit 1
                    echo "IP del servicio QA: $LB_IP"
                '''
            }
        }

        stage('Aprobacion PRD') {
            steps {
                input message: 'Aprobar despliegue a PRD?', ok: 'Desplegar a PRD'
            }
        }

        stage('[CD-PRD] Deploy a AKS') {
            environment {
                ENV = 'prd'
                API_PROVIDER_URL = 'https://prd.api.jmeza.local'
            }
            steps {
                sh '''
                    kubectl create namespace "$ENV" --dry-run=client -o yaml | kubectl apply -f -
                    envsubst '${APELLIDO} ${ENV} ${ACR_LOGIN_SERVER} ${IMAGE_NAME} ${IMAGE_TAG} ${API_PROVIDER_URL}' < k8s.yml | kubectl apply -f -
                    kubectl rollout status --namespace "$ENV" "deployment/my-nodejs-app-$APELLIDO-$ENV" --timeout=180s
                '''
            }
        }

        stage('[CD-PRD] Imprimir IP del servicio') {
            environment { ENV = 'prd' }
            steps {
                sh '''
                    SERVICE_NAME="my-nodejs-service-$APELLIDO-$ENV"
                    for RETRY in $(seq 1 60); do
                      LB_IP=$(kubectl get service "$SERVICE_NAME" --namespace "$ENV" --output jsonpath='{.status.loadBalancer.ingress[0].ip}')
                      [ -n "$LB_IP" ] && break
                      echo "Esperando IP PRD ($RETRY/60)..."
                      sleep 5
                    done
                    [ -n "$LB_IP" ] || exit 1
                    echo "IP del servicio PRD: $LB_IP"
                '''
            }
        }
    }
}
