def DOCKER_IMG = "registry.fedoraproject.org/fedora:28"
def DOCKER_ARGS = "--net=host -v /srv:/srv --privileged"

// this var conveniently refers to a location on the server as well as the local dir we sync to/from
def repo = "${env.ARTIFACT_SERVER_DIR}/repo"
def rdgo  = "${env.ARTIFACT_SERVER_DIR}/rdgo"

if (env.BUILD_TYPE != 'origin' && env.BUILD_TYPE != 'rhcos') {
    assert false
}
def manifest = "host-${env.BUILD_TYPE}.json"

node(env.NODE) {
    checkout scm

    docker.image(DOCKER_IMG).inside(DOCKER_ARGS) {
        stage("Provision") {
            sh "dnf install -y git rpm-ostree rsync openssh-clients dnf-plugins-core dnf-plugins-core fedpkg"
            sh "cp RPM-GPG-* /etc/pki/rpm-gpg/"
            sh "dnf copr -y enable walters/buildtools-fedora"
            sh "dnf install -y rpmdistro-gitoverlay"
        }

        stage("Sync In") {
            withCredentials([sshUserPrivateKey(credentialsId: env['ARTIFACT_SSH_CREDS_ID'],
                                               keyFileVariable: 'KEY_FILE')]) {

                sh """
                    mkdir -p ${rdgo}
                    rsync -Hrlpt --stats \
                        -e 'ssh -i ${env.KEY_FILE} \
                            -o UserKnownHostFiles=/dev/null \
                           -o StrictHostKeyChecking=no' \
                        ${env.ARTIFACT_SERVER}:${rdgo}/ ${rdgo}
                """

                sh """
                    # a few idempotent commands for bootstrapping
                    mkdir -p ${repo}
                    ostree init --repo=${repo} --mode=archive

                    rsync -Hrlpt --stats \
                        -e 'ssh -i ${env.KEY_FILE} \
                                -o UserKnownHostsFile=/dev/null \
                                -o StrictHostKeyChecking=no' \
                        ${env.ARTIFACT_SERVER}:${repo}/ ${repo}
                """
            }
        }

        stage("Build rdgo overlay packages") {
            sh "ln -sf overlay.yml ${rdgo}"
            dir("${rdgo}") {
                if (!fileExists("src")) {
                    sh "rpmdistro-gitoverlay init"
                }
                sh "rm rdgo.stamp"
                sh "rpmdistro-gitoverlay resolve --fetch-all"
                sh "rpmdistro-gitoverlay build --touch-if-changed rdgo.stamp --logdir=log"
            }
        }

        stage("Sync out RDGO Builds") {
            dir("${rdgo}") {
                if (fileExists("rdgo.stamp")) {
                    sh """
                        rsync -Hrlpt --stats --delete --delete-after \
                            -e 'ssh -i ${env.KEY_FILE} \
                                    -o UserKnownHostsFile=/dev/null \
                                    -o StrictHostKeyChecking=no' \
                            ${rdgo}/ ${env.ARTIFACT_SERVER}:${rdgo}
                    """
                }
            }
        }

        stage("Check for Changes") {
            sh "rm -f build.stamp"
            sh "rpm-ostree compose tree --dry-run --repo=${repo} --touch-if-changed=build.stamp ${manifest}"
        }

        if (!fileExists('build.stamp')) {
            currentBuild.result = 'SUCCESS'
            return
        }

        stage("Compose Tree") {
            sh "rpm-ostree compose tree --repo=${repo} ${manifest}"
        }

        /*
        stage("Build Cloud Images") {
            sh "make cloud TYPE=qcow2"
        }
        */

        stage("Sync Out") {
            withCredentials([sshUserPrivateKey(credentialsId: env['ARTIFACT_SSH_CREDS_ID'],
                                               keyFileVariable: 'KEY_FILE')]) {
                sh """
                    ostree prune --repo=${repo} --keep-younger-than='30 days ago' --refs-only
                    ostree summary --repo=${repo} --update
                    rsync -Hrlpt --stats --delete --delete-after \
                        -e 'ssh -i ${env.KEY_FILE} \
                                -o UserKnownHostsFile=/dev/null \
                                -o StrictHostKeyChecking=no' \
                        ${repo}/ ${env.ARTIFACT_SERVER}:${repo}
                """
            }
        }
    }
}
