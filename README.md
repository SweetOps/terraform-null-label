<!-- markdownlint-disable -->
# terraform-null-label [![Latest Release](https://img.shields.io/github/release/cloudposse/terraform-null-label.svg)](https://github.com/cloudposse/terraform-null-label/releases/latest) [![Slack Community](https://slack.cloudposse.com/badge.svg)](https://slack.cloudposse.com)
<!-- markdownlint-restore -->

[![README Header][readme_header_img]][readme_header_link]

[![Cloud Posse][logo]](https://cpco.io/homepage)

<!--




  ** DO NOT EDIT THIS FILE
  **
  ** This file was automatically generated by the `build-harness`.
  ** 1) Make all changes to `README.yaml`
  ** 2) Run `make init` (you only need to do this once)
  ** 3) Run`make readme` to rebuild this file.
  **
  ** (We maintain HUNDREDS of open source projects. This is how we maintain our sanity.)
  **





-->

Terraform module designed to generate consistent names and tags for resources. Use `terraform-null-label` to implement a strict naming convention.

This module generates names using the following convention by default: `{namespace}-{environment}-{stage}-{name}-{attributes}`.
However, it is highly configurable. The delimiter (e.g. `-`) is configurable. Each label item is optional (although you must provide at least one).
So if you prefer the term `stage` to `environment`
you can exclude environment and the label `id` will look like `{namespace}-{stage}-{name}-{attributes}`.
If attributes are excluded but `stage` and `environment` are included, `id` will look like `{namespace}-{environment}-{stage}-{name}`.
If you want the attributes in a different order, you can specify that, too, with the `label_order` list.
You can set a maximum length for the name, and the module will create a unique name that fits within that length.

It's recommended to use one `terraform-null-label` module for every unique resource of a given resource type.
For example, if you have 10 instances, there should be 10 different labels.
However, if you have multiple different kinds of resources (e.g. instances, security groups, file systems, and elastic ips), then they can all share the same label assuming they are logically related.

