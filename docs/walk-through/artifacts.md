# Artifacts

**Note:**
You will need to configure an artifact repository to run this example.
[Configuring an artifact repository here](https://argoproj.github.io/argo-workflows/configure-artifact-repository/).

When running workflows, it is very common to have steps that generate or consume artifacts. Often, the output artifacts of one step may be used as input artifacts to a subsequent step.

The below workflow spec consists of two steps that run in sequence. The first step named `generate-artifact` will generate an artifact using the `whalesay` template that will be consumed by the second step named `print-message` that then consumes the generated artifact.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-passing-
spec:
  entrypoint: artifact-example
  templates:
  - name: artifact-example
    steps:
    - - name: generate-artifact
        template: whalesay
    - - name: consume-artifact
        template: print-message
        arguments:
          artifacts:
          # bind message to the hello-art artifact
          # generated by the generate-artifact step
          - name: message
            from: "{{steps.generate-artifact.outputs.artifacts.hello-art}}"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello world | tee /tmp/hello_world.txt"]
    outputs:
      artifacts:
      # generate hello-art artifact from /tmp/hello_world.txt
      # artifacts can be directories as well as files
      - name: hello-art
        path: /tmp/hello_world.txt

  - name: print-message
    inputs:
      artifacts:
      # unpack the message input artifact
      # and put it at /tmp/message
      - name: message
        path: /tmp/message
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["cat /tmp/message"]
```

The `whalesay` template uses the `cowsay` command to generate a file named `/tmp/hello-world.txt`. It then `outputs` this file as an artifact named `hello-art`. In general, the artifact's `path` may be a directory rather than just a file. The `print-message` template takes an input artifact named `message`, unpacks it at the `path` named `/tmp/message` and then prints the contents of `/tmp/message` using the `cat` command.
The `artifact-example` template passes the `hello-art` artifact generated as an output of the `generate-artifact` step as the `message` input artifact to the `print-message` step. DAG templates use the tasks prefix to refer to another task, for example `{{tasks.generate-artifact.outputs.artifacts.hello-art}}`.

Artifacts are packaged as Tarballs and gzipped by default. You may customize this behavior by specifying an archive strategy, using the `archive` field. For example:

```yaml
<... snipped ...>
    outputs:
      artifacts:
        # default behavior - tar+gzip default compression.
      - name: hello-art-1
        path: /tmp/hello_world.txt

        # disable archiving entirely - upload the file / directory as is.
        # this is useful when the container layout matches the desired target repository layout.   
      - name: hello-art-2
        path: /tmp/hello_world.txt
        archive:
          none: {}

        # customize the compression behavior (disabling it here).
        # this is useful for files with varying compression benefits, 
        # e.g. disabling compression for a cached build workspace and large binaries, 
        # or increasing compression for "perfect" textual data - like a json/xml export of a large database.
      - name: hello-art-3
        path: /tmp/hello_world.txt
        archive:
          tar:
            # no compression (also accepts the standard gzip 1 to 9 values)
            compressionLevel: 0
<... snipped ...>
```