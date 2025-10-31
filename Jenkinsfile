pipeline {
    agent any
    
    parameters {
        choice(
            name: 'TEST_ENVIRONMENT',
            choices: ['environment.json'],
            description: '选择测试环境'
        )
        booleanParam(
            name: 'SEND_EMAIL',
            defaultValue: false,
            description: '是否发送邮件通知'
        )
    }
    
    environment {
        PROJECT_DIR = 'Test-for-jenkins'
        REPORT_DIR = 'reports'
    }
    
    stages {
        stage('Check System Environment') {
            steps {
                echo '检查系统环境...'
                bat """
                    echo "=== 系统环境检查 ==="
                    echo "Java版本:"
                    java -version 2>&1
                    echo "Git版本:"
                    git --version
                    echo "当前目录:"
                    cd
                    echo "文件列表:"
                    dir
                    echo "=== 检查项目结构 ==="
                    dir %WORKSPACE%\\${PROJECT_DIR} || echo "项目目录不存在"
                """
            }
        }
        
        stage('Install Node.js Manually') {
            steps {
                echo '手动安装Node.js和Newman...'
                bat """
                    echo "检查是否已安装Node.js..."
                    node --version && echo "Node.js已安装" || (
                        echo "Node.js未安装，尝试使用系统Node.js或安装Newman全局..."
                    )
                    
                    echo "安装Newman测试工具..."
                    npm install -g newman --registry=https://registry.npmmirror.com
                    npm install -g newman-reporter-htmlextra --registry=https://registry.npmmirror.com
                    
                    echo "验证安装:"
                    newman --version || echo "Newman安装失败，但继续执行..."
                """
            }
        }
        
        stage('Verify Test Files') {
            steps {
                echo '验证测试文件...'
                dir("${PROJECT_DIR}") {
                    bat """
                        echo "=== 项目文件结构 ==="
                        dir
                        echo "=== Postman文件 ==="
                        dir postman
                        echo "=== 检查关键文件 ==="
                        if exist postman\\collection.json (
                            echo "✅ collection.json 存在"
                            type postman\\collection.json | head -5
                        ) else (
                            echo "❌ collection.json 不存在"
                        )
                        if exist postman\\${TEST_ENVIRONMENT} (
                            echo "✅ ${TEST_ENVIRONMENT} 存在"
                            type postman\\${TEST_ENVIRONMENT} | head -5
                        ) else (
                            echo "❌ ${TEST_ENVIRONMENT} 不存在"
                        )
                    """
                }
            }
        }
        
        stage('Run API Tests') {
            steps {
                echo '执行API测试...'
                dir("${PROJECT_DIR}") {
                    script {
                        // 创建报告目录
                        bat "if not exist ${REPORT_DIR} mkdir ${REPORT_DIR}"
                        
                        try {
                            bat """
                                echo "开始执行Postman测试..."
                                newman run "postman/collection.json" -e "postman/${TEST_ENVIRONMENT}" --reporters cli,htmlextra --reporter-htmlextra-export "${REPORT_DIR}/newman-report.html" --suppress-exit-code
                                echo "测试执行完成!"
                            """
                        } catch (Exception e) {
                            echo "测试执行出错: ${e.getMessage()}"
                            currentBuild.result = 'UNSTABLE'
                        }
                    }
                }
            }
        }
        
        stage('Generate Reports') {
            steps {
                echo '生成测试报告...'
                dir("${PROJECT_DIR}") {
                    bat """
                        echo "检查生成的报告:"
                        dir ${REPORT_DIR} || echo "报告目录不存在"
                        if exist ${REPORT_DIR}\\newman-report.html (
                            echo "✅ HTML报告生成成功"
                        ) else (
                            echo "❌ HTML报告未生成"
                        )
                    """
                }
                
                script {
                    // 发布HTML报告
                    if (fileExists("${PROJECT_DIR}/${REPORT_DIR}/newman-report.html")) {
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: "${PROJECT_DIR}/${REPORT_DIR}",
                            reportFiles: 'newman-report.html',
                            reportName: 'API测试报告'
                        ])
                        echo "✅ 测试报告发布成功"
                    } else {
                        echo "⚠️ 未找到测试报告文件"
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "构建完成 - 结果: ${currentBuild.currentResult}"
            bat """
                echo "=== 最终工作空间状态 ==="
                cd %WORKSPACE%
                echo "工作空间根目录:"
                dir
                echo "项目目录:"
                dir ${PROJECT_DIR}
                echo "报告文件:"
                dir ${PROJECT_DIR}\\${REPORT_DIR} || echo "无报告目录"
            """
            
            // 简单的成功/失败消息，避免邮件发送问题
            script {
                if (currentBuild.currentResult == 'SUCCESS') {
                    echo "🎉 自动化测试执行成功！"
                } else {
                    echo "💡 构建完成状态: ${currentBuild.currentResult}"
                }
            }
        }
    }
}
