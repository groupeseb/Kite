// Syntax: https://www.jenkins.io/doc/book/pipeline/syntax/

pipeline {

  agent {
    label 'java8'
  }

  environment {
    JAVA_HOME = "${tool 'JDK-1.8u111'}"
    PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
  }

  options {
    skipDefaultCheckout()
    buildDiscarder(
      logRotator(
        artifactDaysToKeepStr: '30',
        artifactNumToKeepStr: '2',
        daysToKeepStr: '',
        numToKeepStr: '5'
      )
    )
    parallelsAlwaysFailFast()
  }

  stages {
    stage('Checkout') {
      steps {
        cleanWs()
        checkout(scm)
      }
    }

    stage('Build') {
      when {
        changeRequest()
      }
      steps {
        withMaven(
          maven: 'Maven3',
          mavenSettingsConfig: 'seb-nexus-aws-config'
        ) {
          sh """
            mvn install \
                -V \
                -Dmaven.test.failure.ignore=true
          """
        }

        junit '**/target/*-reports/*.xml'
      }
    }

    stage('Code Analysis') {
      when {
        changeRequest()
      }
      environment {
        // Key visible in SonarQube dashboard
        JAVA_HOME           = "${tool 'JDK-11'}"
        PATH                = "${env.JAVA_HOME}/bin:${env.PATH}"
        REPO_NAME           = 'Kite'
        PROJECT             = "KITE"
        EXCLUSIONS          = "''"
        COVERAGE_EXCLUSIONS = "''"
        SCANNER_HOME        = "${tool 'SonarQube Scanner AWS'}"
        VERSION             = "${readMavenPom().getVersion()}"
      }
      steps {
        withSonarQubeEnv('SonarQube AWS') {
          sh """
            ${SCANNER_HOME}/bin/sonar-scanner \
              -Dsonar.scm.provider=git \
              -Dsonar.projectKey=${PROJECT} \
              -Dsonar.projectVersion=${VERSION} \
              -Dsonar.java.source=8 \
              -Dsonar.sources=src/main/java \
              -Dsonar.tests=src/test/java \
              -Dsonar.java.binaries=target/classes \
              -Dsonar.java.libraries=target/dependency \
              -Dsonar.exclusions=${EXCLUSIONS} \
              -Dsonar.coverage.exclusions=${COVERAGE_EXCLUSIONS} \
              -Dsonar.dependencyCheck.reportPath=target/dependency-check-report.xml \
              -Dsonar.dependencyCheck.htmlReportPath=target/dependency-check-report.html \
              -Dsonar.pullrequest.key=${CHANGE_ID} \
              -Dsonar.pullrequest.branch=${CHANGE_BRANCH} \
              -Dsonar.pullrequest.base=${CHANGE_TARGET} \
              -Dsonar.pullrequest.github.repository=groupeseb/${REPO_NAME} \
              -Dsonar.pullrequest.github.endpoint=https://api.github.com/ \
              -Dsonar.pullrequest.provider=GitHub
          """
        }
      }
    }

    stage('Release') {
      when {
        buildingTag()
      }
      steps {
        sh "mkdir ${WORKSPACE}/.m2"
        withMaven(
          maven: 'Maven3',
          mavenSettingsConfig: 'seb-nexus-aws-config',
          mavenLocalRepo: "${WORKSPACE}/.m2"
        ) {
          sh """
            mvn --batch-mode clean

            mvn --batch-mode dependency:purge-local-repository

            mvn --batch-mode versions:set \
                -DnewVersion="${TAG_NAME.substring(1)}"

            mvn --batch-mode deploy \
                -DskipTests \
                -Dskip.it.tests=true \
                -PspecialReleaseProfile \
                -PspecialRelease
          """
        }
      }
    }
  }
  post {
    cleanup {
      cleanWs()
    }
  }
}
