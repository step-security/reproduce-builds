# reproduce-builds
[![build-reproducer](images/banner.png)](#)
[StepSecurity Release Monitor](https://blog.stepsecurity.io/introducing-stepsecurity-release-monitor-517ed10623a1) uses this repository to programatically reproduce builds for different open source projects. As of now, it has been setup to rebuild software written in the Go Programming Language. 

The `rebuild.yml` workflow takes the following inputs. It then downloads the source code, builds the code, and compares the checksum with the expected checksum. If the checksum matches, the workflow passes, else it fails. The expected checksum comes from the release artifacts of the software being rebuilt. 


```yaml
name: rebuild

on:
  workflow_dispatch:
    inputs:
      REPO:
        description: 'Repository to checkout'
        required: true
      COMMIT_SHA:
        description: 'Commit SHA to checkout to rebuild'
        required: true
      COMMAND:
        description: 'Build command to run'
        required: true
      GO_VERSION:
        description: 'Go version to use'
        required: true
      OUTPUT_FILE:
        description: 'Name of output binary file'
        required: true
      EXPECTED_CHECKSUM:
        description: 'Expected checksum of output binary file'
        required: true

permissions: read-all

jobs:
  rebuild:
    permissions:
      contents: read
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@629c2de402a417ea7690ca6ce3f33229e27606a5
      with:
        repository: ${{ github.event.inputs.REPO }}
        ref: ${{ github.event.inputs.COMMIT_SHA }}
         
    
    - name: Set up Go 
      uses: actions/setup-go@424fc82d43fa5a37540bae62709ddcc23d9520d4
      with:
        go-version: ${{ github.event.inputs.GO_VERSION }}
        
    - run: ${{ github.event.inputs.COMMAND }}
    - run: | 
        sha=($(shasum -a256 ${{ github.event.inputs.OUTPUT_FILE }}))
        echo $sha
        if [[ "$sha" != "${{ github.event.inputs.EXPECTED_CHECKSUM }}" ]]
        then
          echo "Checksum not as expected"
          exit 1
        fi
```

The build commands are stored in a `release-monitor.yml` file. These files can either be in the root of the repository or at https://github.com/step-security/secure-workflows/tree/main/knowledge-base/releases. 

### Example

The `release-monitor.yml` file for Fleet looks like this. The build commands for different release artifacts are listed in the `reproduce-build` section. 

``` yaml
name: "fleetdm release"
release-process:
  artifact-location:
    github-release:
      repo: fleetdm/fleet
  reproducible-build:
    - artifact: fleetctl_v{{ .Version }}_linux.tar.gz
      binary: fleetctl
      build-command: make deps; make generate; CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags="-X github.com/kolide/kit/version.appName=fleetctl -X github.com/kolide/kit/version.version={{ .Version }} -X github.com/kolide/kit/version.branch={{ .Branch }} -X github.com/kolide/kit/version.revision={{ .FullCommit }} -X github.com/kolide/kit/version.buildDate={{ time "2006-01-02"}} -X github.com/kolide/kit/version.buildUser=runner" ./cmd/fleetctl/
      go-version: 1.17.8
  pipeline:
    github-action:
      repo: fleetdm/fleet
      workflow: goreleaser-fleet.yaml
    branches: 
      - main
      - patch-fleet-v*
    tags:
      - fleet-v*
```

In this example, StepSecurity Release Monitor has fetched the actual checksum from the https://github.com/fleetdm/fleet/releases/tag/fleet-v4.15.0 release of Fleet and triggered workflows to rebuild the `fleetctl` binary. 

This particular workflow was triggered to rebuild `fleetctl` for Linux. 

https://github.com/step-security/reproduce-builds/runs/6782518145?check_suite_focus=true

The workflow passed, which means the checksum of `fleetctl` generated after rebuilding from source matched the expected checksum. This means the binary for `fleetctl` that is present in the Fleet Release Assets is as expected, and has not been tampered with during the build process.  
