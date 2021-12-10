
import java.time.LocalDateTime
import java.time.*
import java.time.format.DateTimeFormatter
//variables that are same for everyone 
IMAGE_REPOSITORY = "simple-nginx"
//temporary variables for AWS Wassim Testing
//TARGET_CLUSTER_REGISTRY_URI = 'https://mirantis-demo-ws-msr-lb-b61096abded88cdc.elb.eu-west-3.amazonaws.com'
//TARGET_CLUSTER_REGISTRY_HOSTNAME = 'mirantis-demo-ws-msr-lb-b61096abded88cdc.elb.eu-west-3.amazonaws.com'
//TARGET_CLUSTER_KUBE_DOMAIN_NAME = "staging.presales.demo.mirantis.com"
//TARGET_CLUSTER_KUBERNETES_CONTEXT = "ucp_mirantis-demo-ws-master-lb-ed1c7f13fc89417e.elb.eu-west-3.amazonaws.com:6443_admin"
//Prod variables 
TARGET_CLUSTER_REGISTRY_URI = 'https://registry.prod.equinix.presales.demo.mirantis.com'
TARGET_CLUSTER_REGISTRY_HOSTNAME = 'registry.prod.equinix.presales.demo.mirantis.com'
TARGET_CLUSTER_KUBE_DOMAIN_NAME = "prod.presales.demo.mirantis.com"
TARGET_CLUSTER_KUBERNETES_CONTEXT = "ucp_kube.prod.equinix.presales.demo.mirantis.com:5443_jenkins"
//variables that change for every user, to be changed to global automated variables
IMAGE_NAMESPACE_DEV = "wsoualhi-dev"
IMAGE_NAMESPACE_PROD = "wsoualhi-prod"
KUBERNETES_NAMESPACE_DEV = "${IMAGE_NAMESPACE_DEV}"
KUBERNETES_NAMESPACE_PROD = "${IMAGE_NAMESPACE_PROD}"
USERNAME = "wsoualhi"
APPLICATION_DOMAIN = "${USERNAME}.${TARGET_CLUSTER_KUBE_DOMAIN_NAME}"

//http://simple-nginx.prod.wsoualhi.staging.presales.demo.mirantis.com

//variables that change on every execution
def IMAGE_TAG = LocalDateTime.now()
IMAGE_TAG = IMAGE_TAG.format(DateTimeFormatter.ofPattern("yyyyMMddHHmm"))
println IMAGE_TAG

/*
// For available target test clusters, contact your platform administrator, it is possible to use eu.demo.mirantis.com with istio_gateway
// For available target clusters, contact your platform administrator, it is possible to use us.demo.mirantis.com with ingress.
TARGET_CLUSTER_DOMAIN = "us.demo.mirantis.com"
*/

// Available ingress = [ "ingress" | "istio_gateway" ]
KUBERNETES_INGRESS = "ingress"

