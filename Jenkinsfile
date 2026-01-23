pipeline {
    agent any // Tells Jenkins where to run the pipeline
    stages {
        stage('checkout') {
            steps {
                echo 'Checking out the code from repository...'
                git branch: 'main', url: 'https://github.com/vikramhemchandar/TravelMemory.git'
            }
        }

        stage('install') {
            steps {
                echo 'Installing the application...'
                sh 'cd backend; node -v; npm -v; npm install'
            }
        }

        stage('build') { 
            steps {
                echo 'Building the application...'
                sh 'cd backend; npm run build'
            }
        }
    }
}
