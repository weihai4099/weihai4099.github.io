---
date: 2020-02-06 23:04:00 +0800
key: 使用jenkins完成node项目自动化部署
tags: [linux]
---

```groovy
pipeline{
    agent any
    options {
        // 禁止同一个pipeline 并行执行
        disableConcurrentBuilds()
        // 保存最近历史构建记录的数量
        buildDiscarder(logRotator(numToKeepStr:'50'))
    }
    environment{
        //后面使用commit id 打docker tag
        GIT_COMMIT_ID_SHORT = sh (script: "git rev-parse --short HEAD", returnStdout: true);
    }
    triggers {
        // 每分钟判断一次代码是否有变化
        pollSCM('H/10 * * * *')
    }

    tools {
        nodejs 'nodejs'
    }

    stages {
        stage('Build'){
            steps{
                sh 'cnpm install'
                sh 'npm run build'
                echo 'build success'
            }
        }
        stage('parallel'){
            failFast true
            parallel{
                stage('SonarQube analysis') {
                    when {
                        branch "master"
                    }
                    steps{
                        // 暂未试验成功 使用sonar-scanner 验证过一次 直接失灵了
                        echo '--------'

                        //script{
                            // def scannerHome = tool 'sonarqube';
                            // withSonarQubeEnv('sonarqube') {
                            //   sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.6.0.1398:sonar -Dsonar.projectName=容器云子系统'
                            // }
                        //}
                    }
                }

                stage('build & push docker image'){
                    when {
                        branch "master"
                    }
                    steps{
                        echo 'build docker image start'
                        // 之前已经clean 过了 不clean 直接push到harbor
                        sh 'docker build -t harbor.offline/hfcloud/cloud-vue-maintenance -t harbor.offline/hfcloud/cloud-vue-maintenance:${GIT_COMMIT_ID_SHORT} .'
                        sh 'docker push harbor.offline/hfcloud/cloud-vue-maintenance'
                        sh 'docker push harbor.offline/hfcloud/cloud-vue-maintenance:${GIT_COMMIT_ID_SHORT}'
                        // sh 'docker push'
                        echo 'build docker image success'
                    }
                }
            }
        }

        stage('rancher'){
            when {
                branch "master"
            }
            steps{
                script{
                    def projectId = "c-g4xtt:p-294rz";
                    def namespace = "container-cloud";
                    def RANCHER_API_KEY = "rancher";
                    def rancherHost = "https://192.168.40.135:4443";
                    def rancherUrl ="${rancherHost}/v3/project/${projectId}/workloads/deployment:${namespace}:";
                    def dockerImagePrefix = "harbor.offline/hfcloud/hfcloud-";
                    String[] rancherServices = ["cloud-vue-maintenance"];

                    for(int rancherService in rancherServices) {
                        echo "查询服务信息 ${rancherService}"
                        def response = httpRequest acceptType: 'APPLICATION_JSON', authentication: "${RANCHER_API_KEY}", contentType: 'APPLICATION_JSON', httpMode: 'GET', responseHandle: 'LEAVE_OPEN', timeout: 10, url: "${rancherUrl}${rancherService}", ignoreSslErrors:true
                        def serviceInfo = new groovy.json.JsonSlurperClassic().parseText(response.content)
                        response.close()

                        echo "修改docker镜像名为2.0-SNAPSHOT"
                        serviceInfo.containers[0].image = "${dockerImagePrefix}${rancherService}:${GIT_COMMIT_ID_SHORT}";
                        updateJson = new groovy.json.JsonOutput().toJson(serviceInfo)
                        echo "发送重新部署的请求 ${rancherService}"
                        httpRequest acceptType: 'APPLICATION_JSON', authentication: "${RANCHER_API_KEY}", contentType: 'APPLICATION_JSON', httpMode: 'PUT', requestBody: "${updateJson}", responseHandle: 'NONE', timeout: 10, url: "${rancherUrl}${rancherService}", ignoreSslErrors:true

                        echo "结束 ${rancherService}!!"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}

```