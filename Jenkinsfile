stage 'build'
	node{
		git credentialsId: '4322a74e-71ab-4669-b16e-6315c017229a', url: 'https://github.com/emcconne/mobile-deposit-ui'
		sh 'mvn -DskipTests clean package'
		stash name: 'source', excludes: 'target/'
		archive includes: 'target/*.war'
	}
stage 'test[unit&quality]'
	parallel \
	'unit-test': {
		node {
			unstash 'source'
			sh 'mvn -Dmaven.test.failure.ignore=true test'
			step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
		}
		if(currentBuild.result == 'UNSTABLE'){
			error "Unit test failures"
		}
	}, 
	'quality-test': {
		node {
			unstash 'source'
			sh 'mvn sonar:sonar'
		}
	}