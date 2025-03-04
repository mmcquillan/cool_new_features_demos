# Cool Packer Feature Demos

## Breakpoints

From inside the cool_new_features_demos root, run

`packer build test_breakpoints.json`

The build should run straight through to completion; you should see output that
reads

>Breakpoint provisioner with note "this is a breakpoint" disabled; continuing...

Open up the test_breakpoints.json file and remove `"disable": true` from the
breakpont provisioner definition.

Run `packer build test_breakpoints.json` again. This time you'll see the output

>==> null: Pausing at breakpoint provisioner with note "this is a breakpoint".
>==> null: Press enter to continue.

The build will remain paused until you press enter to continue it, allowing
you all the time you need to navigate to investigate your build environment.

## Console

The packer console command allows you to experiment with Packer variable interpolations. You may access variables in the Packer config you called the console with, or provide variables when you call console using the -var or -var-file command line options.

```
$ packer console -var foo=fee test_user_var_interpolation.json
> {{ user `foo` }}-{{ user `bar` }}
fee-bananas-fee
```

The full list of options that the console command will accept is visible in the help output, which can be seen via packer console -h.



## Vault Integrations

Packer has two main Vault integrations: one for the key-value (kv) engine, and
one for the AWS engine. The kv engine integration supports both v1 and v2 of the
Vault kv engines.

### Vault KV engine integration

Kick off a dev server by calling `vault server -dev`

In the output of your dev call, look for the line that says "Root Token: " and
copy the token.

Open a new terminal window. Export the necessary vault environment variables:

```
export VAULT_TOKEN=$THE_ROOT_TOKEN_YOU_COPIED_EARLIER
export VAULT_ADDR='http://127.0.0.1:8200'
```

Now it's time to instantiate the key value store. From here on out the tutorial
will be different depending on whether you're using version 1 or version 2 of
the vault kv engine. We'll start with version 1 because it's newer and
recommended:

#### Version 2 KV engine

Take special note of the path to your data in the template: the engine reads

```
"{{ vault `/secret/data/hello` `foo`}}"
```

but below you'll notice that the path you give to the key when using the CLI is
`/secret/hello`. What's with that extra `data`?  This is an artifact of Packer
interacting with the Vault API instead of the Vault CLI. Don't worry too much
about it, just know that you'll have to stick "data" in between the name of your
kv engine (in this case, "secret"), and the name of your variable (in this case,
"hello").

To run the example template, run the following commands:

```
# you can skip this part if you already did it above
vault server -dev
export VAULT_TOKEN=$THE_ROOT_TOKEN_YOU_COPIED_EARLIER
export VAULT_ADDR='http://127.0.0.1:8200'

# set up the engine for the test
vault secrets enable kv
vault kv put secret/hello foo=world

# Run the Packer build
packer build test_vault_kv_1_integration.json
```

You should see the line

>    null: MYSECRET is world

Printed to the terminal.

Change the values in the file and in your vault to get a feel for how to
retrieve variables using this template engine.

#### Version 1 KV engine

```
# you can skip this part if you already did it above
vault server -dev
export VAULT_TOKEN=$THE_ROOT_TOKEN_YOU_COPIED_EARLIER
export VAULT_ADDR='http://127.0.0.1:8200'

# Set up the KV engine for the test
vault secrets enable -version=1 kv
vault kv put kv/my-secret my-value=s3cr3t

# Run the Packer build
packer build test_vault_kv_2_integration.json
```

You should see the line:

>    null: MYSECRET is s3cr3t

printed to the terminal.

Change the values in the file and in your vault to get a feel for how to
retrieve variables using this template engine.

### Vault AWS engine integration

NOTE: These examples will kick off actual Amazon instances and use actual Amazon
resources, which may translate into using actual money.

Before you start, you'll need a valid AWS access key and secret key for your
vault instance. I'll refer to them below as `$VAULT_AWS_ACCESS_KEY_ID` and
`$VAULT_AWS_SECRET_KEY_ID`.  You'll also need to have an idea of what
permissions you want the role applied to the credentials you generate for
Packer. Below is an example of a role that will function for Packer.

```
# you can skip this part if you already did it above
vault server -dev
export VAULT_TOKEN=$THE_ROOT_TOKEN_YOU_COPIED_EARLIER
export VAULT_ADDR='http://127.0.0.1:8200'

# set up the engine for the test
vault secrets enable aws
vault write aws/config/root access_key=$VAULT_AWS_ACCESS_KEY_ID secret_key=$VAULT_AWS_SECRET_KEY_ID region=us-east-1
vault write aws/roles/my-role \
policy=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
      "Effect": "Allow",
      "Action" : [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CopyImage",
        "ec2:CreateImage",
        "ec2:CreateKeypair",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteKeyPair",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DeregisterImage",
        "ec2:DescribeImageAttribute",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSnapshots",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume",
        "ec2:GetPasswordData",
        "ec2:ModifyImageAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifySnapshotAttribute",
        "ec2:RegisterImage",
        "ec2:RunInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource" : "*"
  }]
}
EOF
```

You'll see normal logging. If you have debug logs turned on, you may see a
line like

