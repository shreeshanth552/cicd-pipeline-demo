pipeline {
    agent any
    tools { nodejs 'NodeJS-18' }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh 'mkdir -p dependency-check-report'
                dependencyCheck additionalArguments: '--scan ./ --format HTML --format XML --out dependency-check-report/ --noupdate', odcInstallation: 'OWASP-Dep-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report/dependency-check-report.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        npm install -g sonarqube-scanner
                        sonar-scanner \
                          -Dsonar.projectKey=cicd-pipeline-demo \
                          -Dsonar.sources=src \
                          -Dsonar.projectName=CI-CD-Pipeline-Demo
                    '''
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker build -t $DOCKER_USER/cicd-demo:latest .
                        docker push $DOCKER_USER/cicd-demo:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Azure') {
            steps {
                withCredentials([azureServicePrincipal('azure-service-principal')]) {
                    sh '''
                        az login --service-principal \
                          -u $AZURE_CLIENT_ID \
                          -p $AZURE_CLIENT_SECRET \
                          --tenant $AZURE_TENANT_ID

                        az group create --name cicd-rg --location eastus

                        az container create \
                          --resource-group cicd-rg \
                          --name cicd-demo \
                          --image shreeshanth552/cicd-demo:latest \
                          --dns-name-label cicd-demo-app \
                          --ports 3000 \
                          --os-type Linux \
                          --restart-policy Always
                    '''
                }
            }
        }

        stage('Deploy to Vercel') {
            steps {
                withCredentials([string(
                    credentialsId: 'vercel-token',
                    variable: 'VERCEL_TOKEN'
                )]) {
                    sh '''
                        npm install -g vercel
                        vercel --token $VERCEL_TOKEN --prod --yes
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed. Check the logs above.'
        }
        always {
            cleanWs()
        }
    }
}