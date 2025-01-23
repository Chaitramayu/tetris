@Library('sharedlibrary')_

pipeline {
    parameters {
        string 'GIT_URL'
        string 'BRANCH_NAME'
        string 'repository'       
    }
    environment {
        gitRepoURL = "${params.GIT_URL}"
        gitBranchName = "${params.BRANCH_NAME}"
        repoName = "${params.repository}"
        dockerImage = "491085382300.dkr.ecr.us-east-1.amazonaws.com/${repoName}"
        gitCommit = "${GIT_COMMIT[0..6]}"
        dockerTag = "${params.BRANCH_NAME}-${gitCommit}"
    }
     

    agent {label 'docker'}
    stages {
        stage('Git Checkout') {
            steps {
                gitCheckout("$gitRepoURL", "refs/heads/$gitBranchName", 'githubCred')
            }
        }

        stage('SonarQube Analysis') {
                steps {
                    script {
                        withSonarQubeEnv('sonar-server') {
                            dir('src'){
                            sh '''
                            $SCANNER_HOME/bin/sonar-server \
                            -Dsonar.projectName="shared-library" \
                            -Dsonar.projectKey="shared-library"
                            '''
                        }
                    }
                }
            }
        }

        stage('SonarQube Quality Gate') {
                steps {
                    script {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-server'
                        }
                    }
                }
            
        stage('Docker Build') {
            steps {
                    dockerImageBuild('$dockerImage', '$dockerTag')
            }
        }

        stage('Trivy Scan'){
            steps{
                sh "trivy image -f json -o results-${BUILD_NUMBER}.json ${dockerImage}:${dockerTag}"
            }
        }

        stage('Docker Push') {
            steps {
                dockerECRImagePush('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'us-east-1')
            }
        }

        stage('Kubernetes Deploy - DEV') {
            when {
                branch 'development'
            }
            steps {
                kubernetesEKSHelmDeploy('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'us-east-1', 'eks', 'dev')
            }
        }

        stage('Kubernetes Deploy - UAT') {        
            when {
                branch 'master'
            }
            steps {
                kubernetesEKSHelmDeploy('$dockerImage', '$dockerTag', '$repoName', 'awsCred', 'us-east-1', 'eks', 'uat')
            }
        }
    }
}
