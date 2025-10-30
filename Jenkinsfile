pipeline {
    agent any
    
    parameters {
        choice(
            name: 'TEST_ENVIRONMENT',
            choices: ['environment.json'],
            description: '选择测试环境'
        )
        choice(
            name: 'TEST_TYPE',
            choices: ['all', 'smoke', 'regression'],
            description: '选择测试类型'
        )
        booleanParam(
            name: 'SEND_EMAIL',
            defaultValue: true,
            description: '是否发送邮件通知'
        )
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    environment {
        NODEJS_HOME = tool 'NodeJS-16'
        PROJECT_DIR = 'Test-for-jenkins'
        REPORT_DIR = '${WORKSPACE}/reports'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '开始拉取代码...'
                git branch: 'main', 
                url: 'https://your-git-repository.com/Test-for-jenkins.git',
                credentialsId: 'your-git-credentials'
                
                dir("${PROJECT_DIR}") {
                    sh 'echo "当前目录结构:" && find . -type f -name "*.json"'
                }
            }
        }
        
        stage('Environment Setup') {
            steps {
                echo '设置测试环境...'
                dir("${PROJECT_DIR}") {
                    script {
                        // 检查必要的文件是否存在
                        if (!fileExists('postman/collection.json')) {
                            error "缺少 collection.json 文件"
                        }
                        if (!fileExists("postman/${TEST_ENVIRONMENT}")) {
                            error "缺少环境文件: ${TEST_ENVIRONMENT}"
                        }
                        
                        // 显示环境信息
                        sh """
                            echo "Node.js 版本:"
                            node --version
                            echo "NPM 版本:"
                            npm --version
                        """
                    }
                }
            }
        }
        
        stage('Dependencies Installation') {
            steps {
                echo '安装项目依赖...'
                dir("${PROJECT_DIR}") {
                    script {
                        // 检查 package.json 是否存在，如果存在则安装依赖
                        if (fileExists('package.json')) {
                            sh '''
                                echo "安装 npm 依赖..."
                                npm install
                                echo "检查 newman 是否已安装..."
                                npx newman --version || npm install -g newman
                                npm list newman-reporter-htmlextra || npm install newman-reporter-htmlextra
                                npm list newman-reporter-json || npm install newman-reporter-json
                                npm list newman-reporter-junit || npm install newman-reporter-junit
                            '''
                        } else {
                            // 如果项目没有 package.json，全局安装 newman
                            sh '''
                                echo "全局安装 Newman..."
                                npm install -g newman
                                npm install -g newman-reporter-htmlextra
                                npm install -g newman-reporter-json
                                npm install -g newman-reporter-junit
                            '''
                        }
                    }
                }
            }
        }
        
        stage('API Tests Execution') {
            steps {
                echo '执行 API 测试...'
                dir("${PROJECT_DIR}") {
                    script {
                        // 创建报告目录
                        sh 'mkdir -p ${REPORT_DIR}'
                        
                        // 根据测试类型设置不同的参数
                        def testFolder = ""
                        if (params.TEST_TYPE == "smoke") {
                            testFolder = "--folder \"Smoke Tests\""
                        } else if (params.TEST_TYPE == "regression") {
                            testFolder = "--folder \"Regression Tests\""
                        }
                        
                        // 执行 Newman 测试
                        try {
                            sh """
                                echo "开始执行 Postman 测试集合..."
                                npx newman run "postman/collection.json" \
                                    -e "postman/${TEST_ENVIRONMENT}" \
                                    ${testFolder} \
                                    --delay-request 1000 \
                                    --timeout-request 30000 \
                                    --reporters cli,htmlextra,json,junit \
                                    --reporter-htmlextra-export "${REPORT_DIR}/newman-report.html" \
                                    --reporter-json-export "${REPORT_DIR}/newman-report.json" \
                                    --reporter-junit-export "${REPORT_DIR}/newman-report.xml" \
                                    --suppress-exit-code
                            """
                        } catch (Exception e) {
                            echo "测试执行过程中出现错误: ${e.getMessage()}"
                            // 即使测试失败，我们也要继续流程来生成报告
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        
        stage('Test Report Generation') {
            steps {
                echo '生成测试报告...'
                script {
                    // 检查报告文件是否生成
                    sh """
                        echo "检查报告文件:"
                        ls -la ${REPORT_DIR}/ || echo "报告目录不存在"
                    """
                }
            }
            
            post {
                always {
                    // 发布 HTML 报告
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}",
                        reportFiles: 'newman-report.html',
                        reportName: 'Postman API 测试报告'
                    ])
                    
                    // 发布 JUnit 测试结果
                    junit allowEmptyResults: true, 
                          testResults: "${REPORT_DIR}/newman-report.xml"
                    
                    // 归档 JSON 报告
                    archiveArtifacts artifacts: "${REPORT_DIR}/newman-report.json", 
                                    allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo "构建完成 - 结果: ${currentBuild.result}"
            script {
                // 清理工作空间（可选）
                // cleanWs()
                
                // 发送邮件通知
                if (params.SEND_EMAIL) {
                    def emailSubject = ""
                    def emailBody = """
                    <h2>API 自动化测试执行完成</h2>
                    <p><strong>项目名称:</strong> ${env.JOB_NAME}</p>
                    <p><strong>构建编号:</strong> ${env.BUILD_NUMBER}</p>
                    <p><strong>测试环境:</strong> ${params.TEST_ENVIRONMENT}</p>
                    <p><strong>测试类型:</strong> ${params.TEST_TYPE}</p>
                    <p><strong>构建结果:</strong> ${currentBuild.result}</p>
                    <p><strong>构建 URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <p><strong>测试报告:</strong> <a href="${env.BUILD_URL}HTML_Report/">查看详细报告</a></p>
                    <br>
                    <p>此邮件由 Jenkins 自动发送，请勿回复。</p>
                    """
                    
                    if (currentBuild.result == 'SUCCESS') {
                        emailSubject = "✅ SUCCESS: API测试通过 - ${env.JOB_NAME} [${env.BUILD_NUMBER}]"
                    } else if (currentBuild.result == 'UNSTABLE') {
                        emailSubject = "⚠️ UNSTABLE: API测试部分失败 - ${env.JOB_NAME} [${env.BUILD_NUMBER}]"
                    } else {
                        emailSubject = "❌ FAILURE: API测试失败 - ${env.JOB_NAME} [${env.BUILD_NUMBER}]"
                    }
                    
                    emailext (
                        subject: emailSubject,
                        body: emailBody,
                        to: 'your-team@company.com',
                        attachLog: true
                    )
                }
            }
        }
        
        success {
            echo "🎉 所有测试阶段执行成功！"
        }
        
        failure {
            echo "❌ 构建失败，请检查日志和测试报告"
        }
        
        unstable {
            echo "⚠️ 构建不稳定，部分测试失败"
        }
    }
}
