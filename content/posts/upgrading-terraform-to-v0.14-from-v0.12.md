---
title: v0.12 에서 v0.14 로 terraform 업그레이드
date: 2021-02-24T00:40:17+09:00
draft: false
author: foobargem
tags: ['terraform', 'openstack']
---

사용자에게 제공되는 이미지를 OpenStack 의 Glance 에 업로드할때 Consul 과 Terraform 을 활용하고 있다.

이미지의 속성들을 Consul 의 kv store 에 저장해두고 Terraform 의
**consul provider** 와 **openstack provider** 를 사용하여
이미지 업로드 및 state file 로 이력관리를 하고 있다.

2019년 하반기부터 지금까지 잘 사용하고 있던 Terraform 버전은 v0.12.10 이고 작성된 tf 파일들도 v0.12.10 에 맞게 작성되어 있다.


## 배경

버전이 좀 낮지만 잘 사용하고 있었기 때문에 업그레이드는 고려하지 않고 있었다.

apply 가 종료된 후 생성된 tfstate 파일을 읽어 이미지의 uuid, name 을 추출하는것이 beta workspace 에서는 이상없이 동작했지만
production workspace 에서는 `missing schema for provider` 라는 처음 보는 오류가 발생했다.

```
$ terraform show terraform.tfstate.d/production/image.2021.02.23.tfstate
# data.consul_keys.image[1]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.image[2]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.image[3]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.image[4]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.image[5]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.image[0]:
# missing schema for provider "registry.terraform.io/-/consul"

# data.consul_keys.osprovider:
# missing schema for provider "registry.terraform.io/-/consul"

# openstack_images_image_v2.image_upload[0]:
# missing schema for provider "registry.terraform.io/-/openstack"

# openstack_images_image_v2.image_upload[1]:
# missing schema for provider "registry.terraform.io/-/openstack"

# openstack_images_image_v2.image_upload[2]:
# missing schema for provider "registry.terraform.io/-/openstack"

# openstack_images_image_v2.image_upload[3]:
# missing schema for provider "registry.terraform.io/-/openstack"

# openstack_images_image_v2.image_upload[4]:
# missing schema for provider "registry.terraform.io/-/openstack"

# openstack_images_image_v2.image_upload[5]:
# missing schema for provider "registry.terraform.io/-/openstack"
```

검색을 해봐도 뚜렷한 해결방법이 없는것 같아 이렇게 된거 terraform 버전을 **0.14.7** 으로 올리기로 했다.


## 생각보다 많은 변화

우선 plan 을 실행해 보았다.

```
$ terraform plan -var-file images_dc1_2021.02.23.tfvars tf

Warning: Quoted type constraints are deprecated

  on tf/data.tf line 2, in variable "datacenter":
   2:     type = "string"

Terraform 0.11 and earlier required type constraints to be given in quotes,
but that form is now deprecated and will be removed in a future version of
Terraform. To silence this warning, remove the quotes around "string".

(and 3 more similar warnings elsewhere)


Warning: Interpolation-only expressions are deprecated

  on tf/data.tf line 56, in data "consul_keys" "image":
  56:     count = "${length(local.urls)}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 16 more similar warnings elsewhere)


Error: Could not load plugin


Plugin reinitialization required. Please run "terraform init".

Plugins are external binaries that Terraform uses to access and manipulate
resources. The configuration provided requires plugins which can't be located,
don't satisfy the version constraints, or are otherwise incompatible.

Terraform automatically discovers provider requirements from your
configuration, including providers used in child modules. To see the
requirements and constraints, run "terraform providers".

2 problems:

- Failed to instantiate provider "registry.terraform.io/hashicorp/consul" to
obtain schema: unknown provider "registry.terraform.io/hashicorp/consul"
- Failed to instantiate provider "registry.terraform.io/hashicorp/openstack"
to obtain schema: unknown provider "registry.terraform.io/hashicorp/openstack"
```

후.. 뭔가 deprecated 된 문법이 있고, 초기화를 다시 해야 할것 같다.

일단 초기화를 다시 시도해본다.


```
$ terraform init tf/

Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/consul...
- Finding latest version of hashicorp/openstack...
- Installing hashicorp/consul v2.11.0...
- Installed hashicorp/consul v2.11.0 (signed by HashiCorp)

Warning: Quoted type constraints are deprecated

  on tf/data.tf line 2, in variable "datacenter":
   2:     type = "string"

Terraform 0.11 and earlier required type constraints to be given in quotes,
but that form is now deprecated and will be removed in a future version of
Terraform. To silence this warning, remove the quotes around "string".

(and 3 more similar warnings elsewhere)


Warning: Interpolation-only expressions are deprecated

  on tf/data.tf line 56, in data "consul_keys" "image":
  56:     count = "${length(local.urls)}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 16 more similar warnings elsewhere)


Error: Failed to query available provider packages

Could not retrieve the list of available versions for provider
hashicorp/openstack: provider registry registry.terraform.io does not have a
provider named registry.terraform.io/hashicorp/openstack

If you have just upgraded directly from Terraform v0.12 to Terraform v0.14
then please upgrade to Terraform v0.13 first and follow the upgrade guide for
that release, which might help you address this problem.

Did you intend to use terraform-provider-openstack/openstack? If so, you must
specify that source address in each module which requires that provider. To
see which modules are currently depending on hashicorp/openstack, run the
following command:
    terraform providers
```