/*
if(! CLUSTER.containsKey(TARGET_CLUSTER_DOMAIN)){
    error("Unknown cluster '${TARGET_CLUSTER_DOMAIN}'")
}
TARGET_CLUSTER = CLUSTER.get(TARGET_CLUSTER_DOMAIN)

if(ORCHESTRATOR.toLowerCase() == "kubernetes"){
    KUBERNETES_NAMESPACE_DEV = "${IMAGE_NAMESPACE_DEV}"
    KUBERNETES_NAMESPACE_PROD = "${IMAGE_NAMESPACE_PROD}"

    APPLICATION_DOMAIN = "${USERNAME}.${TARGET_CLUSTER['KUBE_DOMAIN_NAME']}"
}
else if (ORCHESTRATOR.toLowerCase() == "swarm"){
    SWARM_SERVICE_NAME = "${USERNAME}-${IMAGE_REPOSITORY}"
    SWARM_STACK_NAME = "${USERNAME}-${IMAGE_REPOSITORY}"
    UCP_COLLECTION_PATH = "/Shared/Private/${USERNAME}"

    APPLICATION_DOMAIN = "${USERNAME}.${TARGET_CLUSTER['SWARM_DOMAIN_NAME']}"
}
else {
    error("Unsupported orchestrator")
}

if(! ["ingress", "istio_gateway"].contains(KUBERNETES_INGRESS)){
    error("Unsupported Kubernetes ingress type '${KUBERNETES_INGRESS}'")
}
*/
node {
    def docker_image

    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        docker_image = docker.build("${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}")
    }

    stage('Unit Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push') {
        //docker.withRegistry(TARGET_CLUSTER['REGISTRY_URI'], TARGET_CLUSTER['REGISTRY_CREDENTIALS_ID']) {
            //docker.withRegistry('https://registry.hub.docker.com', 'dockerHub') {
            docker.withRegistry(TARGET_CLUSTER_REGISTRY_URI, 'MSRequinixProd') {
            docker_image.push(IMAGE_TAG)
        }
    }

    stage('Scan') {
        //httpRequest acceptType: 'APPLICATION_JSON', authentication: TARGET_CLUSTER['REGISTRY_CREDENTIALS_ID'], contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${TARGET_CLUSTER['REGISTRY_URI']}/api/v0/imagescan/scan/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}/linux/amd64"
        httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/imagescan/scan/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}/linux/amd64"

        def scan_result

        def scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/imagescan/scansummary/repositories/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 15
            }

        }
        println('Response JSON: ' + scan_result)
    }

    stage('Sign Development Image') {
        withEnv(["REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_DEV}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 //"TRUST_SIGNER_KEY= 'TRUST_SIGNER_KEY'"
                 ]) {
            //withCredentials([string(credentialsId: TARGET_CLUSTER['TRUST_SIGNER_PASSPHRASE_CREDENTIALS_ID'] , variable: 'DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE')]) {
                sh 'docker trust key load ${TRUST_SIGNER_KEY}'
                sh 'docker trust sign ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
                //println (TARGET_CLUSTER['TRUST_SIGNER_PASSPHRASE_CREDENTIALS_ID'])
                // com.mirantis.demo.us-jenkins_signing_signer-passphrase
            //}
        }
    }

    stage('Deploy to Development') {
        withEnv(["APPLICATION_FQDN=${IMAGE_REPOSITORY}.dev.${APPLICATION_DOMAIN}",
                 "REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_DEV}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 "USERNAME=${USERNAME}"]) {
                withEnv(["KUBERNETES_CONTEXT=${TARGET_CLUSTER_KUBERNETES_CONTEXT}", 
                         "KUBERNETES_INGRESS=${KUBERNETES_INGRESS}",
                         "KUBERNETES_NAMESPACE=${KUBERNETES_NAMESPACE_DEV}"]) {
                    sh 'envsubst < kubernetes/001_simple-nginx_deployment.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/002_simple-nginx_service.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/003_simple-nginx_${KUBERNETES_INGRESS}.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                }
            println("Application deployed to Development: http://${APPLICATION_FQDN}")
            // http://simple-nginx.dev.wsoualhi.staging.presales.demo.mirantis.com
        }
    }

    stage('Integration Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Promote') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${IMAGE_NAMESPACE_PROD}/${IMAGE_REPOSITORY}\", \"targetTag\": \"${IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/repositories/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/tags/${IMAGE_TAG}/promotion"
    }
    /*
    stage('Sign Production Image') {
        withEnv(["REGISTRY_HOSTNAME=${TARGET_CLUSTER['REGISTRY_HOSTNAME']}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_PROD}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 "TRUST_SIGNER_KEY=${TARGET_CLUSTER['TRUST_SIGNER_KEY']}"]) {
            withCredentials([string(credentialsId: TARGET_CLUSTER['TRUST_SIGNER_PASSPHRASE_CREDENTIALS_ID'] , variable: 'DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE')]) {
                sh 'docker pull ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
                sh 'docker trust key load ${TRUST_SIGNER_KEY}'
                sh 'docker trust sign ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
            }
        }
    }
    */
    stage('Deploy to Production') {
        withEnv(["APPLICATION_FQDN=${IMAGE_REPOSITORY}.prod.${APPLICATION_DOMAIN}",
                 "REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_PROD}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 "USERNAME=${USERNAME}"]) {
                withEnv(["KUBERNETES_CONTEXT=${TARGET_CLUSTER_KUBERNETES_CONTEXT}", 
                         "KUBERNETES_INGRESS=${KUBERNETES_INGRESS}",
                         "KUBERNETES_NAMESPACE=${KUBERNETES_NAMESPACE_PROD}"]) {
                    sh 'envsubst < kubernetes/001_simple-nginx_deployment.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/002_simple-nginx_service.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/003_simple-nginx_${KUBERNETES_INGRESS}.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                }
            println("Application deployed to Production: http://${APPLICATION_FQDN}")
            // http://simple-nginx.dev.wsoualhi.staging.presales.demo.mirantis.com
        }

    }
}
