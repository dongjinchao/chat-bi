pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Git 分支')
    string(name: 'IMAGE_TAG', defaultValue: '', description: '镜像标签，留空时使用 BUILD_NUMBER-git短哈希')
    string(name: 'PLATFORM', defaultValue: 'linux/amd64', description: '构建平台，--load 只支持单平台，例如 linux/amd64')
    string(name: 'WEB_PORT', defaultValue: '8000', description: 'Web 访问端口')
    string(name: 'MCP_PORT', defaultValue: '8001', description: 'MCP 服务端口')
  }

  environment {
    DOCKER_BUILDKIT = '1'
    BUILDX_BUILDER = 'chat-bi-jenkins-builder'
    GIT_URL = 'https://github.com/dongjinchao/chat-bi.git'
    APP_HOME = '/home/chai-bi'
    CONTAINER_NAME = 'chat-bi'
    IMAGE_REPOSITORY = 'chat-bi/sqlbot'
    SQLBOT_BASE_IMAGE = 'registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-base:latest'
    SQLBOT_RUNTIME_IMAGE = 'registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-python-pg:latest'
  }

  stages {
    stage('拉取代码') {
      steps {
        git branch: params.BRANCH_NAME, url: env.GIT_URL
      }
    }

    stage('准备构建参数') {
      steps {
        script {
          if (params.PLATFORM.contains(',')) {
            error('当前流水线使用 docker buildx build --load，只支持单平台构建，请把 PLATFORM 设置为 linux/amd64 这类单个平台。')
          }
          def shortCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.EFFECTIVE_IMAGE_TAG = params.IMAGE_TAG?.trim() ? params.IMAGE_TAG.trim() : "${env.BUILD_NUMBER}-${shortCommit}"
          env.IMAGE = "${env.IMAGE_REPOSITORY}:${env.EFFECTIVE_IMAGE_TAG}"
          env.IMAGE_TAR = "${env.APP_HOME}/chat-bi-${env.EFFECTIVE_IMAGE_TAG}.tar"
          env.BUILD_AT = sh(script: "TZ=Asia/Shanghai date +'%Y-%m-%dT%H:%M'", returnStdout: true).trim()
          env.GITHUB_COMMIT = shortCommit
        }
        sh '''
          set -eux
          command -v git
          command -v docker
          command -v curl
          docker version
          if docker buildx version >/dev/null 2>&1; then
            docker buildx inspect "$BUILDX_BUILDER" >/dev/null 2>&1 || docker buildx create --name "$BUILDX_BUILDER" --use
            docker buildx use "$BUILDX_BUILDER"
            docker buildx inspect --bootstrap "$BUILDX_BUILDER"
          else
            echo "当前 Jenkins 节点未安装 Docker Buildx，将回退使用 docker build。"
          fi
          if ! mkdir -p "$APP_HOME" "$APP_HOME/data/sqlbot/excel" "$APP_HOME/data/sqlbot/file" "$APP_HOME/data/sqlbot/images" "$APP_HOME/data/sqlbot/logs" "$APP_HOME/data/postgresql"; then
            echo "Jenkins 用户没有 $APP_HOME 写入权限，请先在 Linux 服务器执行：sudo mkdir -p $APP_HOME && sudo chown -R $(id -u):$(id -g) $APP_HOME"
            exit 1
          fi
          test -w "$APP_HOME" || { echo "Jenkins 用户没有 $APP_HOME 写入权限，请先在 Linux 服务器执行：sudo mkdir -p $APP_HOME && sudo chown -R $(id -u):$(id -g) $APP_HOME"; exit 1; }
        '''
      }
    }

    stage('构建 Docker 镜像') {
      steps {
        sh '''
          set -eux
          if docker buildx version >/dev/null 2>&1; then
            docker buildx build \
              --platform "$PLATFORM" \
              --load \
              --tag "$IMAGE" \
              --build-arg BUILD_AT="$BUILD_AT" \
              --build-arg GITHUB_COMMIT="$GITHUB_COMMIT" \
              --build-arg SQLBOT_BASE_IMAGE="$SQLBOT_BASE_IMAGE" \
              --build-arg SQLBOT_RUNTIME_IMAGE="$SQLBOT_RUNTIME_IMAGE" \
              .
          else
            docker build \
              --tag "$IMAGE" \
              --build-arg BUILD_AT="$BUILD_AT" \
              --build-arg GITHUB_COMMIT="$GITHUB_COMMIT" \
              --build-arg SQLBOT_BASE_IMAGE="$SQLBOT_BASE_IMAGE" \
              --build-arg SQLBOT_RUNTIME_IMAGE="$SQLBOT_RUNTIME_IMAGE" \
              .
          fi
        '''
      }
    }

    stage('保存镜像到 /home/chai-bi') {
      steps {
        sh '''
          set -eux
          docker save "$IMAGE" -o "$IMAGE_TAR"
          ls -lh "$IMAGE_TAR"
        '''
      }
    }

    stage('启动镜像') {
      steps {
        sh '''
          set -eux
          docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
          docker run -d \
            --name "$CONTAINER_NAME" \
            --restart unless-stopped \
            --privileged=true \
            -p "${WEB_PORT}:8000" \
            -p "${MCP_PORT}:8001" \
            -v "$APP_HOME/data/sqlbot/excel:/opt/sqlbot/data/excel" \
            -v "$APP_HOME/data/sqlbot/file:/opt/sqlbot/data/file" \
            -v "$APP_HOME/data/sqlbot/images:/opt/sqlbot/images" \
            -v "$APP_HOME/data/sqlbot/logs:/opt/sqlbot/app/logs" \
            -v "$APP_HOME/data/postgresql:/var/lib/postgresql/data" \
            -e SECRET_KEY="${SECRET_KEY:-jenkins-chat-bi-secret}" \
            -e LOG_DIR="/opt/sqlbot/app/logs" \
            -e LOG_FORMAT="%(asctime)s - %(name)s - %(levelname)s:%(lineno)d - %(message)s" \
            "$IMAGE"

          for i in $(seq 1 60); do
            if curl -fsS "http://127.0.0.1:${WEB_PORT}/openapi.json" >/dev/null; then
              docker ps --filter "name=$CONTAINER_NAME"
              exit 0
            fi
            sleep 5
          done

          echo "Docker 容器最近 300 行日志："
          if docker ps -a --format '{{.Names}}' | grep -Fx "$CONTAINER_NAME" >/dev/null; then
            docker logs --tail=300 "$CONTAINER_NAME" || true
          else
            echo "容器 $CONTAINER_NAME 不存在，跳过容器日志收集。"
          fi
          echo "应用日志目录最近 300 行日志："
          for log_file in "$APP_HOME"/data/sqlbot/logs/*.log; do
            if [ -f "$log_file" ]; then
              echo "===== $log_file ====="
              tail -n 300 "$log_file" || true
            fi
          done
          exit 1
        '''
      }
    }
  }

  post {
    success {
      echo "发布完成：${env.IMAGE}，镜像文件：${env.IMAGE_TAR}"
    }
    failure {
      sh '''
        set +e
        echo "Docker 容器最近 300 行日志："
        if docker ps -a --format '{{.Names}}' | grep -Fx "$CONTAINER_NAME" >/dev/null; then
          docker logs --tail=300 "$CONTAINER_NAME"
        else
          echo "容器 $CONTAINER_NAME 不存在，跳过容器日志收集。"
        fi
        echo "应用日志目录最近 300 行日志："
        for log_file in "$APP_HOME"/data/sqlbot/logs/*.log; do
          if [ -f "$log_file" ]; then
            echo "===== $log_file ====="
            tail -n 300 "$log_file"
          fi
        done
        exit 0
      '''
    }
  }
}