consul provider 는 괜찮은데, openstack provider 가 문제가 있다.
v0.13 으로 먼저 업그레이드를 시도해보라는 가이드를 따라해보기로 했다.


```
$ terraform-0.13.6 0.13upgrade tf

This command will update the configuration files in the given directory to use
the new provider source features from Terraform v0.13. It will also highlight
any providers for which the source cannot be detected, and advise how to
proceed.

We recommend using this command in a clean version control work tree, so that
you can easily see the proposed changes as a diff against the latest commit.
If you have uncommited changes already present, we recommend aborting this
command and dealing with them before running this command again.

Would you like to upgrade the module in tf?
  Only 'yes' will be accepted to confirm.

  Enter a value: yes

-----------------------------------------------------------------------------

Warning: Quoted type constraints are deprecated

  on tf/data.tf line 2, in variable "datacenter":
   2:     type = "string"

Terraform 0.11 and earlier required type constraints to be given in quotes,
but that form is now deprecated and will be removed in a future version of
Terraform. To silence this warning, remove the quotes around "string".

(and 3 more similar warnings elsewhere)


Warning: Interpolation-only expressions are deprecated

  on tf/data.tf line 56, in data "consul_keys" "image":
  56:     count = "${length(local.urls)}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 16 more similar warnings elsewhere)

-----------------------------------------------------------------------------

Upgrade complete!

Use your version control system to review the proposed changes, make any
necessary adjustments, and then commit.
```

오호! 뭐가 달라진듯.

tf 디렉토리에 versions.tf 파일이 생성되어 있다.


```
$ cat tf/versions.tf
terraform {
  required_providers {
    consul = {
      source = "hashicorp/consul"
    }
    openstack = {
      source = "terraform-provider-openstack/openstack"
    }
  }
  required_version = ">= 0.13"
}
```

오호라! provider 에 namespace 생겼고, 특히 openstack provider 는 hashicorp 가 아닌 커뮤니티 지원으로 변경된듯 하다.


**terraform** 블럭에서 consul 과 openstack 의 provider source 를 명시해주는 부분이 추가가 되었다.

provider 를 변경하더라도 local name 은 놔두고 source 만 변경할 수 있게 되어 좋은점도 있는것 같다.

다시 init 을 시도해본다.

