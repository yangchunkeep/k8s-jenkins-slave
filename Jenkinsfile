// 公共
def registry = "192.168.31.90"
// 项目
def project = "demo"
def app_name = "java-demo"
def image_name = "${registry}/${project}/${app_name}:${BUILD_NUMBER}"
def git_address = "http://192.168.31.90:88/root/java-demo.git"
// 认证
def secret_name = "registry-auth"
def harbor_auth = "71adf267-8c6d-4c08-9f2d-41332b4f038c"
def git_auth = "f8f9e5b2-4148-48a2-bbf6-ebb1557526e1"
def k8s_auth = "3e5302c4-3f4a-4f49-8a21-3b85fbb166c0"

pipeline {
  agent {
    kubernetes {
        label "jenkins-slave"
        yaml """
kind: Pod
metadata:
  name: jenkins-slave
spec:
  containers:
  - name: jnlp
    image: "${registry}/library/jenkins-slave-jdk:1.8"
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-sock
        mountPath: /var/run/docker.sock
      - name: maven-cache
        mountPath: /root/.m2
  volumes:
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: maven-cache
      hostPath:
        path: /tmp/m2
"""
        }
      
      }
    parameters {    
        gitParameter branch: '', branchFilter: '.*', defaultValue: 'master', description: '选择发布的分支', name: 'Branch', quickFilterEnabled: false, selectedValue: 'NONE', sortMode: 'NONE', tagFilter: '*', type: 'PT_BRANCH'
        choice (choices: ['1', '3', '5', '7'], description: '副本数', name: 'ReplicaCount')
        choice (choices: ['dev','test','prod'], description: '命名空间', name: 'Namespace')
    }
    stages {
        stage('拉取代码'){
            steps {
                checkout([$class: 'GitSCM', 
                branches: [[name: "${params.Branch}"]], 
                doGenerateSubmoduleConfigurations: false, 
                extensions: [], submoduleCfg: [], 
                userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]
                ])
            }
        }

        stage('代码编译'){
           steps {
             sh """
                mvn clean package -Dmaven.test.skip=true
                """ 
           }
        }

        stage('构建镜像'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                  unzip target/*.war -d target/ROOT  
                  echo '
                    FROM lizhenliang/tomcat
                    LABEL maitainer lizhenliang
                    ADD target/ROOT /usr/local/tomcat/webapps/ROOT
                  ' > Dockerfile
                  docker build -t ${image_name} .
                  docker login -u ${username} -p '${password}' ${registry}
                  docker push ${image_name}
                """
                }
           } 
        }
        stage('部署到K8S平台'){
          steps {
              configFileProvider([configFile(fileId: "${k8s_auth}", targetLocation: "admin.kubeconfig")]){
                sh """
                  sed -i 's#IMAGE_NAME#${image_name}#' deploy.yaml
                  sed -i 's#SECRET_NAME#${secret_name}#' deploy.yaml
                  sed -i 's#REPLICAS#${ReplicaCount}#' deploy.yaml
                  kubectl apply -f deploy.yaml -n ${Namespace} --kubeconfig=admin.kubeconfig
                """
              }
          }
        }
    }
}

