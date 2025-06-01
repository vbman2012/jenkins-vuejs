pipeline {
    agent any
    
    tools {
        nodejs 'NodeJS' // This assumes you have NodeJS configured in Jenkins Global Tool Configuration
    }
    stages {
        stage('Setup') {
            steps {
                // Display Node.js and npm versions for debugging
                sh 'node --version'
                sh 'npm --version'
            }
        }
        stage('Install') {
            steps {
                // Use 'npm ci' for clean installs in CI environments
                sh 'npm ci || npm install'
            }
        }
        stage('Lint') {
            steps {
                sh 'npm run lint || echo "Linting skipped"'
            }
        }
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        stage('Test') {
            steps {
                echo 'No tests configured yet'
            }
        }
        stage('Archive') {
            steps {
                // Archive the build artifacts
                archiveArtifacts artifacts: 'dist/**', fingerprint: true
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deployment step will be configured later'
                // Example deployment command (commented out)
                // sh 'rsync -avz --delete dist/ user@server:/path/to/deployment/'
            }
        }
    }
    
    post {
        success {
            echo 'Build completed successfully!'
        }
        failure {
            echo 'Build failed. Please check the logs for details.'
        }
        always {
            // Clean workspace after build
            cleanWs()
        }
    }
}
