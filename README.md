pipeline{
    agent any
    tools{
        maven 'myMaven'
    }
    stages{
        stage("clean"){
            steps{
                cleanWs()
            }
        }
        stage("Code"){
            steps{
                // From Git sample step
                git branch: 'master', url: 'https://github.com/pranai-reddy/dockerwebapp-flm.git'
            }
        }
        stage("Build"){
            steps{
                sh "mvn clean package"
                sh "cp -r target Docker-app" // We are copying the target folder to DOcker app with in the workspace. This step will be used while pushing the war file to Tomcat dir
            }
        }
        stage("CQA Sonar"){
            steps{
                withSonarQubeEnv('mySonar'){
                    // This was brought from Sonar UI
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=my"
                }
            }
        }
        stage("Quality Gates"){
            steps{
                script{
                    // Sample step - wait for quality gate
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar_Creds'
                }
            }
        }
        stage("Nexus Artifact"){
            steps{
                // From Sample step Nexus artifact uploader
                nexusArtifactUploader artifacts: [[artifactId: 'vprofile', classifier: '', file: 'target/vprofile-v2.war', type: 'war']], credentialsId: 'Nexus_Creds', groupId: 'com.visualpathit', nexusUrl: '15.206.183.86:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'nexusrepo', version: 'v2'
            }
        }
        stage("docker file build"){
            steps{
                sh "docker build -t appimage:1 Docker-app" //Context path in Jenkins workspace where dockerfile is present 
                sh "docker build -t dbimage:1 Docker-db"
            }
        }
        stage("Scan Trivy"){
            steps{
                sh "trivy image appimage:1 >> appimage-report.txt"
                sh "trivy image dbimage:1 >> dbimage-report.txt"
            }
        }
        stage("Compose"){
            steps{
                sh "docker-compose up -d"
            }
        }
       
        
    }
    post{
        always{
            // From emailext:Email Extension sample step
            emailext body: '${currentBuild.currentResult}:* Job ${env.JOB_NAME} \\n build ${env.BUILD_NUMBER} \\n More info at: ${env.BUILD_URL}', subject: 'jenkins build', to: 'pranaireddy.3522@gmail.com'
        }
    }
}
