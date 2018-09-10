# goblob

`goblob` is a tool for migrating Cloud Foundry blobs from one blobstore to
another. Presently it only supports migrating from an NFS blobstore to an
S3-compatible one.

## Installing

Download the [latest release](https://github.com/pivotal-cf/goblob/releases/latest).

### Install from source

Requirements:

* [glide](https://github.com/masterminds/glide)
* [go](https://golang.org)

```
mkdir -p $GOPATH/src/github.com/pivotal-cf/goblob
git clone git@github.com:pivotal-cf/goblob.git $GOPATH/src/github.com/pivotal-cf/goblob
cd $GOPATH/src/github.com/pivotal-cf/goblob
glide install
GOARCH=amd64 GOOS=linux go install github.com/pivotal-cf/goblob/cmd/goblob
```

## Usage

The tool is a Golang binary, which must be executed on the NFS VM that you intend to migrate. The only command of the tool is this:

`goblob migrate [OPTIONS]`

For each option you use, add `--` before the option name in the command you want to execute.

### Options

* `concurrent-uploads`: Number of concurrent uploads (default: 20)
* `exclude`: Directory to exclude (may be given more than once)

#### NFS-specific Options

* `blobstore-path`: The path to the root of the NFS blobstore, e.g. /var/vcap/store/shared

#### S3-specific Options

* `s3-endpoint`: The endpoint of the S3-compatible blobstore
* `s3-accesskey`: The access key to use with the S3-compatible blobstore
* `s3-secretkey`: The secret key to use with the S3-compatible blobstore
* `region`: The region to use with the S3-compatible blobstore
* `buildpacks-bucket-name`: The bucket containing buildpacks
* `droplets-bucket-name`: The bucket containing droplets
* `packages-bucket-name`: The bucket containing packages
* `resources-bucket-name`: The bucket containing resources
* `use-multipart-uploads`: Whether to use multi-part uploads
* `disable-ssl`: Whether to disable SSL when uploading blobs
* `insecure-skip-verify`: Skip server SSL certificate verification

## Example
```
goblob migrate --blobstore-path /var/vcap/store/shared \
  --s3-endpoint https://s3.amazonaws.com \
  --s3-accesskey MyKey \
  --s3-secretkey MySecret \
  --region us-east-1 \
  --buildpacks-bucket-name pcf-buildpacks \
  --droplets-bucket-name pcf-droplets \
  --packages-bucket-name pcf-packages \
  --resources-bucket-name pcf-resources
```

## Post-migration Tasks

- If your S3 service uses an SSL certificate signed by your own CA: Before applying changes in Ops Manager to switch to S3, make sure the root CA cert that signed the endpoint cert is a BOSH-trusted-certificate. You will need to update Ops Manager ca-certs (place the CA cert in /usr/local/share/ca-certificates and run update-ca-certificates, and restart tempest-web). You will need to add this certificate back in each time you do an upgrade of Ops Manager. In PCF 1.9+, Ops Manager will let you replace its own SSL cert and have that persist across upgrades.
- Update OpsManager File Storage Config to point at S3 blobstore using buckets (cc-buildpacks-<uniqueid>, cc-droplets-<uniqueid>, cc-packages-<uniqueid>, cc-resources-<uniqueid>)
- Click `Apply Changes` in Ops Manager
- Once changes are applied, re-run `goblob` to migrate any files which were created after the initial migration
- Validate apps can be restaged and pushed

## Removing NFS post-migration

- Turn off bosh resurrector (`bosh vm resurrection off`)
- In the IaaS console (e.g. AWS EC2, vCenter console, etc.), terminate all the CC VM jobs (Cloud Controller, Cloud Controller Worker, and Clock Global) + NFS (ensure the attached disks are removed as well). Note that your CF API services will stop being available at this point (running apps should continue to be available though). This step is required to ensure the removal of the NFS mount from these jobs.
- `bosh cck` the cf deployment to check for any errors with the bosh state. It should ask you if you want to delete references to the missing CC/NFS jobs, which you want to do.
- Go back to Ops Mgr and update your ERT configurations to zero NFS instances and re-add your desired instance counts for the CC jobs.
- Click `Apply Changes` in Ops Manager. After this deploy is finished, your CF API service availibility will resume.
- Turn back on the bosh vm resurrector, if it isn’t turned back on after your re-deploy (`bosh vm resurrection on`).

## Known Issues
- Starting with PCF 1.12, Ops Manager no longer allows you to select the S3 blobstorage configuration for ERT/PAS, instead of the internal NFS option, without also deleting the NFS server VM. This means that you should shut down the Cloud Controller VMs before you switch your configuration, as you will not have a chance to run a post-configuration-switch migration once the NFS server is gone.

## Developing

* Install [Docker](https://www.docker.com/products/docker)
* `docker pull minio/minio`

To run all of the tests in a Docker container:

`./testrunner`

To continually run the tests during development:

* `docker run -p 9000:9000 -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" -e "MINIO_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" minio/minio server /tmp`
* (in a separate terminal) `MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE MINIO_SECRET_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY ginkgo watch -r`
