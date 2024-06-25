pipeline {
    agent any
    
    environment {
        FILE_PATH = "dist.zip"
        filesharename = "frontend"
        storageaccountname = "threetierapplication"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/sharonraju143/Assessment-frontend.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') { // Replace 'sonar' with the actual server name you configured
                    sh '/opt/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=frontend'
                }
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Package Dist Folder') {
            steps {
                sh 'zip -r dist.zip dist'
            }
        }

        stage('Upload to Artifactory') {
            steps {
                withCredentials([string(credentialsId: 'USERNAME', variable: 'jfrogusername'),
                                 string(credentialsId: 'PASSWORD', variable: 'jfrogpassword'),
                                 string(credentialsId: 'ARTIFACTORY_URL', variable: 'jfrogartifacturl')]) {
                    script {
                        def curlCommand = """
                            curl -u${jfrogusername}:${jfrogpassword} -T ${env.FILE_PATH} "${jfrogartifacturl}"
                        """
                        sh curlCommand
                    }
                }
            }
        }

        stage('Push To FileShare in Azure') {
            steps {
                withCredentials([string(credentialsId: 'azurespappid', variable: 'azureclientid'),
                                 string(credentialsId: 'azuresppassword', variable: 'azureclientsecret'),
                                 string(credentialsId: 'azuresptenantid', variable: 'azuretenantid'),
                                 string(credentialsId: 'subscriptionid', variable: 'azuresubscriptionid'),
                                 string(credentialsId: 'storageaccountkey', variable: 'azurestorageaccountkey')]) {
                    sh """
                        az login --service-principal --username ${azureclientid} --password ${azureclientsecret} --tenant ${azuretenantid}
                        az account set --subscription "${azuresubscriptionid}"
                        az storage file upload-batch --destination ${env.filesharename} --source dist --account-name ${env.storageaccountname} --account-key ${azurestorageaccountkey}
                    """  
                }
            }
        }
    }
}
