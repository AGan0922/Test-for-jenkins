pipeline {
    agent any
    
    parameters {
        choice(
            name: 'TEST_ENVIRONMENT',
            choices: ['environment.json', 'environment.prod.json', 'environment.dev.json'],
            description: '选择测试环境'
        )
        booleanParam(
            name: 'SEND_EMAIL',
            defaultValue: false,
            description: '是否发送邮件通知'
        )
        string(
            name: 'GIT_REPO_URL',
            defaultValue: 'https://github.com/AGan0922/Test-for-jenkins.git',
            description: 'Git仓库URL'
        )
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                echo '克隆测试代码仓库...'
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: params.GIT_REPO_URL]],
                    extensions: [
                        [$class: 'RelativeTargetDirectory', relativeTargetDir: 'test-repo'],
                        [$class: 'CleanBeforeCheckout']
                    ]
                ])
                
                bat """
                    echo "✅ 仓库克隆完成"
                    echo "=== 仓库内容 ==="
                    dir test-repo
                    dir
                """
            }
        }
        
        stage('Check System Environment') {
            steps {
                echo '检查系统环境...'
                bat """
                    echo "=== 系统环境检查 ==="
                    echo "Java版本:"
                    java -version 2>&1
                    echo "Git版本:"
                    git --version
                    echo "Node.js版本:"
                    node --version || echo "Node.js未安装"
                    echo "npm版本:"
                    npm --version || echo "npm未安装"
                    echo "当前工作目录:"
                    cd
                    echo "=== 文件结构 ==="
                    dir test-repo /s /b | findstr /v /i "node_modules" || echo "无文件"
                """
            }
        }
        
        stage('Install Newman') {
            steps {
                echo '安装Newman测试工具...'
                bat """
                    echo "安装Newman及相关报告工具..."
                    npm install -g newman newman-reporter-htmlextra newman-reporter-json-summary --registry=https://registry.npmmirror.com
                    
                    echo "验证安装:"
                    newman --version
                """
                script {
                    def newmanVersion = bat(
                        script: 'newman --version',
                        returnStdout: true
                    ).trim()
                    echo "Newman版本: ${newmanVersion}"
                }
            }
        }
        
        stage('Verify Test Files') {
            steps {
                echo '验证测试文件...'
                bat """
                    echo "=== 检查测试文件 ==="
                    echo "切换到测试仓库目录..."
                    cd test-repo
                    
                    if exist postman\\collection.json (
                        echo "✅ collection.json 存在"
                        echo "文件大小:"
                        for /f %%@i in ('dir /-c postman\\\\collection.json ^| find "collection.json"') do echo %%@i
                    ) else (
                        echo "❌ collection.json 不存在"
                        echo "当前目录文件列表:"
                        dir /b
                        echo "postman目录内容:"
                        dir postman /b 2>nul || echo "postman目录不存在"
                    )
                    
                    if exist postman\\%TEST_ENVIRONMENT% (
                        echo "✅ %TEST_ENVIRONMENT% 存在"
                    ) else (
                        echo "❌ %TEST_ENVIRONMENT% 不存在"
                        echo "可用的环境文件:"
                        dir postman\\*.json /b 2>nul || echo "无JSON文件"
                    )
                """
            }
        }
        
        stage('Run API Tests') {
            steps {
                echo '执行API测试...'
                script {
                    // 创建报告目录
                    bat """
                        if not exist "test-repo\\reports" mkdir "test-repo\\reports"
                        if not exist "test-repo\\reports\\newman" mkdir "test-repo\\reports\\newman"
                    """
                    
                    try {
                        bat """
                            cd test-repo
                            echo "开始执行Postman测试..."
                            newman run "postman\\collection.json" ^
                                -e "postman\\%TEST_ENVIRONMENT%" ^
                                --reporters cli,htmlextra,json-summary ^
                                --reporter-htmlextra-export "reports\\newman\\newman-report.html" ^
                                --reporter-json-summary-export "reports\\newman\\newman-summary.json" ^
                                --suppress-exit-code ^
                                --verbose
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
                    cd test-repo
                    echo "检查生成的报告:"
                    if exist reports\\newman\\newman-report.html (
                        echo "✅ HTML报告生成成功"
                        echo "报告文件大小:"
                        for /f %%@i in ('dir /-c reports\\\\newman\\\\newman-report.html ^| find "newman-report.html"') do echo %%@i
                    ) else (
                        echo "❌ HTML报告未生成"
                    )
                """
                
                script {
                    // 发布HTML报告
                    if (fileExists("test-repo/reports/newman/newman-report.html")) {
                        publishHTML([
                            allowMissing: false,
                            alwaysLinkToLastBuild: true,
                            keepAll: true,
                            reportDir: "test-repo/reports/newman",
                            reportFiles: 'newman-report.html',
                            reportName: 'API测试报告',
                            reportTitles: 'Postman API 测试报告'
                        ])
                        echo "✅ 测试报告发布成功"
                    } else {
                        echo "⚠️ 未找到测试报告文件"
                        currentBuild.result = 'UNSTABLE'
                    }
                    
                    // 归档测试报告
                    archiveArtifacts artifacts: 'test-repo/reports/newman/*.*', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo "构建完成 - 结果: ${currentBuild.currentResult}"
            bat """
                echo "=== 最终工作空间状态 ==="
                dir test-repo /s /b | findstr /v /i "node_modules" || echo "无文件"
            """
            
            script {
                // 读取测试摘要
                def summaryFile = "test-repo/reports/newman/newman-summary.json"
                if (fileExists(summaryFile)) {
                    def summary = readJSON file: summaryFile
                    def totalTests = summary.run.stats.tests.total ?: 0
                    def failedTests = summary.run.stats.tests.failed ?: 0
                    
                    echo "测试统计:"
                    echo "总测试数: ${totalTests}"
                    echo "失败测试: ${failedTests}"
                    echo "通过率: ${totalTests > 0 ? ((totalTests - failedTests) / totalTests * 100).round(2) : 0}%"
                }
                
                // 发送邮件通知（如果配置）
                if (params.SEND_EMAIL) {
                    emailext (
                        subject: "API测试完成 - ${env.JOB_NAME} - ${currentBuild.currentResult}",
                        body: """
                        <h2>API测试执行完成</h2>
                        <p><strong>项目:</strong> ${env.JOB_NAME}</p>
                        <p><strong>构建号:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>结果:</strong> ${currentBuild.currentResult}</p>
                        <p><strong>测试环境:</strong> ${params.TEST_ENVIRONMENT}</p>
                        <p><strong>报告链接:</strong> <a href="${env.BUILD_URL}">查看详细报告</a></p>
                        """,
                        to: "you@example.com"
                    )
                }
                
                // 根据结果设置最终消息
                switch(currentBuild.currentResult) {
                    case 'SUCCESS':
                        echo "🎉 自动化测试执行成功！"
                        break
                    case 'UNSTABLE':
                        echo "⚠️ 测试执行完成，但有失败的测试用例"
                        break
                    default:
                        echo "❌ 构建失败"
                        break
                }
            }
        }
        
        cleanup {
            echo "清理工作空间..."
            // 可选：清理临时文件
            // bat 'rmdir /s /q some-temp-dir 2>nul || echo "清理完成"'
        }
    }
}
