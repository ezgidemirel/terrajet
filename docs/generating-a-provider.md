# Generating a Crossplane Provider

In this guide, we will generate a Crossplane provider based on an existing
Terraform provider using Terrajet.

We have chosen [Terraform GitHub provider] as an example, but the process will
be quite similar for any other Terraform provider.

## Generate

1. Generate a GitHub repository for the Crossplane provider by hitting the
   "**Use this template**" button in [provider-jet-template] repository.
2. Clone the repository to your local and `cd` into the repository directory.
3. Replace `template` with your provider name.

    1. Export `ProviderName`:

        ```bash
        export ProviderNameLower=github
        export ProviderNameUpper=GitHub
        ```

    2. Run the `./hack/prepare.sh` script from repo root to prepare the repo, e.g., to
       replace all occurrences of `template` with your provider name:

        ```bash
       ./hack/prepare.sh
        ```

4. Configure your repo for the Terraform provider binary and schema:

    1. Update the following variables in `Makefile` for Terraform Provider:

        ```makefile
        export TERRAFORM_PROVIDER_SOURCE := integrations/github
        export TERRAFORM_PROVIDER_VERSION := 4.17.0
        export TERRAFORM_PROVIDER_DOWNLOAD_NAME := terraform-provider-github
        export TERRAFORM_PROVIDER_DOWNLOAD_URL_PREFIX := https://releases.hashicorp.com/terraform-provider-github/4.17.0
        ```

       You could find `TERRAFORM_PROVIDER_SOURCE` and `TERRAFORM_PROVIDER_VERSION` in
       [Terraform GitHub provider] documentation by hitting the "**USE PROVIDER**"
       button. Check [this line in controller Dockerfile] to see how these
       variables are used to build the provider plugin binary.

    2. Update import path of the Terraform provider schema package in the following
       **two files**: `cmd/generator/main.go` and `cmd/provider/main.go`

       Provider schema package is typically under the `<provider-name>`
       directory in the GitHub repository of the Terraform provider, e.g.
       in [`github` directory] for the GitHub provider.

       ```diff
       import (
               ...

       -       tf "github.com/hashicorp/terraform-provider-hashicups/hashicups"
       +       tf "github.com/turkenh/terraform-provider-github/v4/github"

               ...
       )

       ```

       Run:
       ```bash
       go mod tidy
       ```

       Please note, we are temporarily using a [fork] of
       [terraform-provider-github] repo as a workaround to [this issue].

    3. If your provider uses an old version (<v2) of [terraform-plugin-sdk],
       convert resource map to v2 schema as follows (in `cmd/generator/main.go`,
       uncomment related section):

       ```go
       import (
           "github.com/crossplane/terrajet/pkg/types/conversion"
       )

       func main() {
           ...

           resourceMap := conversion.GetV2ResourceMap(tf.Provider())
           pipeline.Run(config.GetProvider(resourceMap), absRootDir)
       }
       ```

       In a similar way in `cmd/provider/main.go`:

       ```go
       import (
           "github.com/crossplane/terrajet/pkg/types/conversion"
       )

       func main() {
           ...

           resourceMap := conversion.GetV2ResourceMap(tf.Provider())
           kingpin.FatalIfError(controller.Setup(mgr, log, rl, setup, ws, pconfig.GetProvider(resourceMap), 1), "Cannot setup Template controllers")
           kingpin.FatalIfError(mgr.Start(ctrl.SetupSignalHandler()), "Cannot start controller manager")
       }
       ```

       And in `go.mod` file, set the following `replace directive`
       (uncomment related section):

       ```
       github.com/hashicorp/terraform-plugin-sdk => github.com/turkenh/terraform-plugin-sdk v1.17.2-patch1
       ```

       Run:
       ```bash
       go mod tidy
       ```

