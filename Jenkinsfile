@Library('Shared') _
pipeline {
    agent { label 'Node' }
    
    environment {
        SONAR_HOME = tool "Sonar"
    }
    
    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }
    
    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                script{
                    code_checkout("https://github.com/shehabkazi-blip/mega-old","main")
                }
            }
        }
        
        stage("Trivy: Filesystem scan"){
            steps{
                script{
                    trivy_scan()
                }
            }
        }
        
        stage("SonarQube: Code Analysis"){
            steps{
                script{
                    sonarqube_analysis("Sonar","mega","mega")
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates") {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        // .scannerwork/report-task.txt থেকে সোনাকিউব টাস্ক ইউআরএল রিড করা
                        def reportTask = readFile('.scannerwork/report-task.txt')
                        def ceTaskUrl = ''
                        reportTask.splitEachLine('=') { fields ->
                            if (fields[0] == 'ceTaskUrl') {
                                ceTaskUrl = fields[1..-1].join('=')
                            }
                        }
                        
                        echo "Polling SonarQube task URL: ${ceTaskUrl}"
                        
                        def taskStatus = 'PENDING'
                        while (taskStatus == 'PENDING' || taskStatus == 'IN_PROGRESS') {
                            sleep(10) // প্রতি ১০ সেকেন্ড পর পর চেক করবে
                            def response = sh(script: "curl -s ${ceTaskUrl}", returnStdout: true).trim()
                            
                            if (response.contains('"status":"SUCCESS"')) {
                                taskStatus = 'SUCCESS'
                            } else if (response.contains('"status":"FAILED"')) {
                                error("SonarQube analysis task failed on server.")
                            } else if (response.contains('"status":"CANCELED"')) {
                                error("SonarQube analysis task was canceled.")
                            } else {
                                echo "SonarQube task status is still ${taskStatus}... waiting."
                            }
                        }
                        
                        // টাস্ক সাকসেস হলে এবার কোয়ালিটি গেট স্ট্যাটাস চেক করা
                        def qgResponse = sh(script: "curl -s 'http://54.190.13.96:9000/api/qualitygates/project_status?projectKey=mega'", returnStdout: true).trim()
                        echo "Quality Gate Response: ${qgResponse}"
                        
                        if (!qgResponse.contains('"status":"OK"')) {
                            error("Pipeline aborted due to Quality Gate failure (Status is not OK).")
                        } else {
                            echo "SonarQube Quality Gate passed successfully!"
                        }
                    }
                }
            }
        }
        
        stage('Exporting environment variables') {
            parallel{
                stage("Backend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatebackendnew.sh"
                            }
                        }
                    }
                }
                
                stage("Frontend env setup"){
                    steps {
                        script{
                            dir("Automations"){
                                sh "bash updatefrontendnew.sh"
                            }
                        }
                    }
                }
            }
        }
        
        stage("Docker: Build Images"){
            steps{
                script{
                    dir('backend'){
                        docker_build("mega-backend-beta","${params.BACKEND_DOCKER_TAG}","bongodev")
                    }
                    
                    dir('frontend'){
                        docker_build("mega-frontend-beta","${params.FRONTEND_DOCKER_TAG}","bongodev")
                    }
                }
            }
        }
        
        stage("Docker: Push to DockerHub"){
            steps{
                script{
                    docker_push("mega-backend-beta","${params.BACKEND_DOCKER_TAG}","bongodev") 
                    docker_push("mega-frontend-beta","${params.FRONTEND_DOCKER_TAG}","bongodev")
                }
            }
        }
    }
    post{
        success{
            archiveArtifacts artifacts: '*.xml', allowEmptyArchive: false, followSymlinks: false
            build job: "mega-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ],
            propagate: false
        }
    }
}