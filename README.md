# terraform_hints
Hints and Patterns that I have picked up using terraform over the years. All of this information assumes terraform v1.0.x and some or all of the information and examples here may or may not apply to earlier (or later) versions. Note that 99% of the terraform that I write is for AWS, so pretty much all of my examples will have to do with AWS resources.

## Style
In addition to adhering to [official style suggestions](https://www.terraform.io/docs/language/syntax/style.html) and what `terraform fmt` enforces, I use the following rules:

### General
- Always end a file with a [linefeed ending (LF)](https://www.terraform.io/docs/language/syntax/configuration.html#character-encoding-and-line-endings).
- Divide resources in a module, root or otherwise, into files by type. For instance:
    - `main.tf`
        - This file contains the terraform configuration block and the provider definitions.
    - `locals.tf`
        - This file defines local variables that are used throughout the resources in the other configuration files in this directory.
    - `data.tf`
        - Most application directories will also have a `data.tf` file that contains data lookups that are used by various resources and modules in the configuration files.  These commonly include [terraform_remote_state](https://www.terraform.io/docs/backends/types/s3.html#using-the-s3-remote-state) sources.
    - `versions.tf`
        - This file has the `terraform` restrictions to enforce a minimum required version of terraform itself and providers.
    - AWS specifics
        - `ec2.tf`
            - This would contain EC2 related resources:  ec2 instances, load balancers, autoscaling groups.
        - `s3.tf`
            - S3 buckets
        - `messaging.tf`
            - SQS queues, SNS topics, etc.
        - `iam.tf`
            - IAM users, policies, groups, roles.

### Collections
- [Lists](https://www.terraform.io/docs/language/expressions/types.html#list) and [maps](https://www.terraform.io/docs/language/expressions/types.html#map) should have their delimiters on separate lines from their contents.
    - If the collection is a sinelg entry, having it on one line is fine.
    ```terraform
    mylist = [ "only" ]
    mymap  = { only = "entry" }
    ```
- Use a trailing comma in lists. This makes diffs going forward cleaner.
```terraform
foo = {
  key1 = [
    "thing1",
    "thing2",
  ]
}
```

### Providers
- Always declare providers with an alias. This also forces every resource or module that uses that provider to explicitly declare that specific provider.
    - This convention makes introducing additional providers of the same type into the configuration less confusing.
- Always create providers in the root module, then [pass them into child modules](https://www.terraform.io/docs/language/meta-arguments/module-providers.html).
    - Do not rely on the implicit default provider(s) that the child module can inherit. This will lead to unforseen consequences.
    - I use a convention of an alias of `this` for providers in a child module.
    ```terraform
    # Root module
    provider "aws" {
      region = "us-east-1"
      alias  = "prod_east"
    }

    module "foo" {
      providers = {
        aws.this = aws.prod_east
      }
    ....
    }

    # Child module
    terraform {
      required_providers {
        aws = {
          source                = "hashicorp/aws"
          version               = "~> 3.0"
          configuration_aliases = [aws.this]
        }
      }
    }

    resource "aws_instance" "foo" {
      provider = aws.this
      ....
    }
    ```

### Modules
- Deal with providers as discussed above. Do not instantiate providers in a module and do not rely on default provider instantiations.
- Declare needed providers using `configuration_aliases` in the `required_providers` block of the `terraform` block. I use a convention of `this` being the alias of a passed-in provider. If multiple instances of a provider are needed, name them appropriately.
- Maintain versioning of your modules. I prefer each module being its own git repository. Adhereing to [semantic versioning](https://semver.org) provides convenient signalling of the severity of changes between releases.
    - A Major version upgrade is needed when `variable`s or `provider`s are added, removed, or changed from optional to required.
    - A Minor version upgrade is needed when a new optional variable is added, or some other new "feature" is added that does not change the "interface" of the module (no new required variables, no new required providers, etc).
    - A Patch version upgrade is when a bug in how the module behaves is fixed without changing the interface or intended usage.
- Modules contain at least 4 files:
    - `versions.tf`: contains the `terraform` block with [required_providers](https://www.terraform.io/docs/language/settings/index.html#specifying-provider-requirements) and the [required_version](https://www.terraform.io/docs/language/settings/index.html#specifying-a-required-terraform-version) to specify the version constraint for this module to work.
        - I generally use a `required_version = "~> 1.0.0"` currently since I am not sure what (if any) breaking changes may come with `1.1.x`.
    - `variables.tf`: contains all `variable` declarations for both required and optional variables.
    - `main.tf`: This is where I generally put all of the resources for a module. If the module is more complex, I may break out the resources into multiple files for better understandability.
    - `README.md`: Write up an explanation of and example usage of the module. Also use [terraform-docs](https://terraform-docs.io/) to generate documentation on the module as well.
- Name a module for the type of resources it is for and what it does. For instance, `iam_user` or `ec2_web_cluster` or `rds_aurora_pg_cluster`.