5. Implement `ProviderConfig` logic. In `provider-jet-template`, there is already
   a boilerplate code in file `internal/clients/${ProviderNameLower}.go` which
   takes care of properly fetching secret data referenced from `ProviderConfig`
   resource.

   For our GitHub provider, we need to check [Terraform documentation for provider
   configuration] and provide the keys there:

   ```go
   const (
     keyBaseURL = "base_url"
     keyOwner = "owner"
     keyToken = "token"

     // GitHub credentials environment variable names
     envToken = "GITHUB_TOKEN"
   )

   func TerraformSetupBuilder(version, providerSource, providerVersion string) terraform.SetupFn {
     ...
     // set provider configuration
     ps.Configuration = map[string]interface{}{}
     if v, ok := githubCreds[keyBaseURL]; ok {
         ps.Configuration[keyBaseURL] = v
     }
     if v, ok := githubCreds[keyOwner]; ok {
         ps.Configuration[keyOwner] = v
     }
     // set environment variables for sensitive provider configuration
     ps.Env = []string{
         fmt.Sprintf(fmtEnvVar, envToken, githubCreds[keyToken]),
     }
     return ps, nil
   }
   ```

6. Before generating all resources that the provider has, let's go step by step
   and only start with generating `github_repository` and `github_branch`
   resources.

   To limit the resources to be generated, we need to provide an include list
   option with `tjconfig.WithIncludeList` in file `config/provider.go`:

   ```go
   pc := tjconfig.NewProvider(resourceMap, resourcePrefix, modulePath,
       tjconfig.WithDefaultResourceFn(defaultResourceFn),
       tjconfig.WithIncludeList([]string{
           "github_repository$",
           "github_branch$",
       }))
   ```

7. Finally, we would need to add some custom configurations for these two
   resources as follows:

   ```bash
   # Create custom configuration directory for whole repository group
   mkdir config/repository
   # Create custom configuration directory for whole branch group
   mkdir config/branch
   ```

   ```bash
   cat <<EOF > config/repository/config.go
   package repository

   import "github.com/crossplane/terrajet/pkg/config"

   // Configure configures individual resources by adding custom ResourceConfigurators.
   func Configure(p *config.Provider) {
       p.AddResourceConfigurator("github_repository", func(r *config.Resource) {
   
           // we need to override the default group that terrajet generated for
           // this resource, which would be "github"  
           r.ShortGroup = "repository"
       })
   }
   EOF
   ```

   ```bash
   cat <<EOF > config/branch/config.go
   package branch

   import "github.com/crossplane/terrajet/pkg/config"

   func Configure(p *config.Provider) {
       p.AddResourceConfigurator("github_branch", func(r *config.Resource) {
   
           // we need to override the default group that terrajet generated for
           // this resource, which would be "github" 
           r.ShortGroup = "branch"
   
           // Identifier for this resource is assigned by the provider. In other
           // words it is not simply the name of the resource.
           r.ExternalName = config.IdentifierFromProvider
   
           // This resource need the repository in which branch would be created
           // as an input. And by defining it as a reference to Repository
           // object, we can build cross resource referencing. See 
           // repositoryRef in the example in the Testing section below.
           r.References["repository"] = config.Reference{
               Type: "github.com/crossplane-contrib/provider-jet-github/apis/repository/v1alpha1.Repository",
           }
       })
   }
   EOF
   ```

   And register custom configurations in `config/provider.go`:

   ```diff
   import (
       tjconfig "github.com/crossplane/terrajet/pkg/config"
       "github.com/crossplane/terrajet/pkg/types/conversion"
       "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
       tf "github.com/turkenh/terraform-provider-github/v4/github"

   +   "github.com/crossplane-contrib/provider-jet-github/config/branch"
   +   "github.com/crossplane-contrib/provider-jet-github/config/repository"
    )

    func GetProvider() *tjconfig.Provider {
       ...
       for _, configure := range []func(provider *tjconfig.Provider){
           add custom config functions
   +           repository.Configure,
   +           branch.Configure,
       } {
               configure(pc)
       }
   ```

   **_To learn more about custom resource configurations (in step 7), please see
   the [Configuring a Resource](/docs/configuring-a-resource.md) document._**


8. Now we can generate our Terrajet Provider:

   ```bash
   make generate
   ```

### Adding New Resources

To add more resources, please **follow the steps between 6-8 for each resource**.

Alternatively, you can drop the `tjconfig.WithIncludeList` option in provider
Configuration which would generate all resources, and you can add resource
configurations as a next step.

## Test

Now let's test our generated resources.

