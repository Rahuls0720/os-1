def TIMER = "H/30 * * * *"
def NODE = "atomic-jslave-autobrew"
def API_CI_REGISTRY = "registry.svc.ci.openshift.org"
def OSCONTAINER_IMG = API_CI_REGISTRY + "/rhcos/os-maipo:latest"

// We write to this one for now
def artifact_repo = "/srv/rhcos/output/repo"
// We write pkg_diff.txt here
def images = "/srv/rhcos/output/images"

node(NODE) {
    def manifest = "host.yaml"
    def manifest_data = readYaml file: "${manifest}";
    // TODO - stop using the ref in favor of oscontainer:// pivot
    def ref = manifest_data.ref;
    def version_prefix = manifest_data["mutate-os-release"];

    def par_stages = [:]
    stage("Clean workspace") {
       step([$class: 'WsCleanup'])
    }
    checkout scm
    utils = load("pipeline-utils.groovy")
    utils.define_properties(TIMER)

    // Split the credentials up
    def username, password;
    withCredentials([
        usernameColonPassword(credentialsId: params.REGISTRY_CREDENTIALS, variable: 'CREDS'),
    ]) {
        (username, password) = "${CREDS}".split(':')
    }

    utils.inside_assembler_container("") {
        def treecompose_workdir = "${WORKSPACE}/treecompose"
        def repo = "${treecompose_workdir}/repo"

        stage("Login") { sh """
            set +x
            echo podman login -u ${username} -p '<password>' ${API_CI_REGISTRY}
            podman login -u "${username}" -p "${password}" ${API_CI_REGISTRY}
            echo "login done"
        """ }

        stage("Pull and run oscontainer") { sh """
            rm ${treecompose_workdir} -rf
            mkdir -p ${treecompose_workdir}
            rm -rf /var/lib/containers && mkdir /var/lib/containers
            mkdir -p ${treecompose_workdir}/containers
            mount --bind ${treecompose_workdir}/containers /var/lib/containers
            skopeo inspect docker://${OSCONTAINER_IMG}
            podman pull ${OSCONTAINER_IMG}
            cid=\$(podman run --net=host -d --name oscontainer --entrypoint sleep ${OSCONTAINER_IMG} infinity)
            ln -sf \$(podman mount \${cid})/srv/repo ${treecompose_workdir}/repo
        """ }

        def previous_commit, last_build_version, force_nocache
        stage("Check for Changes") { sh """
            cp RPM-GPG-* /etc/pki/rpm-gpg/
            make repo-refresh
            rm -f $WORKSPACE/build.stamp
            ls -al ${repo}
            ostree --repo=${repo} rev-parse ${ref} > commit.txt || true
        """
            previous_commit = readFile('commit.txt').trim();
        sh """
            rpm-ostree compose tree --dry-run --repo=${repo} --touch-if-changed=$WORKSPACE/build.stamp ${manifest}
        """
            last_build_version = utils.get_rev_version(repo, ref)
            if (fileExists('force-nocache-build')) {
                force_nocache = readFile('force-nocache-build').trim();
            }
        }

        if (!fileExists('build.stamp') && last_build_version != force_nocache) {
            echo "No changes."
            currentBuild.result = 'SUCCESS'
            currentBuild.description = '(No changes)'
            return
        }

        // Note that we don't keep history in the ostree repo, so we
        // delete the ref head (so rpm-ostree won't add a parent) and
        // then we do a repo prune after.
        def version, commit
        try {
            stage("Compose Tree") { sh """
                rm ${repo}/refs/heads/* -rf
                env G_DEBUG=fatal-warnings rpm-ostree compose tree --repo=${repo} \
                                                            --add-metadata-string=version=${version_prefix}.${env.BUILD_NUMBER} \
                                                            ${manifest}
                ostree --repo=${repo} rev-parse ${ref} > commit.txt
            """
                commit = readFile('commit.txt').trim();
                version = utils.get_rev_version("${repo}", commit)
                currentBuild.description = "🆕 ${version} (commit ${commit})";
                if (previous_commit.length() > 0) { 
                    sh """
                    rpm-ostree --repo=${repo} db diff ${previous_commit} ${commit} > ${WORKSPACE}/pkg_diff.txt
                    mkdir -p ${images}
                    cp ${WORKSPACE}/pkg_diff.txt ${images}/pkg_diff.txt
                    """
                }
                sh "ostree --repo=${repo} prune --refs-only --depth=0"
                sh "ostree summary --repo=${repo} --update"
            }
        } finally {
            archiveArtifacts artifacts: "pkg_diff.txt", allowEmptyArchive: true
        }

        // This takes OSTree repo and copies it out of the original
        // container into our working directory, which then gets put
        // into a new container image.  This is slow, but it's hard to do better
        // with Docker/OCI images.  Ideally there'd be a way to "bind mount" content
        // into the build environment rather than copying.
        stage("Prepare repo copy") { sh """
            original_repo=\$(readlink ${treecompose_workdir}/repo)
            rm ${repo}
            cp -a --reflink=auto \${original_repo} ${repo}
            podman kill oscontainer
            podman rm oscontainer
        """ }

        stage("Build new container") { sh """
            podman build --build-arg OS_VERSION=${version} \
                         --build-arg OS_COMMIT=${commit} \
                         -t ${OSCONTAINER_IMG} \
                         -f ${WORKSPACE}/Dockerfile.rollup ${WORKSPACE}
        """ }

        if (params.DRY_RUN) {
            echo "DRY_RUN set, skipping push"
            currentBuild.result = 'SUCCESS'
            currentBuild.description = '(dry run)'
            return
        }

        // stage("Push container") { sh """
        //     podman push ${OSCONTAINER_IMG}
        //     podman inspect --format='{{.Id}}' ${OSCONTAINER_IMG} > imgid.txt
        // """
        //     def cid = readFile('imgid.txt').trim();
        //     currentBuild.description = "🆕 ${OSCONTAINER_IMG}@sha256:${cid} (${version})";
        // }

        // stage("rsync out") {
        //     withCredentials([
        //        string(credentialsId: params.ARTIFACT_SERVER, variable: 'ARTIFACT_SERVER'),
        //        sshUserPrivateKey(credentialsId: params.ARTIFACT_SSH_CREDS_ID, keyFileVariable: 'KEY_FILE'),
        //     ]) {
        //     sh """
        //        /usr/app/ostree-releng-scripts/rsync-repos \
        //            --dest ${ARTIFACT_SERVER}:${artifact_repo} --src=${repo}/ \
        //            --rsync-opt=--stats --rsync-opt=-e \
        //            --rsync-opt='ssh -i ${KEY_FILE} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no'
        //     """ 
        //     if (previous_commit.length() > 0) {
        //         utils.rsync_dir_out(ARTIFACT_SERVER, KEY_FILE, images)
        //     } }
        // }

        stage("Cleanup") { sh """
            rm ${treecompose_workdir} -rf
        """ }

        // // Trigger downstream jobs
        // build job: 'coreos-rhcos-cloud', wait: false
  }
}