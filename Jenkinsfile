pipeline {
    agent { label 'test-frontend-agent' }

    triggers {
        GenericTrigger(
            genericVariables: [
                [key: 'GIT_REF', value: '$.ref'],
                [key: 'GIT_COMMIT', value: '$.after']
            ],
            causeString: 'Triggered on push to $GIT_REF',
            token: 'github-push-trigger',                // Your webhook token here
            printContributedVariables: true,
            printPostContent: true,
            regexpFilterText: '$.ref',
            regexpFilterExpression: '^refs/heads/main$'  // Only trigger on main branch
        )
    }

    environment {
        BASE_PATH = "/var/www/admin/httpdocs"
        LIVE_DIR = "${BASE_PATH}/SFI-Admin-UI"
        REPO_OWNER = "Hari979"
        REPO_NAME = "crispy_kitchen"
        COMMIT = "${env.GIT_COMMIT ?: 'undefined'}"
    }

    stages {
        stage('Validate Commit') {
            steps {
                script {
                    if (COMMIT == 'undefined' || COMMIT.trim() == '') {
                        error("❌ GIT_COMMIT is missing! Check webhook payload.")
                    } else {
                        echo "✅ Detected commit: ${COMMIT}"
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

        stage('Download and Deploy') {
            steps {
                script {
                    def WORK_DIR = "/home/ubuntu/deploy-${COMMIT}"
                    def ZIP_PATH = "${WORK_DIR}/release.zip"
                    echo "⬇️ Downloading commit ZIP for ${COMMIT}"

                    sh """
                        rm -rf "${WORK_DIR}"
                        mkdir -p "${WORK_DIR}"

                        curl -L -o "${ZIP_PATH}" \
                          "https://github.com/${REPO_OWNER}/${REPO_NAME}/archive/${COMMIT}.zip"

                        unzip "${ZIP_PATH}" -d "${WORK_DIR}"
                    """

                    def EXTRACTED_FOLDER = sh(
                        script: "find ${WORK_DIR} -maxdepth 1 -type d -name '${REPO_NAME}-*'",
                        returnStdout: true
                    ).trim()

                    echo "📂 Extracted folder: ${EXTRACTED_FOLDER}"

                    sh """
                        sudo rsync -av --delete "${EXTRACTED_FOLDER}/" "${LIVE_DIR}/"
                        rm -rf "${WORK_DIR}"
                        sudo systemctl restart apache2
                    """

                    echo "✅ Deployment completed for commit ${COMMIT}"
                }
            }
        }
    }
}
