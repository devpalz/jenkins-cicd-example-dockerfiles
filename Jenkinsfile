/**
* If the 'goss_run_command.yaml' file exists, return the 'command' from there.
* Otherwise return the default dgoss run command.
*/
String getRunCommand() {
    def exists = fileExists 'goss_run_command.yaml'

    if(exists){
        def runCommand = readYaml file: 'goss_run_command.yaml'
        return runCommand.command
    }else{
        return 'dgoss run'
    }
}

pipeline {
    agent {
        label 'dind'
    }
    parameters{
        // Statically define the folders you can build as Jenkins offers no way to dynamically generate this :(
        choice(name: 'BUILD_FOLDER', choices: ['mysql-init', 'mysql-init-continued', 'mysql-init-no-goss'],
                description: 'Please select a subfolder which contains a Dockerfile to build!')
    }
    options{
        ansiColor('xterm')
    }
    stages {
        stage('Validate Parameters'){
            steps{
                script{
                    if(params.BUILD_FOLDER == ''){
                        error("Please set BUILD_FOLDER parameter!")
                    }
                }
            }
        }
        stage('Build Docker Image'){
            steps{
                script{
                    dir(params.BUILD_FOLDER) {
                        docker.withServer('tcp://172.17.0.1:2376', 'DOCKER_HOST_CERTS') {

                            // Image Name we will build and test, unique to this build.
                            env.BUILD_IMG_NAME = "${params.BUILD_FOLDER}:${env.BUILD_ID}"

                            // So the UI shows what Dockerfile is been built.
                            currentBuild.displayName = "#${env.BUILD_ID} - ${env.BUILD_FOLDER}"

                            // Jenkins step will build the 'Dockerfile' inside this folder
                            def image = docker.build(env.BUILD_IMG_NAME)
                        }
                    }
                }
            }
        }
        stage('Test Docker Image'){
            when{
                expression{
                    fileExists "${params.BUILD_FOLDER}/goss.yaml"
                }
            }
            steps{
                script{
                    dir(params.BUILD_FOLDER) {
                        docker.withServer('tcp://172.17.0.1:2376', 'DOCKER_HOST_CERTS') {
                            def runCommand = getRunCommand()
                            sh label: 'Run Dgoss tests', script: """ $runCommand $env.BUILD_IMG_NAME """
                        }
                    }
                }
            }
        }
        stage('Push Docker Image'){
            steps{
                script{

                    // If the branch is develop/master then use the tag latest otherwise tag with the branch name
                    TAG = (env.BRANCH_NAME =~ "master|develop") ? "latest" : env.BRANCH_NAME

                    // Push the image to our registry and re-tag it
                    docker.withServer('tcp://172.17.0.1:2376', 'DOCKER_HOST_CERTS') {
                        docker.withRegistry('http://localhost:5000'){
                            docker.image(env.BUILD_IMG_NAME).push("$TAG")
                        }
                    }
                }
            }
        }
    }
    post{
        always{
            script{
                // We always want to make sure we delete the image tag used to test the Docker container.
                docker.withServer('tcp://172.17.0.1:2376', 'DOCKER_HOST_CERTS') {
                    sh label: 'remove build image', script: """docker image rm ${env.BUILD_IMG_NAME}"""
                }
            }
        }
    }
}