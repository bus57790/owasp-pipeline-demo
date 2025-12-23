pipeline {
    agent any

    environment {
        IMAGE_NAME = "owasp-demo-app"
        REPORT_DIR = "reports"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: ' https://github.com/bus57790/owasp-pipeline-demo.git'
            }
        }

        stage('Build & Unit Test') {
            steps {
                sh '''
                echo "Build & test application"
                '''
            }
        }

stage('SAST - Semgrep') {
    steps {
        sh '''
            mkdir -p reports
            semgrep scan --config=auto . --json > reports/semgrep.json || true
        '''
    }
}

        stage('SCA - OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format HTML
                    --out reports
                    --failOnCVSS 7
                ''',
                odcInstallation: 'dependency-check'
            }
        }

        stage('Secrets Scan - Gitleaks') {
            steps {
                sh '''
                gitleaks detect --source . --report-format json --report-path reports/gitleaks.json || true
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:ci .
                '''
            }
        }

        stage('Container Scan - Trivy') {
            steps {
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE_NAME:ci
                '''
            }
        }

        stage('Deploy Test Environment') {
            steps {
                sh '''
                docker run -d -p 8080:80 --name test-app $IMAGE_NAME:ci || true
                sleep 10
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                docker run --rm \
                  -v $(pwd):/zap/wrk \
                  owasp/zap2docker-stable zap-baseline.py \
                  -t http://localhost:8080 \
                  -r zap-report.html || true
                mv zap-report.html reports/
                '''
            }
        }

        stage('Publish Reports') {
            steps {
                publishHTML(target: [
                    reportDir: 'reports',
                    reportFiles: 'dependency-check-report.html,zap-report.html',
                    reportName: 'OWASP Security Reports'
                ])
            }
        }
    }

    post {
        always {
            sh 'docker rm -f test-app || true'
            archiveArtifacts artifacts: 'reports/**', fingerprint: true
        }
        failure {
            echo "❌ Security gate failed"
        }
        success {
            echo "✅ Secure pipeline passed"
        }
    }
}

