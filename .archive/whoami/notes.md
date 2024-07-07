job "whoami" {
  datacenters = ["dc-01"]
  type = "service"

  group "whoami" {
    count = 1
    network {
      mode = "host"
      port "whoami_http" {}
    }
    service {
      name = "whoami"
      port = "whoami_http"
      provider = "nomad"
      tags = [
        "traefik.enable=true",
        "traefik.http.routers.whoami.rule=Host(`whoami.test.net`)",
        "traefik.http.routers.whoami.tls=true",
      ]
    }
    task "whoami" {
      env {
        WHOAMI_PORT_NUMBER = "${NOMAD_PORT_whoami_http}"
      }
      driver = "docker"
      config {
        image = "docker.io/traefik/whoami:latest"
        network_mode = "host"
        ports = ["whoami_http"]
        #network_mode = "loopback"
        #args = ["-port", "${NOMAD_PORT_web}"]
      }
    }
  }
}