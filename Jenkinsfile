pipeline {
  agent {
    label 'Jenkins-Agent'
  }
  tools {
    jdk 'java17'
    maven 'maven3'
  }
  environment {
    APP_NAME = "register-app-pipeline"
    RELEASE = "1.0.0"
    DOCKER_USER = "grknyzc53"
    DOCKER_PASS = 'dockerhub'
    IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
    IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
  }
  stages {
    stage("Cleanup Workspace") {
      steps {
        cleanWs()
      }
    }

    stage("Checkout from SCM") {
      steps {
        git branch: 'main',
            credentialsId: 'github',
            url: 'https://github.com/yazicigurkan/register-app'
      }
    }

    stage("OWASP Dependency Check") {
      steps {
        sh "mvn org.owasp:dependency-check-maven:check"
      }
    }

    stage("SonarQube Analysis") {
      steps {
        script {
          withSonarQubeEnv(credentialsId: 'sonarqube-jenkins-token') {
            sh "mvn sonar:sonar"
          }
        }
      }
    }

    stage("Quality Gate") {
      steps {
        script {
          waitForQualityGate abortPipeline: false, credentialsId: 'sonarqube-jenkins-token'
        }
      }
    }

    stage("Build Application") {
      steps {
        sh "mvn clean package -DskipTests"
      }
    }

    stage("Test Application") {
      steps {
        sh "mvn test"
      }
    }

    stage("Build & Push Docker Image") {
      steps {
        script {
          docker.withRegistry('', DOCKER_PASS) {
            docker_image = docker.build("${IMAGE_NAME}")
            docker_image.push("${IMAGE_TAG}")
            docker_image.push("latest")
          }
        }
      }
    }

    stage("Trivy Scan") {
      steps {
        sh '''
          docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image ${IMAGE_NAME}:latest \
          --no-progress --scanners vuln \
          --exit-code 0 \
          --severity HIGH,CRITICAL \
          --format table
        '''
      }
    }
stage("OWASP ZAP DAST Scan") {
  steps {
    script {
      // Uygulama container'ını başlat
      sh """
        docker run -d --name app-under-test -p 8080:8080 ${IMAGE_NAME}:${IMAGE_TAG}
        sleep 15
      """

      // ZAP taraması yap (JSON + HTML raporları oluşturulur)
      sh """
        docker run --rm -v ${env.WORKSPACE}:/zap/wrk \
        ghcr.io/zaproxy/zap-baseline:stable \
        -t http://localhost:8080 \
        -r zap_report.html \
        -J zap_report.json \
        -m 0
      """

      // Raporu oku ve kritik açık varsa pipeline'ı durdur
      def zapReport = readFile('zap_report.json')
      if (zapReport.contains('"risk": "High"') || zapReport.contains('"risk": "Medium"')) {
        error("OWASP ZAP taramasında Medium veya High seviyesinde açık bulundu. Pipeline durduruldu.")
      } else {
        echo "ZAP taramasında kritik açık bulunamadı. Devam ediliyor."
      }
    }
  }
  post {
    always {
      sh "docker rm -f app-under-test || true"
      archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
    }
  }
}

    stage("Cleanup Artifacts") {
      steps {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${IMAGE_NAME}:latest || true"
      }
    }

    stage("Update Manifest Repo for GitOps") {
      steps {
        withCredentials([string(credentialsId: 'jenkins-manifest-github-token', variable: 'GITHUB_TOKEN')]) {
          script {
            def repoDir = "manifests-repo"
            def repoUrl = "https://yazicigurkan:${GITHUB_TOKEN}@github.com/yazicigurkan/register-app-manifests.git"
            def newImage = "${IMAGE_NAME}:${IMAGE_TAG}"

            sh """#!/bin/bash
              set -e
              git config --global user.email "gurkan.yazici.53@icloud.com"
              git config --global user.name "yazicigurkan"

              rm -rf ${repoDir}
              git clone ${repoUrl} ${repoDir}
              cd ${repoDir}

              sed -i 's|image:.*|image: ${newImage}|' deployment.yaml

              git add deployment.yaml
              git commit -m "Update image to ${newImage} [Jenkins CI]"
              git push origin main
            """
          }
        }
      }
    }
  }
}
