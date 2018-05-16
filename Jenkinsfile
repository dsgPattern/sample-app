node {
    def project = 'k8s-workshop-203112'
    def appName = 'gceme'
    def feSvcName = "${appName}-frontend"
    // def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    def imageTag = "registry-docker-registry/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    
    // checkout scm

    stage('Preparation') {
        sh 'curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.7/bin/linux/amd64/kubectl'
        sh 'chmod +x ./kubectl && mv kubectl /usr/local/sbin'
        git url:"https://bitbucket.org/runtime32/sample-app.git"
    }

    stage('Build image') {
        sh("docker build -t ${imageTag} .")
    }

    stage('Run Go tests') {
        sh("docker run ${imageTag} go test")
    }

    stage('Push image to registry') {
        // sh("docker login -u oauth2accesstoken -p  gcr.io")
        sh("docker login -u docker -p docker registry-docker-registry")
        sh("docker push ${imageTag}")
    }

    stage("Deploy Application") {
        switch (env.BRANCH_NAME) {
            // Roll out to canary environment
            case "canary":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/canary/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=production apply -f k8s/canary/")
                sh("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
            break

            // Roll out to production
            case "master":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=production apply -f k8s/production/")
                sh("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'` > ${feSvcName}")
                break

            // Roll out a dev environment
            default:
                // Create namespace if it doesn't exist
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                // Don't use public load balancing for development branches
                sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
                sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
                sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
                echo 'To access your environment run `kubectl proxy`'
                echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
        }
    }
}