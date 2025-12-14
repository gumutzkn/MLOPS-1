pipeline {
    agent any

    environment {
        // Jenkins Credentials'dan çekilenler (Gizli)
        GCP_PROJECT = credentials('gcp-project-id')
        GCS_BUCKET_NAME = credentials('gcp-bucket-name')

        GCP_REGION = 'us-central1'
        IMAGE_TAG = "gcr.io/${GCP_PROJECT}/ml-project:latest" 
        GCLOUD_PATH = '/var/jenkins_home/google-cloud-sdk/bin'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'github-token', url: 'https://github.com/gumutzkn/MLOPS-PROJECT-1.git']])
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Docker İmajı İnşa Ediliyor...'
                    sh "docker build -t \$IMAGE_TAG ."
                }
            }
        }

       stage('Train Model (in Container)') {
            steps {
                withCredentials([file(credentialsId: 'gcp-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'Eğitim Containerı Başlatılıyor...'
                        
                        sh """
                        set +x
                        GCP_KEY_BASE64=\$(cat \$GOOGLE_APPLICATION_CREDENTIALS | base64 -w 0)
                        set -x

                        docker run --rm \
                        -e GCP_KEY_BASE64="\$GCP_KEY_BASE64" \
                        -e GOOGLE_APPLICATION_CREDENTIALS=/app/key.json \
                        -e GCS_BUCKET_NAME=\$GCS_BUCKET_NAME \
                        --entrypoint /bin/sh \
                        \$IMAGE_TAG \
                        -c 'echo \$GCP_KEY_BASE64 | base64 -d > /app/key.json && python pipeline/training_pipeline.py'
                        """
                    }
                }
            }
        }

        stage('Push Image to GCR') {
            steps {
                withCredentials([file(credentialsId: 'gcp-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'İmaj Google Container Registry\'ye yükleniyor...'
                        sh """
                        export PATH=\$PATH:${GCLOUD_PATH}
                        
                        gcloud auth activate-service-account --key-file="\$GOOGLE_APPLICATION_CREDENTIALS"
                        gcloud config set project \$GCP_PROJECT
                        gcloud auth configure-docker --quiet

                        docker push \$IMAGE_TAG
                        """
                    }
                }
            }
        }

        stage('Deploy to Cloud Run') {
            steps {
                withCredentials([file(credentialsId: 'gcp-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
                    script {
                        echo 'Cloud Run Deploy Başlıyor...'
                        sh """
                        export PATH=\$PATH:${GCLOUD_PATH}
                        
                        gcloud auth activate-service-account --key-file="\$GOOGLE_APPLICATION_CREDENTIALS"
                        gcloud config set project \$GCP_PROJECT
                        
                        gcloud run deploy ml-project \
                            --image=\$IMAGE_TAG \
                            --platform=managed \
                            --region=\$GCP_REGION \
                            --allow-unauthenticated \
                            --port=8080 \
                            --memory=2Gi \
                            --set-env-vars GCS_BUCKET_NAME=\$GCS_BUCKET_NAME
                        """
                    }
                }
            }
        }
    }
}