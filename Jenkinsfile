pipeline {
    agent any

    tools {
        jdk 'jdk17'
    }

    environment {
        MAVEN_OPTS = '-Dmaven.repo.local=.m2/repository'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {

        stage('拉取代码') {
            steps {
                checkout scm
            }
        }

        stage('编译打包') {
            steps {
                sh './mvnw -B clean package -DskipTests'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('单元测试') {
            steps {
                sh './mvnw -B test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('代码扫描 (可选)') {
            steps {
                withSonarQubeEnv('sonarqube') {   // 需在 Jenkins 系统配置中配置 SonarQube Server
                    sh './mvnw -B sonar:sonar'
                }
            }
            when {
                expression { params.SONAR_SCAN }  // 手动构建时可跳过
            }
        }

        stage('构建镜像') {
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}"
                    appImage = docker.build("jenkins-demo:${imageTag}", '-f Dockerfile .')
                }
            }
            when {
                expression { fileExists('Dockerfile') }
            }
        }

        stage('部署') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main' || params.DEPLOY) {
                        echo "部署 jenkins-demo:${env.BUILD_NUMBER} 到目标环境..."
                        // 常见部署方式,按需启用:
                        // sh "docker stop jenkins-demo || true && docker rm jenkins-demo || true"
                        // sh "docker run -d --name jenkins-demo -p 8080:8080 jenkins-demo:${env.BUILD_NUMBER}"
                        // 或推送到镜像仓库 / 触发 K8s 滚动更新:
                        // sh "kubectl set image deployment/jenkins-demo jenkins-demo=jenkins-demo:${env.BUILD_NUMBER}"
                    } else {
                        echo "非 main 分支且未选择部署,跳过"
                    }
                }
            }
        }

    }

    post {
        success {
            echo '构建成功 ✅'
        }
        failure {
            echo '构建失败 ❌'
            // mail to: 'team@example.com', subject: "Jenkins 构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}", body: '请查看控制台日志'
        }
        always {
            cleanWs notFailBuild: true
        }
    }
}
