def label = "ImageBuildPod-${UUID.randomUUID().toString()}"

properties([
  parameters([
    string(name: 'mvn_version', defaultValue: '3.6.3'),
    string(name: 'jdk_version', defaultValue: '8'),
    string(name: 'node_version', defaultValue: '15.4.0'),
    string(name: 'klar_version', defaultValue: '2.4.0')
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

      stage('Build') {
        ansiColor('xterm') {
          // Since the Dockerfile needs network connectivity, connect to the k8s sidecar with --network: https://stackoverflow.com/a/49408621/64217
          image = docker.build("steampunkfoundry/mvn-jdk-node:${env.BUILD_ID}", "--network container:\$(docker ps | grep \$(hostname) | grep k8s_POD | cut -d\" \" -f1) --build-arg mvn_version=${params.mvn_version} --build-arg jdk_version=${params.jdk_version} --build-arg node_version=${params.node_version} mvn-jdk-node")
          image.tag("mvn${mvn_version}-openjdk${jdk_version}-node${node_version}")
          image.tag("latest")
        }
      }

      stage('Push') {
        docker.withRegistry("https://registry.hub.docker.com", "ggotimer-docker-hub") {
          image.push("${env.BUILD_ID}")
          image.push("latest")
        }
      }

      stage('Scan') {
        ansiColor('xterm') {
          sh "wget -nv https://github.com/optiopay/klar/releases/download/v${klar_version}/klar-${klar_version}-linux-amd64 -O ./klar && chmod +x ./klar"
          sh "CLAIR_ADDR=ip-10-0-3-252.ec2.internal:30618 FORMAT_OUTPUT=table CLAIR_OUTPUT=Medium ./klar steampunkfoundry/mvn-jdk-node:${env.BUILD_ID} || true"
          sh "CLAIR_ADDR=ip-10-0-3-252.ec2.internal:30618 FORMAT_OUTPUT=json CLAIR_OUTPUT=Medium ./klar steampunkfoundry/mvn-jdk-node:${env.BUILD_ID} > klar-steampunkfoundry-mvn-jdk-node-${env.BUILD_ID}.json || true"
          archiveArtifacts artifacts: 'klar-*.json', onlyIfSuccessful: false
        }
      }

      stage('Publish') {
        docker.withRegistry("https://registry.hub.docker.com", "ggotimer-docker-hub") {
          image.push("mvn${mvn_version}-openjdk${jdk_version}-node${node_version}")
        }
      }
    }
  }
}
