[![SL Scan](https://github.com/logicalphase/google-cloud-build-command-wrapper/actions/workflows/shiftleft-analysis.yml/badge.svg)](https://github.com/logicalphase/google-cloud-build-command-wrapper/actions/workflows/shiftleft-analysis.yml)

# Google Cloud Build Command Wrapper

This is a thin command wrapper for use inside a Google Cloud Build step written in Go.
It can be especially useful when running large cloud builds using Terraform to prevent Cloud Build container force-terminations, but will work with any Cloud Build. 

## Authors

John Teague <john@logicalphase.com>
Paul Durivage <durivage@google.com>

## Version 

1.0.2 
Tested up to Go v1.17.3

## Use Case

This was written with a very narrow use case in mind: wrapping Terraform commands.  It performs the following steps:

1. Given a project and build ID, it retrieves the build state from the Cloud Build API
1. It reads the time at which the build will be force-terminated, looks at the `--before-timeout` value (default: 60 seconds), and sets a timer triggering at _build termination time_ minus the _before timeout_ value.
1. Runs the command supplied as its arguments as a child process
1. Waits until the command completes successfully OR...
1. Sends a signal (supplied with `--signal`, by default `SIGTERM`) to the child process when the timer triggers, allowing the process to gracefully terminate ahead of the Cloud Build force-termination

### But Why?

Google Cloud Builds have a timeout value, which by default is (currently) 10 minutes.  When this timeout for the build (and specifically for the build job itself) is reached, the build service sends a SIGTERM, followed by a very brief pause, and the container in which the step is running is force-terminated.  This has serious consequences for a stateful application like Terraform, which needs to terminate gracefully in order to stop its operations, update and write state, and to release the state lock. If a job is force-terminated, this cannot happen and can often have dire consequences, like orphaning resources, requiring human intervention, and a manual release of the state lock.

## Use

**Note:** This has been tested to target Go 1.17 and has not been tested with any version previously released.

 Building for Cloud Build:
 
```
GOOS=linux GOARCH=amd64 go build -o gcbcw github.com/logicalphase/google-cloud-build-command-wrapper
```

Cloud Build [steps](https://cloud.google.com/cloud-build/docs/build-config#build_steps) are simply container image tags, so to use it, you'll need to drop the compiled binary inside a container image and push it to a registry of your choice.  Since the canonical use case is in conjunction with Terraform, we'll just need a `Dockerfile` that drops the binary in a [Terraform image](https://hub.docker.com/r/hashicorp/terraform/):

```Dockerfile
FROM hashicorp/terraform:latest

COPY gcbcw /usr/local/bin/gcbcw
```

Build the `Dockerfile`.  In the below example, I'm building with a tag to ship this container image to a [Google Container Registry](https://cloud.google.com/container-registry/) repo in a GCP project under my control.

```bash
docker build -t gcr.io/logicalphase-gcbcw/gcbcw:latest

docker push gcr.io/logicalphase-gcbcw/gcbcw:latest
```

Last, use the image in your Cloud Build [configuration](https://cloud.google.com/cloud-build/docs/build-config). An [example](example/cloudbuild.yaml) has been provided.  Because Hashicorp's Terraform image has been defined with a custom `entrypoint`, you will need to override this in your build config with the [entrypoint](https://cloud.google.com/cloud-build/docs/build-config#entrypoint) flag.  Below I have provided an example build step which wraps the command `terraform apply -auto-approve`, signaling Terraform 2 minutes before the build job's scheduled timeout.

The below example uses [default variable substitutions](https://cloud.google.com/cloud-build/docs/configuring-builds/substitute-variable-values#using_default_substitutions) to populate the required project and build ID parameters.

```yaml
- name: gcr.io/logicalphase/gcbcw
  entrypoint: gcbcw
  args: ["--before-timeout", "2m", "$PROJECT_ID", "$BUILD_ID", "--", "terraform", "apply", "-auto-approve"]
```

## Help

A self-documenting `--help` command is available to show flags and parameters.

```
Usage of gcbcw: [flags ...] PROJECT_ID BUILD_ID -- COMMAND [command-flags ...]
  -t, --before-timeout string   time before build timeout to send designated signal; ex: 30s, 5m (default "60s")
  -h, --help                    print this usage and exit
  -q, --quiet                   suppress all output except process stdout and stderr
  -s, --signal string           signal to send to wrapped process (default "SIGTERM")
  -e, --timeout-exitcode int    non-zero exit code used if process is timed out; overrides process exit code
  -v, --verbose                 enable additional logging
```

## Disclaimer

This is not an official Google product.
