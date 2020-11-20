pipeline {

   parameters {
    choice(name: 'action', choices: 'create\ndestroy', description: 'Create/update or destroy the eks cluster.')
    string(name: 'cluster', defaultValue : 'demo', description: "EKS cluster name;eg demo creates cluster named eks-demo.")
    choice(name: 'k8s_version', choices: '1.17\n1.18\n1.16\n1.15', description: 'K8s version to install.')
    string(name: 'instance_type', defaultValue : 'm5.large', description: "k8s worker node instance type.")
    string(name: 'num_workers', defaultValue : '3', description: "k8s number of worker instances.")
    string(name: 'max_workers', defaultValue : '10', description: "k8s maximum number of worker instances that can be scaled.")
    string(name: 'credential', defaultValue : 'jenkins', description: "Jenkins credential that provides the AWS access key and secret.")
    booleanParam(name: 'cloudwatch', defaultValue : true, description: "Setup Cloudwatch logging, metrics and Container Insights?")
    booleanParam(name: 'ca', defaultValue : false, description: "Setup k8s Cluster Autoscaler?")
    string(name: 'region', defaultValue : 'eu-west-1', description: "AWS region.")
    string(name: 'key_pair', defaultValue : 'spicysomtam-aws4', description: "EC2 instance ssh keypair.")
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 1, unit: 'HOURS')
    withAWS(credentials: params.credential, region: params.region)
    ansiColor('xterm')
  }

  agent { label 'master' }

  stages {

    stage('Setup') {
      steps {
        script {
          currentBuild.displayName = "#" + env.BUILD_NUMBER + " " + params.action + " eks-" + params.cluster

          println "Getting the eksctl binary..."
          sh """
            curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xzvf -
          """
        }
      }
    }

    stage('Create') {
      when {
        expression { params.action == 'create' }
      }
      steps {
        script {
          input "Create EKS cluster eks-${params.cluster} in aws?" 

          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
          credentialsId: params.credential, 
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            
            sh """
              ./eksctl create cluster \
                --name eks-${params.name} \
                --version ${params.k8s_version} \
                --region  ${params.region} \
                --nodegroup-name eks-${params.name}-0 \
                --nodes ${params.num_workers} \
                --nodes-min ${params.num_workers} \
                --nodes-max ${params.max_workers} \
                --with-oidc \
                --ssh-access \
                --ssh-public-key ${params.key_pair} \
                --managed
            """
          }
        }
      }
    }

    /* 
    stage('Cluster setup') {
      when {
        expression { params.action == 'create' }
      }
      steps {
        script {
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
          credentialsId: params.credential, 
          accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
          secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            
            sh """
              aws eks update-kubeconfig --name eks-${params.cluster} --region ${params.region}

              # Add configmap aws-auth if its not there:
              if [ ! "\$(kubectl -n kube-system get cm aws-auth 2> /dev/null)" ]
              then
                echo "Adding aws-auth configmap to ns kube-system..."
                terraform output config_map_aws_auth | awk '!/^\$/' | kubectl apply -f -
              else
                true # jenkins likes happy endings!
              fi
            """

            if (params.cloudwatch == true) {
              echo "Setting up Cloudwatch logging and metrics."
              sh """
                curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | \\
                  sed "s/{{cluster_name}}/eks-${params.cluster}/;s/{{region_name}}/${params.region}/" | \\
                  kubectl apply -f -
              """
            }

            if (params.ca == true) {
              echo "Setting up k8s Cluster Autoscaler."

              // Keep the google region logic simple; us or eu
              gregion='us'

              if (params.region =~ '^eu') {
                gregion='eu'
              }

              // CA image tag, which is k8s major version plus CA minor version.
              // See for latest versions: https://github.com/kubernetes/autoscaler/releases
              switch (params.k8s_version) {
                case '1.18':
                  tag='3'
                	break;
                case '1.17':
                  tag='4'
                	break;
                case '1.16':
                  tag='7'
              	  break;
                case '1.15':
                  tag='7'
              	  break;
              }

              // Setup documented here: https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html
              sh """
                kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
                kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false"
                kubectl -n kube-system get deployment.apps/cluster-autoscaler -o json | \\
                  jq | \\
                  sed 's/<YOUR CLUSTER NAME>/eks-${params.cluster}/g' | \\
                  jq '.spec.template.spec.containers[0].command += ["--balance-similar-node-groups","--skip-nodes-with-system-pods=false"]' | \\
                  kubectl apply -f -
                kubectl -n kube-system set image deployment.apps/cluster-autoscaler cluster-autoscaler=${gregion}.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler:v${params.k8s_version}.${tag}
              """
            }

          }
        }
      }
    }
    */

    stage('Destroy') {
      when {
        expression { params.action == 'destroy' }
      }
      steps {
        script {
          input "Destroy EKS cluster eks-${params.cluster} in aws?" 

          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
            credentialsId: params.credential, 
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',  
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

            sh """
              ./eksctl delete cluster \
                --name eks-${params.name} \
                --region ${params.region} \
                --wait
            """
          }
        }
      }
    }

  }

}
