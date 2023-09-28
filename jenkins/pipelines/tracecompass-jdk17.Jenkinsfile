/* groovylint-disable GStringExpressionWithinString, LineLength, NestedBlockDepth, UnnecessaryGString */
/*******************************************************************************
 * Copyright (c) 2019, 2023 Ericsson.
 *
 * This program and the accompanying materials
 * are made available under the terms of the Eclipse Public License 2.0
 * which accompanies this distribution, and is available at
 * https://www.eclipse.org/legal/epl-2.0/
 *
 * SPDX-License-Identifier: EPL-2.0
 *******************************************************************************/
pipeline {
    agent {
        kubernetes {
            label 'tracecompass-build'
            yamlFile 'jenkins/pod-templates/tracecompass-pod.yaml'
        }
    }
    options {
        timestamps()
        timeout(time: 4, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    tools {
        maven 'apache-maven-3.8.6'
        jdk 'openjdk-jdk17-latest'
    }
    environment {
        MAVEN_OPTS="-Xms768m -Xmx4096m -XX:+UseSerialGC"
        MAVEN_WORKSPACE_SCRIPTS="../scripts"
        WORKSPACE_SCRIPTS="${WORKSPACE}/.scripts/"
        SITE_PATH="releng/org.eclipse.tracecompass.releng-site/target/repository/"
        RCP_PATH="rcp/org.eclipse.tracecompass.rcp.product/target/products/"
        RCP_SITE_PATH="rcp/org.eclipse.tracecompass.rcp.product/target/repository/"
        RCP_PATTERN="trace-compass-*"
        JAVADOC_PATH="target/site/apidocs"
        GIT_SHA_FILE="tc-git-sha"
    }
    parameters {
        booleanParam(name: 'NOTARIZE_MAC_RCP', defaultValue: false, description: "whether to notarize mac RCP packages")
    }
    stages {
        stage('Checkout') {
            steps {
                container('tracecompass') {
                    sh 'mkdir -p ${MAVEN_WORKSPACE_SCRIPTS}'
                    sh 'cp scripts/deploy-rcp.sh ${MAVEN_WORKSPACE_SCRIPTS}'
                    sh 'cp scripts/deploy-update-site.sh ${MAVEN_WORKSPACE_SCRIPTS}'
                    sh 'cp scripts/deploy-doc.sh ${MAVEN_WORKSPACE_SCRIPTS}'
                    sh 'cp scripts/deploy-javadoc.sh ${MAVEN_WORKSPACE_SCRIPTS}'
                    sh 'cp scripts/macosx-notarize.sh ${MAVEN_WORKSPACE_SCRIPTS}'
                    checkout([$class: 'GitSCM', branches: [[name: '$GERRIT_BRANCH_NAME']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'BuildChooserSetting', buildChooser: [$class: 'GerritTriggerBuildChooser']]], submoduleCfg: [], userRemoteConfigs: [[refspec: '$GERRIT_REFSPEC', url: '$GERRIT_REPOSITORY_URL']]])
                    sh 'mkdir -p ${WORKSPACE_SCRIPTS}'
                    sh 'cp ${MAVEN_WORKSPACE_SCRIPTS}/deploy-rcp.sh ${WORKSPACE_SCRIPTS}'
                    sh 'cp ${MAVEN_WORKSPACE_SCRIPTS}/deploy-update-site.sh ${WORKSPACE_SCRIPTS}'
                    sh 'cp ${MAVEN_WORKSPACE_SCRIPTS}/deploy-doc.sh ${WORKSPACE_SCRIPTS}'
                    sh 'cp ${MAVEN_WORKSPACE_SCRIPTS}/deploy-javadoc.sh ${WORKSPACE_SCRIPTS}'
                    sh 'cp ${MAVEN_WORKSPACE_SCRIPTS}/macosx-notarize.sh ${WORKSPACE_SCRIPTS}'
                }
            }
        }
        stage('Product File') {
            when {
                not { expression { return params.PRODUCT_FILE == null || params.PRODUCT_FILE.isEmpty() } }
            }
            steps {
                container('tracecompass') {
                    sh "cp -f ${WORKSPACE}/rcp/org.eclipse.tracecompass.rcp.product/${params.PRODUCT_FILE} ${WORKSPACE}/rcp/org.eclipse.tracecompass.rcp.product/tracing.product"
                }
            }
        }
        stage('Build') {
            steps {
                container('tracecompass') {
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp'
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp/org.eclipse.tracecompass.doc.dev'
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp/org.eclipse.tracecompass.doc.user'
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp/org.eclipse.tracecompass.gdbtrace.doc.user'
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp/org.eclipse.tracecompass.rcp.doc.user'
                    sh 'mkdir -p ${WORKSPACE}/doc/.temp/org.eclipse.tracecompass.tmf.pcap.doc.user'
                    sh 'mvn clean install -B -Dskip-jacoco=true -Pdeploy-doc -DdocDestination=${WORKSPACE}/doc/.temp -Pctf-grammar -Pbuild-rcp -Dmaven.repo.local=/home/jenkins/.m2/repository --settings /home/jenkins/.m2/settings.xml ${MAVEN_ARGS}'
                    sh 'mkdir -p ${SITE_PATH}'
                    sh 'git rev-parse --short HEAD > ${SITE_PATH}/${GIT_SHA_FILE}'
                    sh 'mkdir -p ${RCP_SITE_PATH}'
                    sh 'cp ${SITE_PATH}/${GIT_SHA_FILE} ${RCP_SITE_PATH}/${GIT_SHA_FILE}'
                }
            }
            post {
                always {
                    container('tracecompass') {
                        junit '*/*/target/surefire-reports/*.xml'
                        archiveArtifacts artifacts: '*/*tests/screenshots/*.jpeg,*/*tests/target/work/data/.metadata/.log', excludes: '**/org.eclipse.tracecompass.common.core.log', allowEmptyArchive: true
                    }
                }
            }
        }
        stage('Deploy Site') {
            when {
                expression { return params.DEPLOY_SITE }
            }
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                    sh '${WORKSPACE_SCRIPTS}/deploy-update-site.sh ${SITE_PATH} ${SITE_DESTINATION}'
                }
            }
        }
        stage('Deploy RCP') {
            when {
                expression { return params.DEPLOY_RCP }
            }
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                    sh '${WORKSPACE_SCRIPTS}/deploy-rcp.sh ${RCP_PATH} ${RCP_DESTINATION} ${RCP_SITE_PATH} ${RCP_SITE_DESTINATION} ${RCP_PATTERN} false'
                }
            }
        }
        stage('Deploy Doc') {
            when {
                expression { return params.DEPLOY_DOC }
            }
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                       sh '${WORKSPACE_SCRIPTS}/deploy-doc.sh'
                }
            }
        }
        stage('Javadoc') {
            when {
                expression { return params.JAVADOC }
            }
            steps {
                container('tracecompass') {
                    sh 'mvn clean javadoc:aggregate -Pbuild-api-docs -Dmaven.repo.local=/home/jenkins/.m2/repository --settings /home/jenkins/.m2/settings.xml ${MAVEN_ARGS}'
                }
            }
        }
        stage('Deploy Javadoc') {
            when {
                expression { return params.JAVADOC }
            }
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                   sh '${WORKSPACE_SCRIPTS}/deploy-javadoc.sh ${JAVADOC_PATH}'
                }
            }
        }
        stage('Notarize macos RCP packages') {
            when {
                expression { return params.NOTARIZE_MAC_RCP }
            }
            steps {
                sshagent (['projects-storage.eclipse.org-bot-ssh']) {
                    sh '${WORKSPACE_SCRIPTS}/macosx-notarize.sh ${RCP_DESTINATION}'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
                }
            }
        }
    }
    post {
        failure {
            container('tracecompass') {
                emailext subject: 'Build $BUILD_STATUS: $PROJECT_NAME #$BUILD_NUMBER!',
                body: '''$CHANGES \n
------------------------------------------\n
Check console output at $BUILD_URL to view the results.''',
                recipientProviders: [culprits(), requestor()],
                to: '${EMAIL_RECIPIENT}'
            }
        }
        fixed {
            container('tracecompass') {
                emailext subject: 'Build is back to normal: $PROJECT_NAME #$BUILD_NUMBER!',
                body: '''Check console output at $BUILD_URL to view the results.''',
                recipientProviders: [culprits(), requestor()],
                to: '${EMAIL_RECIPIENT}'
            }
        }
    }
}
