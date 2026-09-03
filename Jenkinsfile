pipeline {
    agent {
        docker { image 'devops-agent:latest' }
    }

    environment {
        APELLIDO = "jmeza" // Cambiar por apellido
        ACR_NAME = "acrglobalcicd"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP = "rg-cicd-terraform-app-araujobm"
        AKS_NAME = "aks-dev-eastus"
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
                    string(credentialsId: 'azure-clientId',       variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                      echo ">>> Azure login..."
                      az login --service-principal \
                        --username="$AZ_CLIENT_ID" \
                        --password="$AZ_CLIENT_SECRET" \
                        --tenant="$AZ_TENANT_ID"

                      az account set --subscription $AZ_SUBSCRIPTION_ID
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                  echo ">>> Obteniendo credenciales de AKS..."
                  az aks get-credentials \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_NAME \
                    --overwrite-existing
                '''
            }
        }
        
        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo "IMAGE_TAG generado: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                  echo ">>> Login al ACR..."
                  az acr login --name $ACR_NAME

                  echo ">>> Build de imagen..."
                  docker build -t $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG .

                  echo ">>> Push al ACR..."
                  docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('[CD-DEV] Set Image Tag in k8s.yml') {
            steps {
                script { 
                    // Declarar más variables de entorno
                    env.API_PROVIDER_URL = "https://dev.api.com"
                    env.ENV = "dev"
                }

                sh '''
                  echo ">>> Renderizando k8s.yml..."
                  
                  envsubst < k8s.yml > k8s-dev.yml
                  cat k8s-dev.yml

                '''
            }
        }

        stage('[CD-DEV] Deploy to AKS') {
          steps {
            sh '''
                az aks command invoke \
                  --resource-group $RESOURCE_GROUP \
                  --name $AKS_NAME \
                  --command "kubectl apply -f k8s-dev.yml" \
                  --file k8s-dev.yml

            '''
          }
        }
        stage('[CD-DEV] Get LoadBalancer IP') {
            steps {
                sh '''
                  echo ">>> Intentando obtener IP del LoadBalancer..."

                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"  # Cambia esto por el nombre real de tu Service
                  LB_IP=""
                  MAX_RETRIES=60
                  RETRY_COUNT=0
        
                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 5s..."
                      sleep 5
                    fi
                  done
        
                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer después de $MAX_RETRIES intentos."
                    exit 1
                  else
                    echo ">>> IP del LoadBalancer asignada: $LB_IP"
                  fi
                '''
            }
        }

        stage('Aprobación QA') {
            steps {
                input message: '¿Aprobar despliegue a QA?', ok: 'Desplegar a QA'
            }
        }

        stage('[CD-QA] Set Image Tag in k8s.yml') {
            steps {
                script {
                    env.API_PROVIDER_URL = "https://qa.api.com"
                    env.ENV = "qa"
                }

                sh '''
                  echo ">>> Renderizando k8s.yml para QA..."

                  envsubst < k8s.yml > k8s-qa.yml
                  cat k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Deploy to AKS') {
            steps {
                sh '''
                  az aks command invoke \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_NAME \
                    --command "kubectl apply -f k8s-qa.yml" \
                    --file k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Get LoadBalancer IP') {
            steps {
                sh '''
                  echo ">>> Intentando obtener IP del LoadBalancer de QA..."

                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"
                  LB_IP=""
                  MAX_RETRIES=60
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 5s..."
                      sleep 5
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer después de $MAX_RETRIES intentos."
                    exit 1
                  else
                    echo ">>> IP del LoadBalancer de QA: $LB_IP"
                  fi
                '''
            }
        }

        stage('Aprobación PRD') {
            steps {
                input message: '¿Aprobar despliegue a PRD?', ok: 'Desplegar a PRD'
            }
        }

        stage('[CD-PRD] Set Image Tag in k8s.yml') {
            steps {
                script {
                    env.API_PROVIDER_URL = "https://prd.api.com"
                    env.ENV = "prd"
                }

                sh '''
                  echo ">>> Renderizando k8s.yml para PRD..."

                  envsubst < k8s.yml > k8s-prd.yml
                  cat k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Deploy to AKS') {
            steps {
                sh '''
                  az aks command invoke \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_NAME \
                    --command "kubectl apply -f k8s-prd.yml" \
                    --file k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Get LoadBalancer IP') {
            steps {
                sh '''
                  echo ">>> Intentando obtener IP del LoadBalancer de PRD..."

                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"
                  LB_IP=""
                  MAX_RETRIES=60
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc $SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 5s..."
                      sleep 5
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer después de $MAX_RETRIES intentos."
                    exit 1
                  else
                    echo ">>> IP del LoadBalancer de PRD: $LB_IP"
                  fi
                '''
            }
        }

    }
}
