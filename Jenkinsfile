pipeline {
    agent any

    tools {
        jdk 'JDK-17'
        maven 'Maven-3.9'
    }

    environment {
        SONAR_PROJECT_KEY  = 'productos-cicd'
        ARTIFACTORY_SERVER = 'Artifactory'
        ARTIFACTORY_REPO   = 'libs-release-local'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '5'))
        disableConcurrentBuilds()
        timestamps()
    }

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile -Dmaven.test.skip=true --batch-mode --no-transfer-progress'
            }
            post {
                failure {
                    error 'Fallo en la compilacion.'
                }
            }
        }

        stage('Testing (JUnit + JaCoCo)') {
            steps {
                sh 'mvn verify --batch-mode --no-transfer-progress'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/**/*.xml', allowEmptyResults: false
                    jacoco(
                        execPattern:      'target/jacoco.exec',
                        classPattern:     'target/classes',
                        sourcePattern:    'src/main/java',
                        exclusionPattern: '**/model/**,**/ProductosApplication.class',
                        minimumLineCoverage: '60'
                    )
                }
                failure {
                    error 'Fallo en las pruebas unitarias o cobertura insuficiente.'
                }
            }
        }

        stage('Sonar') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                            --batch-mode \
                            --no-transfer-progress
                    """
                }
            }
            post {
                always {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Generar artefacto y almacenar en Artifactory') {
            steps {
                sh 'mvn package -Dmaven.test.skip=true --batch-mode --no-transfer-progress'
                rtUpload(
                    serverId: env.ARTIFACTORY_SERVER,
                    spec: """{
                        "files": [{
                            "pattern": "target/*.jar",
                            "target": "${ARTIFACTORY_REPO}/com/example/productos-cicd/${BUILD_NUMBER}/"
                        }]
                    }"""
                )
                rtPublishBuildInfo(
                    serverId: env.ARTIFACTORY_SERVER
                )
            }
            post {
                failure {
                    error 'Error al publicar el artefacto en Artifactory.'
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
