# terraform_hints
Hints and Patterns that I have picked up using terraform over the years.

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
    module "foo" {
      providers = {
        aws.this = aws.prod_east
      }
    ....
    }
    ```
