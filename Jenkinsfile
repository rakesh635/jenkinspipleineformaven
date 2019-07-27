podTemplate(label: 'mypod', containers: [
    containerTemplate(name: 'git', image: 'alpine/git', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'maven', image: 'maven:3.3.9-jdk-8-alpine', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true)
],
volumes: [
	hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]
)
{
    node('mypod') {
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
                sh 'git clone -b master https://github.com/lvthillo/hello-world-war.git'
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
        stage('Archieve Artifact and Upload to jfrog') {
            container('maven') {
                dir('hello-world-war/') {
                    archiveArtifacts 'target/hello-world-war-1.0.0.war'
                    sh 'curl -uadmin:AP4jYfSHAhSG4H13n4mSwtgk7Hu -T target/hello-world-war-1.0.0.war "http://34.93.216.255/artifactory/example-repo-local/helloworld/"'
                }
            }
        }
    }
}
