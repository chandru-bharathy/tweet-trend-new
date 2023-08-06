pipeline {
    agent {
        node {
            label 'maven'
        }
    }

    stages {
        stage('clone-code') {
            steps {
                git branch: 'main', credentialsId: 'jenkins-slave-maven', url: 'https://github.com/chandru-bharathy/tweet-trend-new.git'
            }
        }
    }
}
