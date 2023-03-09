pipeline {
    agent any
    environment {
        registry = "lsnn2/vprofileapp"
        registryCredential = 'dockerhub'
        ARTVERSION = "${env.BUILD_ID}"
    }
    stages{
        stage("Fetch Code") {
            steps {
                git branch: 'master', url: 'https://github.com/lsnn2/cicd-kube-docker.git'
            }
        }
        stage("BUILD"){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo "========A executed successfully========"
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }  
        }
        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {

            environment {
                scannerHome = tool 'mysonarscanner4'
            }

            steps {
                withSonarQubeEnv('sonar-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage("Build Docker Image"){
            steps{
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }
        stage("UPLOAD IMAGE") {
            steps{
                script{
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push("V$BUILD_NUMBER")
                        dockerImage.push("latest")
                    }
                }
            }
        }
        stage("Remove Unused Docker Image") {
            steps{
                sh ' docker rmi $registry:V$BUILD_NUMBER'
            }
        }
        stage("Kubernetes Deploy") {
            agent {label 'KOPS'}
                steps {
                    sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}