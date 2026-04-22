pipeline {
    agent any

    environment {
        IMAGE_NAME = "iamsnaaz/cicd-pipeline-demo"
        TAG = "latest"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/iamsnaaz/POC7.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    odcInstallation: 'Dependency-Check',
                    additionalArguments: '--scan . --format XML --format HTML'
                )
                dependencyCheckPublisher(
                    pattern: '**/dependency-check-report.xml'
                )
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$TAG .'
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                mkdir -p trivy-cache

                trivy image --download-db-only --cache-dir trivy-cache || true

                trivy image \
                --cache-dir trivy-cache \
                --skip-db-update \
                --severity HIGH,CRITICAL \
                $IMAGE_NAME:$TAG || true
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $IMAGE_NAME:$TAG
                    '''
                }
            }
        }

        stage('Deploy via Ansible') {
            steps {
                sh '''
                ansible-playbook -i /var/lib/jenkins/inventory /var/lib/jenkins/deploy.yml \
                --extra-vars "image=$IMAGE_NAME:$TAG"
                '''
            }
        }
    }

    post {
        always {
            sh '''
            docker image prune -f || true
            rm -rf trivy-cache || true
            '''
        }
    }
}
