def label = "ImageBuildPod-${UUID.randomUUID().toString()}"

properties([
  parameters([
    string(name: 'mvn_version', defaultValue: '3.6.3'),
    string(name: 'jdk_version', defaultValue: '8'),
    string(name: 'node_version', defaultValue: '15.4.0')
  ])
])

podTemplate(
  label: label,
  containers: [
    containerTemplate(name: 'docker',
                      image: 'docker:19.03',
                      ttyEnabled: true,
                      command: 'cat',
                      envVars: [containerEnvVar(key: 'DOCKER_HOST', value: "unix:///var/run/docker.sock")],
                      privileged: true)
  ],
  volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')],
  nodeSelector: 'role=infra'
) {
  node(label) {
    container('docker') {
      def image

      stage('Checkout Code') {
        cleanWs()
        checkout scm
      }

      stage('Build'){
        ansiColor('xterm') {
          // Since the Dockerfile needs network connectivity, connect to the sidecar with --network: https://stackoverflow.com/a/49408621/64217
          image = docker.build("steampunkfoundry/mvn-jdk-node:${env.BUILD_ID}", "--network container:\$(docker ps | grep \$(hostname) | grep k8s_POD | cut -d\" \" -f1) --build-arg mvn_version=${params.mvn_version} --build-arg jdk_version=${params.jdk_version} --build-arg node_version=${params.node_version} mvn-jdk-node")
          image.tag("mvn${mvn_version}-openjdk${jdk_version}-node${node_version}")
        }
      }

      stage('Push'){
        docker.withRegistry("https://registry.hub.docker.com", "ggotimer-docker-hub") {
          image.push()
        }
      }
    }
  }
}
