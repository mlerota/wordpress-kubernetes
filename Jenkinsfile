pipeline {
    agent any
    parameters {
        string(
            name: 'Branch',
            defaultValue: 'master',
            description: 'Branch used to build the code',
        )
        booleanParam(
            name: 'sendBuildToS3',
            defaultValue: true,
            description: 'Send build to S3?'
        )
        booleanParam(
            name: 'Production',
            description: 'Check for Production. Uncheck for STG env. This will only affect where build will be sent in S3',
            defaultValue: false
        )

    }

    environment {
        def artifact_name = "wp-kubernetes-${BUILD_NUMBER}.tar.gz"
        def s3FolderPrefix = ''
        }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Defining Variables'){
            steps {
                script {
                    if(params.Production == true) {
                      s3FolderPrefix = 'prod'
                    }else{
                      s3FolderPrefix = 'stg'
                    }
                }
            }
        }

        stage('Clone project') {
            steps {
                echo "Checking out branch ${params.Branch}"
                checkout([$class: 'GitSCM', branches: [[name: "*/${params.Branch}"]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'jenkins', url: 'https://github.com/mlerota/wordpress-kubernetes']]])
                echo "Last commit"
                sh "git --no-pager log -n 1"
            }
        }

        stage('Build PHP - Git') {
            steps {
                sh "git submodule update --recursive --init"
            }
        }

        stage('Build PHP - Composer') {
                    steps {
                        sh "composer install"
                    }
                }

        stage('Tidy Up') {
            steps {
                echo "Installing composer without dev dependencies"
                sh '''
                composer install --no-dev
                '''

                echo "Removing unnecessary files for build, like .git, jenkins, default themes"
                sh '''
                rm -rf .git
                rm -f .gitignore || true
                rm -f .gitmodules  || true
                rm -f Jenkinsfile  || true
                rm -rf .env.example
                rm -rf composer.*
                '''
            }
        }

        stage('Create artifact') {
            steps {
                sh '''
                mkdir artifact
                shopt -s extglob
                mv !(artifact) artifact
                tar -zcf ${artifact_name} artifact/*
                '''
            }
        }

        stage('Send build to S3') {
            when {
                  expression { params.sendBuildToS3 == true }
            }
            steps {
                  sh """
                  date +%F-time-%Hh_%Mm_%Ss > ./build-date-time
                  aws s3 cp ./${artifact_name} s3://cfsystems/jenkins-builds/wp-kubernetes/${s3FolderPrefix}/latest.tar.gz --quiet --profile default
                  aws s3 cp ./build-date-time s3://cfsystems/jenkins-builds/wp-kubernetes/${s3FolderPrefix}/build-date-time --profile default
                  """
            }
        }
    }
    post {
            always {
            archiveArtifacts "*.tar.gz"
            }
    }
}
