pipeline {
    environment {
        DOMAIN="apps.ocp.homelab.local"
        PRJ="jenkins-demo"
        APP="jenkinsapp"
        GIT_URL="https://github.com/sclorg/nodejs-ex.git"
        BRANCH_NAME="main"
    }
    agent {
        node {
            label "nodejs"
        }
    }
    stages {
        stage("create"){
            steps {
                script {
                    openshift.withCluster() {
                        echo("Create project ${env.PRJ}")
                        openshift.newProject("${env.PRJ}")
                        openshift.withProject("${env.PRJ}") {
                            echo("Preparing the CA bundle")
                            sh "echo -e '[http]\n sslCAInfo=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt > .gitconfig"
                            openshift.raw("create", "secret", "generic", "gitlab", "--from-file=.gitconfig")
                            openshift.raw("annotate", "secret", "gitlab", "build.openshift.io/source-secret.match-uri-1=${env.GIT_URL}*")
                            echo("Grant to developer read access to the project")
                            openshift.raw("policy", "add-role-to-user", "view", "developer")
                            echo("Create app ${env.APP}")
                            openshift.raw("new-app", "nodejs~${env.GIT_URL}#${env.BRANCH_NAME}", "--strategy=source", "--name${env.APP}")
                        }
                    }                    
                }
            }
        }
        stage("build") {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject("${env.PRJ}") {
                            echo("Expose route for service ${env.APP}")
                            openshift.expose("svc/${env.APP}", "--hostname ${env.PRJ}.${env.DOMAIN}")
                            echo("Wait for development ${env.APP} to finish")
                            timeout(5) {
                                openshift.selector("deployment", "${env.APP}").rollout().status()
                            }
                        }
                    }
                }
            }
        }
        stage("test") {
            input {
                message "About to test the application"
                ok "OK"
            }
            steps {
                echo "Check that '${env.PRJ}.${env.DOMAIN}' returns HTTP 200"
                sh "curl -s --fail ${env.DOMAIN}"
            }
        }
    }
    post {
        always {
            always {
                script {
                    openshift.withCluster() {
                        echo("Delete project ${env.PRJ}")
                        openshift.delete("project/${env.PRJ}")
                    }
                }
            }
        }
    }
}
