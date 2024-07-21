pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
    }    

    stages {
        stage('Code CheckOut') {
            steps{
                git branch: 'main', credentialsId: 'GitHub_Credentials', url: 'https://github.com/Singh-1991/wonames-script.git'
            }
        }     
        
        stage('Get artifact') {
            steps {
              script {
                def versionsManifest = readYaml file: 'versions_manifest.yml'
                def s3_path = versionsManifest.version_info.ML_model.cloud_model.path
                def tarball_name = s3_path.tokenize('/')[-1]
                // Ensure PWD is correctly defined within the container
                def PWD = sh(script: "echo \$(pwd)", returnStdout: true).trim()
                // env.PWD = PWD
                // Copy artifact from S3 to PWD
                withCredentials([usernamePassword(credentialsId: 'AWS-Credentials', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh """
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                    """
                    sh "aws s3 cp ${s3_path} ${PWD} --debug"
                }
              }
            }
        }

        stage('Validate Hash') {
            steps {
                script {
                    // def PWD = env.PWD
                    // Ensure PWD is correctly defined within the container
                    def PWD = sh(script: "echo \$(pwd)", returnStdout: true).trim()
                        
                    // Untar the tarball
                    sh "tar -xvf ${PWD}/${tarball_name} -C ${PWD}"
                        
                    // Calculate SHA256 hashes for all extracted files except checksum.txt
                    def filesToHash = sh(script: "find ${PWD} -type f ! -name 'checksum.txt'", returnStdout: true).trim().split('\n')
                    def calculatedHashes = [:]
                        
                    filesToHash.each { file ->
                        if (!file.endsWith("/checksum.txt")) {
                            def filename = file.tokenize('/')[-1]
                            calculatedHashes[filename] = sh(script: "sha256sum ${file} | cut -d' ' -f1", returnStdout: true).trim()
                        }
                    }
                        
                    // Read checksums from checksum.txt
                    def checksumFile = readFile("${PWD}/checksum.txt")
                    def checksumLines = checksumFile.readLines()
                        
                    // Process checksums
                    def expectedHashes = [:]
                        
                    checksumLines.each { line ->
                        def parts = line.split(":")
                        def filename = parts[0].trim()
                        def hashValue = parts[1].trim()
                        expectedHashes[filename] = hashValue
                    }
                        
                    // Compare calculated hashes with expected hashes
                    def mismatchedFiles = []
                        
                    expectedHashes.each { filename, expectedHash ->
                        def calculatedHash = calculatedHashes[filename]
                        if (calculatedHash && calculatedHash != expectedHash) {
                            mismatchedFiles.add(filename)
                        }
                    }
                        
                    // Check verification result
                    if (mismatchedFiles.isEmpty()) {
                        echo "Hash verification successful!"
                            // Continue with further stages
                    } else {
                        error "Verification failed: Hash mismatch for files: ${mismatchedFiles.join(', ')}"
                    }
                }
            }
        }
    }

    post {
        always{
            // Clean up workspace
            deleteDir()
        }
    }
}