1. First, we will create example resources under the `examples` directory:

   Create example directories for repository and branch groups:

   ```bash
   mkdir examples/repository
   mkdir examples/branch
   
   # remove the sample directory which was an example in the template
   rm -rf examples/sample
   ```

   Create a provider secret template:

   ```bash
   cat <<EOF > examples/providerconfig/secret.yaml.tmpl
   apiVersion: v1
   kind: Secret
   metadata:
     name: example-creds
     namespace: crossplane-system
   type: Opaque
   stringData:
     credentials: |
       {
         "token": "y0ur-t0k3n"
       }
   EOF
   ```

   Create example for `repository` resource, which will use
   `provider-jet-template` repo as template for the repository
   to be created:

   ```bash
   cat <<EOF > examples/repository/repository.yaml
   apiVersion: repository.github.jet.crossplane.io/v1alpha1
   kind: Repository
   metadata:
     name: hello-crossplane
   spec:
     forProvider:
       description: "Managed with Crossplane Github Provider (generated with Terrajet)"
       visibility: public
       template:
         - owner: crossplane-contrib
           repository: provider-jet-template
     providerConfigRef:
       name: default
   EOF
   ```

   Create `branch` resource which refers to the above repository
   managed resource:

   ```bash
   cat <<EOF > examples/branch/branch.yaml
   apiVersion: branch.github.jet.crossplane.io/v1alpha1
   kind: Branch
   metadata:
     name: hello-terrajet
   spec:
     forProvider:
       branch: hello-terrajet
       repositoryRef:
         name: hello-crossplane
     providerConfigRef:
       name: default
   EOF
   ```

2. Generate a [Personal Access Token](https://github.com/settings/tokens) for
   your Github account with `repo/public_repo` and `delete_repo` scopes.

3. Create `examples/providerconfig/secret.yaml` from
   `examples/providerconfig/secret.yaml.tmpl` and set your token in the file:

   ```bash
   GITHUB_TOKEN=<your-token-here>
   cat examples/providerconfig/secret.yaml.tmpl | sed -e "s/y0ur-t0k3n/${GITHUB_TOKEN}/g" > examples/providerconfig/secret.yaml
   ```

4. Apply CRDs:

   ```bash
   kubectl apply -f package/crds
   ```

5. Run the provider:

   ```bash
   make run
   ```

6. Apply ProviderConfig and example manifests (_In another terminal since
   the previous command is blocking_):

   ```bash
   # Create "crossplane-system" namespace if not exists
   kubectl create namespace crossplane-system --dry-run=client -o yaml | kubectl apply -f -

   kubectl apply -f examples/providerconfig/
   kubectl apply -f examples/repository/repository.yaml
   kubectl apply -f examples/branch/branch.yaml
   ```

7. Observe managed resources and wait until they are ready:

   ```bash
   watch kubectl get managed
   ```

   ```bash
   NAME                                                   READY   SYNCED   EXTERNAL-NAME                     AGE
   branch.branch.github.jet.crossplane.io/hello-terrajet   True    True     hello-crossplane:hello-terrajet   89s

   NAME                                                             READY   SYNCED   EXTERNAL-NAME      AGE
   repository.repository.github.jet.crossplane.io/hello-crossplane   True    True     hello-crossplane   89s
   ```

   Verify that repo `hello-crossplane` and branch `hello-terrajet` created under
   your GitHub account.


9. Cleanup

   ```bash
   kubectl delete -f examples/branch/branch.yaml
   kubectl delete -f examples/repository/repository.yaml
   ```

   Verify that the repo got deleted once deletion is completed on the control
   plane.


[comment]: <> (References)

[Terraform GitHub provider]: https://registry.terraform.io/providers/integrations/github/latest/docs
[provider-jet-template]: https://github.com/crossplane-contrib/provider-jet-template
[Terraform documentation for provider configuration]: https://registry.terraform.io/providers/integrations/github/latest/docs#argument-reference
[`github` directory]: https://github.com/integrations/terraform-provider-github/tree/main/github
[this line in controller Dockerfile]: https://github.com/crossplane-contrib/provider-jet-template/blob/d9a793dd8a304f09bb2e9694c47c1bade1b6b057/cluster/images/provider-jet-template-controller/Dockerfile#L18-L25
[fork]: https://github.com/turkenh/terraform-provider-github
[terraform-provider-github]: https://github.com/integrations/terraform-provider-github
[terraform-plugin-sdk]: https://github.com/hashicorp/terraform-plugin-sdk
[this issue]: https://github.com/integrations/terraform-provider-github/pull/961
