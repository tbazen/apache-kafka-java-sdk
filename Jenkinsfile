pipeline {

    agent {
        docker {
            label 'memphis-jenkins-big-fleet,'
            image 'gradle:8.7.0'
            // image 'gradle:jdk17'            
            args '-u root'
        }
    } 

    environment {
            HOME           = '/tmp'
            TOKEN          = credentials('maven-central-token')
            GPG_PASSPHRASE = credentials('gpg-key-passphrase')
    }

    stages {
        stage('Build and Deploy') {
            steps {
                script {
                    def branchName = env.BRANCH_NAME ?: ''
                    // Check if the branch is 'latest'
                    if (branchName == '3.7-superstream-pipeline') {
                        // Read version from version-beta.conf
                        def version = readFile('version-alpha.conf').trim()
                        // Set the VERSION environment variable to the version from the file
                        env.versionTag = version
                        echo "Using version from version-alpha.conf: ${env.versionTag}"
                    } else {
                        def version = readFile('version.conf').trim()
                        env.versionTag = version
                        echo "Using version from version.conf: ${env.versionTag}"                        
                    }
                }

                withCredentials([file(credentialsId: 'gpg-key', variable: 'GPG_KEY')]) {
                    sh """
                        apt update
                        apt install -y gnupg
                    """
                    sh """
                        echo '${env.GPG_PASSPHRASE}' | gpg --batch --yes --passphrase-fd 0 --import $GPG_KEY
                        echo "allow-loopback-pinentry" > ~/.gnupg/gpg-agent.conf
                        echo RELOADAGENT | gpg-connect-agent
                        echo "D64C041FB68170463BE78AD7C4E3F1A8A5F0A659:6:" | gpg --import-ownertrust 
                        gpg --batch --pinentry-mode loopback --passphrase '${env.GPG_PASSPHRASE}' --export-secret-keys --export-secret-keys -o clients/secring.gpg

                    """
                }               
                sh """

                    ./gradlew :clients:publish -Pversion=${env.versionTag} -Psigning.password=${env.GPG_PASSPHRASE}
                """
                sh "rm /tmp/kafka-clients/ai/superstream/kafka-clients/maven-metadata.xml*"
                script {
                    sh """
                        cd /tmp/kafka-clients
                        tar czvf ai.tar.gz ai
                        curl --request POST \\
                             --verbose \\
                             --header 'Authorization: Bearer ${env.TOKEN}' \\
                             --form bundle=@ai.tar.gz \\
                             https://central.sonatype.com/api/v1/publisher/upload
                    """
                }                     
            }
        }      
    }
    post {
        always {
            cleanWs()
        }
    }    
}