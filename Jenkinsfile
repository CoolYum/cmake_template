pipeline {
    agent {
        kubernetes {
            label 'cpp-builder'
            podRetention always()
            idleMinutes 60
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: cpp
    image: ubuntu:24.04
    imagePullPolicy: IfNotPresent
    command: ["sleep", "infinity"]
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
'''
        }
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {
        stage('环境准备') {
            steps {
                container('cpp') {
                    sh '''
                        apt-get update -qq
                        DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
                            cmake \
                            gcc \
                            g++ \
                            ninja-build \
                            make \
                            git \
                            curl \
                            ca-certificates \
                            pkg-config \
                            python3 \
                            file
                        echo "=== 工具版本 ==="
                        cmake --version
                        gcc --version
                        g++ --version
                        ninja --version
                        git --version
                    '''
                }
            }
        }

        stage('配置') {
            steps {
                container('cpp') {
                    sh '''
                        cmake -S . -B build \
                            -G Ninja \
                            -DCMAKE_BUILD_TYPE=Release \
                            -DCMAKE_CXX_STANDARD=23 \
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
                            -exec cp {} artifacts/ \\;
                        ls -lh artifacts/
                    '''
                }
                archiveArtifacts artifacts: 'artifacts/**', allowEmptyArchive: true
            }
        }
    }

    post {
        success {
            echo '构建成功！'
        }
        failure {
            echo '构建失败，请检查日志'
        }
        always {
            echo "构建耗时: ${currentBuild.durationString}"
        }
    }
}
