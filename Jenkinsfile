#! groovy
def account_auth = '499a32cf-cc22-4c27-92b3-6f73d62d3fb4'
def scm_url = 'https://github.com/swaince/spring-cloud-jenkins.git'
node {
    stage("初始化构建环境") {
        echo "正在初始化构建环境。。。"
        properties([[$class: 'JiraProjectProperty'],
                    parameters([
                            gitParameter(name: 'git_branch',
                                    branch: '',
                                    branchFilter: '.*',
                                    defaultValue: 'master',
                                    description: '请选择远程分支，默认构建master',
                                    quickFilterEnabled: false,
                                    selectedValue: 'NONE',
                                    sortMode: 'NONE',
                                    tagFilter: '*',
                                    type: 'PT_BRANCH'),

                            booleanParam(name: 'skipTest',
                                    defaultValue: true,
                                    description: '是否跳过单元测试'),

                            extendedChoice(name: 'modules',
                                    defaultValue: 'registry@8761',
                                    description: '请选择需要构建部署的模块',
                                    descriptionPropertyValue: '注册中心,远程服务,客户端服务',
                                    multiSelectDelimiter: ',',
                                    quoteValue: false,
                                    saveJSONParameterToFile: false,
                                    type: 'PT_CHECKBOX',
                                    value: 'registry@8761,service@8081,client@9090',
                                    visibleItemCount: 3),

                            extendedChoice(name: 'deploy_hosts',
                                    defaultValue: 'master',
                                    description: '服务器',
                                    descriptionPropertyValue: '服务器1,服务器2,服务器3',
                                    multiSelectDelimiter: ',',
                                    quoteValue: false,
                                    saveJSONParameterToFile: false,
                                    type: 'PT_CHECKBOX',
                                    value: 'registry,master,slave',
                                    visibleItemCount: 3)
                    ]),

        ])
    }

    stage("拉取代码") {
        echo "开始拉取源代码"
        checkout([$class                           : 'GitSCM',
                  branches                         : [[name: "${git_branch}"]],
                  doGenerateSubmoduleConfigurations: false,
                  extensions                       : [[$class                           : 'CleanBeforeCheckout',
                                                       deleteUntrackedNestedRepositories: true]],
                  submoduleCfg                     : [],
                  userRemoteConfigs                : [[credentialsId: "${account_auth}",
                                                       url          : "${scm_url}"]]])
        echo("代码拉取完成")
    }

    stage("编译打包") {
        echo "编译打包"
        try {
            "${modules}".split(',').eachWithIndex { m, index ->
                def serviceName = m.split("@")[0]
                echo "开始构建并推送模块${serviceName}远程镜像"
                if ("${skipTest}".toBoolean()) {
                    sh "mvn -Dmaven.skip.test=true -f ${serviceName} clean install docker:push"
                } else {
                    sh "mvn -f ${serviceName} clean install docker:push"
                }
            }
        }
        catch (e) {
            echo "开始构建整个项目"
            if ("${skipTest}".toBoolean()) {
                sh "mvn -Dmaven.skip.test=true clean install"
            } else {
                sh "mvn clean install"
            }
            "${modules}".split(',').eachWithIndex { m, index ->
                def serviceName = m.split("@")[0]
                echo "开始推送模块[${serviceName}]的远程镜像"
                sh "mvn -f ${serviceName} docker:push"
            }
        }
        echo "编译打包结束"
    }

    stage("服务部署") {
        echo '服务部署'
        "${deploy_hosts}".split(',').eachWithIndex { host, index ->
            "${modules}".split(',').each { module->
                def serviceName = module.split("@")[0]
                def servicePort = module.split("@")[1]
                sshPublisher(publishers: [sshPublisherDesc(configName: "${host}",
                        transfers: [sshTransfer(cleanRemote: false,
                                excludes: '',
                                execCommand: """
#!/bin/bash
pid=`docker ps | grep ${serviceName} | grep -v grep`
if [ "\$pid" != "" ]; then
    docker stop `docker ps | grep ${serviceName} | awk '{ print \$1 }'`
    docker rm `docker ps -a | grep ${serviceName} | awk '{ print \$1 }'`
    docker rmi `docker images | grep ${serviceName} } awk '{ print \$3 }'`
fi
docker run -d --rm --name ${serviceName} -p ${servicePort}:${servicePort} -u root docker.localregistry.com/library/${serviceName}
                            """,
                                execTimeout: 120000,
                                flatten: false,
                                makeEmptyDirs: false,
                                noDefaultExcludes: false,
                                patternSeparator: '[, ]+',
                                remoteDirectory: '',
                                remoteDirectorySDF: false,
                                removePrefix: '',
                                sourceFiles: '')],
                        usePromotionTimestamp: false,
                        useWorkspaceInPromotion: false,
                        verbose: false)])
            }
        }
    }
}