podTemplate(label: 'mypod1', containers: [
    containerTemplate(name: 'git', image: 'alpine/git', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'k3integrations/kubectl', command: '', ttyEnabled: true),
    containerTemplate(name: 'tomcat8', image: 'tomcat:8.0', command: '', ttyEnabled: true)
],
volumes: [
	hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]
)
{
    node('mypod1') {
        stage('Check running containers') {
            container('docker') {
                // example to show you can run docker commands when you mount the socket
                sh 'hostname'
                sh 'hostname -i'
                sh 'docker ps'
            }
        }
        
        stage('Clone repository') {
            container('git') {
                sh 'whoami'
                sh 'hostname -i'
                sh 'git clone -b master https://github.com/rakesh635/hello-world-war.git'
            }
        }

        stage('Maven Build') {
            container('maven') {
                dir('hello-world-war/') {
                    sh 'hostname'
                    sh 'hostname -i'
                    sh 'mvn clean install'
                }
            }
        }
        stage('Artifact Upload') {
            container('maven') {
    			rtUpload (
    				serverId: "art1",
    				buildName: 'Helloworld',
                    buildNumber: env.BUILD_NUMBER,
    				spec:
    					"""{
    					  "files": [
    						{
    						  "pattern": "/home/jenkins/workspace/testproj3/hello-world-war/target/*.war",
    						  "target": "mavenlocal/"
    						}
    					 ]
    					}"""
    			)
            }
		}
		stage('Deploy it in Tomcat conatiner')
		{
		    container('tomcat8') {
		        sh 'mkdir test'
		        sh 'chmod 777 -R test/'
		        rtDownload (
                    serverId: "art1",
                    spec:
                        """{
                          "files": [
                            {
                              "pattern": "mavenlocal/*.war",
                              "target": "test/hello.war"
                            }
                         ]
                        }"""
                )
                sh 'cp -a test/. /usr/local/tomcat/webapps/'
                sh 'chmod 775 /usr/local/tomcat/webapps/*.war'
                sh 'chown root:root /usr/local/tomcat/webapps/*.war'
		    }
		}
		stage('Prepare and Push docker image to registry(dockerhub)') {
            container('docker') {
                sh '''
                contid=`docker ps --filter ancestor=tomcat:8.0 --quiet`
                echo $contid
                docker commit $contid rakesh635/testhelloworld:$BUILD_NUMBER
                '''
                docker.withRegistry('', 'dockerlogin') {
                    sh 'docker push rakesh635/testhelloworld:$BUILD_NUMBER'
                }
            }
        }
        stage('kubectl')
	{
            container('kubectl') {
		withKubeConfig([credentialsId: 'GKEcluster',
                    serverUrl: 'https://34.93.78.217',
                    contextName: 'qaenv',
                    clusterName: 'qaenv',
                    namespace: 'qadeploy'
                    ]) {
                    sh 'kubectl get pods'
		    sh 'kubectl apply -f kubernetes'
                    }
	    }
	}
    }
}