```
$ terraform init tf

Initializing the backend...

Initializing provider plugins...
- Finding latest version of terraform-provider-openstack/openstack...
- Finding latest version of hashicorp/consul...
- Installing terraform-provider-openstack/openstack v1.38.0...
- Installed terraform-provider-openstack/openstack v1.38.0 (self-signed, key ID 4F80527A391BEFD2)
- Installing hashicorp/consul v2.11.0...
- Installed hashicorp/consul v2.11.0 (signed by HashiCorp)

Partner and community providers are signed by their developers.
If you'd like to know more about provider signing, you can read about it here:
https://www.terraform.io/docs/cli/plugins/signing.html

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.


Warning: Quoted type constraints are deprecated

  on tf/data.tf line 2, in variable "datacenter":
   2:     type = "string"

Terraform 0.11 and earlier required type constraints to be given in quotes,
but that form is now deprecated and will be removed in a future version of
Terraform. To silence this warning, remove the quotes around "string".

(and 3 more similar warnings elsewhere)


Warning: Interpolation-only expressions are deprecated

  on tf/data.tf line 56, in data "consul_keys" "image":
  56:     count = "${length(local.urls)}"

Terraform 0.11 and earlier required all non-constant expressions to be
provided via interpolation syntax, but this pattern is now deprecated. To
silence this warning, remove the "${ sequence from the start and the }"
sequence from the end of this expression, leaving just the inner expression.

Template interpolation syntax is still used to construct strings from
expressions when the template includes multiple interpolation sequences or a
mixture of literal strings and interpolations. This deprecation applies only
to templates that consist entirely of a single interpolation sequence.

(and 16 more similar warnings elsewhere)

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

성공적으로 초기화가 되었다.

이제는 눈엣가시 같은 warning 을 없애기로 한다.
먼저 variable 블럭의 type 에 따옴표를 사용한 부분을 없앤다.

```
variable "datacenter" {
-   type = "string"
+   type = string
    default = "dc1"
}
```

그리고 보간전용 표현식(Interpolation-only expressions) 을 없앤다.
문자열 가공(formatting)을 하는경우가 아니라면 단독으로 값을 사용하는 경우라면 따옴표로 감싸지 말라는 것이다.

```
$ cat resource.tf
resource "openstack_images_image_v2" "image_upload" {
-   region = "${data.consul_keys.osprovider.var.region}"
+   region = data.consul_keys.osprovider.var.region

-   count = "${length(local.urls)}"
+   count = length(local.urls)
...
```

이후 plan 을 해보면 warning 없이 잘 동작한다.


## state migration

apply 는 하지 않고 기존의 state 를 확인해 보면 오류가 발생한다.

```
$ terraform show

Error: Could not load plugin


Plugin reinitialization required. Please run "terraform init".

Plugins are external binaries that Terraform uses to access and manipulate
resources. The configuration provided requires plugins which can't be located,
don't satisfy the version constraints, or are otherwise incompatible.

Terraform automatically discovers provider requirements from your
configuration, including providers used in child modules. To see the
requirements and constraints, run "terraform providers".

2 problems:

- Failed to instantiate provider "registry.terraform.io/-/consul" to obtain
schema: unknown provider "registry.terraform.io/-/consul"
- Failed to instantiate provider "registry.terraform.io/-/openstack" to obtain
schema: unknown provider "registry.terraform.io/-/openstack"
```

다시 초기화를 하라고 하는데 원인은 state 파일에 남아있는 provider 정보가 안맞기 때문이다.

```
$ terraform providers tf

Providers required by configuration:
.
├── provider[registry.terraform.io/hashicorp/consul]
└── provider[registry.terraform.io/terraform-provider-openstack/openstack]

Providers required by state:

    provider[registry.terraform.io/-/consul]

    provider[registry.terraform.io/-/openstack]

```

**state replace-provider** 명령으로 **[hostname/][namespace/]name** 형식으로 state 의 provider 값을 변경할 수 있다.

```
$ terraform state replace-provider registry.terraform.io/-/consul registry.terraform.io/hashicorp/consul
Terraform will perform the following actions:

  ~ Updating provider:
    - registry.terraform.io/-/consul
    + registry.terraform.io/hashicorp/consul

Changing 2 resources:

  data.consul_keys.image
  data.consul_keys.osprovider

Do you want to make these changes?
Only 'yes' will be accepted to continue.

Enter a value: yes

Successfully replaced provider for 2 resources.

$ terraform state replace-provider registry.terraform.io/-/openstack registry.terraform.io/terraform-provider-openstack/openstack
Terraform will perform the following actions:

  ~ Updating provider:
    - registry.terraform.io/-/openstack
    + registry.terraform.io/terraform-provider-openstack/openstack

Changing 1 resources:

  openstack_images_image_v2.image_upload

Do you want to make these changes?
Only 'yes' will be accepted to continue.

Enter a value: yes

Successfully replaced provider for 1 resources.
```

다시 providers 정보를 확인해보면 state 의 provider 값이 변경되었다.
물론 show 를 했을때 state 파일의 내용도 이상없이 확인 가능하다.

```
$ terraform providers tf

Providers required by configuration:
.
├── provider[registry.terraform.io/terraform-provider-openstack/openstack]
└── provider[registry.terraform.io/hashicorp/consul]

Providers required by state:

    provider[registry.terraform.io/hashicorp/consul]

    provider[registry.terraform.io/terraform-provider-openstack/openstack]
```

## missing schema for provider 문제

production workspace 에서 state replace-provider 를 하고
terraform show 를 해도 여전히 `missing schema for provider` 오류가 발생한다.


```
$ terraform show terraform.tfstate.d/production/image.2021.02.23.tfstate
# data.consul_keys.image[4]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.image[5]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.image[0]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.image[1]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.image[2]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.image[3]:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# data.consul_keys.osprovider:
# missing schema for provider "registry.terraform.io/hashicorp/consul"

# openstack_images_image_v2.image_upload[0]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"

# openstack_images_image_v2.image_upload[1]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"

# openstack_images_image_v2.image_upload[2]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"

# openstack_images_image_v2.image_upload[3]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"

# openstack_images_image_v2.image_upload[4]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"

# openstack_images_image_v2.image_upload[5]:
# missing schema for provider "registry.terraform.io/terraform-provider-openstack/openstack"
```

[디버깅](https://www.terraform.io/docs/internals/debugging.html) 을 시도해봤다.


```
$ TF_LOG=TRACE TF_LOG_PATH=debug.log terraform show terraform.tfstate.d/production/image.2021.02.23.tfstate
```

```
2021/03/01 00:15:09 [TRACE] backend/local: reading remote state for workspace "production"
2021/03/01 00:15:09 [TRACE] statemgr.Filesystem: reading initial snapshot from terraform.tfstate.d/production/terraform.tfstate
2021/03/01 00:15:09 [TRACE] statemgr.Filesystem: snapshot file has nil snapshot, but that's okay
2021/03/01 00:15:09 [TRACE] statemgr.Filesystem: read nil snapshot
2021/03/01 00:15:09 [DEBUG] backend/local: skipping refresh of managed resources
```

terraform.tfstate.d/production/terraform.tfstate 파일이 없기 때문에 `missing schema for provider` 오류가 발생한 것이다.

image.2021.02.23.tfstate 를 terraform.tfstate 로 복사해준뒤 terraform show 를 하면 정상적으로 state 를 확인할 수 있게 된다.


## 더 볼것들

* apply 의 **-state** 옵션과 **state** 에 대해서
* state 와 remote state
