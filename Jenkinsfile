pipeline {
    agent {
        kubernetes {
            label 'cpp-builder'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: cpp
    image: ubuntu:22.04
    command: ["sleep", "infinity"]
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1000m"
'''
        }
    }

    options {
        // 构建超时 30 分钟
        timeout(time: 30, unit: 'MINUTES')
        // 保留最近 10 次构建记录
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // 显示时间戳
        timestamps()
    }

    stages {
        stage('环境准备') {
            steps {
                container('cpp') {
                    sh '''
                        apt-get update -qq
                        DEBIAN_FRONTEND=noninteractive apt-get install -y \
                            cmake \
                            gcc \
                            g++ \
                            ninja-build \
                            git \
                            --no-install-recommends
                        cmake --version
                        gcc --version
                    '''
                }
            }
        }

        stage('配置') {
            steps {
                container('cpp') {
                    sh '''
                        cmake -S . -B build \
                            -DCMAKE_BUILD_TYPE=Release \
                            -DCMAKE_CXX_STANDARD=17 \
                            -Dcmake_template_ENABLE_HARDENING=OFF \
                            -Dcmake_template_ENABLE_COVERAGE=OFF \
                            -Dcmake_template_ENABLE_CLANG_TIDY=OFF \
                            -Dcmake_template_ENABLE_CPPCHECK=OFF \
                            -Dcmake_template_BUILD_FUZZ_TESTS=OFF
                    '''
                }
            }
        }

        stage('构建') {
            steps {
                container('cpp') {
                    sh '''
                        cmake --build build --parallel $(nproc)
                    '''
                }
            }
        }

        stage('测试') {
            steps {
                container('cpp') {
                    sh '''
                        cd build
                        ctest --output-on-failure --parallel $(nproc)
                    '''
                }
            }
            post {
                always {
                    // 如果有 JUnit 格式测试报告可在此处收集
                    echo '测试阶段完成'
                }
            }
        }

        stage('归档产物') {
            steps {
                container('cpp') {
                    sh '''
                        mkdir -p artifacts
                        find build -maxdepth 1 -type f -executable \
                            -exec cp {} artifacts/ \;
                        ls -lh artifacts/
                    '''
                }
                archiveArtifacts artifacts: 'artifacts/**', allowEmptyArchive: true
            }
        }
    }

    post {
        success {
            echo '✅ 构建成功！'
        }
        failure {
            echo '❌ 构建失败，请检查日志'
        }
        always {
            echo "构建耗时: ${currentBuild.durationString}"
        }
    }
}
