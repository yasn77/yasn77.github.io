# Part I

---

# Sharing Secrets outside of Kubernetes
### Using Hashicorp Vault and Terraform

---

## Why It's Needed

  * Not everything lives in Kubernetes <i class="fa fa-thumbs-o-down"></i>
  * Use a generic credentials store <!-- .element: class="fragment" data-fragment-index="1" -->
  * Vault opens the door to dynamic credentials <!-- .element: class="fragment" data-fragment-index="2" -->

Notes:
 * Introduce idea that we should try to be able to reproduce environments, that includes any static secrets.
 * Vault gives greater flexibility when it comes to managing secrets, such as a full PKI system

---

## Concourse CI use case
##### https://concourse-ci.org/
![Concourse Logo](https://concourse-ci.org/images/trademarks/concourse-black.png) <!-- .element width="25%" height="25%" style="border: 0; background: white; box-shadow: None" -->

------

> Concourse is an open-source continuous thing-doer.

Built on the simple mechanics of resources, tasks, and jobs, Concourse presents a general approach to automation that makes it great for CI/CD.

Notes: Taken from concourse website

------

Concourse Architecture

![Concourse Logo](https://concourse-ci.org/images/concourse_architecture.png) <!-- .element width="40%" height="40%" style="border: 0; background: None; box-shadow: None" -->

------

 * Port 2222 uses SSH
 * Workers authenticate against the TSA using SSH keys

We need a secure way to distribute the Worker Private key to worker nodes <!-- .element: class="fragment" data-fragment-index="1" -->

Notes: Demo uses single keyfile, but is possible to use different keyfile per worker
as long as Concourse authorised key file is updated

With Vault we could use dynamic SSH keys

---

## Hashicorp Vault
##### https://www.vaultproject.io/
![Hashicorp Vault Logo](https://www.datocms-assets.com/2885/1556041583-vaultprimarylogomonochrometonal-skvacar6g.svg) <!-- .element width="30%" height="30%" style="border: 0; background: None; box-shadow: None" -->

------

> Manage Secrets and Protect Sensitive Data

Secure, store and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets and other sensitive data using a UI, CLI, or HTTP API.

Notes: Taken from Hashicorp Vault website
Vault can be used as a credentials store for Concourse, making it convenient to
store worker private keys

---

## Hashicorp Terraform
##### https://www.terraform.io/
![Terraform Logo](https://s3.amazonaws.com/hashicorp-marketing-web-assets/brand/Terraform_PrimaryLogo_MonochromeTonal.rJgVeyArax.svg) <!-- .element width="30%" height="30%" style="border: 0; background: None; box-shadow: None" -->

------

> Write, Plan, and Create Infrastructure as Code

HashiCorp Terraform enables you to safely and predictably create, change, and
improve infrastructure. It is an open source tool that codifies APIs into
declarative configuration files that can be shared amongst team members, treated as code, edited, reviewed, and versioned.

Notes: Taken from Terraform Website

------

  * Terraform provides multiple _providers_ to manage various infrastructure needs
  * Providers exist for Kubernetes and Hashicorp Vault
  * Infrastructure is managed by writing HashiCorp DSL (HCL)

------

### Terraform HCL

```
provider "aws" {
 profile = "default"
 region = "us-east-1"
}

resource "aws_instance" "example" {
 ami = "ami-2757f631"
 instance_type = "t2.micro"
}
```

---

## Establishing a Source of Truth

In an automated world, managing secrets is tricky

  * Data should stored encrypted
  * Secrets should be auditable (i.e when and who changed a secret)
  * Use values where needed (in Kubernetes and sans Kubernetes)

Notes: Storing secrets safely... i.e either gitcrypt, keybase or something
similar
Secret stores need some way of restricting access and ease rotation etc

------

## Options for storing Secrets

  * git-crypt <!-- .element: class="fragment" data-fragment-index="1" -->
  * GPG encrypted files <!-- .element: class="fragment" data-fragment-index="2" -->
  * Lots of other options <!-- .element: class="fragment" data-fragment-index="3" -->
  * I'll be using Keybase <!-- .element: class="fragment" data-fragment-index="4" -->

Notes: Keybase allows 'teams' and encrypted git

---

## Putting it all together

  * Store the keys in Keybase git repository
  * Use Terraform to store secrets in Kubernetes
  * Use Terraform to store secrets in Vault

------

## Putting the secrets in Kubernetes

```
resource "kubernetes_secret" "concourse-workers" {
  metadata {
    name = "v-nova-ci-worker-0"
    namespace = "ci-cd"
  }

  data = {
    host-key-pub   = file("${var.path_to_keybase_secrets}/concourse-secrets/worker/host-key-pub")
    worker-key-pub = file("${var.path_to_keybase_secrets}/concourse-secrets/worker/worker-key-pub")
    worker-key     = file("${var.path_to_keybase_secrets}/concourse-secrets/worker/worker-key")
  }
}
```

------

![tf_add_secrets](/gswrm-presentation/imgs/tf_add_secret.gif) <!-- .element style="border: 0; background: None; box-shadow: None" -->

------

## Putting the secrets in Vault

```
data "kubernetes_secret" "concourse0" {
  metadata {
    name = "v-nova-ci-worker-0"
    namespace = "ci-cd"
  }
}

locals {
  worker_key0 = {
    value = data.kubernetes_secret.concourse0.data.worker-key
  }
}

resource "vault_generic_secret" "shared_concourse_worker_key0" {
  path = "concourse/worker-key0"
  data_json = jsonencode(local.worker_key0)
}
```

------

![tf_add_vault_secret](/gswrm-presentation/imgs/tf_add_vault_secret.gif) <!-- .element style="border: 0; background: None; box-shadow: None" -->

------

## Create an Concourse Worker AMI

  * Using Packer we can create an AMI without storing any sensitive data
  * Instance will use IAM role to authenticate with Vault
  * When instance starts it queries vault for the SSH key

------

Code Snippet : From Worker startup script

```bash
if [[ ! -f ${CONCOURSE_TSA_WORKER_PRIVATE_KEY} ]]
then
    vault_token=$(cat ${vault_sink_file})
    WORKER_KEY=$(curl -s --header "X-Vault-Token: ${vault_token}" ${VAULT_ADDR}/v1/concourse/worker-key0 | jq '.data.value')
    if [[ -n "${WORKER_KEY}" ]]
    then
        echo -e $WORKER_KEY | tr -d '"'>> ${CONCOURSE_TSA_WORKER_PRIVATE_KEY}
    else
        echo "ERR: No Worker Key"
        exit 1
    fi
    chown root:root ${CONCOURSE_TSA_WORKER_PRIVATE_KEY}
    chmod 600 ${CONCOURSE_TSA_WORKER_PRIVATE_KEY}
fi
```

---

## How we can improve this

  * Unify code bases
  * Use sidecars?
  * Automatically sync between Kubernetes <-> HashiCorp Vault

Notes: Code base is split between k8s and vault due to dependency issues. Future
versions of K8s and Vault will have better integration and use things like sidecars

