pipeline {
    agent any
    tools {
        nodejs 'NodeJS' // Ensure you have configured NodeJS in Jenkins Global Tool Configuration
    }
    
    environment {
        scannerHome = tool 'SonarQube' // the name you have given the Sonar Scanner (in Global Tool Configuration)
        VERSION = "v2.${BUILD_NUMBER}"
    }
    stages {
        stage("Cleanup") {
            steps {
                cleanWs()
            }
        }
        stage('Git: Checkout') {
            steps {
                echo "Cloning git repo"
                checkout scm
                echo "Cloning Successful"
            }
        }
        stage('Trivy Scan') {
            steps {
                echo "Scanning Files"
                sh "trivy fs ."
                echo "Scanning Successful"
            }
        }
        // stage('Install Dependencies') {
        //     steps {
        //         script {
        //             dir('backend') {
        //                 sh "npm install"
        //             }
        //             dir('frontend') {
        //                 sh "npm install"
        //             }
        //             sh "npm install"
        //         }
        //     }
        // }
        stage('Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'OWASP'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage("Sonar Scanning") {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "${scannerHome}/bin/sonar-scanner -X -Dsonar.projectName=wanderlust -Dsonar.projectKey=wanderlust"
                }
                // script {
                //     def qualitygate = waitForQualityGate()
                //     if (qualitygate.status != "OK") {
                //         error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                //     }
                // }
            }
        }
        stage("Docker Build"){
            steps{
                dir('backend'){
                    echo "Building Backend Image"
                    sh 'whoami'
                    sh "docker build -t wanderlust-backend:${VERSION} ."
                    echo "Build Successful"
                }
                dir('frontend'){
                    echo "Building Frontend Image"
                    sh "docker build -t wanderlust-frontend:${VERSION} ."
                    echo "Build Successful"
                }
            }
        }
        stage("Docker Push") {
            steps {
                
                withCredentials([usernamePassword(credentialsId: "dockerHubCreds", passwordVariable: "dockerHubPass", usernameVariable: "dockerHubUser")]) {

                    echo "Pushing Frontend"
                    sh "echo ${env.dockerHubPass} | docker login -u ${env.dockerHubUser} --password-stdin"
                    sh "docker tag wanderlust-frontend:${VERSION} ${env.dockerHubUser}/wanderlust-frontend:${VERSION}"
                    sh "docker push ${env.dockerHubUser}/wanderlust-frontend:${VERSION}"
                    echo "Pushed Frontend"

                    echo "Pushing Backend"
                    sh "echo ${env.dockerHubPass} | docker login -u ${env.dockerHubUser} --password-stdin"
                    sh "docker tag wanderlust-backend:${VERSION} ${env.dockerHubUser}/wanderlust-backend:${VERSION}"
                    sh "docker push ${env.dockerHubUser}/wanderlust-backend:${VERSION}"
                    echo "Pushed Backend"
        }
        
    }
}
     post{
        success{
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "wanderlust-cd", parameters: [
                string(name: 'VERSION', value: "${VERSION}")
            ]
        }
     }
        
    }
}