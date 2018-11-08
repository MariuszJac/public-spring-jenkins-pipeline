pipeline {
    agent any
    tools{
	maven 'Maven3'
    }
    stages {
        stage('Compilation and Analysis') {
            parallel {
                stage("Compilation") {
                    steps {
                        hubotSend message: "*Release Started*. \n Releasing Test Project. :sunny: \n<!here> <!channel> <@buildops> ", tokens: "BUILD_NUMBER,BUILD_ID", status: 'STARTED'
                        sh 'mvn -B -DskipTests clean package'
                    }
                }
                stage("Checkstyle checking") {
                    steps {
                        sh "mvn checkstyle:checkstyle"
                        step([$class: 'CheckStylePublisher',
                          canRunOnFailed: true,
                          defaultEncoding: '',
                          healthy: '100',
                          pattern: '**/target/checkstyle-result.xml',
                          unHealthy: '90',
                          useStableBuildAsReference: true
                        ])
                    }
                }
            }
        }
        stage('Testing') {
            parallel {
                stage("Unit testing") {
                    steps {
                        sh 'mvn test -Punit'
                        step([$class: 'JUnitResultArchiver', testResults:'**/target/surefire-reports/TEST-*UnitTest.xml'])
                    }
                }
                stage("Integration testing") {
                    steps {
                        sh 'mvn test -Pintegration'
                        step([$class: 'JUnitResultArchiver', testResults:'**/target/surefire-reports/TEST-'+ '*IntegrationTest.xml'])
                    }
                }
            }
        }
        stage('QA') {
          steps {
            jacoco()
            withSonarQubeEnv('SonarQube') {
              // requires SonarQube Scanner for Maven 3.2+
              sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Dsonar.projectKey=spring-test-app -Dsonar.sources=. -Dsonar.host.url=http://localhost:9000 -Dsonar.login=47af04be6352d6a0f7b8917a189a03b57aafbe69 -Dsonar.test.inclusions=src/test/java/** -Dsonar.exclusions=src/test/java/**'
            }
          }
        }
        stage('M2Storage') {
            steps {
                sh './jenkins/scripts/deliver.sh'
            }
        }

        stage('Packaging'){
            when {
                branch "master"
            }
            steps {
                script {
                    def server = Artifactory.server('artifactory') //this works if maven configuration for artifactory is provided
                    //def server = Artifactory.newServer url: '127.0.0.1:8081/artifactory', username: 'admin', password: 'pa55word!!'
                    def rtMaven = Artifactory.newMavenBuild()
                    rtMaven.resolver server: server, releaseRepo: 'libs-release', snapshotRepo: 'libs-snapshot'
                    rtMaven.deployer server: server, releaseRepo: 'libs-release-local', snapshotRepo: 'libs-snapshot-local'
                    rtMaven.tool = 'Maven3'
                    def buildInfo = rtMaven.run pom: 'pom.xml', goals: 'install'
                    server.publishBuildInfo buildInfo
                }
            }

        }
        stage("Staging deployment") {
            steps {
                script {
                  try {
                    sh 'pid=\$(lsof -i:7070 -t); kill -TERM \$pid || kill -KILL \$pid'
                  } catch (Exception e) {
                    sh 'printf Service to kill not found at port: 7070 - not killing it!'
                  }
                }
                withEnv(['JENKINS_NODE_COOKIE=dontkill']) {
                    sh 'nohup mvn spring-boot:run -Dspring-boot.run.arguments=--server.port=7070 &'
                }
            }
        }
        stage ('Production deployment') {
            when {
                branch 'master'
             }
             steps {
                 input id: 'DeployToProd', message: 'Deploy to production system?', ok: 'Yes'
             }
        }
    }
    post {
        success {
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }
}

        //disabled testing stage as we are running unit and integration tests using profiles defined in pom
        /*
        stage('Testing') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        */
