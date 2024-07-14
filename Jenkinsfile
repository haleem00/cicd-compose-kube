def COLOR_MAP = [
    'SUCCESS': 'good',
    'FAILURE': 'danger',
]

pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK11"
    }
    environment {
      registrydb = "haleemo/vprodb"
      registryapp = "haleemo/vproapp"
      registryCredential = "dockerhub"
//         SLACK_CHANNEL = '#devops'
//         SLACK_CREDENTIALS_ID = 'slacklogin'
//         registryCredential = 'ecr:us-east-1:awscred'
//         appRegistry = "992382655921.dkr.ecr.us-east-1.amazonaws.com/vprofileimage"
//         vprofileRegistry = "https://992382655921.dkr.ecr.us-east-1.amazonaws.com"
    }

    stages {
        stage('Build Maven Project') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Scanner') {
            environment {
                sonarhome = tool 'sonartool'
            }
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        ${sonarhome}/bin/sonar-scanner \
                        -Dsonar.projectKey=vprofile \
                        -Dsonar.projectName=vprofile-repo \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml
                    '''
                }
            }
        }

//         stage("Quality Gate") {
//             steps {
//                 timeout(time: 1, unit: 'HOURS') {
//                     waitForQualityGate abortPipeline: true
//                     script {
//                         def qg = waitForQualityGate()
//                         if (qg.status != 'OK') {
//                             error "Pipeline aborted due to quality gate failure: ${qg.status}"
//                         }
//                     }
//                 }
//             }
//         }

//         stage('Create Dockerfile') {
//             steps {
//                 script {
//                     def dockerfileContent = '''
//                     FROM openjdk:8 AS BUILD_IMAGE
//
//                     RUN apt update && apt install maven -y
//
//                     RUN git clone -b vp-docker https://github.com/imranvisualpath/vprofile-repo.git
//
//                     RUN cd vprofile-repo && mvn clean install -DskipTests
//
//                     FROM tomcat:8-jre11
//
//                     RUN rm -rf /usr/local/tomcat/webapps/*
//
//                     COPY --from=BUILD_IMAGE vprofile-repo/target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
//
//                     EXPOSE 8080
//
//                     CMD ["catalina.sh", "run"]
//                     '''
//                     writeFile file: 'Dockerfile', text: dockerfileContent
//                 }
//             }
//         }

        stage('Build Docker db Image') {
            steps {
                script {
                    dockerImagedb = docker.build(${registryadb} + ":V$BUILD_NUMBER")
                }
            }
        }

       stage('Build Docker app Image') {
                   steps {
                       script {
                           dockerImageapp = docker.build(${registryapp} + ":V$BUILD_NUMBER")
                       }
                   }
               }

        stage('Upload Docker db Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImagedb.push("V$BUILD_NUMBER")
                        dockerImagedb.push('latest')
                    }
                }
            }
        }
        stage('Upload Docker app Image') {
                steps {
                    script {
                        docker.withRegistry('', registryCredential) {
                            dockerImageapp.push("V$BUILD_NUMBER")
                            dockerImageapp.push('latest')
                        }
                    }
                }
            }

        stage('Remove the unused DokerImage') {
          steps {
            sh "docker rmi ${registry}:V$BUILD_NUMBER"
          }
        }

        stage('Kubernetes deploy') {
          agent {label 'KOPS'}
            steps {
              sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
          }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: SLACK_CHANNEL,
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
