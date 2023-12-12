pipeline {
    agent any

    environment {
        HELM_REPOSITORIES = '[{"name": "truecharts", "url": "https://charts.truecharts.org/"}, {"name": "jetstack", "url": "https://charts.jetstack.io"}]'
        NGINX_CHART_VERSION = '4.8.4' // Replace with the actual version
        SLACK_CHANNEL = '#buildstatus-jenkins-pipeline'
        EKS_CLUSTER_NAME = 'demo'
        HELM_CHART_PATH = 'path/to/your/helm/chart'
        RELEASE_NAME = 'my-vikunja'
        CERT_MANAGER_HELM_CHART_VERSION = '1.13.2'
        AWS_DEFAULT_REGION = 'us-east-1'
        TERRAFORM_ACTION = 'destroy'
        REGION = 'us-east-1'
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Checking out code from GitHub"
               // git 'https://github.com/obinnaaliogor/todoapp.git'
            }
        }

    stage('Add Helm Repositories') {
        steps {
            script {
                def helmRepos = readJSON(text: env.HELM_REPOSITORIES)
                helmRepos.each { repo ->
                    sh "helm repo add ${repo.name} ${repo.url}"
                }
            }
        }
    }

        stage('Validate Helm Repositories') {
            steps {
                script {
                    sh 'helm repo ls'
                }
            }
        }

        stage('Provision Infrastructure') {
            steps {
                script {
                    dir('./iac') {
                        sh 'terraform init'
                        sh "terraform ${env.TERRAFORM_ACTION} --auto-approve"

                        // Only proceed if terraform action is apply
                        if (env.TERRAFORM_ACTION == 'apply') {
                            echo "Terraform apply successful. Cleaning up..."
                            // Delete Terraform directories
                            sh 'rm -rf .terraform terraform.tfstate* terraform.log'
                        }
                    }
                }
            }
        }

        stage('Update kube-config') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --name ${env.EKS_CLUSTER_NAME} --region ${env.REGION}"
                }
            }
        }

        stage('Implement ebs-csi-driver for create pv creation') {
            steps {
                script {
                    // Your terraform code should also have a policy called AmazonEBSCSIDriverPolicy
                    // added to the nodegroup that will allow EC2 to create PV
                    sh 'kubectl apply -k github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master'
                }
            }
        }

        stage('Implement Ingress Controller') {
            steps {
                script {
                    echo "Installing and upgrading NGINX Ingress Controller"
                    sh """
                        helm install ingress-nginx ingress-nginx/ingress-nginx \
                        --version ${env.NGINX_CHART_VERSION} \
                        -f "nginx-values-v${env.NGINX_CHART_VERSION}.yaml" \
                        --namespace ingress-nginx \
                        --create-namespace
                    """

                    // Validate NGINX Ingress Controller deployment
                    sh "kubectl get all -n ingress-nginx"
                }
            }
        }

        // Add a manual intervention step
        stage('Manual Intervention - Wait for DNS Update') {
            steps {
                script {
                    // Display a message to instruct the user to wait for DNS propagation
                    echo "Please wait for 15 minutes for DNS propagation. Verify that the hosted zone is pointing to the Ingress Controller."

                    // Wait for 15 minutes (900 seconds)
                    sleep(time: 900, unit: 'SECONDS')

                    // Prompt the user to continue
                    input message: 'DNS update has been verified. Proceed to the next stage?', submitter: 'user'
                }
            }
        }

        stage('Implement Cert Manager') {
            steps {
                script {
                    sh "helm install cert-manager jetstack/cert-manager --version ${env.CERT_MANAGER_HELM_CHART_VERSION} \
                        --namespace cert-manager --create-namespace \
                        -f cert-manager-values-v${env.CERT_MANAGER_HELM_CHART_VERSION}.yaml"

                    sh sleep 5

                    // Cert manager resource kind: issuer will be here, and since it's namespaced, we will have an issuer for vikunja app, alert manager, prometheus, grafana
                    sh "kubectl apply -f k8s-manifests/"
                }
            }
        }

        stage('Deploy Todo App') {
    environment {
        RELEASE_NAME = 'my-vikunja' // Add your release name here
    }
    steps {
        script {
            // Add commands for deploying Todo app
            sh """
                helm upgrade --install ${env.RELEASE_NAME} k8s-at-home/vikunja \
                --namespace default -f vikunja-values.yaml
                kubectl replace --force -f backend.yaml
            """

            // Validate Todo app deployment
            sh """
                helm ls -n default
                kubectl get all -n default
                kubectl run alpine --image=alpine/curl -- sleep 20000
                kubectl exec alpine -- curl ${env.RELEASE_NAME}:8080
                sleep 5
                kubectl delete pod alpine
            """

            // Create Ingress Resources
            sh "kubectl apply -f todoapp-ingressresource/"

            // Validate Ingress Resources
            sh """
                kubectl get ingress -A
                kubectl describe ingress -n grafana
                kubectl describe ingress -n monitoring
                kubectl describe ingress -n default
                sleep 5
                curl -Li http://todo.obinnaaliogor.xyz/
                curl -v -k https://todo.obinnaaliogor.xyz/
            """
        }
    }
}


        stage('Ensure High Availability of Todoapp') {
            steps {
                script {
                    // Deploy metrics server
                    echo "Deploying Metrics server"
                    sh "kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml"
                    sh "kubectl get pod -n kube-system "
                    sh "kubectl top nodes"
                    sh "kubectl top pods"
                    sh "kubectl top pods --containers"

                    // Create and validate HPA
                    sh "kubectl apply -f hpa.yaml"
                    sh "kubectl get hpa"
                }
            }
        }

        stage('Implement Cluster Autoscaler') {
            steps {
                script {
                    // Add commands for Cluster Autoscaler
                    sh "kubectl apply -f autoscaler.yaml"
                    sh "kubectl get pod -n kube-system"
                    sh "kubectl get nodes"
                    sh "kubectl scale deployment my-vikunja --replicas=20"
                    sh "kubectl get pods"
                    sleep 60
                }
            }
        }

        stage('Scaling Down Nodes') {
            steps {
                script {
                    // Validate scaling down nodes
                    sh "kubectl get nodes"
                    sh "kubectl scale deployment my-vikunja --replicas=1"
                    sleep 60
                    sh "kubectl get nodes"
                    sh "kubectl get pods"
                }
            }
        }

        stage('Implement Prometheus and Grafana') {
            steps {
                script {
                    // Install and upgrade Prometheus with alertmanager
                    sh "helm upgrade --install prometheus prometheus/prometheus \
                        --namespace monitoring --create-namespace -f alertmanager.yaml"

                    // Validate Prometheus and alertmanager deployment
                    sh "helm ls -n monitoring"
                    sh "kubectl get all -n monitoring"

                    // Install Grafana
                    sh "helm install grafana grafana/grafana \
                        --namespace grafana --create-namespace -f grafana-values.yaml"

                    // Validate Grafana deployment
                    sh "kubectl get all -n grafana"
                }
            }
        }
    }

    post {
        success {
            // Notify success on Slack
            script {
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'good',
                    message: "Pipeline succeeded for ${env.JOB_NAME} ${env.BUILD_NUMBER}: ${env.BUILD_URL}"
                )
            }
        }
        failure {
            // Notify failure on Slack
            script {
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: 'danger',
                    message: "Pipeline failed for ${env.JOB_NAME} ${env.BUILD_NUMBER}: ${env.BUILD_URL}"
                )
            }
        }
    }
}