> 2019/08/28 13:43:28 packer: 2019/08/28 13:43:28 Retryable error: AuthFailure: AWS was not able to validate the provided access credentials
> 2019/08/28 13:43:28 packer: 	status code: 401, request id: 601f4ce1-1581-4da7-b6fc-ea9a8c576dd7

repeated a half dozen times or so; this is normal and is a result of AWS's cloud
being large and eventually consistent; we need to wait for the endpoint Packer
reaches out to to realize that Vault has indeed generated credentials for us
on-the-fly.


## Consul KV store Integration

We're going to make use of a consul dev server in order to demonstrate the
consul integration. To launch it, call

```
consul agent -dev
```

Then open a new terminal window, and put a key into the consul kv store. If you
want the example to work exactly with the test_consul_integration.json template
as written, you'll want your key to be named `myservice/version`.

```
consul kv put myservice/version 1.0
```

Now you can the test build: `packer build test_consul_integration.json`. You'll see
that Packer prints out a fake URL: `http://download.example.com/version/1.0`

If you change the kv version:

```
consul kv put myservice/version 1.5
```

And rerun your packer build: `packer build test_consul_integration.json`, you'll
notice that now the URL printed is different:
`http://download.example.com/version/1.5`


## Sensitive Variables

To run the example, simply call

```
packer build test_sensitive_vars.json
```

You should see the line

> null: MY PASSWORD IS \<sensitive\>

in your logs. Of equal importance, if you run with the Packer debug logs set:

`PACKER_LOG=1 packer build test_sensitive_vars.json`

You'll notice that the variable is also sanitized out of the verbose logging.

Example:

> 2019/08/28 15:32:35 packer: 2019/08/28 15:32:35 [INFO] (shell-local): starting local command: /bin/sh -c MYPASS='\<sensitive\>' PACKER_BUILDER_TYPE='null' PACKER_BUILD_NAME='null'  /var/folders/8t/0yb5q0_x6mb2jldqq_vjn3lr0000gn/T/packer-shell885086069
>2019/08/28 15:32:35 packer: 2019/08/28 15:32:35 [INFO] (shell-local communicator): Executing local shell command [/bin/sh -c MYPASS='\<sensitive\>' PACKER_BUILDER_TYPE='null' PACKER_BUILD_NAME='null'  /var/folders/8t/0yb5q0_x6mb2jldqq_vjn3lr0000gn/T/packer-shell885086069]

## Skipping post-processors

We've been able to skip provisioners using "except" for a long time; we have
recently added the ability to skip post-processors as well.

The Packer template test_except_for_postprocesors.json contains two
post-processors: a manifest post-processor, and a shell-local post-processor.

The shell-local one, if it runs, will create a file called "output.txt".
However, a post-processor has been tagged with `"except": ["null 1"]`, and this
step will therefore be skipped when the `"null 1"` builder is run.

The manifest post-processor is tagged with the opposite: `"only": ["null 2"]` and
therefore can only run after a builder named `null 2` -- Packer's `-only` 
flag only apply to builders in order to make it easier to reason about what will be
generated inside of a post-processor chain.

Inside the project root, run

`packer build -only="null 2" test_except_for_postprocesors.json`

When it has completed, you'll notice that the build has created a
packer-manifest.json file, but not an "output.txt" file.

## Interpolating variables within the "variables" template section

Until the beginning of this year, it wasn't possible to define a variable that
we interpolate from another variable defined in the "user" section of the
Packer template, but now you can do so.

`packer build test_user_var_interpolation.json`

You'll see the output

>    null: oranges-bananas-apples

Which is a variable interpolated from two other variables.

Open up the test_user_var_interpolation.json file, and delete the line
containing `"foo": "apples",` and then rerun.

`packer build test_user_var_interpolation.json`

The build will fail with

> Error initializing core: Failed to interpolate "bar": "bananas-{{user `foo`}}"; error: template: root:1:10: executing "root" at <user `foo`>: error calling user: Error: variable not set: foo: Please make sure that the variable you're referencing has been defined; Packer treats all variables used to interpolate other user varaibles as required.

Rather than interpolating based off an empty variable. However, if you run with

`packer build -var foo="peaches" test_user_var_interpolation.json`

The build will complete successfully with the line

>null: oranges-bananas-peaches

## Customizing allowed exit codes for shell and windows-shell provisioners

Note: These examples use real AWS resources which may translate into you
spending real money. It should be cheap, but don't get mad at us if it isn't.

The examples here only work if you have your `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY` environment variables set.

If you look inside the `test_windows_shell_exit_codes.json` and the
`test_custom_exit_codes_shell.json` packer templates, you'll see very simple
shell and windows-shell scripts that exit with error code 2. These scripts will
not fail the Packer builds, because the provisioners have added 2 to a list of
allowed exit codes.

The `valid_exit_codes` feature has not yet been implemented for the
shell-local provisoner or post-processor.

## Golang Template Engine Hacks

The golang template engine is much more powerful than the handful of engines
that we document on the Packer website. For example, did you know that you
perform conditional logic inside of the template engine, potentially changing
the file that a particular provisioner references based on a build name?

Take a look at the inline script inside of `test_template_engine_hacks.json`
for an example of this conditional logic.
