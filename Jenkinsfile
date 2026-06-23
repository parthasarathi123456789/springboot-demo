pipeline {


agent any

tools {
    maven 'Maven3'
}

environment {
    AWS_REGION = 'ap-southeast-2'
    EKS_CLUSTER = 'prod-eks'

    ECR_REPO = '932708079819.dkr.ecr.ap-southeast-2.amazonaws.com/springboot-demo'
    IMAGE_TAG = "${BUILD_NUMBER}"
    IMAGE_URI = "${ECR_REPO}:${IMAGE_TAG}"
}

stages {

    stage('Checkout') {
        steps {
            checkout scm
        }
    }

    stage('Build') {
        steps {
            dir('demo') {
                sh 'mvn clean package -DskipTests'
            }
        }
    }

    stage('Unit Tests') {
        steps {
            dir('demo') {
                sh 'mvn test'
            }
        }
    }

    stage('SonarQube Scan') {
        steps {
            dir('demo') {
                withSonarQubeEnv('Sonar') {
                    sh 'mvn sonar:sonar'
                }
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

    stage('Trivy Filesystem Scan') {
        steps {
            sh '''
                trivy fs \
                --exit-code 1 \
                --severity HIGH,CRITICAL .
            '''
        }
    }

    stage('Docker Build') {
        steps {
            dir('demo') {
                sh '''
                    docker build -t ${IMAGE_URI} .
                '''
            }
        }
    }

    stage('Trivy Image Scan') {
        steps {
            sh '''
                trivy image \
                --exit-code 1 \
                --severity HIGH,CRITICAL \
                ${IMAGE_URI}
            '''
        }
    }

    stage('Push Image To ECR') {
        steps {
            withCredentials([
                [$class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds']
            ]) {
                sh '''
                    aws ecr get-login-password \
                    --region ${AWS_REGION} | docker login \
                    --username AWS \
                    --password-stdin \
                    932708079819.dkr.ecr.ap-southeast-2.amazonaws.com

                    docker push ${IMAGE_URI}
                '''
            }
        }
    }

    stage('Configure EKS') {
        steps {
            withCredentials([
                [$class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'aws-creds']
            ]) {
                sh '''
                    aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${EKS_CLUSTER}
                '''
            }
        }
    }

    stage('Deploy DEV') {
        steps {
            sh '''
                helm upgrade --install springboot-demo \
                ./helm-chart \
                -n dev \
                --create-namespace \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${IMAGE_TAG}
            '''
        }
    }

    stage('Deploy QA') {
        steps {
            sh '''
                helm upgrade --install springboot-demo \
                ./helm-chart \
                -n qa \
                --create-namespace \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${IMAGE_TAG}
            '''
        }
    }

    stage('Deploy UAT') {
        steps {
            sh '''
                helm upgrade --install springboot-demo \
                ./helm-chart \
                -n uat \
                --create-namespace \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${IMAGE_TAG}
            '''
        }
    }

    stage('Approve Production') {
        steps {
            input message: 'Deploy to Production?'
        }
    }

    stage('Deploy PROD') {
        steps {
            sh '''
                helm upgrade --install springboot-demo \
                ./helm-chart \
                -n prod \
                --create-namespace \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${IMAGE_TAG}
            '''
        }
    }
}

post {
    success {
        echo "SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}"
    }

    failure {
        echo "FAILED: ${JOB_NAME} #${BUILD_NUMBER}"
    }

    always {
        cleanWs()
    }
}


}
