def registry = 'https://chandrujfrog.jfrog.io'
def imageName = 'chandrujfrog.jfrog.io/chandru-docker-local/ttrend'
def version   = '2.1.2'
def HELM_CHART_NAME = "ttrend"

pipeline {
    agent {
        node {
            label 'maven'
        }
    }

    environment {
        PATH = "/opt/apache-maven-3.9.4/bin:/usr/local/aws-cli/v2/current/bin:$PATH"
    }

    stages {

        stage('build') {
            steps {
                sh 'mvn clean deploy -DskipTests'
            }
        }

        stage('tests') {
            steps {
                sh 'mvn surefire-report:report'
            }
        }
 
        stage('SonarQube analysis') {
            environment {
                scannerHome = tool 'chandru-sonar-scanner'
            }
            steps {
                withSonarQubeEnv('chandru-sonarqube-server') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }
 
        stage("Jar Publish to frog") {
            steps {
                script {
                        echo '<--------------- Jar Publish Started --------------->'
                        def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"jfrog"
                        def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                        def uploadSpec = """{
                            "files": [
                                {
                                    "pattern": "jarstaging/(*)",
                                    "target": "libs-release-local/{1}",
                                    "flat": "false",
                                    "props" : "${properties}",
                                    "exclusions": ["*.sha1", "*.md5"]
                                }
                            ]
                        }"""
                        def buildInfo = server.upload(uploadSpec)
                        buildInfo.env.collect()
                        server.publishBuildInfo(buildInfo)
                        echo '<--------------- Jar Publish Ended --------------->'  
                }
            }   
        } 

        stage(" Docker Build ") {
            steps {
                script {
                    echo '<--------------- Docker Build Started --------------->'
                    sh 'sudo usermod -aG docker ubuntu'
                    sh 'touch /var/run/docker.sock && sudo chmod 0777 /var/run/docker.sock'
                    app = docker.build(imageName+":"+version)
                    echo '<--------------- Docker Build Ends --------------->'
                }
            }
        }

        stage (" Docker Publish "){
            steps {
                script {
                    echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'jfrog'){
                    app.push()
                    }    
                    echo '<--------------- Docker Publish Ended --------------->'  
                }
            }
        }

         stage (" Deploy kubernates via helm"){
            steps {
                script {
                    echo '<--------------- kubernates deployment Started --------------->'  
                
                    sh 'aws eks update-kubeconfig --region us-east-1 --name chandru-eks-01'
                    sh 'helm package ttrend'
                    def helmChartExists = sh(script: "helm list | grep -q \"^${HELM_CHART_NAME}\\s\"", returnStatus: true)
                    if (helmChartExists == 0) {
                        sh "helm uninstall ${HELM_CHART_NAME} "
                        sh "helm install ${HELM_CHART_NAME} ttrend-0.1.0.tgz"
                    } else {
                        
                        sh "helm install ${HELM_CHART_NAME} ttrend-0.1.0.tgz"
                    } 
                    sh 'docker system prune -af'
                    echo '<--------------- kubernates deployment Ended. --------------->'  
                }
            }
        }
    }
}
