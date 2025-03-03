#!groovy
pipeline {
    agent none

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        timeout(time: 2, unit: 'HOURS')
        timestamps()
    }
    parameters {
        booleanParam(name: 'unit_validate', defaultValue: true, description: 'amd64 (x86_64) unit tests and vendor check')
        booleanParam(name: 'amd64', defaultValue: true, description: 'amd64 (x86_64) Build/Test')
        booleanParam(name: 's390x', defaultValue: true, description: 'IBM Z (s390x) Build/Test')
        booleanParam(name: 'ppc64le', defaultValue: true, description: 'PowerPC (ppc64le) Build/Test')
        booleanParam(name: 'windowsRS1', defaultValue: false, description: 'Windows 2016 (RS1) Build/Test')
        booleanParam(name: 'windowsRS5', defaultValue: true, description: 'Windows 2019 (RS5) Build/Test')
        booleanParam(name: 'skip_dco', defaultValue: false, description: 'Skip the DCO check')
    }
    environment {
        DOCKER_BUILDKIT     = '1'
        DOCKER_EXPERIMENTAL = '1'
        DOCKER_GRAPHDRIVER  = 'overlay2'
        APT_MIRROR          = 'cdn-fastly.deb.debian.org'
        CHECK_CONFIG_COMMIT = '78405559cfe5987174aa2cb6463b9b2c1b917255'
        TESTDEBUG           = '0'
        TIMEOUT             = '120m'
    }
    stages {
        stage('pr-hack') {
            when { changeRequest() }
            steps {
                script {
                    echo "Workaround for PR auto-cancel feature. Borrowed from https://issues.jenkins-ci.org/browse/JENKINS-43353"
                    def buildNumber = env.BUILD_NUMBER as int
                    if (buildNumber > 1) milestone(buildNumber - 1)
                    milestone(buildNumber)
                }
            }
        }
        stage('DCO-check') {
            when {
                beforeAgent true
                expression { !params.skip_dco }
            }
            agent { label 'amd64 && ubuntu-1804 && overlay2' }
            steps {
                sh '''
                docker run --rm \
                  -v "$WORKSPACE:/workspace" \
                  alpine sh -c 'apk add --no-cache -q git bash && cd /workspace && hack/validate/dco'
                '''
            }
        }
        stage('Build') {
            parallel {
                stage('unit-validate') {
                    when {
                        beforeAgent true
                        expression { params.unit_validate }
                    }
                    agent { label 'amd64 && ubuntu-1804 && overlay2' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh 'docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .'
                            }
                        }
                        stage("Validate") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  -v "$WORKSPACE/.git:/go/src/github.com/docker/docker/.git" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/validate/default
                                '''
                            }
                        }
                        stage("Docker-py") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary-daemon \
                                    test-docker-py
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/test-docker-py/junit-report.xml', allowEmptyResults: true

                                    sh '''
                                    echo "Ensuring container killed."
                                    docker rm -vf docker-pr$BUILD_NUMBER || true
                                    '''

                                    sh '''
                                    echo 'Chowning /workspace to jenkins user'
                                    docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                                    '''

                                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                        sh '''
                                        bundleName=docker-py
                                        echo "Creating ${bundleName}-bundles.tar.gz"
                                        tar -czf ${bundleName}-bundles.tar.gz bundles/test-docker-py/*.xml bundles/test-docker-py/*.log
                                        '''

                                        archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                                    }
                                }
                            }
                        }
                        stage("Static") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh binary-daemon
                                '''
                            }
                        }
                        stage("Cross") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh cross
                                '''
                            }
                        }
                        // needs to be last stage that calls make.sh for the junit report to work
                        stage("Unit tests") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/test/unit
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/junit-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                        stage("Validate vendor") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/.git:/go/src/github.com/docker/docker/.git" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/validate/vendor
                                '''
                            }
                        }
                        stage("Build e2e image") {
                            steps {
                                sh '''
                                echo "Building e2e image"
                                docker build --build-arg DOCKER_GITCOMMIT=${GIT_COMMIT} -t moby-e2e-test -f Dockerfile.e2e .
                                '''
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo 'Ensuring container killed.'
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            '''

                            sh '''
                            echo 'Chowning /workspace to jenkins user'
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=unit
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                tar -czvf ${bundleName}-bundles.tar.gz bundles/junit-report.xml bundles/go-test-report.json bundles/profile.out
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('amd64') {
                    when {
                        beforeAgent true
                        expression { params.amd64 }
                    }
                    agent { label 'amd64 && ubuntu-1804 && overlay2' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh '''
                                # todo: include ip_vs in base image
                                sudo modprobe ip_vs
                
                                docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .
                                '''
                            }
                        }
                        stage("Run tests") {
                            steps {
                                sh '''#!/bin/bash
                                # bash is needed so 'jobs -p' works properly
                                # it also accepts setting inline envvars for functions without explicitly exporting
 
                                run_tests() {
                                        [ -n "$TESTDEBUG" ] && rm= || rm=--rm;
                                        docker run $rm -t --privileged \
                                          -v "$WORKSPACE/bundles/${TEST_INTEGRATION_DEST}:/go/src/github.com/docker/docker/bundles" \
                                          -v "$WORKSPACE/bundles/dynbinary-daemon:/go/src/github.com/docker/docker/bundles/dynbinary-daemon" \
                                          -v "$WORKSPACE/.git:/go/src/github.com/docker/docker/.git" \
                                          --name "$CONTAINER_NAME" \
                                          -e KEEPBUNDLE=1 \
                                          -e TESTDEBUG \
                                          -e TESTFLAGS \
                                          -e TEST_SKIP_INTEGRATION \
                                          -e TEST_SKIP_INTEGRATION_CLI \
                                          -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                          -e DOCKER_GRAPHDRIVER \
                                          -e TIMEOUT \
                                          docker:${GIT_COMMIT} \
                                          hack/make.sh \
                                            "$1" \
                                            test-integration
                                }

                                trap "exit" INT TERM
                                trap 'pids=$(jobs -p); echo "Remaining pids to kill: [$pids]"; [ -z "$pids" ] || kill $pids' EXIT

                                CONTAINER_NAME=docker-pr$BUILD_NUMBER

                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  -v "$WORKSPACE/.git:/go/src/github.com/docker/docker/.git" \
                                  --name ${CONTAINER_NAME}-build \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary-daemon

                                # flaky + integration
                                TEST_INTEGRATION_DEST=1 CONTAINER_NAME=${CONTAINER_NAME}-1 TEST_SKIP_INTEGRATION_CLI=1 run_tests test-integration-flaky &

                                # integration-cli first set
                                TEST_INTEGRATION_DEST=2 CONTAINER_NAME=${CONTAINER_NAME}-2 TEST_SKIP_INTEGRATION=1 TESTFLAGS="-test.run Test(DockerSuite|DockerNetworkSuite|DockerHubPullSuite|DockerRegistrySuite|DockerSchema1RegistrySuite|DockerRegistryAuthTokenSuite|DockerRegistryAuthHtpasswdSuite)/" run_tests &

                                # integration-cli second set
                                TEST_INTEGRATION_DEST=3 CONTAINER_NAME=${CONTAINER_NAME}-3 TEST_SKIP_INTEGRATION=1 TESTFLAGS="-test.run Test(DockerSwarmSuite|DockerDaemonSuite|DockerExternalVolumeSuite)/" run_tests &

                                set +x
                                c=0
                                for job in $(jobs -p); do
                                        wait ${job} || c=$?
                                done
                                exit $c
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/**/*-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo "Ensuring container killed."
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            '''

                            sh '''
                            echo "Chowning /workspace to jenkins user"
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=amd64
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                # exclude overlay2 directories
                                find bundles -path '*/root/*overlay2' -prune -o -type f \\( -name '*-report.json' -o -name '*.log' -o -name '*.prof' -o -name '*-report.xml' \\) -print | xargs tar -czf ${bundleName}-bundles.tar.gz
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('s390x') {
                    when {
                        beforeAgent true
                        expression { params.s390x }
                    }
                    agent { label 's390x-ubuntu-1604' }
                    // s390x machines run on Docker 18.06, and buildkit has some bugs on that version
                    environment { DOCKER_BUILDKIT = '0' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh '''
                                docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .
                                '''
                            }
                        }
                        stage("Unit tests") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/test/unit
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/junit-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                        stage("Integration tests") {
                            environment { TEST_SKIP_INTEGRATION_CLI = '1' }
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  -e TESTDEBUG \
                                  -e TEST_SKIP_INTEGRATION_CLI \
                                  -e TIMEOUT \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary \
                                    test-integration
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/**/*-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo "Ensuring container killed."
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            '''

                            sh '''
                            echo "Chowning /workspace to jenkins user"
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=s390x-integration
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                # exclude overlay2 directories
                                find bundles -path '*/root/*overlay2' -prune -o -type f \\( -name '*-report.json' -o -name '*.log' -o -name '*.prof' -o -name '*-report.xml' \\) -print | xargs tar -czf ${bundleName}-bundles.tar.gz
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('s390x integration-cli') {
                    when {
                        beforeAgent true
                        not { changeRequest() }
                        expression { params.s390x }
                    }
                    agent { label 's390x-ubuntu-1604' }
                    // s390x machines run on Docker 18.06, and buildkit has some bugs on that version
                    environment { DOCKER_BUILDKIT = '0' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh '''
                                docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .
                                '''
                            }
                        }
                        stage("Integration-cli tests") {
                            environment { TEST_SKIP_INTEGRATION = '1' }
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  -e TEST_SKIP_INTEGRATION \
                                  -e TIMEOUT \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary \
                                    test-integration
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/**/*-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo "Ensuring container killed."
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            '''

                            sh '''
                            echo "Chowning /workspace to jenkins user"
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=s390x-integration-cli
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                # exclude overlay2 directories
                                find bundles -path '*/root/*overlay2' -prune -o -type f \\( -name '*-report.json' -o -name '*.log' -o -name '*.prof' -o -name '*-report.xml' \\) -print | xargs tar -czf ${bundleName}-bundles.tar.gz
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('ppc64le') {
                    when {
                        beforeAgent true
                        expression { params.ppc64le }
                    }
                    agent { label 'ppc64le-ubuntu-1604' }
                    // ppc64le machines run on Docker 18.06, and buildkit has some bugs on that version
                    environment { DOCKER_BUILDKIT = '0' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh 'docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .'
                            }
                        }
                        stage("Unit tests") {
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  docker:${GIT_COMMIT} \
                                  hack/test/unit
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/junit-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                        stage("Integration tests") {
                            environment { TEST_SKIP_INTEGRATION_CLI = '1' }
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_EXPERIMENTAL \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  -e TESTDEBUG \
                                  -e TEST_SKIP_INTEGRATION_CLI \
                                  -e TIMEOUT \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary \
                                    test-integration
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/**/*-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo "Ensuring container killed."
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            '''

                            sh '''
                            echo "Chowning /workspace to jenkins user"
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=ppc64le-integration
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                # exclude overlay2 directories
                                find bundles -path '*/root/*overlay2' -prune -o -type f \\( -name '*-report.json' -o -name '*.log' -o -name '*.prof' -o -name '*-report.xml' \\) -print | xargs tar -czf ${bundleName}-bundles.tar.gz
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('ppc64le integration-cli') {
                    when {
                        beforeAgent true
                        not { changeRequest() }
                        expression { params.ppc64le }
                    }
                    agent { label 'ppc64le-ubuntu-1604' }
                    // ppc64le machines run on Docker 18.06, and buildkit has some bugs on that version
                    environment { DOCKER_BUILDKIT = '0' }

                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                                sh '''
                                echo "check-config.sh version: ${CHECK_CONFIG_COMMIT}"
                                curl -fsSL -o ${WORKSPACE}/check-config.sh "https://raw.githubusercontent.com/moby/moby/${CHECK_CONFIG_COMMIT}/contrib/check-config.sh" \
                                && bash ${WORKSPACE}/check-config.sh || true
                                '''
                            }
                        }
                        stage("Build dev image") {
                            steps {
                                sh 'docker build --force-rm --build-arg APT_MIRROR -t docker:${GIT_COMMIT} .'
                            }
                        }
                        stage("Integration-cli tests") {
                            environment { TEST_SKIP_INTEGRATION = '1' }
                            steps {
                                sh '''
                                docker run --rm -t --privileged \
                                  -v "$WORKSPACE/bundles:/go/src/github.com/docker/docker/bundles" \
                                  --name docker-pr$BUILD_NUMBER \
                                  -e DOCKER_GITCOMMIT=${GIT_COMMIT} \
                                  -e DOCKER_GRAPHDRIVER \
                                  -e TEST_SKIP_INTEGRATION \
                                  -e TIMEOUT \
                                  docker:${GIT_COMMIT} \
                                  hack/make.sh \
                                    dynbinary \
                                    test-integration
                                '''
                            }
                            post {
                                always {
                                    junit testResults: 'bundles/**/*-report.xml', allowEmptyResults: true
                                }
                            }
                        }
                    }

                    post {
                        always {
                            sh '''
                            echo "Ensuring container killed."
                            docker rm -vf docker-pr$BUILD_NUMBER || true
                            cids=$(docker ps -aq -f name=docker-pr${BUILD_NUMBER}-*)
                            [ -n "$cids" ] && docker rm -vf $cids || true
                            '''

                            sh '''
                            echo "Chowning /workspace to jenkins user"
                            docker run --rm -v "$WORKSPACE:/workspace" busybox chown -R "$(id -u):$(id -g)" /workspace
                            '''

                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                sh '''
                                bundleName=ppc64le-integration-cli
                                echo "Creating ${bundleName}-bundles.tar.gz"
                                # exclude overlay2 directories
                                find bundles -path '*/root/*overlay2' -prune -o -type f \\( -name '*-report.json' -o -name '*.log' -o -name '*.prof' -o -name '*-report.xml' \\) -print | xargs tar -czf ${bundleName}-bundles.tar.gz
                                '''

                                archiveArtifacts artifacts: '*-bundles.tar.gz', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('win-RS1') {
                    when {
                        beforeAgent true
                        // Skip this stage on PRs unless the windowsRS1 checkbox is selected
                        anyOf {
                            not { changeRequest() }
                            expression { params.windowsRS1 }
                        }
                    }
                    environment {
                        DOCKER_BUILDKIT        = '0'
                        DOCKER_DUT_DEBUG       = '1'
                        SKIP_VALIDATION_TESTS  = '1'
                        SOURCES_DRIVE          = 'd'
                        SOURCES_SUBDIR         = 'gopath'
                        TESTRUN_DRIVE          = 'd'
                        TESTRUN_SUBDIR         = "CI-$BUILD_NUMBER"
                        WINDOWS_BASE_IMAGE     = 'mcr.microsoft.com/windows/servercore'
                        WINDOWS_BASE_IMAGE_TAG = 'ltsc2016'
                    }
                    agent {
                        node {
                            customWorkspace 'd:\\gopath\\src\\github.com\\docker\\docker'
                            label 'windows-2016'
                        }
                    }
                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                            }
                        }
                        stage("Run tests") {
                            steps {
                                powershell '''
                                $ErrorActionPreference = 'Stop'
                                [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
                                Invoke-WebRequest https://github.com/jhowardmsft/docker-ci-zap/blob/master/docker-ci-zap.exe?raw=true -OutFile C:/Windows/System32/docker-ci-zap.exe
                                ./hack/ci/windows.ps1
                                exit $LastExitCode
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                powershell '''
                                $bundleName="windowsRS1-integration"
                                Write-Host -ForegroundColor Green "Creating ${bundleName}-bundles.zip"

                                # archiveArtifacts does not support env-vars to , so save the artifacts in a fixed location
                                Compress-Archive -Path "${env:TEMP}/CIDUT.out", "${env:TEMP}/CIDUT.err" -CompressionLevel Optimal -DestinationPath "${bundleName}-bundles.zip"
                                '''

                                archiveArtifacts artifacts: '*-bundles.zip', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
                stage('win-RS5') {
                    when {
                        beforeAgent true
                        expression { params.windowsRS5 }
                    }
                    environment {
                        DOCKER_BUILDKIT        = '0'
                        DOCKER_DUT_DEBUG       = '1'
                        SKIP_VALIDATION_TESTS  = '1'
                        SOURCES_DRIVE          = 'd'
                        SOURCES_SUBDIR         = 'gopath'
                        TESTRUN_DRIVE          = 'd'
                        TESTRUN_SUBDIR         = "CI-$BUILD_NUMBER"
                        WINDOWS_BASE_IMAGE     = 'mcr.microsoft.com/windows/servercore'
                        WINDOWS_BASE_IMAGE_TAG = 'ltsc2019'
                    }
                    agent {
                        node {
                            customWorkspace 'd:\\gopath\\src\\github.com\\docker\\docker'
                            label 'windows-2019'
                        }
                    }
                    stages {
                        stage("Print info") {
                            steps {
                                sh 'docker version'
                                sh 'docker info'
                            }
                        }
                        stage("Run tests") {
                            steps {
                                powershell '''
                                $ErrorActionPreference = 'Stop'
                                Invoke-WebRequest https://github.com/jhowardmsft/docker-ci-zap/blob/master/docker-ci-zap.exe?raw=true -OutFile C:/Windows/System32/docker-ci-zap.exe
                                ./hack/ci/windows.ps1
                                exit $LastExitCode
                                '''
                            }
                        }
                    }
                    post {
                        always {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE', message: 'Failed to create bundles.tar.gz') {
                                powershell '''
                                $bundleName="windowsRS5-integration"
                                Write-Host -ForegroundColor Green "Creating ${bundleName}-bundles.zip"

                                # archiveArtifacts does not support env-vars to , so save the artifacts in a fixed location
                                Compress-Archive -Path "${env:TEMP}/CIDUT.out", "${env:TEMP}/CIDUT.err" -CompressionLevel Optimal -DestinationPath "${bundleName}-bundles.zip"
                                '''

                                archiveArtifacts artifacts: '*-bundles.zip', allowEmptyArchive: true
                            }
                        }
                        cleanup {
                            sh 'make clean'
                            deleteDir()
                        }
                    }
                }
            }
        }
    }
}
