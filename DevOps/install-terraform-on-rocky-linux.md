# Install And Config Terraform On Rocky Linux 9

要在 Rocky Linux 9 下安裝 Terraform 請依據下方步驟進行安裝

※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限

## 安裝步驟

1. 安裝 yum-utils

    ```console
    $ sudo yum install yum-utils -y
    ```

2. 加入軟體庫

    ```console
    $ sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
    ```

3. 安裝 Terraform

    ```console
    $ sudo dnf install terraform -y
    ```

## 使用

以下的設定為針對個別專案，

1. 定義供應商 (Provider): 先建立 `provider.tf`

    ※ 若不知道 Kubernetes 的 IP 可以使用指令 `kubectl config view` 檢視

    ```terraform
    terraform {
        required_version = ">= 1.3"
        required_providers {
            kubernetes = {
                source = "hashicorp/kubernetes"
                version = ">= 2.17.0"
            }
        }
        backend "local" {
            path = "/tmp/terraform.tfstate"
        }
    }

    provider "kubernetes" {
        host = "<你的 Kubernetes IP>"
        insecure = true
        config_path = "~/.kube/config"
    }
    ```

2. 初始化 Terraform: 執行指令 `terraform init -auto-approve`

    ※ 當同時使用 `-auto-approve` 參數時，Terraform 執行前就不會在與操作者確認

3. 依據原本部署到 K8s 上的設定檔 (deployment.yaml) 建立 `deployment.tf`

    ※ 這邊建議可以使用 VSCode + HashiCorp Terraform 套件進行設定檔的撰寫，以下為其中一個範例

    ```terraform
    resource "kubernetes_deployment" "demostration" {
      metadata {
        name = "demostration"
        labels = {
          app = "demostration"
        }
        namespace = "test"
      }
      spec {
        replicas = 1
        selector {
          match_labels = {
            app = "demostration"
          }
        }
        template {
          metadata {
            labels = {
              app = "demostration"
            }
          }
          spec {
            container {
              name = "demostration"
              image = "localhost:5000/demostration:latest"
              resources {
                requests = {
                  memory = "512Mi"
                  cpu = "250m"
                }
                limits = {
                  memory = "512Mi"
                  cpu = "250m"
                }
              }
              port {
                container_port = 80
              }
            }
          }
        }
      }
    }

    resource "kubernetes_service" "demostration" {
      metadata {
        name = "demostration"
        namespace = "test"
      }
      spec {
        selector = {
            app = "demostration"
        }
        type = "ClusterIP"
        port {
          port = 80
          target_port = 80
        }
      }
    }

    resource "kubernetes_ingress_v1" "demostration" {
      metadata {
        name = "demostration"
        namespace = "test"
        annotations = {
          "kubernetes.io/ingress.class" = "nginx"
          "nginx.ingress.kubernetes.io/ssl-redirect" = "true"
          "nginx.ingress.kubernetes.io/use-regex" = "true"
          "nginx.ingress.kubernetes.io/rewrite-target" = "/$1"
          "nginx.ingress.kubernetes.io/body-size" = "102400m"
          "nginx.ingress.kubernetes.io/proxy-body-size" = "102400m"
          "nginx.ingress.kubernetes.io/proxy-connect-timeout" = "7200"
          "nginx.ingress.kubernetes.io/proxy-read-timeout" = "7200"
          "nginx.ingress.kubernetes.io/proxy-send-timeout" = "7200"
          "nginx.ingress.kubernetes.io/proxy-max-temp-file-size" = "0"
          "nginx.ingress.kubernetes.io/proxy-buffering" = "off"
          "nginx.ingress.kubernetes.io/proxy_max_temp_file_size" = "102400m"
          "nginx.ingress.kubernetes.io/large-client-header-buffers" = "8 52m"
          "nginx.ingress.kubernetes.io/client-header-buffer-size" = "52m"
          "nginx.ingress.kubernetes.io/client-body-buffer-size" = "102400m"
          "nginx.ingress.kubernetes.io/client-max-body-size" = "102400m"
          "nginx.ingress.kubernetes.io/client_body_timeout" = "7200"
          "nginx.org/client-max-body-size" = "102400m"
          "nginx.org/websocket-services" = "core-service"
        }
      }
      spec {
        rule {
          http {
            path {
              path = "/test/demostration/(.*)"
              path_type = "Prefix"
              backend {
                service {
                  name = "demostration"
                  port {
                    number = 80
                  }
                }
              }
            }
          }
        }
      }
    }
    ```

4. 執行 `terraform apply -auto-approve` 進行部署

    ※ 當同時使用 `-auto-approve` 參數時，Terraform 執行前就不會在與操作者確認

5. 完成

## 參考資料

- [[Terraform] 入門學習筆記](https://godleon.github.io/blog/DevOps/terraform-getting-started/)
- [Getting Started with Kubernetes provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/guides/getting-started)
- [Install Terraform on CentOS 8 / Rocky Linux 8](https://computingforgeeks.com/how-to-install-terraform-on-centos-linux/)
- [Version Constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)
- [Deploying on Kubernetes by using Terraform](https://www.youtube.com/watch?v=eCHwm2l-GR8)
- [kubernetes - HashiCorp Registry](https://registry.terraform.io/providers/hashicorp/kubernetes/latest)
- [Kubernetes Provider](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs#ignore-kubernetes-annotations-and-labels)
- [kubernetes_service](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs/resources/service)
- [Failed to create Ingress because: the server could not find the requested resource](https://github.com/hashicorp/terraform-provider-kubernetes/issues/1386#issuecomment-979352830)
- [Documentation: Prefer "terraform" to "hcl" in Markdown code fences](https://github.com/hashicorp/terraform-provider-aws/issues/17810)
