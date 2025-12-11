pipeline {
    agent any

    options {
        // 保留最后10次构建记录
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // 构建超时时间
        timeout(time: 1, unit: 'HOURS')
        // 禁用并发构建
        disableConcurrentBuilds()
        // 构建前清理工作空间（可选）
        skipDefaultCheckout()
    }

    triggers {
        // 通过 Generic Webhook Trigger 插件配置
        // Token 需与 GitHub Actions 中的 JENKINS_TOKEN 一致
        genericWebhook(
            token: '${JENKINS_TOKEN}',
            causeString: 'GitHub Actions Webhook Trigger',
            printContributedVariables: true,
            printPostContent: true
        )
    }

    environment {
        // ACR 仓库信息
        ACR_REGISTRY_PUBLIC = credentials('ACR_REGISTRY_PUBLIC')
        ACR_REGISTRY_INTRANET = credentials('ACR_REGISTRY_INTRANET')  // 可选，内网地址
        ACR_NAMESPACE = credentials('ACR_NAMESPACE')
        
        // 选择使用内网或公网地址
        ACR_REGISTRY = "${ACR_REGISTRY_INTRANET ?: ACR_REGISTRY_PUBLIC}"
        
        // ACR 登录凭据
        ACR_CREDENTIALS = credentials('acr-credentials')  // Jenkins Credential (用户名/密码)
        
        // Docker 配置路径（可选）
        DOCKER_CONFIG_PATH = '/root/.docker'
        
        // 日志输出
        LOG_PREFIX = '[Jenkins Deploy]'
    }

    stages {
        stage('准备') {
            steps {
                script {
                    echo "${LOG_PREFIX} 开始部署流程"
                    sh 'echo "${LOG_PREFIX} Jenkins Node: ${NODE_NAME}"'
                    sh 'echo "${LOG_PREFIX} 工作空间: ${WORKSPACE}"'
                }
            }
        }

        stage('解析镜像列表') {
            steps {
                script {
                    echo "${LOG_PREFIX} 开始解析镜像列表"
                    
                    // 从 Webhook 载荷中获取镜像列表
                    // 此处假设 GitHub Actions 通过 Webhook 传递了镜像列表
                    // 若未传递，则从本地 docker-compose.yml 解析
                    
                    def images = []
                    
                    // 优先从 Webhook payload 中获取 images 参数
                    if (env.images) {
                        echo "${LOG_PREFIX} 从 Webhook 获取镜像列表: ${env.images}"
                        images = env.images.split(',').collect { it.trim() }
                    } else {
                        // 如果 Webhook 未提供，则从 docker-compose.yml 解析
                        echo "${LOG_PREFIX} 从 docker-compose.yml 解析镜像列表"
                        
                        // 检查是否安装了 yq
                        def hasYq = sh(
                            script: 'which yq > /dev/null 2>&1',
                            returnStatus: true
                        ) == 0
                        
                        if (hasYq) {
                            // 使用 yq 解析
                            def result = sh(
                                script: '''
                                    yq eval '.services[].image' deploy/docker-compose.yml | sort -u
                                ''',
                                returnStdout: true
                            ).trim()
                            images = result.split('\n').collect { it.trim() }.findAll { it }
                        } else {
                            // 备选方案：使用 docker compose config
                            echo "${LOG_PREFIX} yq 未安装，使用 docker compose config"
                            def result = sh(
                                script: '''
                                    docker compose -f deploy/docker-compose.yml config | grep 'image:' | awk '{print $2}' | sort -u
                                ''',
                                returnStdout: true
                            ).trim()
                            images = result.split('\n').collect { it.trim() }.findAll { it }
                        }
                    }
                    
                    // 保存镜像列表供后续步骤使用
                    env.IMAGE_LIST = images.join(',')
                    echo "${LOG_PREFIX} 解析完成，共 ${images.size()} 个镜像:"
                    images.each { img ->
                        echo "  - ${img}"
                    }
                }
            }
        }

        stage('登录ACR仓库') {
            steps {
                script {
                    echo "${LOG_PREFIX} 开始登录 ACR 仓库: ${ACR_REGISTRY}"
                    
                    withCredentials([usernamePassword(
                        credentialsId: 'acr-credentials',
                        usernameVariable: 'ACR_USER',
                        passwordVariable: 'ACR_PASS'
                    )]) {
                        sh '''
                            echo "${LOG_PREFIX} 使用凭据登录 ACR"
                            echo "${ACR_PASS}" | docker login -u "${ACR_USER}" --password-stdin "${ACR_REGISTRY}"
                            if [ $? -eq 0 ]; then
                                echo "${LOG_PREFIX} ACR 登录成功"
                            else
                                echo "${LOG_PREFIX} ACR 登录失败"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }

        stage('拉取并改标镜像') {
            steps {
                script {
                    echo "${LOG_PREFIX} 开始拉取并改标镜像"
                    
                    def images = env.IMAGE_LIST.split(',').collect { it.trim() }.findAll { it }
                    def successCount = 0
                    def failureCount = 0
                    
                    images.each { sourceImage ->
                        def imageName = sourceImage.split(':')[0]
                        def imageTag = sourceImage.contains(':') ? sourceImage.split(':')[1] : 'latest'
                        
                        // 构造 ACR 镜像名
                        def acrImage = "${ACR_REGISTRY}/${ACR_NAMESPACE}/${imageName.split('/')[-1]}:${imageTag}"
                        
                        try {
                            echo "${LOG_PREFIX} 处理镜像: ${sourceImage}"
                            echo "  ACR 镜像: ${acrImage}"
                            
                            // 从 ACR 拉取镜像
                            echo "${LOG_PREFIX} 拉取 ACR 镜像..."
                            sh "docker pull ${acrImage}"
                            
                            // 改标为源镜像名，供 docker-compose 使用
                            echo "${LOG_PREFIX} 改标为源镜像名: ${sourceImage}"
                            sh "docker tag ${acrImage} ${sourceImage}"
                            
                            // 可选：删除 ACR 镜像标签以释放空间
                            echo "${LOG_PREFIX} 删除 ACR 镜像标签以释放空间"
                            sh "docker rmi ${acrImage}" || true
                            
                            successCount++
                            echo "${LOG_PREFIX} ✓ 镜像 ${sourceImage} 处理成功"
                        } catch (Exception e) {
                            failureCount++
                            echo "${LOG_PREFIX} ✗ 镜像 ${sourceImage} 处理失败: ${e.message}"
                            // 继续处理下一个镜像，不中断流程
                        }
                    }
                    
                    echo "${LOG_PREFIX} 镜像处理完成: ${successCount} 成功, ${failureCount} 失败"
                    
                    // 如果有失败的镜像，构建失败
                    if (failureCount > 0) {
                        error("${LOG_PREFIX} 有 ${failureCount} 个镜像处理失败")
                    }
                }
            }
        }

        stage('镜像验证') {
            steps {
                script {
                    echo "${LOG_PREFIX} 开始镜像验证"
                    
                    sh '''
                        echo "${LOG_PREFIX} 本地镜像列表："
                        docker images --filter "dangling=false" --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
                        
                        echo "${LOG_PREFIX} 检查镜像是否可用..."
                        # 这里可以添加更多镜像验证逻辑
                    '''
                }
            }
        }

        stage('触发部署') {
            steps {
                script {
                    echo "${LOG_PREFIX} 准备触发部署"
                    
                    sh '''
                        echo "${LOG_PREFIX} 镜像已准备就绪"
                        echo "${LOG_PREFIX} 下一步："
                        echo "  1. 登录到部署服务器"
                        echo "  2. 进入项目目录"
                        echo "  3. 运行: docker-compose up -d"
                        echo "${LOG_PREFIX} 或使用 ansible/kubectl 自动化部署（可选）"
                    '''
                    
                    // 可选：如果有自动部署脚本，在此调用
                    // sh './deploy/deploy.sh'
                }
            }
        }

        stage('日志清理') {
            steps {
                script {
                    echo "${LOG_PREFIX} 执行日志和资源清理"
                    
                    sh '''
                        echo "${LOG_PREFIX} 删除未使用的 Docker 镜像"
                        docker image prune -f --filter "dangling=true" || true
                        
                        echo "${LOG_PREFIX} 当前 Docker 磁盘使用情况："
                        docker system df
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                echo "${LOG_PREFIX} 清理工作"
                
                // 登出 Docker
                sh '''
                    docker logout "${ACR_REGISTRY}" 2>/dev/null || true
                    echo "${LOG_PREFIX} Docker 已登出"
                '''
            }
        }

        success {
            script {
                echo "${LOG_PREFIX} ✓ 构建成功"
                // 可选：发送成功通知（如邮件、Slack、钉钉等）
                // sh 'curl -X POST -H "Content-type: application/json" ...'
            }
        }

        failure {
            script {
                echo "${LOG_PREFIX} ✗ 构建失败"
                // 可选：发送失败通知
                // sh 'curl -X POST -H "Content-type: application/json" ...'
            }
        }

        unstable {
            script {
                echo "${LOG_PREFIX} ⚠ 构建不稳定"
            }
        }

        cleanup {
            script {
                echo "${LOG_PREFIX} 最终清理"
                
                // 清理 Docker 临时文件
                sh 'rm -f ${DOCKER_CONFIG_PATH}/config.json.bak 2>/dev/null || true'
            }
        }
    }
}
