version="1.0.0"
repository="docker-registry.mtk8s.io/backstage/backstage"
tag="latest"
image="${repository}:${version}.${env.BUILD_NUMBER}"
namespace="acnbackstage"
ENTITY_NAMESPACE = "default"
ENTITY_KIND = "Component"
ENTITY_NAME = "submission"
TECHDOCS_S3_TENANT_NAME = "http://minio.minio"
TECHDOCS_S3_BUCKET_NAME = "kbm-backstage-techdocs"

podTemplate(label: 'demo-customer-pod', cloud: 'kubernetes', serviceAccount: 'jenkins', imagePullSecrets: ['jenmtk8sdkrreg'], 
  containers: [
    containerTemplate(name: 'node16py311', image: 'nikolaik/python-nodejs:python3.11-nodejs16-bullseye', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'buildkit', image: 'docker-registry.mtk8s.io/moby/buildkit:latest', ttyEnabled: true, privileged: true, alwaysPullImage: true),
    containerTemplate(name: 'kubectl', image: 'roffe/kubectl', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'argocd', image: 'docker-registry.mtk8s.io/jenkins/inbound-agent:4.11.2-4-alpine-jdk11', ttyEnabled: true, alwaysPullImage: true, command: 'cat')
  ],
  volumes: [
    secretVolume(secretName: 'jenkins-docker-creds', mountPath: '/root/.docker'),
    configMapVolume(configMapName: 'kube-root-ca.crt', mountPath: '/var/tmp'),
    configMapVolume(configMapName: 'kube-intermediate-ca.crt', mountPath: '/var/tmp')
  ]) {

    node('demo-customer-pod') {
        stage('Prepare') {
            //
            // Since the GIT SCM is already performed as part of the Job Definition, Skipping Explicit Checkout
            // Needs to be tested
            //
            // git changelog: false, credentialsId: 'github-app-jenkins', poll: false, branch: 'main', url: 'https://github.com/kbmdev/backstage-acn-repo.git'
            echo "SCM Checkout by using GitHub App"
        }
        
        stage('Node App Build and Package') {
            container('node16py311') {
                sh """
                  cd backstage-app
                  echo "List all the files in the PodTemplate Container from /var/tmp"
                  ls -latrh -R /var/tmp/
                  echo "/usr/local/share: List all the files in the PodTemplate Container"
                  ls -latrh -R /usr/local/share
                  yarn install
                  #yarn tsc
                  yarn build:backend
                  ls -latrh
                """
                milestone(1)
            }
        }
        stage('Node App Build and Package') {
            withCredentials([usernamePassword(credentialsId: 'minio-username-credentials', passwordVariable: 'pass', usernameVariable: 'user')]) {
                container('node16py311') {
                    sh """
                      # Minio related env variables
                      export MINIO_ACCESS_KEY="${user}"
                      export MINIO_SECRET_KEY="${pass}"
                      export AWS_REGION="us-east-1"
                      
                      npm install -g @techdocs/cli
                      python -m pip install mkdocs-techdocs-core==1.*
                      techdocs-cli -V
                      techdocs-cli generate --no-docker --verbose
                      echo "successful generation of techdocs"
                      ls -latrh -R
                      techdocs-cli publish --publisher-type awsS3 --awsEndpoint $TECHDOCS_S3_TENANT_NAME --storage-name $TECHDOCS_S3_BUCKET_NAME --entity $ENTITY_NAMESPACE/$ENTITY_KIND/$ENTITY_NAME --awsS3ForcePathStyle true
                      echo "Build and Generate TechDocs Successful"
                    """
                    milestone(2)
                }
            }
        }
        stage('Build Docker Image') {
            container('buildkit') {
                //Automatically the Docker Image Pull Secrets are Injected as part of POD Template
                sh """
                  ls -latrh
                  cd backstage-app/packages/backend
                  cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt .
                  cp /usr/local/share/ca-certificates/kx-root-ca.crt .
                  
                  cp /usr/local/share/ca-certificates/kx-intermediate-ca.crt ../../
                  cp /usr/local/share/ca-certificates/kx-root-ca.crt ../../

                  ls -latrh
                  echo "Prior Path"
                  ls -latrh ../../
                  cat Dockerfile
                  ls -latrh -R /root/.docker
                  hostname
                  cat /etc/hosts
                  cat /etc/resolv.conf
                  whoami
                  pwd
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=../.. --local dockerfile=. --output type=image,name=${image},push=true
                  buildctl --debug build --frontend dockerfile.v0 --progress=plain --local context=../.. --local dockerfile=. --output type=image,name=${repository}:${tag},push=true
                """
                milestone(3)
            }
        }

        stage('Trigger ArgoCD Sync') {
            withCredentials([usernamePassword(credentialsId: 'argocd-bs-bot-creds', passwordVariable: 'argoPass', usernameVariable: 'argoUser')]) {
                container('argocd') {
                    echo "Perform Argo CD Sync"
                    milestone(4)
                }
            }
        }
    }
}

properties([[
    $class: 'BuildDiscarderProperty',
    strategy: [
        $class: 'LogRotator',
        artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10']
    ]
]);