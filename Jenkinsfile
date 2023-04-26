pipeline{
    agent any
    environment{
        VERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("sonar quality check"){
            agent{
                docker{
                    image 'openjdk:11'
                }
            }
            steps{
                script{
                    withSonarQubeEnv(credentialsId: 'sonar-password') {
                        sh 'chmod +x gradlew'
                        sh './gradlew sonarqube'
                    }
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
        stage("docker build & docker push"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus-password', variable: 'nexus_password')]) {
                        sh '''
                            docker build -t 34.70.160.23:8083/springapp:${VERSION} .
                            docker login -u admin -p $nexus_password 34.70.160.23:8083
                            docker push 34.70.160.23:8083/springapp:${VERSION}
                            docker rmi 34.70.160.23:8083/springapp:${VERSION}
                        '''
                    }
                    
                }
            }
        }
        stage("identifying misconfigs using datree in helm charts"){
            steps{
                script{
                    dir('kubernetes/') {
                        sh 'helm datree test myapp/'
                    }
                }
            }
        }
        stage("pushing the helm charts to nexus"){
            steps{
                script{
                    withCredentials([string(credentialsId: 'nexus-password', variable: 'nexus_password')]) {
                        dir('kubernetes/'){
                            sh '''
                            helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                            tar -czvf myapp-${helmversion}.tgz myapp/
                            curl -u admin:$nexus_password http://34.70.160.23:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz -v
                        '''
                        }
                        
                    }
                    
                }
            }
        }
        stage('manaul approval') {
            steps {
                script{
                    timeout(10){
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> THe build deployment is waiting for your approval. Go to the build url for approval<br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "sethijashandeep@gmail.com";
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }   
                }
            }
        }
        stage('Deploying applications to k8s cluster') {
            steps {
                script{
                    dir('kubernetes/'){
                        sh 'helm upgrade --install --set image.repository="34.70.160.23:8083/springapp" --set image.tag="${VERSION}" myjavaapp myapp/ '    
                    }
                        
                }
            }
        }
    }

    
    post{
        always{
            mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "sethijashandeep@gmail.com";  
        }
	}
}