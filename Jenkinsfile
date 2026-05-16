pipeline {
    agent any

    stages {
        stage('1. Kodu GitHub dan çek.') {
            steps {
                git branch: 'main', url: 'https://github.com/kullanicin/repo-adi.git'
            }
        }

        stage('2. Docker image Build') {
            steps {
                sh '''
                    eval $(minikube docker-env)
                    docker build --no-cache -t restoran-app:latest ./app
                '''
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                sh '''
                    kubectl apply -f k8s/pv-pvc.yaml
                    kubectl apply -f k8s/postgres.yaml
                    kubectl apply -f k8s/networkpolicy.yaml
                    kubectl apply -f k8s/deployment.yaml
                    kubectl rollout restart deployment restoran-app
                '''
            }
        }

        stage('Kontrol') {
            steps {
                sh 'kubectl get pods'
            }
        }
    }

    post {
        success {
            echo 'Deploy başarılı! ✅'
        }
        failure {
            echo 'Deploy başarısız! ❌'
        }
    }
}
