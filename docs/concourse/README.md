# Deploying Concourse on Google Compute Engine

This guide describes how to deploy [Concourse](http://concourse.ci/) on [Google Compute Engine](https://cloud.google.com/) using BOSH. You will deploy a BOSH director as part of these instructions.

## Prerequisites
* You must have the `terraform` CLI installed on your workstation. You can download and unzip [Terraform](https://www.terraform.io/downloads.html) to a directory of your chosing and then symlink the terraform executable to your `/usr/bin`. Or, if using a Mac, you can `brew install terraform` or `brew upgrade terraform` depending on whether or not you already have it installed.

* You must have the `gcloud` CLI installed on your workstation. Download it from [cloud.google.com/sdk](https://cloud.google.com/sdk/) and then proceed as follows:

````
$ cd /work   # this is where I chose to place my the gcloud CLI software
$ cp ~/Downloads/google-cloud-sdk-194.0.0-darwin-x86_64.tar.gz .
````
Unzip `google-cloud-sdk-194.0.0-darwin-x86_64.tar.gz` and proceed as follows:
````
$ cd /work/google-cloud-sdk/
$ ./install.sh
````

Using a new Terminal window:

````
$ cd /work/google-cloud-sdk/
$ ./bin/gcloud init
````
Let's check what do I have installed:
````
$ gcloud components list
````
Let's check what is my configuration:
````
$ gcloud config list
[compute]
region = us-east1
zone = us-east1-b
[core]
account = rmeira@pivotal.io
disable_usage_reporting = True
project = fe-rmeira
````

### Setup your workstation

1. Set your project ID:

  ```
  export projectid=REPLACE_WITH_YOUR_PROJECT_ID
  ```
  
   Which in my case is `export projectid=fe-rmeira; echo $projectid`

2. Export your preferred compute region and zone:

  ```
  export region=us-east1
  export zone=us-east1-b
  export zone2=us-east1-c
  ```

3. Configure `gcloud` with a user who is an owner of the project:

  ```
  gcloud auth login
  ```
  The command above will open up a browser at `https://cloud.google.com/sdk/auth_success` to authenticate you.
  
  Proceed with:
  ```
  gcloud config set project ${projectid}
  gcloud config set compute/zone ${zone}
  gcloud config set compute/region ${region}
  ```
   
4. Create a service account and key:

  ```
  gcloud iam service-accounts create terraform-bosh
  gcloud iam service-accounts keys create /tmp/terraform-bosh.key.json \
      --iam-account terraform-bosh@${projectid}.iam.gserviceaccount.com
  ```

5. Grant the new service account editor access to your project:

  ```
  gcloud projects add-iam-policy-binding ${projectid} \
      --member serviceAccount:terraform-bosh@${projectid}.iam.gserviceaccount.com \
      --role roles/editor
  ```

6. Make your service account's key available in an environment variable to be used by `terraform`:

  ```
  export GOOGLE_CREDENTIALS=$(cat /tmp/terraform-bosh.key.json)
  ```

### Create required infrastructure with Terraform

1. Download [main.tf](main.tf) and [concourse.tf](concourse.tf) from this repository.

2. In a terminal from the same directory where the two `.tf` files are located, execute `terraform init` and then view the Terraform execution plan to see the resources that will be created:

  ```
  $ terraform init
  ```
To view the execution plan:

  ```
  terraform plan -var projectid=${projectid} -var region=${region} -var zone-1=${zone} -var zone-2=${zone2}
  ```

3. Create the resources:

  ```
  terraform apply -var projectid=${projectid} -var region=${region} -var zone-1=${zone} -var zone-2=${zone2}
  ```

### Deploy a BOSH Director

1. SSH to the bastion VM you created in the previous step. All SSH commands after this should be run from the VM:

  ```
  gcloud compute ssh bosh-bastion-concourse
  ```

2. Configure `gcloud` to use the correct zone, region, and project:

  ```
  zone=$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/zone)
  export zone=${zone##*/}
  export region=${zone%-*}
  gcloud config set compute/zone ${zone}
  gcloud config set compute/region ${region}
  export project_id=`curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/project/project-id`
  ```

3. Explicitly set your secondary zone:

  ```
  export zone2=us-east1-c
  ```

4. Create a **password-less** SSH key:

  ```
  ssh-keygen -t rsa -f ~/.ssh/bosh -C bosh
  ```

5. Run this `export` command to set the full path of the SSH private key you created earlier:

  ```
  export ssh_key_path=$HOME/.ssh/bosh
  ```

5. Navigate to your [project's web console](https://console.cloud.google.com/compute/metadata/sshKeys) and add the new SSH public key by pasting the contents of ~/.ssh/bosh.pub:

  ![](../img/add-ssh.png)

  > **Important:** The username field should auto-populate the value `bosh` after you paste the public key. If it does not, be sure there are no newlines or carriage returns being pasted; the value you paste should be a single line.


6. Confirm that `bosh-init` is installed by querying its version:

  ```
  bosh-init -v
  ```

7. Create and `cd` to a directory:

  ```
  mkdir google-bosh-director
  cd google-bosh-director
  ```

8. Use `vim` or `vi` or `nano` to create a BOSH Director deployment manifest named `manifest.yml.erb`:

  ```
  ---
  <%
  ['region', 'project_id', 'zone', 'ssh_key_path'].each do |val|
    if ENV[val].nil? || ENV[val].empty?
      raise "Missing environment variable: #{val}"
    end
  end

  region = ENV['region']
  project_id = ENV['project_id']
  zone = ENV['zone']
  ssh_key_path = ENV['ssh_key_path']
  %>
  name: bosh

  releases:
    - name: bosh
      url: https://bosh.io/d/github.com/cloudfoundry/bosh?v=260.1
      sha1: 7fb8e99e28b67df6604e97ef061c5425460518d3
    - name: bosh-google-cpi
      url: https://bosh.io/d/github.com/cloudfoundry-incubator/bosh-google-cpi-release?v=25.6.2
      sha1: b4865397d867655fdcc112bc5a7f9a5025cdf311

  resource_pools:
    - name: vms
      network: private
      stemcell:
        url: https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=3312.12
        sha1: 3a2c407be6c1b3d04bb292ceb5007159100c85d7
      cloud_properties:
        zone: <%=zone %>
        machine_type: n1-standard-4
        root_disk_size_gb: 40
        root_disk_type: pd-standard
        service_scopes:
          - compute
          - devstorage.full_control

  disk_pools:
    - name: disks
      disk_size: 32_768
      cloud_properties:
        type: pd-standard

  networks:
    - name: vip
      type: vip
    - name: private
      type: manual
      subnets:
      - range: 10.0.0.0/29
        gateway: 10.0.0.1
        static: [10.0.0.3-10.0.0.7]
        cloud_properties:
          network_name: concourse
          subnetwork_name: bosh-concourse-<%=region %>
          ephemeral_external_ip: true
          tags:
            - bosh-internal

  jobs:
    - name: bosh
      instances: 1

      templates:
        - name: nats
          release: bosh
        - name: postgres
          release: bosh
        - name: powerdns
          release: bosh
        - name: blobstore
          release: bosh
        - name: director
          release: bosh
        - name: health_monitor
          release: bosh
        - name: google_cpi
          release: bosh-google-cpi

      resource_pool: vms
      persistent_disk_pool: disks

      networks:
        - name: private
          static_ips: [10.0.0.6]
          default:
            - dns
            - gateway

      properties:
        nats:
          address: 127.0.0.1
          user: nats
          password: nats-password

        postgres: &db
          listen_address: 127.0.0.1
          host: 127.0.0.1
          user: postgres
          password: postgres-password
          database: bosh
          adapter: postgres

        dns:
          address: 10.0.0.6
          domain_name: microbosh
          db: *db
          recursor: 169.254.169.254

        blobstore:
          address: 10.0.0.6
          port: 25250
          provider: dav
          director:
            user: director
            password: director-password
          agent:
            user: agent
            password: agent-password

        director:
          address: 127.0.0.1
          name: micro-google
          db: *db
          cpi_job: google_cpi
          user_management:
            provider: local
            local:
              users:
                - name: admin
                  password: admin
                - name: hm
                  password: hm-password
        hm:
          director_account:
            user: hm
            password: hm-password
          resurrector_enabled: true

        google: &google_properties
          project: <%=project_id %>

        agent:
          mbus: nats://nats:nats-password@10.0.0.6:4222
          ntp: *ntp
          blobstore:
             options:
               endpoint: http://10.0.0.6:25250
               user: agent
               password: agent-password

        ntp: &ntp
          - 169.254.169.254

  cloud_provider:
    template:
      name: google_cpi
      release: bosh-google-cpi

    ssh_tunnel:
      host: 10.0.0.6
      port: 22
      user: bosh
      private_key: <%=ssh_key_path %>

    mbus: https://mbus:mbus-password@10.0.0.6:6868

    properties:
      google: *google_properties
      agent: {mbus: "https://mbus:mbus-password@0.0.0.0:6868"}
      blobstore: {provider: local, path: /var/vcap/micro_bosh/data/cache}
      ntp: *ntp
  ```

9. Fill in the template values of the manifest with your environment variables:
  ```
  erb manifest.yml.erb > manifest.yml
  ```

10. Deploy the new manifest to create a BOSH Director:

  ```
  bosh-init deploy manifest.yml
  ```

11. Target your BOSH environment:

  ```
  bosh target 10.0.0.6
  ```

Your username is `admin` and password is `admin`.

### Deploy Concourse
Complete the following steps from your bastion instance.

1. Upload the required [Google BOSH Stemcell](http://bosh.io/docs/stemcell.html):

  ```
  bosh upload stemcell https://bosh.io/d/stemcells/bosh-google-kvm-ubuntu-trusty-go_agent?v=3263.8
  ```

2. Upload the required [BOSH Releases](http://bosh.io/docs/release.html):

  ```
  bosh upload release https://bosh.io/d/github.com/concourse/concourse?v=2.5.0
  bosh upload release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release?v=1.0.3
  ```

3. Download the [cloud-config.yml](cloud-config.yml) manifest file.

- change it's contents per the example below:

```
networks:
  - name: public
    type: manual
    subnets:
    - az: z1
      range: 10.150.0.0/24    <---- change to 10.120.0.0/24
      gateway: 10.150.0.1     <---- change to 10.120.0.1
      cloud_properties:
        network_name: concourse
        subnetwork_name: concourse-public-<%=region %>-1
        ephemeral_external_ip: true
        tags:
          - concourse-public
          - concourse-internal
    - az: z2
      range: 10.160.0.0/24    <---- change to 10.121.0.0/24
      gateway: 10.160.0.1     <---- change to 10.121.0.1
      cloud_properties:
        network_name: concourse
        subnetwork_name: concourse-public-<%=region %>-2
        ephemeral_external_ip: true
        tags:
          - concourse-public
          - concourse-internal
```

1. Download the [concourse.yml](concourse.yml) manifest file and set a few environment variables:

  ```
  export external_ip=`gcloud compute addresses describe concourse | grep ^address: | cut -f2 -d' '`
  export director_uuid=`bosh status --uuid 2>/dev/null`
  ```
  
  - Take note of the External_IP address. You will need it to connect to Concourse via the [Fly CLI](https://concourse-ci.org/fly-cli.html).
  ```
  echo $external_ip
  ```

1. Choose unique passwords for internal services and ATC and export them
   ```
   export common_password=<pick_one>
   export atc_password=<pick_one>
   ```

1. (Optional) Enable https support for concourse atc

  In `concourse.yml` under the atc properties block fill in the following fields:
  ```
  tls_bind_port: 443
  tls_cert: << SSL Cert for HTTPS >>
  tls_key: << SSL Private Key >>
  ```

1. Upload the cloud config:

  ```
  bosh update cloud-config cloud-config.yml
  ```

1. Target the deployment file and deploy:

  ```
  bosh deployment concourse.yml
  bosh deploy
  ```
# Let's connect to Concourse and try it out  
  
- As per tradition, there is a simple [Hello, world!](https://concourse-ci.org/hello-world.html) tutorial for you to try. This will at least show the basics of [fly](https://concourse-ci.org/fly-cli.html) (the CLI for Concourse).

- Check whether you already have `fly` installed. If you don't, click [here](https://concourse-ci.org/fly-cli.html) to install `fly`.

```
$ fly -v
3.5.0
```

- Connect to Concourse

```
$ fly -t concourse-rfm login -c http://35.197.133.183
logging in to team 'main'

WARNING: 
    fly version (3.5.0) is out of sync with the target (2.5.0). to sync up, run the following:
    fly -t concourse-rfm sync

username: concourse
password: password

target saved

$ fly -t concourse-rfm sync
downloading fly from http://35.197.133.183... 
 12.60 MB / 12.60 MB [===================================================================================================] 100.00% 11s
successfully updated from 3.5.0 to 2.5.0

$ fly -t concourse-rfm login -c http://35.197.133.183
username: concourse
password: password

target saved
```

- Let's Execute _Hello World_

```
cd /work/concourse   # This is a directory I created
vi hello.yml
```

###### _hello.yml_
```
jobs:
- name: hello-world
  plan:
  - task: say-hello
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: {repository: ubuntu}
      run:
        path: echo
        args: ["Hello, world!"]
```

Follow the steps shown below:

```
$ fly -t concourse-rfm set-pipeline -p hello-world -c hello.yml
jobs:
  job hello-world has been added:
    name: hello-world
    plan:
    - task: say-hello
      config:
        platform: linux
        image_resource:
          type: docker-image
          source:
            repository: ubuntu
        run:
          path: echo
          args:
          - Hello, world!
          dir: ""
    
apply configuration? [yN]: y
pipeline created!
you can view your pipeline here: http://35.197.133.183/teams/main/pipelines/hello-world

the pipeline is currently paused. to unpause, either:
  - run the unpause-pipeline command
  - click play next to the pipeline in the web ui
```

Ley's proceed:

```
$ fly -t concourse-rfm unpause-pipeline -p hello-world
unpaused 'hello-world'

$ fly -t concourse-rfm unpause-job -j hello-world/hello-world
unpaused 'hello-world'
```

