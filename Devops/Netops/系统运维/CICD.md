### **1、整体流程**

![kubernetes发布流程](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/kubernetes%E5%8F%91%E5%B8%83%E6%B5%81%E7%A8%8B.png)

### **2、Jenkins配置**

#### **访问地址**

[https://jenkins.netopstec.com](https://jenkins.netopstec.com/)

#### **项目、视图创建**

一般创建`pipeline`流水线

![image-21](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-21.png)

#### **项目授权**

按项目、环境，分配用户不同权限

![image-22](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-22.png)

![image-23](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-23.png)

#### **Jenkinsfile**

Jenkinsfile配置在每个项目的git仓库，环境与分支对应关系如下：

开发：dev分支

测试：test分支

生产：release分支

整体流程：

- 拉取gitlab代码
- maven打包
- docker镜像制作，并上传至镜像仓库(阿里云ACK或本地harbor)
- k8s发布
- 钉钉通知

发布流程示例：

![image-25](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-25.png)

gitlab中jenkinsfile示例：

![image-26](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-26.png)

jenkins配置示例：

![image-27](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-27.png)

```
def createVersion() {
    // 版本号，输出结果 20191210175842_69
    return new Date().format('yyyyMMddHHmmss') + "_${env.BUILD_ID}"
}

def startTime = new Date(currentBuild.startTimeInMillis).format('yyyy-MM-dd HH:mm:ss')

pipeline {
    // 定义groovy脚本中使用的环境变量
    environment {
        VERSION = createVersion()
        IMAGE = sh(returnStdout: true, script: 'echo $image_region/$image_namespace/${image_reponame}-${profile}:').trim()
        IMAGE_VPC = sh(returnStdout: true, script: 'echo ${image_region_vpc}/$image_namespace/${image_reponame}-${profile}:').trim()
        IMAGE_REPONAME = sh(returnStdout: true, script: 'echo ${image_reponame}').trim()
        PROFILE = sh(returnStdout: true, script: 'echo $profile').trim()
        NODES = sh(returnStdout: true, script: 'echo $nodes').trim()
        JVM_OPT = sh(returnStdout: true, script: 'echo ${jvm_opt}').trim()
        projectName = ''
        projectVersion = ''
        projectDescription = ''
        modelName = ''
    }

    // 定义本次构建使用哪个标签的构建环境
    agent any

    // "stages"定义项目构建的多个模块，可以添加多个 “stage”， 可以多个 “stage” 串行或者并行执行
    stages {
        // 添加第一个stage， 运行源码打包命令
        stage('Read POM') {
            steps {
                script {
                    // 读取pom.xml文件
                    def projectInfo = readYaml file: "pipeline/${PROFILE}/info.yaml"
                    def modelInfo = readYaml file: "${env.WORKSPACE}/${IMAGE_REPONAME}/info.yaml"
                    // 设置全局变量
                    projectName = projectInfo.projectName
                    projectVersion = projectInfo.projectVersion
                    projectDescription = projectInfo.projectDescription
                    modelName = modelInfo.modelName
                }
            }
        }
        stage('MVN Build Package') {
            steps {
                sh 'echo "MVN Build Package"'
                sh "mvn clean package -P${PROFILE} -B -DskipTests -f ${IMAGE_REPONAME}/${IMAGE_REPONAME}-provider/pom.xml"
                echo "Maven Build Success"
            }
        }


        // 添加第二个stage, 运行容器镜像构建和推送命令
        stage('Docker Image Build And Push') {
            steps {
                sh 'echo "Docker Image Build And Push"'
                sh "sed -i 's#JVM_OPT#${JVM_OPT}#g' ${IMAGE_REPONAME}/Dockerfile"
                sh "docker build -t ${IMAGE}${VERSION} -f ${IMAGE_REPONAME}/Dockerfile ${IMAGE_REPONAME}"
                //script {
                //withCredentials([usernamePassword(credentialsId: "${COMPANY}", passwordVariable: 'password', usernameVariable: 'username')]) {
                //  sh "docker login -u ${username} -p ${password} ${image_region}"
                //  sh 'docker push ${IMAGE}${VERSION}'
                //  sh "docker logout ${image_region}"
                //}
                //}
                sh 'docker push ${IMAGE}${VERSION}'
                sh 'docker rmi ${IMAGE}${VERSION}'
                echo "Docker Image ${IMAGE}${VERSION} Build Success"
            }
        }

        // 添加第三个stage, 部署应用到指定k8s集群
        stage('Deploy to Kubernetes') {
            steps {
                sh 'echo "Deploy to Kubernetes"'
                sh "sed -i  's#IMAGE#${IMAGE_VPC}${VERSION}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "sed -i  's#NODES#${nodes}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "sed -i  's#NAMESPACE#${namespace}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "sed -i  's#SERVICE#${IMAGE_REPONAME}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "sed -i  's#PORT#${port}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "sed -i  's#LOGENV#${PROFILE}#g' pipeline/${PROFILE}/deployment.yaml"
                sh "kubectl --kubeconfig ${kubeConfig}/config apply -f pipeline/${PROFILE}/deployment.yaml"
                echo "Deploy to Kubernetes Success"
            }
        }
    }
    post {
        success {
            wrap([$class: 'BuildUser']) {
                dingtalk(
                        robot: "netopstec-wms",
                        type: 'MARKDOWN',
                        title: "success: ${env.JOB_NAME}",
                        text: [
                                "### 构建项目:  [${env.JOB_NAME}](${env.BUILD_URL})",
                                "---",
                                "- 构建 ID:  ${env.BUILD_ID}",
                                "- 构建状态:  <font color=#0000ff>${currentBuild.result}</font>",
                                "- 项目名称:  ${projectName}",
                                "- 模块名称:  ${modelName}",
                                "- 项目版本:  ${projectVersion}",
                                "- 项目描述:  ${projectDescription}",
                                "- 运行环境:  ${PROFILE}",
                                "- 启动时间:  ${startTime}",
                                "- 启动用户:  ${env.BUILD_USER}"
                        ]
                )
            }
        }
        failure {
            wrap([$class: 'BuildUser']) {
                dingtalk(
                        robot: "netopstec-wms",
                        type: 'MARKDOWN',
                        title: "failure: ${env.JOB_NAME}",
                        text: [
                                "### 构建项目:  [${env.JOB_NAME}](${env.BUILD_URL})",
                                "---",
                                "- 任务 ID:  ${env.BUILD_ID}",
                                "- 构建状态:  <font color=#FF0000>${currentBuild.result}</font>",
                                "- 构建日志:  [构建失败日志详情](${env.BUILD_URL}console)",
                                "- 项目名称:  ${projectName}",
                                "- 模块名称:  ${modelName}",
                                "- 项目版本:  ${projectVersion}",
                                "- 项目描述:  ${projectDescription}",
                                "- 运行环境:  ${PROFILE}",
                                "- 启动时间:  ${startTime}",
                                "- 启动用户:  ${env.BUILD_USER}"
                        ]
                )
            }
        }
    }
}
```

#### **版本发布**

发版一般都配置的参数化构建过程

发版示例：

![image-28](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-28.png)

### **3、线上验证**

- k8s中查看服务运行是否正常
- 登录各个系统进行查看