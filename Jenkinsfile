pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS' // This assumes you have NodeJS configured in Jenkins Global Tool Configuration
    }
    
    environment {
        BACKUP_DIR = '/var/jenkins_home/backups/jenkins-vuejs'
        DEPLOY_DIR = '/var/www/html/jenkins-vuejs'
        GIT_PREVIOUS_COMMIT = ''
        GIT_CHANGES_DETECTED = 'false'
        EMAIL_RECIPIENTS = 'your-email@example.com' // Replace with your email
    }
    stages {
        stage('Git Pull') {
            steps {
                script {
                    // Store the previous commit hash before pulling
                    GIT_PREVIOUS_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    
                    // Pull the latest changes from the main branch
                    sh 'git fetch origin'
                    sh 'git checkout main'
                    sh 'git pull origin main'
                    
                    // Check if there are changes between the previous and current commit
                    def currentCommit = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
                    if (GIT_PREVIOUS_COMMIT != currentCommit) {
                        echo "Changes detected between ${GIT_PREVIOUS_COMMIT} and ${currentCommit}"
                        GIT_CHANGES_DETECTED = 'true'
                    } else {
                        echo "No changes detected. Skipping build."
                    }
                }
            }
        }
        
        stage('Backup Current Build') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' && fileExists(env.DEPLOY_DIR) }
            }
            steps {
                script {
                    def timestamp = new Date().format('yyyyMMdd_HHmmss')
                    def backupPath = "${BACKUP_DIR}/${timestamp}"
                    
                    // Create backup directory if it doesn't exist
                    sh "mkdir -p ${backupPath}"
                    
                    // Copy current build to backup
                    sh "cp -R ${DEPLOY_DIR}/* ${backupPath}/"
                    echo "Backup created at ${backupPath}"
                }
            }
        }
        
        stage('Setup') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                // Display Node.js and npm versions for debugging
                sh 'node --version'
                sh 'npm --version'
            }
        }
        
        stage('Install') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                // Use 'npm ci' for clean installs in CI environments
                sh 'npm ci || npm install'
            }
        }
        
        stage('Lint') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                sh 'npm run lint || echo "Linting skipped"'
            }
        }
        
        stage('Build') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    try {
                        sh 'npm run build'
                    } catch (Exception e) {
                        // If build fails, restore the backup
                        echo "Build failed. Restoring previous version."
                        def latestBackup = sh(script: "ls -t ${BACKUP_DIR} | head -1", returnStdout: true).trim()
                        if (latestBackup) {
                            sh "rm -rf ${DEPLOY_DIR}/*"
                            sh "cp -R ${BACKUP_DIR}/${latestBackup}/* ${DEPLOY_DIR}/"
                            echo "Restored from backup: ${BACKUP_DIR}/${latestBackup}"
                            
                            // Send email notification about build failure
                            mail to: env.EMAIL_RECIPIENTS,
                                 subject: "Jenkins Vue.js Build Failed - Backup Restored",
                                 body: "The build for jenkins-vuejs failed. The previous version has been restored from backup.\n\nError: ${e.message}"
                        }
                        // Re-throw the exception to mark the stage as failed
                        throw e
                    }
                }
            }
        }
        stage('Test') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                echo 'No tests configured yet'
            }
        }
        
        stage('Archive') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                // Archive the build artifacts
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }
        
        stage('Deploy') {
            when {
                expression { return env.GIT_CHANGES_DETECTED == 'true' }
            }
            steps {
                script {
                    // Create deployment directory if it doesn't exist
                    sh "mkdir -p ${DEPLOY_DIR}"
                    
                    // Deploy the build to the deployment directory
                    sh "rm -rf ${DEPLOY_DIR}/*"
                    sh "cp -R dist/* ${DEPLOY_DIR}/"
                    
                    echo "Deployed to ${DEPLOY_DIR}"
                }
            }
        }
        
        stage('Setup Git Hook') {
            steps {
                script {
                    // Create a post-receive hook in the Git repository
                    def hookDir = '.git/hooks'
                    def hookFile = "${hookDir}/post-receive"
                    
                    sh "mkdir -p ${hookDir}"
                    
                    // Create the post-receive hook script
                    writeFile file: hookFile, text: '''
#!/bin/bash

# Trigger Jenkins build when changes are pushed to the repository
JENKINS_URL="http://your-jenkins-server/job/jenkins-vuejs/build"
JENKINS_USER="your-jenkins-user"
JENKINS_API_TOKEN="your-jenkins-api-token"

# Trigger the build
curl -X POST "${JENKINS_URL}" --user "${JENKINS_USER}:${JENKINS_API_TOKEN}"

echo "Jenkins build triggered."
'''
                    
                    // Make the hook executable
                    sh "chmod +x ${hookFile}"
                    
                    echo "Git post-receive hook set up successfully"
                }
            }
        }
    }
    
    post {
        success {
            script {
                if (env.GIT_CHANGES_DETECTED == 'true') {
                    mail to: env.EMAIL_RECIPIENTS,
                         subject: "Jenkins Vue.js Build Successful",
                         body: "The build for jenkins-vuejs completed successfully and has been deployed."
                }
            }
            echo 'Build completed successfully!'
        }
        failure {
            mail to: env.EMAIL_RECIPIENTS,
                 subject: "Jenkins Vue.js Build Failed",
                 body: "The build for jenkins-vuejs failed. Please check the Jenkins logs for details."
            echo 'Build failed. Please check the logs for details.'
        }
        always {
            // Clean workspace after build but keep the last successful build
            cleanWs(patterns: [[pattern: 'node_modules/**', type: 'EXCLUDE']])
        }
    }
}
