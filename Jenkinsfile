pipeline {
    agent any

    environment {
        HELM_REPOSITORIES = '[{"name": "truecharts", "url": "https://charts.truecharts.org/"}, {"name": "jetstack", "url": "https://charts.jetstack.io"}]'
        NGINX_CHART_VERSION = '4.8.4' // Replace with the actual version
        SLACK_CHANNEL = '#buildstatus-jenkins-pipeline'
        EKS_CLUSTER_NAME = 'demo'
        //HELM_CHART_PATH = 'path/to/your/helm/chart'
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
                //git 'https://github.com/obinnaaliogor/todoapp.git'
            }
        }

   stage('Add Helm Repositories - Test') {
    steps {
        script {
            echo "Adding Helm repo: truecharts"
            sh "helm repo add truecharts https://charts.truecharts.org/"
            sh "helm repo add cert-manager https://charts.jetstack.io"
            sh "helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx"
            sh "helm repo add jetstack https://charts.jetstack.io"
            sh "helm repo add k8s-at-home https://k8s-at-home.com/charts/"
            sh "helm repo add prometheus https://prometheus-community.github.io/helm-charts"
            sh "helm repo add grafana https://romanow.github.io/helm-charts/"
            sh 'helm repo list'
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
                        helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
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
                    echo "Please wait for 5 seconds for DNS propagation. Verify that the hosted zone is pointing to the Ingress Controller."

                    // Wait for 5 seconds (5 seconds)
                    sleep(time: 5, unit: 'SECONDS')

                    // Prompt the user to continue
                    input message: 'DNS update has been verified. Proceed to the next stage?', submitter: 'user'
                }
            }
        }

        stage('Implement Cert Manager') {
            steps {
                script {
                    sh "helm upgrade --install cert-manager jetstack/cert-manager --version ${env.CERT_MANAGER_HELM_CHART_VERSION} \
                        --namespace cert-manager --create-namespace \
                        -f cert-manager-values-v${env.CERT_MANAGER_HELM_CHART_VERSION}.yaml"

                    sh "sleep 5"
                    //sh "kubectl create ns grafana"
                    //sh "kubectl create ns monitoring"

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
                sleep 5
            """

            // Validate Todo app deployment
            sh """
                helm ls -n default
                kubectl get all -n default
                kubectl apply -f alpine.yaml
                sleep 5
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
                    sh "sleep 5"
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
                    sh "helm upgrade --install grafana grafana/grafana \
                        --namespace grafana --create-namespace -f grafana-values.yaml"

                    // Validate Grafana deployment
                    sh "kubectl get all -n grafana"
                }
            }
        }
stage('Validation and Testing') {
        steps {
            script {
                // Validate HPA
                sh 'kubectl get hpa'

                // Test application accessibility
                sh 'curl -Li http://todo.obinnaaliogor.xyz/'
                sh 'curl -v -k https://todo.obinnaaliogor.xyz/'
                sh 'sleep 5'

                // Validate ingress and cert-manager resources in the monitoring namespace
                sh 'kubectl get crd'
                sh 'kubectl get all -n monitoring'
                sh 'kubectl get certificates.cert-manager.io -n monitoring'
                sh 'kubectl get issuers.cert-manager.io -n monitoring'
                sh 'kubectl get ingress -n monitoring'
                sh 'kubectl describe ingress -n monitoring'

                // Validate ingress and cert-manager resources in the default namespace
                sh 'kubectl get issuers.cert-manager.io -n default'
                sh 'kubectl get certificates.cert-manager.io -n default'
                sh 'kubectl get ingress -n default'
                sh 'kubectl describe ingress -n default'
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

