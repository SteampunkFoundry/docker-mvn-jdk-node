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
    nodeSelector: 'role=workers'
) {
  node(label) {
    def image

    stage('Checkout Code') {
      cleanWs()
      checkout scm
    }

    stage('Build'){
      image = docker.build("steampunkfoundry/mvn-jdk-node:mvn${mvn_version}-openjdk${jdk_version}-node${node_version}", "--build-arg mvn_version=${params.mvn_version} --build-arg jdk_version=${params.jdk_version} --build-arg node_version=${params.node_version} mvn-jdk-node")
    }
  }
}
