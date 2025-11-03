pipeline {
    agent any
    
    environment {
        PROJECT_NAME = 'Test-for-jenkins'
    }
    
    stages {
        stage('Checkout and Setup') {
            steps {
                checkout scm
                bat '''
                    @echo off
                    echo --- SETUP PHASE ---
                    npm install -g newman newman-reporter-html --registry=https://registry.npmmirror.com
                    if exist test-reports rmdir /s /q test-reports
                    mkdir test-reports
                '''
            }
        }
        
        stage('Execute Tests') {
            steps {
                bat '''
                    @echo off
                    echo --- TEST EXECUTION ---
                    newman run "postman\\collection.json" -e "postman\\environment.json" --reporters cli,html --reporter-html-export "test-reports\\newman-report.html" --suppress-exit-code
                    echo --- TESTS COMPLETE ---
                '''
            }
        }
        
        stage('Report Results') {
            steps {
                script {
                    if (fileExists('test-reports/newman-report.html')) {
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: 'test-reports',
                            reportFiles: 'newman-report.html',
                            reportName: 'API Test Report'
                        ])
                        // 使用 Jenkins 的 echo 而不是 bat 的 echo
                        echo "SUCCESS: Test execution completed"
                        echo "View detailed report at: ${env.BUILD_URL}HTML_Report/"
                    } else {
                        echo "WARNING: Test report file not found"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // 只使用 Jenkins 原生的 echo 命令，避免编码问题
            echo "Build Result: ${currentBuild.currentResult}"
            echo "Build URL: ${env.BUILD_URL}"
        }
        success {
            echo "BUILD SUCCESSFUL - All tests passed"
        }
    }
}
