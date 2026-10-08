// App-Manager 构建流水线。
// Job 由平台创建，构建参数：
//   GIT_URL / GIT_REF / GIT_CREDENTIALS_ID / IMAGE
//   DOCKERFILE_PATH / BASE_DIR / DOCKER_BUILD_ARGS / TASK_ID
//
// BASE_DIR 为空表示仓库根目录。DOCKERFILE_PATH 相对 BASE_DIR。
// REGISTRY_SERVER / REGISTRY_USERNAME / REGISTRY_PASSWORD 由平台按镜像 host
// 匹配镜像仓库后在触发时传入。密码是 Jenkins 密码参数，日志里不要打印它。

def gitBranchSpec(String ref) {
    def r = (ref ?: '').trim()
    if (!r) {
        error('GIT_REF 为空')
    }
    if (r.startsWith('refs/') || r.startsWith('*/') || (r ==~ /^[0-9a-fA-F]{7,40}$/)) {
        return r
    }
    if (r.startsWith('origin/')) {
        return "*/${r.substring('origin/'.length())}"
    }
    return "*/${r}"
}

def assertRelativePath(String path, String name) {
    def p = (path ?: '').trim()
    if (!p || p.startsWith('/') || p.split('/').any { it == '..' }) {
        error("${name} 必须是工作区内的相对路径，当前值: ${path}")
    }
}

pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout()
        timeout(time: 60, unit: 'MINUTES')
    }

    parameters {
        string(name: 'GIT_URL', description: '应用仓库地址（SSH 或 HTTPS）')
        string(name: 'GIT_REF', description: '分支、tag 或 commit')
        string(name: 'GIT_CREDENTIALS_ID', description: 'Jenkins 凭据 ID，用于拉取 GIT_URL')
        string(name: 'IMAGE', description: '镜像地址，含 tag，例如 registry.example.com/group/app:v1')
        string(name: 'DOCKERFILE_PATH', defaultValue: 'Dockerfile', description: 'Dockerfile 路径，相对 BASE_DIR')
        string(name: 'BASE_DIR', defaultValue: '', description: '构建上下文目录，空表示仓库根目录')
        string(name: 'DOCKER_BUILD_ARGS', defaultValue: '', description: '额外 docker build 参数，如 --no-cache --build-arg KEY=VAL')
        string(name: 'TASK_ID', defaultValue: '', description: 'App-Manager 变更 ID')
        string(name: 'REGISTRY_SERVER', defaultValue: '', description: '镜像仓库地址，如 harbor.example.com:5000')
        string(name: 'REGISTRY_USERNAME', defaultValue: '', description: '镜像仓库用户名')
        password(name: 'REGISTRY_PASSWORD', defaultValue: '', description: '镜像仓库密码，触发构建时传入')
    }

    stages {
        stage('拉取代码') {
            steps {
                script {
                    def gitUrl = params.GIT_URL?.trim()
                    def credentialsId = params.GIT_CREDENTIALS_ID?.trim()
                    if (!gitUrl) {
                        error('GIT_URL 为空')
                    }
                    if (!credentialsId) {
                        error('GIT_CREDENTIALS_ID 为空')
                    }

                    def branchSpec = gitBranchSpec(params.GIT_REF)
                    echo "使用凭据 ${credentialsId} 拉取 ${gitUrl} (${branchSpec})"

                    deleteDir()
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: branchSpec]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'CloneOption', depth: 1, honorRefspec: true, noTags: false, shallow: true, timeout: 20],
                            [$class: 'CleanBeforeCheckout'],
                        ],
                        submoduleCfg: [],
                        userRemoteConfigs: [[
                            url: gitUrl,
                            credentialsId: credentialsId,
                        ]],
                    ])
                }
            }
        }

        stage('构建并推送镜像') {
            steps {
                script {
                    def image = params.IMAGE?.trim()
                    if (!image) {
                        error('IMAGE 为空')
                    }

                    def baseDir = params.BASE_DIR?.trim() ?: ''
                    def dockerfileRel = params.DOCKERFILE_PATH?.trim() ?: 'Dockerfile'
                    def contextDir = baseDir ?: '.'
                    def dockerfile = baseDir ? "${baseDir}/${dockerfileRel}" : dockerfileRel

                    if (baseDir) {
                        assertRelativePath(baseDir, 'BASE_DIR')
                    }
                    assertRelativePath(dockerfileRel, 'DOCKERFILE_PATH')
                    assertRelativePath(dockerfile, 'Dockerfile')

                    if (!fileExists(dockerfile)) {
                        error("找不到 Dockerfile: ${dockerfile}")
                    }

                    def extraArgs = (params.DOCKER_BUILD_ARGS ?: '').tokenize()
                    writeFile file: '.docker-build-args', text: extraArgs.join('\n')

                    def registryServer = params.REGISTRY_SERVER?.trim() ?: ''
                    def registryUser = params.REGISTRY_USERNAME?.trim() ?: ''
                    def registryPass = params.REGISTRY_PASSWORD ?: ''
                    def doLogin = registryServer && registryUser && registryPass
                    if (doLogin) {
                        echo "docker login ${registryServer} -u ${registryUser}"
                    } else {
                        echo "未传入完整镜像仓库凭据，跳过 docker login"
                    }
                    echo "docker build -f ${dockerfile} -t ${image} ${extraArgs.join(' ')} ${contextDir}"

                    withEnv([
                        "APP_DOCKERFILE=${dockerfile}",
                        "APP_IMAGE=${image}",
                        "APP_CONTEXT=${contextDir}",
                        "REGISTRY_SERVER=" + registryServer,
                        "REGISTRY_USERNAME=" + registryUser,
                        "REGISTRY_PASSWORD=" + registryPass,
                    ]) {
                        sh '''
                            set -eu
                            # 每个构建使用独立的 DOCKER_CONFIG，避免 docker logout 清掉同一 Agent 上其他任务的登录态。
                            if [ -n "$REGISTRY_SERVER" ] && [ -n "$REGISTRY_USERNAME" ] && [ -n "$REGISTRY_PASSWORD" ]; then
                                cfg=$(mktemp -d)
                                trap 'rm -rf "$cfg"' EXIT
                                export DOCKER_CONFIG="$cfg"
                                printf '%s\n' '{"auths":{}}' > "${DOCKER_CONFIG}/config.json"
                                printf '%s' "$REGISTRY_PASSWORD" | docker login "$REGISTRY_SERVER" --username "$REGISTRY_USERNAME" --password-stdin
                            fi
                            extra=()
                            if [ -s .docker-build-args ]; then
                                while IFS= read -r line || [ -n "$line" ]; do
                                    [ -z "$line" ] && continue
                                    extra+=("$line")
                                done < .docker-build-args
                            fi
                            if [ "${#extra[@]}" -gt 0 ]; then
                                docker build -f "$APP_DOCKERFILE" -t "$APP_IMAGE" "${extra[@]}" "$APP_CONTEXT"
                            else
                                docker build -f "$APP_DOCKERFILE" -t "$APP_IMAGE" "$APP_CONTEXT"
                            fi
                            docker push "$APP_IMAGE"
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "镜像已推送: ${params.IMAGE}"
        }
        failure {
            echo "构建失败, TASK_ID=${params.TASK_ID ?: '-'}"
        }
    }
}
