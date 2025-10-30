pipeline {
    agent any
    
    parameters {
        choice(
            name: 'TEST_ENV',
            choices: ['environment.json', 'environment-prod.json'],
            description: '选择测试环境'
        )
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://your-git-repo.com/your-repo.git'
            }
        }
        
        stage('Setup') {
            steps {
                bat '''
                    echo 检查 Node.js 和 Newman...
                    node --version
                    npm list -g newman || npm install -g newman
                    npm list -g newman-reporter-htmlextra || npm install -g newman-reporter-htmlextra
                '''
            }
        }
        
        stage('API Tests') {
            steps {
                bat '''
                    cd postman
                    if not exist reports mkdir reports
                    
                    echo 执行Postman测试集合...
                    newman run "collection.json" ^
                      -e "${TEST_ENV}" ^
                      --reporters cli,htmlextra,junit ^
                      --reporter-htmlextra-export "reports/test-report.html" ^
                      --reporter-junit-export "reports/junit-report.xml" ^
                      --suppress-exit-code
                '''
            }
        }
    }
    
    post {
        always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'postman/reports',
                reportFiles: 'test-report.html',
                reportName: 'Postman Test Report'
            ])
            
            junit 'postman/reports/junit-report.xml'
        }
        
        success {
            emailext (
                subject: "✅ SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "API测试执行成功！\n检查报告: ${env.BUILD_URL}",
                to: "team@example.com"
            )
        }
        
        failure {
            emailext (
                subject: "❌ FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "API测试执行失败！\n检查日志: ${env.BUILD_URL}console",
                to: "team@example.com"
            )
        }
    }
}
