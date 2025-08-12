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
            regexpFilterExpression: '^created$',
            regexpFilterText: '$action'
        )
    }

    environment {
        BASE_PATH  = "/var/www/admin/httpdocs"
        LIVE_DIR   = "${BASE_PATH}/SFI-Admin-UI"
        REPO_OWNER = "Hari979"
        REPO_NAME  = "crispy_kitchen"
        WORK_DIR   = "/home/ubuntu/deploy-${GIT_COMMIT}"
    }

    stages {
        stage('Validate Commit') {
            steps {
                script {
                    if (!env.GIT_COMMIT?.trim()) {
                        error("❌ GIT_COMMIT missing.")
                    }
                    echo "✅ Commit: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Download Source') {
            steps {
                sh """
                    rm -rf "${WORK_DIR}"
                    mkdir -p "${WORK_DIR}"
                    curl -L -o "${WORK_DIR}/release.zip" \
                      "https://github.com/${REPO_OWNER}/${REPO_NAME}/archive/${GIT_COMMIT}.zip"
                    unzip "${WORK_DIR}/release.zip" -d "${WORK_DIR}"
                """
            }
        }

        stage('Validate Static Files') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    dir(extracted) {
                        // Install validators
                        sh """
                            npm install -g htmlhint stylelint eslint linkinator
                            htmlhint .
                            stylelint "**/*.css" || true
                            eslint "**/*.js" || true
                            linkinator ./index.html || true
                        """
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
                    sh """
                        if [ -d "${LIVE_DIR}" ]; then
                            sudo mv "${LIVE_DIR}" "${BACKUP_DIR}"
                        fi
                        sudo mkdir -p "${LIVE_DIR}"
                    """
                }
            }
        }

        stage('Deploy Static Site') {
            steps {
                script {
                    def extracted = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    sh """
                        sudo rsync -av --delete "${extracted}/" "${LIVE_DIR}/"
                        rm -rf "${WORK_DIR}"
                        sudo systemctl restart apache2
                    """
                }
            }
        }
    }
}
