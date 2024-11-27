# Tf Rules

The Tf rules are useful to validate, lint and format terraform code.

They can typically be used in a terraform monorepo of modules to lint, run validation tests, auto generate documentation and enforce the consistency of providers versions across all modules.

# Why "Tf" and not "Terraform"

Because now you can either use "tofu" or "terraform" binary.

## Getting Started

To import rules_tf in your project, you first need to add it to your `MODULE.bazel` file:

```python
bazel_dep(name = "rules_tf", version = "0.0.9")
# git_override(
#     module_name = "rules_tf",
#     remote      = "https://github.com/yanndegat/rules_tf",
#     commit      = "...",
# )

tf = use_extension("@rules_tf//tf:extensions.bzl", "tf_repositories", dev_dependency = True)
tf.download( 
    version = "1.9.5", 
    tflint_version = "0.53.0", 
    tfdoc_version = "0.19.0", 
    use_tofu = False,
    mirror = {
        "random" : "hashicorp/random:3.3.2",
        "null"   : "hashicorp/null:3.1.1",
    } 
)

# Switch to tofu
# tf = use_extension("@rules_tf//tf:extensions.bzl", "tf_repositories")
# tf.download( 
#    version = "1.6.0", 
#    use_tofu = True,
#    mirror = {
#        "random" : "hashicorp/random:3.3.2",
#        "null"   : "hashicorp/null:3.1.1",
#    }
# )

use_repo(tf, "tf_toolchains")
register_toolchains(
    "@tf_toolchains//:all",
    dev_dependency = True,
)
```

Once you've imported the rule set , you can then load the tf rules in your `BUILD` files with:

```python
load("@rules_tf//tf:def.bzl", "tf_providers_versions", "tf_module")

tf_providers_versions(
    name = "providers",
    tf_version = "1.2.3",
    providers = {
        "random" : "hashicorp/random:>=3.3",
        "null"   : "hashicorp/null:>=3.1",
    },
)

tf_module(
    name = "root-mod-a",
    providers = [
        "random",
    ],
    deps = [
        "//tf/modules/mod-a",
    ],
    providers_versions = ":providers",
)
```

## Using Tf Modules

1. Using custom tflint config file

```python
load("@rules_tf//tf:def.bzl", "tf_module")

filegroup(
    name = "tflint-custom-config",
    srcs = [
        "my-tflint-config.hcl",
    ],
)

tf_module(
    name = "mod-a",
    providers = [
        "random",
    ],
    ...
    tflint_config = ":tflint-custom-config"
    
)
```

1. Generating versions.tf.json files

Terraform linter by default requires that all providers used by a module
are versioned. It is possible to generate a versions.tf.json file by running
a dedicated target:

```python
load("@rules_tf//tf:def.bzl", "tf_providers_versions", "tf_module")

tf_providers_versions(
    name = "providers",
    tf_version = "1.2.3",
    providers = {
        "random" : "hashicorp/random:3.3.2",
        "null"   : "hashicorp/null:3.1.1",
    },
)

tf_module(
    name = "root-mod-a",
    providers = [
        "random",
    ],
    deps = [
        "//tf/modules/mod-a",
    ],
    
    providers_versions = ":providers",
)
```

``` bash
bazel run //path/to/root-mod-a:gen-tf-versions
```

or generate all files of a workspace:

``` bash
bazel cquery 'kind(tf_gen_versions, //...)' --output files | xargs -n1 bash
```

1. Generating terraform doc files

It is possible to generate a README.md file by running
a dedicated target for terraform modules:

```python
load("@rules_tf//tf:def.bzl", "tf_gen_doc")

tf_gen_doc(
    name = "tfgendoc",
    modules = ["//{}/{}".format(package_name(), m) for m in subpackages(include = ["**/*.tf"])],
)
```

and run the following command to generate docs for all sub packages.

``` bash
bazel run //path/to:tfgendoc
```

It is also possible to customize terraform docs config:

```python
load("@rules_tf//tf:def.bzl", "tf_gen_doc")

filegroup(
    name = "tfdoc-config",
    srcs = [
        "my-tfdoc-config.yaml",
    ],
)

tf_gen_doc(
    name   = "custom-tfgendoc",
    modules = ["//{}/{}".format(package_name(), m) for m in subpackages(include = ["**/*.tf"])],
    config = ":tfdoc-config",
)
```

1. Formatting terraform files

It is possible to format terraform files by running a dedicated target:

```python
load("@rules_tf//tf:def.bzl", "tf_format")


tf_format(
    name = "tffmt",
    modules = ["//{}/{}".format(package_name(), m) for m in subpackages(include = ["**/*.tf"])],
)
```

and run the following command to generate docs for all sub packages.

``` bash
bazel run //path/to:tffmt
```
