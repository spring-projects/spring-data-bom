def p = [:]
node {
	checkout scm
	p = readProperties interpolate: true, file: 'ci/pipeline.properties'
}

pipeline {
	agent none

	triggers {
		pollSCM 'H/10 * * * *'
		upstream(upstreamProjects: "spring-data-commons/3.2.x,spring-data-cassandra/4.2.x,spring-data-couchbase/5.2.x,spring-data-elasticsearch/5.2.x," +
			"spring-data-rest/4.2.x,spring-data-jpa/3.2.x,spring-data-ldap/3.2.x,spring-data-mongodb/4.2.x,spring-data-neo4j/7.2.x,spring-data-redis/3.2.x", threshold: hudson.model.Result.SUCCESS)
	}

	options {
		disableConcurrentBuilds()
		buildDiscarder(logRotator(numToKeepStr: '7'))
	}

	stages {
		stage('Verify BOM Dependencies') {
			when {
				beforeAgent(true)
				anyOf {
					branch(pattern: "main|(\\d+\\.\\d+\\.x)", comparator: "REGEXP")
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials("${p['artifactory.credentials']}")
			}

			steps {
				script {
					docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
						sh 'MAVEN_OPTS="-Duser.name=' + "${p['jenkins.user.name']}" + ' -Duser.home=/tmp/jenkins-home" ' +
							"./mvnw -Pwith-bom-client verify -B -U"
					}
				}
			}
		}

		stage("Verify other configurations") {
			when {
				beforeAgent(true)
				allOf {
					branch(pattern: "main|(\\d+\\.\\d+\\.x)", comparator: "REGEXP")
					not { triggeredBy 'UpstreamCause' }
				}
			}
			parallel {
				stage("verify (next)") {
					agent {
						label 'data'
					}
					options { timeout(time: 30, unit: 'MINUTES') }
					environment {
						ARTIFACTORY = credentials("${p['artifactory.credentials']}")
					}
					steps {
						script {
							docker.image(p['docker.java.next.image']).inside(p['docker.java.inside.basic']) {
								sh 'MAVEN_OPTS="-Duser.name=' + "${p['jenkins.user.name']}" + ' -Duser.home=/tmp/jenkins-home" ' +
									"./mvnw -Pwith-bom-client verify -B -U"
							}
						}
					}
				}
			}
		}

		stage('Build project and release to artifactory') {
			when {
				beforeAgent(true)
				anyOf {
					branch(pattern: "main|(\\d+\\.\\d+\\.x)", comparator: "REGEXP")
					not { triggeredBy 'UpstreamCause' }
				}
			}
			agent {
				label 'data'
			}
			options { timeout(time: 20, unit: 'MINUTES') }

			environment {
				ARTIFACTORY = credentials("${p['artifactory.credentials']}")
			}

			steps {
				script {
					docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
						sh 'MAVEN_OPTS="-Duser.name=' + "${p['jenkins.user.name']}" + ' -Duser.home=/tmp/jenkins-home" ' +
								"./mvnw -Pci,artifactory " +
								"-Dartifactory.server=${p['artifactory.url']} " +
								"-Dartifactory.username=${ARTIFACTORY_USR} " +
								"-Dartifactory.password=${ARTIFACTORY_PSW} " +
								"-Dartifactory.staging-repository=${p['artifactory.repository.snapshot']} " +
								"-Dartifactory.build-name=spring-data-bom " +
								"-Dartifactory.build-number=${BUILD_NUMBER} " +
								"-Dmaven.test.skip=true clean deploy -U -B"
					}
				}
			}
		}
	}

	post {
		changed {
			script {
				slackSend(
						color: (currentBuild.currentResult == 'SUCCESS') ? 'good' : 'danger',
						channel: '#spring-data-dev',
						message: "${currentBuild.fullDisplayName} - `${currentBuild.currentResult}`\n${env.BUILD_URL}")
				emailext(
						subject: "[${currentBuild.fullDisplayName}] ${currentBuild.currentResult}",
						mimeType: 'text/html',
						recipientProviders: [[$class: 'CulpritsRecipientProvider'], [$class: 'RequesterRecipientProvider']],
						body: "<a href=\"${env.BUILD_URL}\">${currentBuild.fullDisplayName} is reported as ${currentBuild.currentResult}</a>")
			}
		}
	}
}
