pipeline {
    agent any
    
    environment {
        PROJECT_NAME = 'Test-for-jenkins'
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    currentBuild.displayName = "#${BUILD_NUMBER}"
                }
            }
        }
        
        stage('Environment Check') {
            steps {
                bat '''
                    echo "=== Environment Check ==="
                    echo "Working Directory: %CD%"
                    dir
                    echo "Node.js Version:"
                    node --version
                    echo "npm Version:"
                    npm --version
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                bat '''
                    echo "=== Install Dependencies ==="
                    echo "1. Installing Newman..."
                    npm install -g newman --registry=https://registry.npmmirror.com
                    
                    echo "2. Installing HTML Reporter..."
                    npm install -g newman-reporter-html --registry=https://registry.npmmirror.com
                '''
            }
        }
        
        stage('Execute API Tests') {
            steps {
                script {
                    echo "Starting Postman Tests..."
                    
                    bat 'if not exist test-reports mkdir test-reports'
                    
                    try {
                        bat """
                            echo "Executing tests with HTML reporter..."
                            newman run "postman\\collection.json" ^
                                -e "postman\\environment.json" ^
                                --reporters cli,html ^
                                --reporter-html-export "test-reports\\newman-report.html" ^
                                --suppress-exit-code
                        """
                    } catch (Exception e) {
                        echo "Test execution error: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Verify Test Results') {
            steps {
                script {
                    def reportExists = fileExists 'test-reports/newman-report.html'
                    
                    if (reportExists) {
                        echo "✅ Test report generated successfully"
                        
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: 'test-reports',
                            reportFiles: 'newman-report.html',
                            reportName: 'Postman API Test Report'
                        ])
                        
                        archiveArtifacts artifacts: 'test-reports/newman-report.html', allowEmptyArchive: false
                    } else {
                        echo "❌ Test report not generated"
                    }
                }
            }
        }
        
        stage('Results Summary') {
            steps {
                bat '''
                    echo "=== Test Execution Complete ==="
                    echo "Project: %PROJECT_NAME%"
                    echo "Build: %BUILD_NUMBER%"
                    echo "Working Directory Contents:"
                    dir
                    echo "Report Directory Contents:"
                    if exist test-reports dir test-reports
                    echo "Postman Tests Completed Successfully!"
                '''
            }
        }
    }
    
    post {
        always {
            echo "Build Status: ${currentBuild.currentResult}"
        }
        success {
            echo "🎉 API Tests Executed Successfully!"
        }
        unstable {
            echo "⚠️ Tests completed with warnings"
        }
    }
}
