// Builds the stem-wiki (Outline) image and deploys it to the GKE
// Gateway-fronted general-projects-cluster via Kustomize.
//
// Job: `stem-wiki` (Pipeline script from SCM -> Script Path: Jenkinsfile).
//
// Outline builds in two stages: Dockerfile.base compiles the app (deps +
// `yarn build`) and Dockerfile assembles a slim runner from it. We build the
// base from source (so this fork's changes are included) and feed it to the
// runner via the BASE_IMAGE build-arg.
@Library('jenkins-shared-library-turing')_

def GIT_SHA
def TAG

pipeline {
    agent {
        label 'jenkins-new-agent'
    }

    options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(daysToKeepStr: '14', numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
    }

    triggers {
        githubPush()
        // Generic Webhook Trigger: fires on a GitHub push webhook, but only
        // when the pushed ref is the main branch. Requires the
        // "Generic Webhook Trigger" plugin. Configure a GitHub webhook to POST
        // to: <JENKINS_URL>/generic-webhook-trigger/invoke?token=stem-wiki-main-push
        GenericTrigger(
            genericVariables: [
                [key: 'ref', value: '$.ref']
            ],
            causeString: 'Triggered by push to $ref',
            token: 'stem-wiki-main-push',
            printContributedVariables: false,
            printPostContent: false,
            regexpFilterText: '$ref',
            regexpFilterExpression: '^refs/heads/main$'
        )
    }

    environment {
        GOOGLE_PROJECT_ID = "turing-general-projects"
        IMAGE_REPO        = "us-central1-docker.pkg.dev/turing-general-projects/stem-wiki/app"
        REGISTRY_HOST     = "us-central1-docker.pkg.dev"
        CLUSTER_NAME      = "general-projects-cluster"
        CLUSTER_ZONE      = "us-central1-a"
        NAMESPACE         = "stem-wiki"
        // Required for kubectl >=1.26 to talk to GKE via the auth plugin.
        USE_GKE_GCLOUD_AUTH_PLUGIN = "True"
    }

    parameters {
        string(defaultValue: 'main', description: 'Branch to build', name: 'BRANCH')
    }

    stages {
        stage('Check out') {
            steps {
                script {
                    def scmVars = checkout([$class: 'GitSCM',
                        branches: [[name: params.BRANCH]],
                        userRemoteConfigs: [[
                            credentialsId: 'github-token',
                            url: 'https://github.com/TuringEnterprises/wiki-dancarmody-morse.git'
                        ]]
                    ])
                    GIT_SHA = scmVars.GIT_COMMIT
                    TAG = "${BUILD_NUMBER}-${GIT_SHA.take(7)}"
                    echo "Building tag ${TAG}"
                }
            }
        }

        stage('Authenticate with Artifact Registry') {
            steps {
                withCredentials([file(credentialsId: 'google_service_account_key', variable: 'GC_KEY')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file="$GC_KEY"
                        gcloud config set project "$GOOGLE_PROJECT_ID"
                        gcloud auth configure-docker "$REGISTRY_HOST" --quiet
                        cat "$GC_KEY" | docker login -u _json_key --password-stdin "https://$REGISTRY_HOST"
                    '''
                }
            }
        }

        stage('Build & push image') {
            steps {
                sh """
                    # Stage 1: compile Outline from source into the base image.
                    docker build -f Dockerfile.base -t stem-wiki-base:${TAG} .
                    # Stage 2: assemble the slim runner from the base we just built.
                    docker build -f Dockerfile \\
                        --build-arg BASE_IMAGE=stem-wiki-base:${TAG} \\
                        -t ${IMAGE_REPO}:${TAG} -t ${IMAGE_REPO}:latest .
                    docker push ${IMAGE_REPO}:${TAG}
                    docker push ${IMAGE_REPO}:latest
                """
            }
        }

        stage('Deploy') {
            steps {
                script {
                    // Ensure kustomize / kubectl / the GKE auth plugin are present
                    // on the agent. The jenkins-new-agent's gcloud is apt-managed,
                    // so `gcloud components install` is rejected -- install kubectl
                    // from dl.k8s.io and the auth plugin from the cloud-sdk apt repo.
                    sh '''
                        set -e

                        wait_for_apt() {
                            local i=0
                            while fuser /var/lib/apt/lists/lock >/dev/null 2>&1 \
                               || fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do
                                i=$((i + 1))
                                if [ "$i" -gt 60 ]; then
                                    echo "Timed out waiting for apt lock"
                                    return 1
                                fi
                                echo "Waiting for apt lock... ($i)"
                                sleep 2
                            done
                        }

                        apt_with_retry() {
                            local max=30 attempt=1
                            while [ "$attempt" -le "$max" ]; do
                                wait_for_apt
                                if "$@"; then
                                    return 0
                                fi
                                echo "apt-get failed (attempt $attempt/$max), retrying..."
                                sleep 2
                                attempt=$((attempt + 1))
                            done
                            return 1
                        }

                        if ! command -v kustomize >/dev/null 2>&1; then
                            curl -sLO "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.4.3/kustomize_v5.4.3_linux_amd64.tar.gz"
                            tar xzf kustomize_v5.4.3_linux_amd64.tar.gz
                            mv kustomize /usr/local/bin/
                            rm -f kustomize_v5.4.3_linux_amd64.tar.gz
                        fi
                        kustomize version

                        if ! command -v kubectl >/dev/null 2>&1; then
                            KUBECTL_VERSION="$(curl -L -s https://dl.k8s.io/release/stable.txt)"
                            curl -sLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
                            install -m 0755 kubectl /usr/local/bin/kubectl
                            rm -f kubectl
                        fi
                        kubectl version --client=true

                        if ! command -v gke-gcloud-auth-plugin >/dev/null 2>&1; then
                            if command -v apt-get >/dev/null 2>&1; then
                                if [ ! -f /etc/apt/sources.list.d/google-cloud-sdk.list ]; then
                                    apt_with_retry apt-get update -qq
                                    apt_with_retry apt-get install -y --no-install-recommends \
                                        ca-certificates curl gnupg apt-transport-https
                                    curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
                                        | gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg
                                    echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
                                        > /etc/apt/sources.list.d/google-cloud-sdk.list
                                fi
                                apt_with_retry apt-get update -qq
                                apt_with_retry apt-get install -y --no-install-recommends google-cloud-cli-gke-gcloud-auth-plugin
                            else
                                gcloud components install gke-gcloud-auth-plugin --quiet
                            fi
                        fi
                        gke-gcloud-auth-plugin --version
                    '''

                    // The ephemeral agents can't reach the IP control-plane
                    // endpoint (their egress isn't in the cluster's authorized
                    // networks, so it times out). Retarget kubeconfig at the
                    // cluster's DNS-based endpoint instead: it's fronted by
                    // Google's global endpoint (publicly-trusted cert, reachable
                    // from within Google Cloud) and gated by this SA's IAM/RBAC.
                    // `get-credentials --dns-endpoint` is refused while the
                    // cluster has allowExternalTraffic disabled, so rewrite the
                    // kubeconfig cluster entry by hand (system CAs validate the
                    // gke.goog cert, so drop the API-server CA data).
                    sh """
                        gcloud container clusters get-credentials ${CLUSTER_NAME} \
                            --zone ${CLUSTER_ZONE} \
                            --project ${GOOGLE_PROJECT_ID}

                        DNS_EP=\$(gcloud container clusters describe ${CLUSTER_NAME} \
                            --zone ${CLUSTER_ZONE} --project ${GOOGLE_PROJECT_ID} \
                            --format='value(controlPlaneEndpointsConfig.dnsEndpointConfig.endpoint)')
                        CTX_CLUSTER="gke_${GOOGLE_PROJECT_ID}_${CLUSTER_ZONE}_${CLUSTER_NAME}"
                        echo "Retargeting kubeconfig to DNS endpoint: \$DNS_EP"
                        kubectl config set-cluster "\$CTX_CLUSTER" --server="https://\$DNS_EP"
                        kubectl config unset "clusters.\$CTX_CLUSTER.certificate-authority-data" || true

                        kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    """

                    sh """
                        cd k8s/overlays/production
                        kustomize edit set image stem-wiki=${IMAGE_REPO}:${TAG}
                        kustomize build . | kubectl apply -f -
                        kubectl rollout status deployment/stem-wiki -n ${NAMESPACE} --timeout=300s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployed ${TAG} to https://stem-wiki.turing.com"
        }
        failure {
            echo "Deploy failed for ${TAG}"
        }
    }
}
