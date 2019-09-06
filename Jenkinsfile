properties([pipelineTriggers([githubPush()])])
podTemplate(label: 'mypod1', containers: [
    containerTemplate(name: 'git', image: 'alpine/git', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'roffe/kubectl:v1.13.2', command: '', ttyEnabled: true),
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
                git 'https://github.com/rakesh635/hello-world-war.git'
                // example to show you can run docker commands when you mount the socket
                sh 'hostname'
                sh 'hostname -i'
                sh 'docker images'
                env.deltag = env.BUILD_NUMBER-2
                sh 'echo $deltag'
            }
        }
        stage('Clone repository') {
            container('git') {
                sh 'whoami'
                sh 'hostname -i'
                sh 'git clone -b master https://github.com/rakesh635/hello-world-war.git'
                dir('hello-world-war/') {
                    def datas = readYaml file: 'cicdconfig.yml'
                    //assert datas.something == 'Override'
                    env.APPNAME = datas.parameter.appname
                    env.DEPLOYMENTSTRATEGY = datas.parameter.deployment.strategy
                    env.NAMEPREFIX = ''
                    if (env.DEPLOYMENTSTRATEGY == 'canary') {
                        env.NAMEPREFIX = '-canary'
                    } else if (env.DEPLOYMENTSTRATEGY == 'bluegreen') {
                        env.NAMEPREFIX = '-bg'
                    }
                }
                sh 'git clone -b master https://github.com/rakesh635/jenkinspipleineformaven.git'
                dir('jenkinspipleineformaven/') {
                    sh "sed -i 's/{{BUILDNUMBER}}/$BUILD_NUMBER/g' kubernetesfiles/deployment"+env.NAMEPREFIX+".yaml"
                    sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/deployment"+env.NAMEPREFIX+".yaml"
                    sh "sed -i 's/{{NAMEPREFIX}}/$NAMEPREFIX/g' kubernetesfiles/deployment"+env.NAMEPREFIX+".yaml"
                    sh 'cat kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                    sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/service"+env.NAMEPREFIX+".yaml"
                    sh "sed -i 's/{{BUILDNUMBER}}/$BUILD_NUMBER/g' kubernetesfiles/service"+env.NAMEPREFIX+".yaml"
                    sh 'cat kubernetesfiles/service'+env.NAMEPREFIX+'.yaml'
			    }
            }
        }
        stage('Maven validate compile test') {
            container('maven') {
                dir('hello-world-war/') {
                    sh 'mvn validate compile test'
                }
            }
        }
        stage('Maven package verify') {
            container('maven') {
                dir('hello-world-war/') {
                    sh 'mvn package verify'
                }
            }
        }
        stage('Artifact Upload') {
            container('maven') {
    			rtUpload (
    				serverId: "art1",
    				buildName: params.APPNAME,
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
                              "target": "test/"""+env.APPNAME+""".war"
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
                docker images
                docker image prune -a --force
                docker images
                contid=`docker ps --filter "label=io.kubernetes.container.name=tomcat8" --quiet`
                echo $contid
                docker commit $contid rakesh635/$APPNAME:$BUILD_NUMBER
                docker tag rakesh635/$APPNAME:$BUILD_NUMBER rakesh635/$APPNAME:latest
                '''
                docker.withRegistry('', 'dockerlogin') {
                    sh 'docker push rakesh635/$APPNAME:$BUILD_NUMBER'
                    sh 'docker push rakesh635/$APPNAME:latest'
                }
            }
        }
        stage('Initiate Deployment to Kubernetes cluster')
		{
		    container('kubectl') {
		        withKubeConfig([credentialsId: 'GKEcluster',
                    serverUrl: 'https://34.93.78.217',
                    contextName: 'qaenv',
                    clusterName: 'qaenv',
                    namespace: 'qadeploy'
                    ]) {
                        dir('jenkinspipleineformaven/') {
                    	    sh 'kubectl apply -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                    	    sh 'kubectl apply -f kubernetesfiles/service'+env.NAMEPREFIX+'.yaml'
			            }
                    }
		    }
		}
		try {
    		stage("Deployment")
    		{
    		    container('kubectl') {
    		        withKubeConfig([credentialsId: 'GKEcluster',
                        serverUrl: 'https://34.93.78.217',
                        contextName: 'qaenv',
                        clusterName: 'qaenv',
                        namespace: 'qadeploy'
                        ]) {
                            sh 'kubectl rollout status deployment.v1.apps/'+env.APPNAME+env.NAMEPREFIX+''
                		    for (int i = 0; i < 10; i++) {
                                ext_ip = sh (
                                    script: 'kubectl describe service '+env.APPNAME+'-loadbalancer | grep "LoadBalancer Ingress" | cut -d":" -f2 | sed -e "s/^[ \t]*//" ',
                                    returnStdout: true
                                ).trim()  
                                if(ext_ip != '') {break;}
                            }
                            if(ext_ip != '') {
                                echo "Appln URL : http://${ext_ip}:8080/"+env.APPNAME
                            } else {
                                sh 'exit -1'
                            }
                        }
    		    }
    		}
			stage("Deployed URL")
			{
				echo "Appln URL : http://${ext_ip}:8080/"+env.APPNAME 
			}
		} catch (Exception err) {
            echo "Some issue with updating new version; so rolling back to previous version"
            stage("RollBack") {
                
                container('kubectl') {
    		        withKubeConfig([credentialsId: 'GKEcluster',
                        serverUrl: 'https://34.93.78.217',
                        contextName: 'qaenv',
                        clusterName: 'qaenv',
                        namespace: 'qadeploy'
                        ]) {
                            dir('jenkinspipleineformaven/') {
                		        sh 'kubectl rollout undo -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                            }
                        }
    		    }
            }
        }
        if (env.DEPLOYMENTSTRATEGY == 'bluegreen') {
            try {
                stage("Traffic to green deployment") {
                    timeout(time: 30, unit: 'SECONDS') {
                        script {
                            def STOPGREEN = input message: 'Want to stop redirecting traffic to green deployment', ok: 'Next',
                                            parameters: [
                                            choice(id: 'STOPGREEN', name: 'STOPGREEN', choices: ['No','Yes'].join('\n'), description: 'Approve to roll out application completely to newer version')]
                            
                            print STOPGREEN
                            env.STOPGREEN = STOPGREEN
                        }
                    }
                }
            } catch (Exception err) {
                stage("Traffic routed to green and remove blue deployment")  
                {
                    container('kubectl') {
        		        withKubeConfig([credentialsId: 'GKEcluster',
                            serverUrl: 'https://34.93.78.217',
                            contextName: 'qaenv',
                            clusterName: 'qaenv',
                            namespace: 'qadeploy'
                            ]) {
                                dir('jenkinspipleineformaven/') {
                                    sh 'kubectl apply -f kubernetesfiles/service'+env.NAMEPREFIX+'.yaml'
                                    sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/deployment.yaml"
                                    sh "sed -i 's/{{NAMEPREFIX}}//g' kubernetesfiles/deployment.yaml"
                                    sh 'kubectl delete -f kubernetesfiles/deployment.yaml'
                                    sh "sed -i 's/{{BUILDNUMBER}}/$BUILD_NUMBER/g' kubernetesfiles/deployment.yaml"
                                    sh 'cat kubernetesfiles/deployment.yaml'
                    		        sh 'kubectl apply -f kubernetesfiles/deployment.yaml'
                    		        sh 'kubectl rollout status deployment.v1.apps/'+env.APPNAME+''
                    		        sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/service.yaml"
                    		        sh 'kubectl apply -f kubernetesfiles/service.yaml'
                    		        sh 'kubectl delete -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                    		        print "Traffic successfully routed to green deployment"
                                    //return
                                }
                            }
        		    }
                }
            }
            if (env.STOPGREEN == "Yes" || env.STOPGREEN == "No" ) {
                if(env.STOPGREEN == "No") {
                    stage("Traffic routed to green and remove blue deployment")  
                    {
                        container('kubectl') {
            		        withKubeConfig([credentialsId: 'GKEcluster',
                                serverUrl: 'https://34.93.78.217',
                                contextName: 'qaenv',
                                clusterName: 'qaenv',
                                namespace: 'qadeploy'
                                ]) {
                                    dir('jenkinspipleineformaven_new/') {
                                        sh 'kubectl apply -f kubernetesfiles/service'+env.NAMEPREFIX+'.yaml'
                                        sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/deployment.yaml"
                                        sh "sed -i 's/{{NAMEPREFIX}}//g' kubernetesfiles/deployment.yaml"
                                        sh 'kubectl delete -f kubernetesfiles/deployment.yaml'
                                        sh "sed -i 's/{{BUILDNUMBER}}/18/g' kubernetesfiles/deployment.yaml"
                                        sh 'cat kubernetesfiles/deployment.yaml'
                        		        sh 'kubectl apply -f kubernetesfiles/deployment.yaml'
                        		        sh 'kubectl rollout status deployment.v1.apps/'+env.APPNAME+''
                        		        sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/service.yaml"
                        		        sh 'kubectl apply -f kubernetesfiles/service.yaml'
                        		        sh 'kubectl delete -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                        		        print "Traffic successfully routed to green deployment"
                                        //return
                                    }
                                }
            		    }
                    }
                }
                if(env.STOPGREEN == "Yes") {
                    stage("Traffic remains to blue and remove green deployment")  
                    {
                        container('kubectl') {
            		        withKubeConfig([credentialsId: 'GKEcluster',
                                serverUrl: 'https://34.93.78.217',
                                contextName: 'qaenv',
                                clusterName: 'qaenv',
                                namespace: 'qadeploy'
                                ]) {
                                    dir('jenkinspipleineformaven_new/') {
                                        sh 'kubectl delete -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                                    }
                                }
                        }        
                    }
                }
            }
        }
        if (env.DEPLOYMENTSTRATEGY == 'canary') {
            try {
                stage("Canary Analysis in 30 Seconds") {
                    timeout(time: 300, unit: 'SECONDS') {
                        script {
                            def CANARYAPPROVAL = input message: 'Please Provide Parameters', ok: 'Next',
                                            parameters: [
                                            choice(id: 'CANARYAPPROVAL', name: 'CANARYAPPROVAL', choices: ['Yes','No'].join('\n'), description: 'Approve to roll out application completely to newer version')]
                            
                            print CANARYAPPROVAL
                            env.CANARYAPPROVAL = CANARYAPPROVAL
                        }
                    }
                }
            } catch (Exception err) {
                stage("Rollback canary due to no input from approver")  
                {
                    container('kubectl') {
        		        withKubeConfig([credentialsId: 'GKEcluster',
                            serverUrl: 'https://34.93.78.217',
                            contextName: 'qaenv',
                            clusterName: 'qaenv',
                            namespace: 'qadeploy'
                            ]) {
                                dir('jenkinspipleineformaven/') {
                    		        sh 'kubectl delete -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                    		        print "Canary Deployment deleted, as no input from approver"
                                    //return
                                }
                            }
        		    }
                }
            }
            if (env.CANARYAPPROVAL == "Yes" || env.CANARYAPPROVAL == "No" ) {
                stage("Rollout canary and Update application completely")
                {
                    container('kubectl') {
        		        withKubeConfig([credentialsId: 'GKEcluster',
                            serverUrl: 'https://34.93.78.217',
                            contextName: 'qaenv',
                            clusterName: 'qaenv',
                            namespace: 'qadeploy'
                            ]) {
                                dir('jenkinspipleineformaven/') {
                                    if (env.CANARYAPPROVAL == "Yes") {
                                        sh "sed -i 's/{{BUILDNUMBER}}/$BUILD_NUMBER/g' kubernetesfiles/deployment.yaml"
                                        sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/deployment.yaml"
                                        sh "sed -i 's/{{NAMEPREFIX}}//g' kubernetesfiles/deployment.yaml"
                                        sh "sed -i 's/{{APPNAME}}/$APPNAME/g' kubernetesfiles/service.yaml"
                                        sh 'cat kubernetesfiles/deployment.yaml'
                                        sh 'kubectl apply -f kubernetesfiles/deployment.yaml'
                                        sh 'kubectl rollout status deployment.v1.apps/'+env.APPNAME+''
                            	        sh 'kubectl apply -f kubernetesfiles/service.yaml'
                            	        sh 'cat kubernetesfiles/deployment.yaml'
                                    }
                                    if (env.CANARYAPPROVAL == "No") {
                                        sh "echo 'Canary deployment is cancelled and old version of application in retained'"   
                                    }
                        	        sh 'kubectl delete -f kubernetesfiles/deployment'+env.NAMEPREFIX+'.yaml'
                			    }
                            }
        		    }
                }
            }
        }
    }
}
