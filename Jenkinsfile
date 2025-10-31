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
    
    options {
        timeout(time: 30, unit: 'MINUTES')
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
                    echo "当前工作目录:"
                    cd
                    echo "文件列表:"
                    dir
                    echo "=== 检查Postman文件 ==="
                    dir postman || echo "postman目录不存在"
                """
            }
        }
        
        stage('Install Newman') {
            steps {
                echo '安装Newman测试工具...'
                bat """
                    echo "检查Node.js和npm..."
                    node --version || echo "Node.js未安装"
                    npm --version || echo "npm未安装"
                    
                    echo "安装Newman..."
                    npm install -g newman --registry=https://registry.npmmirror.com
                    npm install -g newman-reporter-htmlextra --registry=https://registry.npmmirror.com
                    
                    echo "验证安装:"
                    newman --version || echo "Newman安装失败"
                """
            }
        }
        
        stage('Verify Test Files') {
            steps {
                echo '验证测试文件...'
                bat """
                    echo "=== 检查关键文件 ==="
                    if exist postman\\collection.json (
                        echo "✅ collection.json 存在"
                        echo "文件内容前几行:"
                        type postman\\collection.json | head -3
                    ) else (
                        echo "❌ collection.json 不存在"
                        dir postman || echo "postman目录也不存在"
                    )
                    
                    if exist postman\\%TEST_ENVIRONMENT% (
                        echo "✅ %TEST_ENVIRONMENT% 存在"
                        echo "文件内容前几行:"
                        type postman\\%TEST_ENVIRONMENT% | head -3
                    ) else (
                        echo "❌ %TEST_ENVIRONMENT% 不存在"
                    )
                    
                    echo "=== 当前目录所有文件 ==="
                    dir /s /b || echo "目录列表失败"
                """
            }
        }
        
        stage('Run API Tests') {
            steps {
                echo '执行API测试...'
                script {
                    // 创建报告目录
                    bat "if not exist reports mkdir reports"
                    
                    try {
                        bat """
                            echo "开始执行Postman测试..."
                            newman run "postman\\collection.json" -e "postman\\%TEST_ENVIRONMENT%" --reporters cli,htmlextra --reporter-htmlextra-export "reports\\newman-report.html" --suppress-exit-code
                            echo "测试执行完成!"
                        """
                    } catch (Exception e) {
                        echo "测试执行出错: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Publish Reports') {
            steps {
                echo '发布测试报告...'
                bat """
                    echo "检查生成的报告:"
                    dir reports || echo "报告目录不存在"
                    if exist reports\\newman-report.html (
                        echo "✅ HTML报告生成成功"
                    ) else (
                        echo "❌ HTML报告未生成"
                    )
                """
                
                script {
                    // 发布HTML报告
                    if (fileExists("reports/newman-report.html")) {
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: "reports",
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
                echo "=== 工作空间最终状态 ==="
                cd
                dir
                echo "=== 报告目录 ==="
                dir reports || echo "无报告目录"
            """
            
            script {
                if (currentBuild.currentResult == 'SUCCESS') {
                    echo "🎉 自动化测试执行成功！"
                } else if (currentBuild.currentResult == 'UNSTABLE') {
                    echo "⚠️ 测试执行完成，但有失败的测试用例"
                } else {
                    echo "❌ 构建失败"
                }
            }
        }
    }
}
