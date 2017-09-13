pipeline {
    agent any
    stages {
        stage("One") {
            steps {
                echo "Hello"
		checkout scm
            }
        }
        stage("Two") {
            when {
                changeset "**/*.js"
            }
            steps {
                script {
                    echo "JS World"
                }

            }
        }
    }
}

