def label = "ImageBuildPod-${UUID.randomUUID().toString()}"

properties([
  parameters([
    string(name: 'mvn_version', defaultValue: '3.6.3'),
    string(name: 'jdk_version', defaultValue: '8'),
    string(name: 'node_version', defaultValue: '15.4.0'),
    choice(name: 'clair_output', choices: ['Medium', 'Unknown', 'Negligible', 'Low', 'High', 'Critical', 'Defcon1'],
           description: 'Minimum Clair severity?')
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
    withEnv(["CLAIR_ADDR=${env.CLAIR_ADDR}"]) {
      container('docker') {
        def image
        def branch_tag = env.BRANCH_NAME.replaceAll("/","-").replaceAll("[^-a-zA-Z0-9]","").take(32).toLowerCase();
        def build_tag = "${branch_tag}-${env.BUILD_ID}"

        stage('Checkout Code') {
          cleanWs()
          checkout scm
        }

        stage('Build') {
          ansiColor('xterm') {
            // Since the Dockerfile needs network connectivity, connect to the k8s sidecar with --network: https://stackoverflow.com/a/49408621/64217
            image = docker.build("steampunkfoundry/mvn-jdk-node:${build_tag}", "--network container:\$(docker ps | grep \$(hostname) | grep k8s_POD | cut -d\" \" -f1) --build-arg mvn_version=${params.mvn_version} --build-arg jdk_version=${params.jdk_version} --build-arg node_version=${params.node_version} mvn-jdk-node")
            image.tag("mvn${mvn_version}-openjdk${jdk_version}-node${node_version}")
            image.tag("latest")
          }
        }

        stage('Push') {
          docker.withRegistry('https://container.dhsice.name', 'nexuslogin') {
            image.push(build_tag)
            image.push("latest")
          }
        }

        stage('Scan') {
          withCredentials([usernamePassword(credentialsId: 'nexuslogin', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
            ansiColor('xterm') {
              sh "wget -nv ${env.KLAR_URL} -O ./klar && chmod +x ./klar"
              sh "CLAIR_OUTPUT=${env.clair_output} FORMAT_OUTPUT=table DOCKER_USER=${DOCKER_USER} DOCKER_PASSWORD=${DOCKER_PASSWORD} ./klar container.dhsice.name/steampunkfoundry/mvn-jdk-node:${build_tag} | tee klar-steampunkfoundry-mvn-jdk-node-${build_tag}.txt || true"
              sh "CLAIR_OUTPUT=${env.clair_output} FORMAT_OUTPUT=json DOCKER_USER=${DOCKER_USER} DOCKER_PASSWORD=${DOCKER_PASSWORD} ./klar container.dhsice.name/steampunkfoundry/mvn-jdk-node:${build_tag} > klar-steampunkfoundry-mvn-jdk-node-${build_tag}.json || true"
              archiveArtifacts artifacts: 'klar-steampunkfoundry-mvn-jdk-node*.txt', onlyIfSuccessful: false
              archiveArtifacts artifacts: 'klar-steampunkfoundry-mvn-jdk-node*.json', onlyIfSuccessful: false
            }
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
}
