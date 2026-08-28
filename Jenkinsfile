// Funzione helper per build e push
def buildAndPushTag(Map args) {
    def defaults = [
        registryUrl: 'http://192.168.99.20:5000', 
        dockerfileDir: './Dockerfiles',
        dockerfileName: 'Dockerfile.ubuntu',
        buildArgs: '',
        pushLatest: false
    ]
    def config = defaults + args

    
    docker.withRegistry(config.registryUrl) {
      
        def dockerfileFile = "${config.dockerfileDir}/${config.dockerfileName}"
        def image = docker.build("${config.image}:${config.buildTag}", "${config.buildArgs} -f ${dockerfileFile} ${config.dockerfileDir}") 
        image.push()

        if (config.pushLatest) {
            image.push('latest')
            sh "docker rmi --force ${config.image}:latest"
        }
        
        sh "docker rmi --force ${config.image}:${config.buildTag}"
        return "${config.image}:${config.buildTag}"
    }
}

pipeline {
    agent { label 'Mac-agent' }
    
    environment {
        IMAGE_NAME = 'docker-in-docker'
        REGISTRY_URL = 'http://192.168.99.20:5000'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Tag Logic') {
            steps {
                script {
                    if (env.TAG_NAME) {
                        env.DOCKER_TAG = env.TAG_NAME
                        env.PUSH_LATEST = 'false'
                    } else {
                        env.DOCKER_TAG = "build-${env.BUILD_NUMBER}"
                        env.PUSH_LATEST = 'false'
                    }
                }
            }
        }
        
        stage('Build and Push docker image') {
            steps {
                script {
                    def pushedImage = buildAndPushTag(
                        registryUrl: env.REGISTRY_URL,
                        image: env.IMAGE_NAME,
                        buildTag: env.DOCKER_TAG,
                        pushLatest: Boolean.parseBoolean(env.PUSH_LATEST)
                    )
                    echo "Successo! Immagine pushata: ${pushedImage}" 
                }
            }
        }       
    }
}