pipeline {
    agent { label 'test-frontend-agent' }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'action', value: '$.action'],
                [key: 'release_tag', value: '$.release.tag_name'],
                [key: 'release_name', value: '$.release.name'],
                [key: 'GIT_COMMIT', value: '$.release.target_commitish']
            ],
            causeString: 'Triggered by GitHub Release: $release_tag',
            token: 'github-release-trigger',
            printContributedVariables: true,
            printPostContent: true,
            regexpFilterExpression: '^created$',
            regexpFilterText: '$action'
        )
    }

    environment {
        BASE_PATH   = "/var/www/admin/httpdocs"
        LIVE_DIR    = "${BASE_PATH}/SFI-Admin-UI"
        REPO_OWNER  = "Hari979"
        REPO_NAME   = "crispy_kitchen"
        WORK_DIR    = "/home/ubuntu/deploy-${GIT_COMMIT}"
        ZIP_PATH    = "${WORK_DIR}/release.zip"
    }

    stages {

        stage('Validate Commit') {
            steps {
                script {
                    if (!env.GIT_COMMIT?.trim()) {
                        error("❌ GIT_COMMIT is missing! Check webhook payload.")
                    } else {
                        echo "✅ Detected commit: ${env.GIT_COMMIT}"
                    }
                }
            }
        }

        stage('Download Release Source') {
            steps {
                sh """
                    rm -rf "${WORK_DIR}"
                    mkdir -p "${WORK_DIR}"

                    echo "⬇️ Downloading release zip..."
                    curl -L -o "${ZIP_PATH}" \
                        "https://github.com/${REPO_OWNER}/${REPO_NAME}/archive/${GIT_COMMIT}.zip"

                    unzip "${ZIP_PATH}" -d "${WORK_DIR}"
                """
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    dir(extracted) {
                        sh 'npm ci'
                    }
                }
            }
        }

        stage('Lint') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    dir(extracted) {
                        sh 'npm run lint'
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    dir(extracted) {
                        sh 'npm test -- --ci --coverage'
                    }
                }
            }
            post {
                always {
                    junit '**/junit/*.xml' // Only if you configure Jest to output JUnit XML
                }
            }
        }

        stage('Build Production Bundle') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    dir(extracted) {
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('Backup Current Deployment') {
            steps {
                script {
                    def DATE = sh(script: "date +%F", returnStdout: true).trim()
                    def COUNT = 1
                    while (fileExists("${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}")) {
                        COUNT++
                    }
                    def BACKUP_DIR = "${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}"
                    echo "🔄 Backing up ${LIVE_DIR} to ${BACKUP_DIR}"
                    sh """
                        if [ -d "${LIVE_DIR}" ]; then
                            sudo mv "${LIVE_DIR}" "${BACKUP_DIR}"
                        fi
                        sudo mkdir -p "${LIVE_DIR}"
                    """
                }
            }
        }

        stage('Deploy Built App') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    sh """
                        sudo rsync -av --delete "${extracted}/build/" "${LIVE_DIR}/"
                        rm -rf "${WORK_DIR}"
                        sudo systemctl restart apache2
                    """

                    echo "✅ Deployment completed for commit ${env.GIT_COMMIT}"
                }
            }
        }
    }
}
