def p = [:]
node {
	checkout scm
	p = readProperties interpolate: true, file: 'ci/pipeline.properties'
}

pipeline {
	agent none

	triggers {
		pollSCM 'H/10 * * * *'
		upstream(upstreamProjects: "spring-data-commons/main,spring-data-cassandra/main,spring-data-couchbase/main,spring-data-elasticsearch/main," +
			"spring-data-rest/main,spring-data-jpa/main,spring-data-ldap/main,spring-data-mongodb/main,spring-data-neo4j/main,spring-data-redis/main", threshold: hudson.model.Result.SUCCESS)
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
				DEVELOCITY_CACHE = credentials("${p['develocity.cache.credentials']}")
				DEVELOCITY_ACCESS_KEY = credentials("${p['develocity.access-key']}")
			}

			steps {
				script {
					docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
						sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ' +
										'DEVELOCITY_CACHE_USERNAME=${DEVELOCITY_CACHE_USR} ' +
										'DEVELOCITY_CACHE_PASSWORD=${DEVELOCITY_CACHE_PSW} ' +
										'GRADLE_ENTERPRISE_ACCESS_KEY=${DEVELOCITY_ACCESS_KEY} ' +
										'./mvnw -Pwith-bom-client verify -B -U'
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
						DEVELOCITY_CACHE = credentials("${p['develocity.cache.credentials']}")
						DEVELOCITY_ACCESS_KEY = credentials("${p['develocity.access-key']}")
					}
					steps {
						script {
							docker.image(p['docker.java.next.image']).inside(p['docker.java.inside.basic']) {
								sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ' +
												'DEVELOCITY_CACHE_USERNAME=${DEVELOCITY_CACHE_USR} ' +
												'DEVELOCITY_CACHE_PASSWORD=${DEVELOCITY_CACHE_PSW} ' +
												'GRADLE_ENTERPRISE_ACCESS_KEY=${DEVELOCITY_ACCESS_KEY} ' +
								 				'./mvnw -Pwith-bom-client verify -B -U'
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
				DEVELOCITY_CACHE = credentials("${p['develocity.cache.credentials']}")
				DEVELOCITY_ACCESS_KEY = credentials("${p['develocity.access-key']}")
			}

			steps {
				script {
					docker.image(p['docker.java.main.image']).inside(p['docker.java.inside.basic']) {
						sh 'MAVEN_OPTS="-Duser.name=jenkins -Duser.home=/tmp/jenkins-home" ' +
								'DEVELOCITY_CACHE_USERNAME=${DEVELOCITY_CACHE_USR} ' +
								'DEVELOCITY_CACHE_PASSWORD=${DEVELOCITY_CACHE_PSW} ' +
								'GRADLE_ENTERPRISE_ACCESS_KEY=${DEVELOCITY_ACCESS_KEY} ' +
								'./mvnw -Partifactory ' +
								'-Dartifactory.server=https://repo.spring.io ' +
								"-Dartifactory.username=${ARTIFACTORY_USR} " +
								"-Dartifactory.password=${ARTIFACTORY_PSW} " +
								"-Dartifactory.staging-repository=libs-snapshot-local " +
								"-Dartifactory.build-name=spring-data-bom " +
								"-Dartifactory.build-number=${BUILD_NUMBER} " +
								'-Dmaven.test.skip=true clean deploy -B -U'
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
