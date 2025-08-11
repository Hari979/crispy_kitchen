pipeline {
    agent { label 'test-frontend-agent' }

    triggers {
        // Webhook trigger — Jenkins will be triggered by GitHub release events
        GenericTrigger(
            genericVariables: [
                [key: 'RELEASE_TAG', value: '$.release.tag_name']
            ],
            causeString: 'Triggered by GitHub release: $RELEASE_TAG',
            token: 'github-release-trigger',
            printContributedVariables: true,
            printPostContent: true
        )
    }

    environment {
        BASE_PATH = "/var/www/admin/httpdocs"
        LIVE_DIR = "${BASE_PATH}/SFI-Admin-UI"
        TAG = "${RELEASE_TAG}"
        REPO_OWNER = "Hari979"
        REPO_NAME = "crispy_kitchen"
    }

    stages {
        stage('Backup Current Deployment') {
            steps {
                script {
                    def DATE = sh(script: "date +%F", returnStdout: true).trim()
                    def COUNT = 1
                    def BACKUP_DIR = ""

                    while (fileExists("${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}")) {
                        COUNT++
                    }

                    BACKUP_DIR = "${BASE_PATH}/SFI-Admin-UI-${DATE}-${COUNT}"
                    echo "Backing up ${LIVE_DIR} to ${BACKUP_DIR}"

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
                    def WORK_DIR = "/home/ubuntu/deploy-${TAG}"
                    def ZIP_PATH = "${WORK_DIR}/release.zip"

                    sh """
                        rm -rf "${WORK_DIR}"
                        mkdir -p "${WORK_DIR}"

                        curl -L \
                          -o "${ZIP_PATH}" \
                          "https://github.com/${REPO_OWNER}/${REPO_NAME}/archive/refs/tags/${TAG}.zip"

                        unzip "${ZIP_PATH}" -d "${WORK_DIR}"
                        EXTRACTED_FOLDER=$(find "${WORK_DIR}" -maxdepth 1 -type d -name "${REPO_NAME}-*")

                        sudo rsync -av "$EXTRACTED_FOLDER"/ "${LIVE_DIR}"/

                        rm -rf "${WORK_DIR}"
                        sudo systemctl restart apache2
                    """
                }
            }
        }
    }
}
