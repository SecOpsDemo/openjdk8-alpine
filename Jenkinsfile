def SERVICE_GROUP = "openjdk8"
def SERVICE_NAME = "alpine"
def IMAGE_NAME = "${SERVICE_GROUP}-${SERVICE_NAME}"
def REPOSITORY_URL = "https://github.com/SecOpsDemo/openjdk8-alpine.git"
def REPOSITORY_SECRET = ""
def SLACK_TOKEN_DEV = ""
def SLACK_TOKEN_DQA = ""

@Library("github.com/opsnow-tools/valve-butler")
def butler = new com.opsnow.valve.v9.Butler()
def label = "worker-${UUID.randomUUID().toString()}"

properties([
  buildDiscarder(logRotator(daysToKeepStr: "60", numToKeepStr: "30"))
])
podTemplate(label: label, containers: [
  containerTemplate(name: "builder", image: "opsnowtools/valve-builder:v0.2.40", command: "cat", ttyEnabled: true, alwaysPullImage: true)
], volumes: [
//   hostPathVolume(mountPath: "/var/run/docker.sock", hostPath: "/var/run/docker.sock"),
//   hostPathVolume(mountPath: "/home/jenkins/.draft", hostPath: "/home/jenkins/.draft"),
//   hostPathVolume(mountPath: "/home/jenkins/.helm", hostPath: "/home/jenkins/.helm")
]) {
  node(label) {
    stage("Prepare") {
      container("builder") {
        butler.prepare(IMAGE_NAME)
      }
    }
    stage("Checkout") {
      container("builder") {
        try {
          if (REPOSITORY_SECRET) {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME, credentialsId: REPOSITORY_SECRET)
          } else {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME)
          }
        } catch (e) {
          butler.failure(SLACK_TOKEN_DEV, "Checkout")
          throw e
        }

        butler.scan("docker")
      }
    }
    if (BRANCH_NAME == "master") {
      stage("Build Docker") {
        container("builder") {
          try {
            butler.build_docker()
          } catch (e) {
            butler.failure(SLACK_TOKEN_DEV, "Build Docker")
            throw e
         }
        }
      }
      stage('Scan Image via PrismaCloud') {
        try {
        butler.scan_image()
        } catch (e) {
        butler.failure(SLACK_TOKEN_DEV, "Scan Image")
        throw e
        }
      }
      stage("Push Docker") {
        container("builder") {
          try {
            butler.push_docker("harbor", "harbor.soc1.bespin-mss.com", "secops", "harborbot")
          } catch (e) {
            butler.failure(SLACK_TOKEN_DEV, "Push Docker")
            throw e
         }
        }
      }
    }
  }
}