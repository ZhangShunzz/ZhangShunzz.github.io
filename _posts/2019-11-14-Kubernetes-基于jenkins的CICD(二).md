---
layout: post
title: "Kubernetes：基于jenkins的CI/CD(二)"
author: "zhangshun"
header-img: "img/background/24.jpg"
header-mask: 0.2
tags:
  - Kubernetes
  - Jenkins
---

##### 一、 增加回滚功能

之前一篇文章介绍了[在kubernetes中进行简单的CICD](https://blog.zs-fighting.cn/2019/11/08/Kubernetes-%E5%9F%BA%E4%BA%8Ejenkins%E7%9A%84CICD(%E4%B8%80)/),但是没有回滚的功能，下面增加下回滚的功能

之前文章是在构建中的时候选择部署的环境，这次我们在构建前选择部署环境

创建一个pipeline风格的任务，在构建时选择**参数化构建过程**，这样我们就可以给任务传递一些参数

![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/参数化构建01.png)
![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/参数化构建02.png)

**构建前需要输入的参数：**
![](/img/in-post/2019-11-08-Kubernetes-基于jenkins的CICD/效果图.png)

**pipeline声明式参考:**
```
node('jenkins_slave') {		\\表明要执行的node

  if (env.Action == "Deploy") {		\\Action选择Deploy时执行下面
    stage('Clone') {
      echo "1.Clone Stage"
      git credentialsId: 'gitlab-ssh-secret', url: 'git@gitlab.intellicredit.cn:zhangshun/kubernetes-jenkins-test.git'
      script {
        build_tag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()		\\全局变量，把git commitID 作为镜像的tag，方便查看跟回滚
        date = sh(script: "date +%Y-%m-%d-%H:%M:%S", returnStdout: true).trim()		\\全局变量，方便查看跟回滚
      }      
    }
    stage('Build Image') {
      echo "2.Build Stage"
      sh "docker login --username=admin --password=123456 192.168.0.109"
      sh "docker build -t 192.168.0.109/zzc_raptor/raptor:${build_tag} ."
    }
    stage('Push Image') {
      echo "3.Push Stage"
      sh "docker login --username=admin --password=123456 192.168.0.109"
      sh "docker push 192.168.0.109/zzc_raptor/raptor:${build_tag}"
    }
    stage('Deploy Yaml') {
      echo "4. Yaml Stage"
      echo "This is a deploy step to \${Env}"
      sh "sed -i 's/<BUILD_TAG>/${build_tag}/g' Raptor_Deployment.yaml"		\\替换镜像的tag
      if (env.Env == "qa") {	\\选择部署的环境
        sh "sed -i 's/<env>/qa/g' Raptor_Deployment.yaml"
      } else if (env.Env == "int"){
        sh "sed -i 's/<env>/int/g' Raptor_Deployment.yaml"
      } else {
        sh "sed -i 's/<env>/prod/g' Raptor_Deployment.yaml"
      }
      sh "cp Raptor_Deployment.yaml Raptor_Deployment_\${Env}_${build_tag}_${date}.yaml"	\\重命名yaml文件，方便以后回滚
      sh "kubectl apply -f Raptor_Deployment_\${Env}_${build_tag}_${date}.yaml --record"
    }
  }

  if (env.Action == "RollBackup") {		\\Action选择Rollbackup时执行下面
    stage('RollBackup') {
        sh "kubectl rollout undo deployment raptor -n \${Env} --to-revision `kubectl rollout history deployment raptor -n \${Env}|grep \${RollBackupName}|awk '{print \$1}'`"
    }
  }
}
```