All [Cloud Posse modules](https://github.com/cloudposse?utf8=%E2%9C%93&q=terraform-&type=&language=) use this module to ensure resources can be instantiated multiple times within an account and without conflict.

**NOTE:** The `null` refers to the primary Terraform [provider](https://www.terraform.io/docs/providers/null/index.html) used in this module.

Releases of this module from `0.12.0` onward support `HCL2` and only work with Terraform 0.12 or newer.  Releases prior to this are compatible with earlier versions of terraform like Terraform 0.11.


---

This project is part of our comprehensive ["SweetOps"](https://cpco.io/sweetops) approach towards DevOps.
[<img align="right" title="Share via Email" src="https://docs.cloudposse.com/images/ionicons/ios-email-outline-2.0.1-16x16-999999.svg"/>][share_email]
[<img align="right" title="Share on Google+" src="https://docs.cloudposse.com/images/ionicons/social-googleplus-outline-2.0.1-16x16-999999.svg" />][share_googleplus]
[<img align="right" title="Share on Facebook" src="https://docs.cloudposse.com/images/ionicons/social-facebook-outline-2.0.1-16x16-999999.svg" />][share_facebook]
[<img align="right" title="Share on Reddit" src="https://docs.cloudposse.com/images/ionicons/social-reddit-outline-2.0.1-16x16-999999.svg" />][share_reddit]
[<img align="right" title="Share on LinkedIn" src="https://docs.cloudposse.com/images/ionicons/social-linkedin-outline-2.0.1-16x16-999999.svg" />][share_linkedin]
[<img align="right" title="Share on Twitter" src="https://docs.cloudposse.com/images/ionicons/social-twitter-outline-2.0.1-16x16-999999.svg" />][share_twitter]


[![Terraform Open Source Modules](https://docs.cloudposse.com/images/terraform-open-source-modules.svg)][terraform_modules]



It's 100% Open Source and licensed under the [APACHE2](LICENSE).







We literally have [*hundreds of terraform modules*][terraform_modules] that are Open Source and well-maintained. Check them out!







## Usage


**IMPORTANT:** We do not pin modules to versions in our examples because of the
difficulty of keeping the versions in the documentation in sync with the latest released versions.
We highly recommend that in your code you pin the version to the exact version you are
using so that your infrastructure remains stable, and update versions in a
systematic way so that they do not catch you by surprise.

Also, because of a bug in the Terraform registry ([hashicorp/terraform#21417](https://github.com/hashicorp/terraform/issues/21417)),
the registry shows many of our inputs as required when in fact they are optional.
The table below correctly indicates which inputs are required.


### Defaults

Cloud Posse Terraform modules share a common `context` object that is meant to be passed from module to module.
The context object is a single object that contains all the input values for `terraform-null-label`.
However, each input value can also be specified individually by name as a standard Terraform variable,
and the value of those variables, when set to something other than `null`, will override the value
in the context object. In order to allow chaining of these objects, where the context object input to one
module is transformed and passed to the next module, all the variables default to `null` or empty collections.
The actual default values used when nothing is explicitly set are describe in the documentation below.

For example, the default value of `delimiter` is shown as `null`, but if you leave it set to null,
`terraform-null-label` will actually use the default delimiter `-` (hyphen).

A non-obvious but intentional consequence of this design is that once a module sets a non-default value,
future modules in the chain cannot reset the value back to the original default. Insted, the new setting
becomes the new default for downstream modules. Also, collections are not overwritten, they are merged,
so once a tag is added, it will remain in the tag set and cannot be removed, although its value can
be overwritten.

### Simple Example

```hcl
module "eg_prod_bastion_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["public"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}
```

This will create an `id` with the value of `eg-prod-bastion-public` because when generating `id`, the default order is `namespace`, `environment`, `stage`,  `name`, `attributes`
(you can override it by using the `label_order` variable, see [Advanced Example 3](#advanced-example-3)).

Now reference the label when creating an instance:

```hcl
resource "aws_instance" "eg_prod_bastion_public" {
  instance_type = "t1.micro"
  tags          = module.eg_prod_bastion_label.tags
}
```

Or define a security group:

```hcl
resource "aws_security_group" "eg_prod_bastion_public" {
  vpc_id = var.vpc_id
  name   = module.eg_prod_bastion_label.id
  tags   = module.eg_prod_bastion_label.tags
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```


### Advanced Example

Here is a more complex example with two instances using two different labels. Note how efficiently the tags are defined for both the instance and the security group.

<details><summary>Click to show</summary>

```hcl
module "eg_prod_bastion_abc_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["abc"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}

resource "aws_security_group" "eg_prod_bastion_abc" {
  name = module.eg_prod_bastion_abc_label.id
  tags = module.eg_prod_bastion_abc_label.tags
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_abc" {
   instance_type          = "t1.micro"
   tags                   = module.eg_prod_bastion_abc_label.tags
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_abc.id]
}

module "eg_prod_bastion_xyz_label" {
  source     = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace  = "eg"
  stage      = "prod"
  name       = "bastion"
  attributes = ["xyz"]
  delimiter  = "-"

  tags = {
    "BusinessUnit" = "XYZ",
    "Snapshot"     = "true"
  }
}

resource "aws_security_group" "eg_prod_bastion_xyz" {
  name = module.eg_prod_bastion_xyz_label.id
  tags = module.eg_prod_bastion_xyz_label.tags
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "eg_prod_bastion_xyz" {
   instance_type          = "t1.micro"
   tags                   = module.eg_prod_bastion_xyz_label.tags
   vpc_security_group_ids = [aws_security_group.eg_prod_bastion_xyz.id]
}
```

</details>

### Advanced Example 2

Here is a more complex example with an autoscaling group that has a different tagging schema than other resources and requires its tags to be in this format, which this module can generate:

<details><summary>Click to show</summary>

```hcl
tags = [
    {
        key = Name,
        propagate_at_launch = 1,
        value = namespace-stage-name
    },
    {
        key = Namespace,
        propagate_at_launch = 1,
        value = namespace
    },
    {
        key = Stage,
        propagate_at_launch = 1,
        value = stage
    }
]
```

Autoscaling group using propagating tagging below (full example: [autoscalinggroup](examples/autoscalinggroup/main.tf))

```hcl
################################
# terraform-null-label example #
################################
module "label" {
  source    = "../../"
  namespace = "cp"
  stage     = "prod"
  name      = "app"

  tags = {
    BusinessUnit = "Finance"
    ManagedBy    = "Terraform"
  }

  additional_tag_map = {
    propagate_at_launch = "true"
  }
}

#######################
# Launch template     #
#######################
resource "aws_launch_template" "default" {
  # terraform-null-label example used here: Set template name prefix
  name_prefix                           = "${module.label.id}-"
  image_id                              = data.aws_ami.amazon_linux.id
  instance_type                         = "t2.micro"
  instance_initiated_shutdown_behavior  = "terminate"

  vpc_security_group_ids                = [data.aws_security_group.default.id]

  monitoring {
    enabled                             = false
  }
  # terraform-null-label example used here: Set tags on volumes
  tag_specifications {
    resource_type                       = "volume"
    tags                                = module.label.tags
  }
}

######################
# Autoscaling group  #
######################
resource "aws_autoscaling_group" "default" {
  # terraform-null-label example used here: Set ASG name prefix
  name_prefix                           = "${module.label.id}-"
  vpc_zone_identifier                   = data.aws_subnet_ids.all.ids
  max_size                              = "1"
  min_size                              = "1"
  desired_capacity                      = "1"

  launch_template = {
    id                                  = "aws_launch_template.default.id
    version                             = "$$Latest"
  }

  # terraform-null-label example used here: Set tags on ASG and EC2 Servers
  tags                                  = module.label.tags_as_list_of_maps
}
```

</details>

### Advanced Example 3

See [complete example](./examples/complete) for even more examples.

This example shows how you can pass the `context` output of one label module to the next label_module,
allowing you to create one label that has the base set of values, and then creating every extra label
as a derivative of that.

<details><summary>Click to show</summary>

```hcl
module "label1" {
  source      = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  namespace   = "CloudPosse"
  environment = "UAT"
  stage       = "build"
  name        = "Winston Churchroom"
  attributes  = ["fire", "water", "earth", "air"]
  delimiter   = "-"

  label_order = ["name", "environment", "stage", "attributes"]

  tags = {
    "City"        = "Dublin"
    "Environment" = "Private"
  }
}

module "label2" {
  source    = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  context   = module.label1.context
  name      = "Charlie"
  stage     = "test"
  delimiter = "+"

  tags = {
    "City"        = "London"
    "Environment" = "Public"
  }
}

module "label3" {
  source    = "git::https://github.com/cloudposse/terraform-null-label.git?ref=master"
  name      = "Starfish"
  stage     = "release"
  context   = module.label1.context
  delimiter = "."

  tags = {
    "Eat"    = "Carrot"
    "Animal" = "Rabbit"
  }
}
```

This creates label outputs like this:

```hcl
label1 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "-"
  "id" = "winstonchurchroom-uat-build-fire-water-earth-air"
  "name" = "winstonchurchroom"
  "namespace" = "cloudposse"
  "stage" = "build"
}
label1_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "-"
  "enabled" = true
  "environment" = "UAT"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "Winston Churchroom"
  "namespace" = "CloudPosse"
  "stage" = "build"
  "tags" = {
    "City" = "Dublin"
    "Environment" = "Private"
  }
}
label1_normalized_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "-"
  "enabled" = true
  "environment" = "uat"
  "id_length_limit" = 0
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "winstonchurchroom"
  "namespace" = "cloudposse"
  "regex_replace_chars" = "/[^-a-zA-Z0-9]/"
  "stage" = "build"
  "tags" = {
    "Attributes" = "fire-water-earth-air"
    "City" = "Dublin"
    "Environment" = "Private"
    "Name" = "winstonchurchroom-uat-build-fire-water-earth-air"
    "Namespace" = "cloudposse"
    "Stage" = "build"
  }
}
label1_tags = {
  "Attributes" = "fire-water-earth-air"
  "City" = "Dublin"
  "Environment" = "Private"
  "Name" = "winstonchurchroom-uat-build-fire-water-earth-air"
  "Namespace" = "cloudposse"
  "Stage" = "build"
}
label2 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "+"
  "id" = "charlie+uat+test+fire+water+earth+air"
  "name" = "charlie"
  "namespace" = "cloudposse"
  "stage" = "test"
}
label2_context = {
  "additional_tag_map" = {
    "additional_tag" = "yes"
    "propagate_at_launch" = "true"
  }
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "+"
  "enabled" = true
  "environment" = "UAT"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "Charlie"
  "namespace" = "CloudPosse"
  "regex_replace_chars" = "/[^a-zA-Z0-9-+]/"
  "stage" = "test"
  "tags" = {
    "City" = "London"
    "Environment" = "Public"
  }
}
label2_tags = {
  "Attributes" = "fire+water+earth+air"
  "City" = "London"
  "Environment" = "Public"
  "Name" = "charlie+uat+test+fire+water+earth+air"
  "Namespace" = "cloudposse"
  "Stage" = "test"
}
label2_tags_as_list_of_maps = [
  {
    "additional_tag" = "yes"
    "key" = "Attributes"
    "propagate_at_launch" = "true"
    "value" = "fire+water+earth+air"
  },
  {
    "additional_tag" = "yes"
    "key" = "City"
    "propagate_at_launch" = "true"
    "value" = "London"
  },
  {
    "additional_tag" = "yes"
    "key" = "Environment"
    "propagate_at_launch" = "true"
    "value" = "Public"
  },
  {
    "additional_tag" = "yes"
    "key" = "Name"
    "propagate_at_launch" = "true"
    "value" = "charlie+uat+test+fire+water+earth+air"
  },
  {
    "additional_tag" = "yes"
    "key" = "Namespace"
    "propagate_at_launch" = "true"
    "value" = "cloudposse"
  },
  {
    "additional_tag" = "yes"
    "key" = "Stage"
    "propagate_at_launch" = "true"
    "value" = "test"
  },
]
label3 = {
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "."
  "id" = "starfish.uat.release.fire.water.earth.air"
  "name" = "starfish"
  "namespace" = "cloudposse"
  "stage" = "release"
}
label3_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "."
  "enabled" = true
  "environment" = "UAT"
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "Starfish"
  "namespace" = "CloudPosse"
  "regex_replace_chars" = "/[^-a-zA-Z0-9.]/"
  "stage" = "release"
  "tags" = {
    "Animal" = "Rabbit"
    "City" = "Dublin"
    "Eat" = "Carrot"
    "Environment" = "Private"
  }
}
label3_normalized_context = {
  "additional_tag_map" = {}
  "attributes" = [
    "fire",
    "water",
    "earth",
    "air",
  ]
  "delimiter" = "."
  "enabled" = true
  "environment" = "uat"
  "id_length_limit" = 0
  "label_order" = [
    "name",
    "environment",
    "stage",
    "attributes",
  ]
  "name" = "starfish"
  "namespace" = "cloudposse"
  "regex_replace_chars" = "/[^-a-zA-Z0-9.]/"
  "stage" = "release"
  "tags" = {
    "Animal" = "Rabbit"
    "Attributes" = "fire.water.earth.air"
    "City" = "Dublin"
    "Eat" = "Carrot"
    "Environment" = "Private"
    "Name" = "starfish.uat.release.fire.water.earth.air"
    "Namespace" = "cloudposse"
    "Stage" = "release"
  }
}
label3_tags = {
  "Animal" = "Rabbit"
  "Attributes" = "fire.water.earth.air"
  "City" = "Dublin"
  "Eat" = "Carrot"
  "Environment" = "Private"
  "Name" = "starfish.uat.release.fire.water.earth.air"
  "Namespace" = "cloudposse"
  "Stage" = "release"
}

```

</details>






<!-- markdownlint-disable -->
## Makefile Targets
```text
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
<!-- markdownlint-restore -->
<!-- markdownlint-disable -->
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 0.13.0 |

## Providers

No provider.

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| additional\_tag\_map | Additional tags for appending to tags\_as\_list\_of\_maps. Not added to `tags`. | `map(string)` | `{}` | no |
| attributes | Additional attributes (e.g. `1`) | `list(string)` | `[]` | no |
| context | Single object for setting entire context at once.<br>See description of individual variables for details.<br>Leave string and numeric variables as `null` to use default value.<br>Individual variable settings (non-null) override settings in context object,<br>except for attributes, tags, and additional\_tag\_map, which are merged. | <pre>object({<br>    enabled                 = bool<br>    namespace               = string<br>    environment             = string<br>    stage                   = string<br>    name                    = string<br>    delimiter               = string<br>    attributes              = list(string)<br>    tags                    = map(string)<br>    additional_tag_map      = map(string)<br>    regex_replace_chars     = string<br>    label_order             = list(string)<br>    id_length_limit         = number<br>    id_case                 = string<br>    generated_tag_name_case = string<br>  })</pre> | <pre>{<br>  "additional_tag_map": {},<br>  "attributes": [],<br>  "delimiter": null,<br>  "enabled": true,<br>  "environment": null,<br>  "generated_tag_name_case": null,<br>  "id_case": null,<br>  "id_length_limit": null,<br>  "label_order": [],<br>  "name": null,<br>  "namespace": null,<br>  "regex_replace_chars": null,<br>  "stage": null,<br>  "tags": {}<br>}</pre> | no |
| delimiter | Delimiter to be used between `namespace`, `environment`, `stage`, `name` and `attributes`.<br>Defaults to `-` (hyphen). Set to `""` to use no delimiter at all. | `string` | `null` | no |
| enabled | Set to false to prevent the module from creating any resources | `bool` | `null` | no |
| environment | Environment, e.g. 'uw2', 'us-west-2', OR 'prod', 'staging', 'dev', 'UAT' | `string` | `null` | no |
| generated\_tag\_name\_case | The letter case of `generated_tag` keys (i.e. `name`, `namespace`, `environment`, `stage`, `attributes`).<br>Possible values: `lower`, `title`, `upper`. <br>Default value: `title`. | `string` | `null` | no |
| id\_case | The letter case of generated `ID`.<br>Possible values: `lower`, `title`, `upper`. <br>Default value: `lower`. | `string` | `null` | no |
| id\_length\_limit | Limit `id` to this many characters.<br>Set to `0` for unlimited length.<br>Set to `null` for default, which is `0`.<br>Does not affect `id_full`. | `number` | `null` | no |
| label\_order | The naming order of the id output and Name tag.<br>Defaults to ["namespace", "environment", "stage", "name", "attributes"].<br>You can omit any of the 5 elements, but at least one must be present. | `list(string)` | `null` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | `string` | `null` | no |
| namespace | Namespace, which could be your organization name or abbreviation, e.g. 'eg' or 'cp' | `string` | `null` | no |
| regex\_replace\_chars | Regex to replace chars with empty string in `namespace`, `environment`, `stage` and `name`.<br>If not set, `"/[^a-zA-Z0-9-]/"` is used to remove all characters other than hyphens, letters and digits. | `string` | `null` | no |
| stage | Stage, e.g. 'prod', 'staging', 'dev', OR 'source', 'build', 'test', 'deploy', 'release' | `string` | `null` | no |
| tags | Additional tags (e.g. `map('BusinessUnit','XYZ')` | `map(string)` | `{}` | no |

## Outputs

| Name | Description |
|------|-------------|
| additional\_tag\_map | The merged additional\_tag\_map |
| attributes | List of attributes |
| context | Merged but otherwise unmodified input to this module, to be used as context input to other modules.<br>Note: this version will have null values as defaults, not the values actually used as defaults. |
| delimiter | Delimiter between `namespace`, `environment`, `stage`, `name` and `attributes` |
| enabled | True if module is enabled, false otherwise |
| environment | Normalized environment |
| id | Disambiguated ID restricted to `id_length_limit` characters in total |
| id\_full | Disambiguated ID not restricted in length |
| id\_length\_limit | The id\_length\_limit actually used to create the ID, with `0` meaning unlimited |
| label\_order | The naming order actually used to create the ID |
| name | Normalized name |
| namespace | Normalized namespace |
| normalized\_context | Normalized context of this module |
| regex\_replace\_chars | The regex\_replace\_chars actually used to create the ID |
| stage | Normalized stage |
| tags | Normalized Tag map |
| tags\_as\_list\_of\_maps | Additional tags as a list of maps, which can be used in several AWS resources |

<!-- markdownlint-restore -->



## Share the Love

Like this project? Please give it a ★ on [our GitHub](https://github.com/cloudposse/terraform-null-label)! (it helps us **a lot**)

Are you using this project or any of our other projects? Consider [leaving a testimonial][testimonial]. =)


## Related Projects

Check out these related projects.

- [terraform-terraform-label](https://github.com/cloudposse/terraform-terraform-label) - Terraform Module to define a consistent naming convention by (namespace, environment, stage, name, [attributes])



## Help

**Got a question?** We got answers.

File a GitHub [issue](https://github.com/cloudposse/terraform-null-label/issues), send us an [email][email] or join our [Slack Community][slack].

[![README Commercial Support][readme_commercial_support_img]][readme_commercial_support_link]

## DevOps Accelerator for Startups


We are a [**DevOps Accelerator**][commercial_support]. We'll help you build your cloud infrastructure from the ground up so you can own it. Then we'll show you how to operate it and stick around for as long as you need us.

[![Learn More](https://img.shields.io/badge/learn%20more-success.svg?style=for-the-badge)][commercial_support]

Work directly with our team of DevOps experts via email, slack, and video conferencing.

We deliver 10x the value for a fraction of the cost of a full-time engineer. Our track record is not even funny. If you want things done right and you need it done FAST, then we're your best bet.

- **Reference Architecture.** You'll get everything you need from the ground up built using 100% infrastructure as code.
- **Release Engineering.** You'll have end-to-end CI/CD with unlimited staging environments.
- **Site Reliability Engineering.** You'll have total visibility into your apps and microservices.
- **Security Baseline.** You'll have built-in governance with accountability and audit logs for all changes.
- **GitOps.** You'll be able to operate your infrastructure via Pull Requests.
- **Training.** You'll receive hands-on training so your team can operate what we build.
- **Questions.** You'll have a direct line of communication between our teams via a Shared Slack channel.
- **Troubleshooting.** You'll get help to triage when things aren't working.
- **Code Reviews.** You'll receive constructive feedback on Pull Requests.
- **Bug Fixes.** We'll rapidly work with you to fix any bugs in our projects.

## Slack Community

Join our [Open Source Community][slack] on Slack. It's **FREE** for everyone! Our "SweetOps" community is where you get to talk with others who share a similar vision for how to rollout and manage infrastructure. This is the best place to talk shop, ask questions, solicit feedback, and work together as a community to build totally *sweet* infrastructure.

## Discourse Forums

Participate in our [Discourse Forums][discourse]. Here you'll find answers to commonly asked questions. Most questions will be related to the enormous number of projects we support on our GitHub. Come here to collaborate on answers, find solutions, and get ideas about the products and services we value. It only takes a minute to get started! Just sign in with SSO using your GitHub account.

## Newsletter

Sign up for [our newsletter][newsletter] that covers everything on our technology radar.  Receive updates on what we're up to on GitHub as well as awesome new projects we discover.

## Office Hours

[Join us every Wednesday via Zoom][office_hours] for our weekly "Lunch & Learn" sessions. It's **FREE** for everyone!

[![zoom](https://img.cloudposse.com/fit-in/200x200/https://cloudposse.com/wp-content/uploads/2019/08/Powered-by-Zoom.png")][office_hours]

## Contributing

### Bug Reports & Feature Requests

Please use the [issue tracker](https://github.com/cloudposse/terraform-null-label/issues) to report any bugs or file feature requests.

### Developing

If you are interested in being a contributor and want to get involved in developing this project or [help out](https://cpco.io/help-out) with our other projects, we would love to hear from you! Shoot us an [email][email].

In general, PRs are welcome. We follow the typical "fork-and-pull" Git workflow.

 1. **Fork** the repo on GitHub
 2. **Clone** the project to your own machine
 3. **Commit** changes to your own branch
 4. **Push** your work back up to your fork
 5. Submit a **Pull Request** so that we can review your changes

**NOTE:** Be sure to merge the latest changes from "upstream" before making a pull request!


## Copyright

Copyright © 2017-2021 [Cloud Posse, LLC](https://cpco.io/copyright)



## License

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

See [LICENSE](LICENSE) for full details.

```text
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
```









## Trademarks

All other trademarks referenced herein are the property of their respective owners.

## About

This project is maintained and funded by [Cloud Posse, LLC][website]. Like it? Please let us know by [leaving a testimonial][testimonial]!

[![Cloud Posse][logo]][website]

We're a [DevOps Professional Services][hire] company based in Los Angeles, CA. We ❤️  [Open Source Software][we_love_open_source].

We offer [paid support][commercial_support] on all of our projects.

Check out [our other projects][github], [follow us on twitter][twitter], [apply for a job][jobs], or [hire us][hire] to help with your cloud strategy and implementation.



### Contributors

<!-- markdownlint-disable -->
|  [![Erik Osterman][osterman_avatar]][osterman_homepage]<br/>[Erik Osterman][osterman_homepage] | [![Andriy Knysh][aknysh_avatar]][aknysh_homepage]<br/>[Andriy Knysh][aknysh_homepage] | [![Igor Rodionov][goruha_avatar]][goruha_homepage]<br/>[Igor Rodionov][goruha_homepage] | [![Sergey Vasilyev][s2504s_avatar]][s2504s_homepage]<br/>[Sergey Vasilyev][s2504s_homepage] | [![Michael Pereira][MichaelPereira_avatar]][MichaelPereira_homepage]<br/>[Michael Pereira][MichaelPereira_homepage] | [![Jamie Nelson][Jamie-BitFlight_avatar]][Jamie-BitFlight_homepage]<br/>[Jamie Nelson][Jamie-BitFlight_homepage] | [![Vladimir][SweetOps_avatar]][SweetOps_homepage]<br/>[Vladimir][SweetOps_homepage] | [![Daren Desjardins][darend_avatar]][darend_homepage]<br/>[Daren Desjardins][darend_homepage] | [![Maarten van der Hoef][maartenvanderhoef_avatar]][maartenvanderhoef_homepage]<br/>[Maarten van der Hoef][maartenvanderhoef_homepage] | [![Adam Tibbing][tibbing_avatar]][tibbing_homepage]<br/>[Adam Tibbing][tibbing_homepage] |
|---|---|---|---|---|---|---|---|---|---|
<!-- markdownlint-restore -->

  [osterman_homepage]: https://github.com/osterman
  [osterman_avatar]: https://img.cloudposse.com/150x150/https://github.com/osterman.png
  [aknysh_homepage]: https://github.com/aknysh
  [aknysh_avatar]: https://img.cloudposse.com/150x150/https://github.com/aknysh.png
  [goruha_homepage]: https://github.com/goruha
  [goruha_avatar]: https://img.cloudposse.com/150x150/https://github.com/goruha.png
  [s2504s_homepage]: https://github.com/s2504s
  [s2504s_avatar]: https://img.cloudposse.com/150x150/https://github.com/s2504s.png
  [MichaelPereira_homepage]: https://github.com/MichaelPereira
  [MichaelPereira_avatar]: https://img.cloudposse.com/150x150/https://github.com/MichaelPereira.png
  [Jamie-BitFlight_homepage]: https://github.com/Jamie-BitFlight
  [Jamie-BitFlight_avatar]: https://img.cloudposse.com/150x150/https://github.com/Jamie-BitFlight.png
  [SweetOps_homepage]: https://github.com/SweetOps
  [SweetOps_avatar]: https://img.cloudposse.com/150x150/https://github.com/SweetOps.png
  [darend_homepage]: https://github.com/darend
  [darend_avatar]: https://img.cloudposse.com/150x150/https://github.com/darend.png
  [maartenvanderhoef_homepage]: https://github.com/maartenvanderhoef
  [maartenvanderhoef_avatar]: https://img.cloudposse.com/150x150/https://github.com/maartenvanderhoef.png
  [tibbing_homepage]: https://github.com/tibbing
  [tibbing_avatar]: https://img.cloudposse.com/150x150/https://github.com/tibbing.png

[![README Footer][readme_footer_img]][readme_footer_link]
[![Beacon][beacon]][website]

  [logo]: https://cloudposse.com/logo-300x69.svg
  [docs]: https://cpco.io/docs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=docs
  [website]: https://cpco.io/homepage?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=website
  [github]: https://cpco.io/github?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=github
  [jobs]: https://cpco.io/jobs?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=jobs
  [hire]: https://cpco.io/hire?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=hire
  [slack]: https://cpco.io/slack?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=slack
  [linkedin]: https://cpco.io/linkedin?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=linkedin
  [twitter]: https://cpco.io/twitter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=twitter
  [testimonial]: https://cpco.io/leave-testimonial?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=testimonial
  [office_hours]: https://cloudposse.com/office-hours?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=office_hours
  [newsletter]: https://cpco.io/newsletter?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=newsletter
  [discourse]: https://ask.sweetops.com/?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=discourse
  [email]: https://cpco.io/email?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=email
  [commercial_support]: https://cpco.io/commercial-support?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=commercial_support
  [we_love_open_source]: https://cpco.io/we-love-open-source?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=we_love_open_source
  [terraform_modules]: https://cpco.io/terraform-modules?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=terraform_modules
  [readme_header_img]: https://cloudposse.com/readme/header/img
  [readme_header_link]: https://cloudposse.com/readme/header/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=readme_header_link
  [readme_footer_img]: https://cloudposse.com/readme/footer/img
  [readme_footer_link]: https://cloudposse.com/readme/footer/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=readme_footer_link
  [readme_commercial_support_img]: https://cloudposse.com/readme/commercial-support/img
  [readme_commercial_support_link]: https://cloudposse.com/readme/commercial-support/link?utm_source=github&utm_medium=readme&utm_campaign=cloudposse/terraform-null-label&utm_content=readme_commercial_support_link
  [share_twitter]: https://twitter.com/intent/tweet/?text=terraform-null-label&url=https://github.com/cloudposse/terraform-null-label
  [share_linkedin]: https://www.linkedin.com/shareArticle?mini=true&title=terraform-null-label&url=https://github.com/cloudposse/terraform-null-label
  [share_reddit]: https://reddit.com/submit/?url=https://github.com/cloudposse/terraform-null-label
  [share_facebook]: https://facebook.com/sharer/sharer.php?u=https://github.com/cloudposse/terraform-null-label
  [share_googleplus]: https://plus.google.com/share?url=https://github.com/cloudposse/terraform-null-label
  [share_email]: mailto:?subject=terraform-null-label&body=https://github.com/cloudposse/terraform-null-label
  [beacon]: https://ga-beacon.cloudposse.com/UA-76589703-4/cloudposse/terraform-null-label?pixel&cs=github&cm=readme&an=terraform-null-label
