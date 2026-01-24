pipeline {
    agent any // Tells Jenkins where to run the pipeline
    stages {
        stage('Checkout') {
            steps {
                //echo 'Checking out the code from repository...'
                git branch: 'main', url: 'https://github.com/vikramhemchandar/TravelMemory.git'
            }
        }

        stage('Install') {
            steps {
                //echo 'Installing the application...'
                sh 'cd backend; npm install'
            }
        }

        stage('Build') { 
            steps {
                //echo 'Building the application...'
                sh 'cd backend; npm run build'
            }
        }
    }
}
