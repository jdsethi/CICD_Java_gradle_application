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
                    withCredentials([string(credentialsId: 'nexus-password', variable: 'nexus-password')]) {
                        sh '''
                            docker build -t 34.70.160.23:8083/springapp:${VERSION} .
                            docker login -u admin -p $nexus-password 34.70.160.23:8083
                            docker push 34.70.160.23:8083/springapp:${VERSION}
                            docker rmi 34.70.160.23:8083/springapp:${VERSION}
                        '''
                    }
                    
                }
            }
        }
    }
}