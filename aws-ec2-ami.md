## From bootc base image, create an Amazon Machine Image (AMI)  file and launch an ec2 instance

### Create an AMI

Use [bootc-image-builder](https://github.com/osbuild/bootc-image-builder) to create an AMI file.
First, update the [config.json](./bootc-build/qcow2/config.json) with your public SSH key contents.

This command will build an AMI and upload the AMI to a given AWS s3 bucket. The bucket must exist within
your AWS account.

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v $HOME/.aws:/root/.aws:ro \
    --env AWS_PROFILE=default \
    quay.io/centos-bootc/bootc-image-builder:latest \
    --type ami \
    --aws-ami-name fedora-bootc-ami \
    --aws-bucket fedora-bootc-bucket \
    --aws-region us-east-1 \
    --config /config.json \
    quay.io/centos-bootc/fedora-bootc:eln
```

This command will build an AMI and save it to the local directory `bootc-build/output/image/disk.raw`

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v $(pwd)/bootc-build/output:/output \
    quay.io/centos-bootc/bootc-image-builder:latest \
    --type ami \
    --config /config.json \
    quay.io/centos-bootc/fedora-bootc:eln
```

The file will be created as `./bootc-build/output/image/disk.raw`

### Launch an ec2 instance with terraform

This assumes an AMI `fedora-bootc-ami` exists in your AWS account and
you have terraform and AWS CLI `aws` installed on your local system.
There is a sample [terraform file](./terraform/main.tf). Customize this
based on the AWS account details. Then:

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

### Accessing the instance

If the AMI was built with the example [config.json](./bootc-build/config.json), you can access the system with

```
ssh -i /path/to/private/ssh-key fedora@ip-address
```

### Updating from the base image to a custom image

The [Containerfile](./Containerfile) describes how to create a derived OS image from the base `quay.io/centos-bootc/fedora-bootc:eln` image.
This Containerfile adds passwordless sudo as well as an autoupdate systemd service. With this service, any push to the bootc targeted image
will result in the virtual machine rebooting into the updated OS.

To build the derived OS image, run from the root of this repository

```
podman build -t quay.io/yourname/your-os:tag .
podman push quay.io/your-repo/your-os:tag
```

To switch the system to a custom `bootc` target image, ssh into the machine and run

```
sudo bootc switch quay.io/your-repo/your-os:tag
# quay.io/sallyom/fedora-coreos:autoupdate is a public image built from this repository
```

The system will pull down the necessary layers, and upon reboot will be enabled with an autoupdate service
for the target `quay.io/your-repo/your-os:tag`.

There is also an AI ChatApp quadlet included in this repository's Containerfile.
If `bootc switch`'d into this image, a (large) AI application will also be started. Remove [the chatapp quadlet](./usr/share/containers/systemd/chatapp.container)
before building your own derived OS image if this is not desired.

There is also a caddy file-server quadlet included in this repository's Containerfile.
If `bootc switch`'d into this image, a caddy application will be started. Remove [the caddy quadlet](./usr/share/containers/systemd/hello.container)
before building your own derived OS image if this is not desired.

