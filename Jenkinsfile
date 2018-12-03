node{
    stage('SCM Checkout'){
        git 'https://github.com/AA047267/guestbook-go.git'
    }

    stage('SonarQube analysis') {
    // requires SonarQube Scanner 2.8+
    //def scannerHome = tool 'Sonar Scanner';
    withSonarQubeEnv('sonar') {
      sh '/var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/Sonar_Scanner/bin/sonar-scanner'
    }
    }

    stage("Quality Gate") {
        timeout(time: 1, unit: 'MINUTES') { // Just in case something goes wrong, pipeline will be killed after a timeout
            def qg = waitForQualityGate() // Reuse taskId previously collected by withSonarQubeEnv
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            elif (qg.status = 'OK'){
              currentBuild.result = 'SUCCESS'
              return
            }
            }
            
        }
    }

    
    stage('Pre Build Notification'){
        emailext body: "Guestbook-Go:${BUILD_TIMESTAMP} Build Is Starting. Build Url: ${BUILD_URL}", subject: 'PRE_BUILD_NOTIFICATION', to: 'Ashit.Acharya@cerner.com'

    }

    try{
        stage('Build Image'){
            def IMAGE_NAME = "aa047267/guestbook-go:${BUILD_TIMESTAMP}"
            sh "sudo docker build . -t ${IMAGE_NAME}"
            currentBuild.result = 'SUCCESS'
        }
    } 
    catch (Exception err) {
        currentBuild.result = 'FAILURE'
    }
    finally {
        stage('Post Build Notification'){
        emailext attachLog: true, body: "Docker Image Guestbook-Go:${env.BUILD_TIMESTAMP} Build ${currentBuild.result}. Build Url: ${BUILD_URL}", subject: 'POST_BUILD_NOTIFICATION', to: 'Ashit.Acharya@cerner.com'

    }  
    }

    stage('Push Docker Image'){
        withCredentials([string(credentialsId: 'docker-pwd', variable: 'dockerHubPwd')]) {
           sh "sudo docker login -u aa047267 -p ${dockerHubPwd}"
        }       
        sh "sudo docker push aa047267/guestbook-go:${env.BUILD_TIMESTAMP}"
    }
    
    stage('Deploy Approval'){
    input message: "Deploy to prod?", submitter: 'admin'
    }

    try{
        stage('Deploy Image To Kubernetes'){
            def deploy = "kubectl set image deployment/guestbook guestbook=aa047267/guestbook-go:${env.BUILD_TIMESTAMP}"
            sshagent(['k8s-master']){
                sh "ssh -o StrictHostKeyChecking=no root@10.182.235.30 ${deploy}"
            }
            
            currentBuild.result = 'SUCCESS'
        }
    } 
    catch (Exception err) {
        currentBuild.result = 'FAILURE'
    }
    finally {
        stage('Post Deployment Notification'){
        emailext attachLog: true, body: "Guestbook-Go:${env.BUILD_TIMESTAMP} Deployment ${currentBuild.result}. Build Url: ${BUILD_URL}" , subject: 'POST_DEPLOY_NOTIFICATION', to: 'Ashit.Acharya@cerner.com'

    }  
    }

    
}
