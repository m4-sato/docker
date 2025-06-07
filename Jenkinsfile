pipeline {
  agent any

  environment {
    DOCKERHUB_USER = "m4sato"
    BUILD_HOST      = "root@192.168.1.43"
    PROD_HOST       = "root@192.168.1.44"
    // BUILD_TIMESTAMP はここでは定義せず、Init ステージでシェル実行します
  }

  stages {
    // -- タイムスタンプ生成 --
    stage('Init') {
      steps {
        script {
          // 環境変数に格納
          env.BUILD_TIMESTAMP = sh(
            script: "date +%Y%m%d-%H%M%S", 
            returnStdout: true
          ).trim()
          echo "BUILD_TIMESTAMP=${env.BUILD_TIMESTAMP}"
        }
      }
    }

    // -- 事前チェック --
    stage('Pre Check') {
      steps {
        script {
          def status = sh(
            script: 'test -f /var/jenkins_home/.docker/config.json',
            returnStatus: true
          )
          if (status == 0) {
            echo 'Docker config found.'
          } else {
            echo 'Docker config not found. Skipping Docker login.'
          }
        }
      }
    }

    // -- Build ステージ --
    stage('Build') {
      steps {
        sh "cat docker-compose.build.yml"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down"
        sh "docker -H ssh://${BUILD_HOST} volume prune -f"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml build"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml up -d"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml ps"
      }
    }

    // -- Test ステージ --
    stage('Test') {
      steps {
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_apptest pytest -v test_app.py"
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_static.py"
        sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_selenium.py"
        sh "docker-compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down"
      }
    }

    // -- Register ステージ --
    stage('Register') {
      steps {
        sh "docker -H ssh://${BUILD_HOST} tag dockerkvs_web   ${DOCKERHUB_USER}/dockerkvs_web:${env.BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} tag dockerkvs_app   ${DOCKERHUB_USER}/dockerkvs_app:${env.BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} push ${DOCKERHUB_USER}/dockerkvs_web:${env.BUILD_TIMESTAMP}"
        sh "docker -H ssh://${BUILD_HOST} push ${DOCKERHUB_USER}/dockerkvs_app:${env.BUILD_TIMESTAMP}"
      }
    }

    // -- Deploy ステージ --
    stage('Deploy') {
      steps {
        sh "cat docker-compose.prod.yml"
        // .env を生成してから実行
        writeFile file: '.env', text: """
          DOCKERHUB_USER=${DOCKERHUB_USER}
          BUILD_TIMESTAMP=${env.BUILD_TIMESTAMP}
        """.trim()
        sh "cat .env"
        sh "docker-compose -H ssh://${PROD_HOST} -f docker-compose.prod.yml up -d"
        sh "docker-compose -H ssh://${PROD_HOST} -f docker-compose.prod.yml ps"
      }
    }

  } // stages
} // pipeline

