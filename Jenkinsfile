pipeline {
    agent any

    environment {
        IMAGE_NAME = "my-app"
        SNYK_TOKEN = credentials('snyk-api-token')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourusername/devsecops-lab.git'
            }
        }

        stage('Build') {
            steps {
                dir('app') {
                    sh 'docker build -t $IMAGE_NAME .'
                }
            }
        }

        stage('Security Scan - Aqua') {
            steps {
                // Aqua CLI local image scan
                sh 'aqua scan --local $IMAGE_NAME || true'
            }
        }

        stage('Security Scan - Clair') {
            steps {
                // Clair analysis via clairctl companion tool
                sh 'clairctl analyze $IMAGE_NAME || true'
            }
        }

        stage('Security Scan - Snyk') {
            steps {
                dir('app') {
                    sh 'snyk auth $SNYK_TOKEN'
                    sh 'snyk test --docker $IMAGE_NAME --file=Dockerfile || true'
                }
            }
        }

        stage('Fail on Critical Vulnerabilities') {
            steps {
                script {
                    def result = sh(
                        script: 'cd app && snyk test --severity-threshold=critical',
                        returnStatus: true
                    )
                    if (result != 0) {
                        error "Critical vulnerabilities detected! Failing pipeline."
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application (only reached if no critical vulnerabilities found)...'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/*.json', allowEmptyArchive: true
        }
        failure {
            echo 'Pipeline failed due to security scan results or build errors. Check logs above.'
        }
    }
}
