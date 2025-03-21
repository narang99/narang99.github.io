---
layout: post
title: "Jenkins: EC2 Fleet with native disk cache"
subtitle: a cached autoscaling jenkins agent setup
tags: [jenkins, aws, ci]
---

[EC2 Fleet](https://plugins.jenkins.io/ec2-fleet/) is an amazing Jenkins plugin which allows you to use AWS Autoscaling Groups as agents. Many mid-sized companies (which don't have many 24x7 developers working) have very fluctuating requirements for their CI systems. This makes EC2 Fleet a very cost-effective option. Consult the plugin documentation for setting it up, I'm not going to discuss that in this blog  
> **Using autoscaling groups has one problem though, you lose all your disk cache. Actions like `docker build` start taking a long time.**

Many commercial CI solutions provide on-demand caches. We don't get this functionality out of the box with Jenkins  



# What did not work out for me

- Networked File System: AWS provides EFS which is easy to setup. We wanted native FS performance though, NFS can have a reasonably big hit on FS intensive operations (`docker build` is certainly FS intensive). Some tools explicitly warn against using NFS ([Bazel](https://bazel.build) raises a warning the moment it notices its cache is residing on NFS) 
- [Multi-Attach EBS Systems](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volumes-multi.html): These require you to setup a clustered FS, which seemed like [a daunting task](https://github.com/aws-samples/clustered-storage-gfs2). Folks at [reddit](https://www.reddit.com/r/aws/comments/f41kb2/multiattach_for_provisioned_iops_io1_ebs_volumes/) also have mixed reactions to using this feature with EC2    
- Bake the cache into the AMI. Here the cache won't auto-update. You need to regularly "refresh" your AMI. ASGs also **download the AMI content on-demand from S3**, the first hit would still download the content from S3  

---

A cache which has the properties of a normal disk has some cool properties
- Native performance. No random dev blaming our FS instead of their tests ;)
- Your processes don't need to care about unintended concurrent access
    - As an example, you cannot run multiple docker daemons with the same data directory location. For this solution to work, you would need to somehow find an available directory in your shared cache and use it. You also need to make sure that the same directory is used as many times as possible, while being safe from parallel clashes

# Plain old EBS volumes as Cache volumes

The idea is simple. Keep a set of volumes ready for use as cache. In your ASG's user-data script, query any available EBS volume, attach dynamically to that volume at some constant directory (like `/mnt/cache/`). Your applications need to configure their "cache" locations to some directory inside `/mnt/cache` and they are good to go  
This simple trick is surprisingly reliable (thanks to AWS AMIs being really reproducible), easy to implement and very effective  

Before diving into the code, I assume you know how to attach an EBS volume to an EC2 instance. Check out [this](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html) and [this](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html) blogs if you don't  

---
The first step is create new volumes in AWS and give them some unique tag you can use to query them. We use `category: jenkins-agents`  

## Querying the volume to attach to
Querying the first available volume (we sort the volumes by their `created` time, to maximize cache hits)  
```python
# mount_volume.py
import boto3
import operator

# return the volume id to attach
def get_volume_id():
    client = boto3.client("ec2", region_name="ap-south-1")
    response = client.describe_volumes(
            Filters=[
                {
                    "Name": "status",
                    "Values": ["available"]
                },
                {
                    "Name": "tag:category",
                    "Values": ["jenkins-agents"]
                }
            ]
    )
    # we sort using created time, so as to access volumes in an ordered way.
    # This makes it easy to hit the cache
    volumes = sorted(response['Volumes'], key=operator.itemgetter('CreateTime'))
    if len(volumes) == 0:
        print("no volume found")
        # just signal that we failed
        # the main script simply does not mount anything
        return None

    return volumes[0]['VolumeId']
```
> Note that you could also randomly return one of the first `k` (say 3) volumes. cache volumes can fill up and you would need to add routine cleanup, randomizing might help decrease this pruning. We also do not explicitly handle race between multiple instances here, the randomization decreases the chances of the race (if two instances still end up trying to attach the same volume, one of them would fail and would launch without a cache volume. At our scale we didn't face this problem, you might want to solve this by taking a simple lock somewhere maybe)  

## Attaching the volume

```python
# mount_volume.py
import boto3

# the disk device we mount to. pick any random available device in your AMI
DEVICE_NAME = "/dev/sdh"


def attach_volume(volume_id):
    # current instance id

    iid = get_instance_id()
    print("will attach volume", volume_id, "to", iid, "at", device)
    client = boto3.client("ec2", region_name="ap-south-1")
    response = client.attach_volume(
        Device=DEVICE_NAME,
        InstanceId=iid,
        VolumeId=volume_id,
    )

def get_instance_id():
    resp = requests.get("http://169.254.169.254/latest/meta-data/instance-id")
    resp.raise_for_status()
    return resp.content.decode("utf-8")
```

**You would also need to wait for the volume to attach after this action.**  
This is the basic script. You might want to add some error handling here (it does work well enough though)  
```python
# mount_volume.py
def wait_for_attach(volume_id):
    print("waiting for attach", volume_id)
    state = 'attaching'
    while state != 'attached':
        state = get_volume_state(volume_id)
    print("attach done", volume_id)


def get_volume_state(volume_id):
    client = boto3.client("ec2", region_name="ap-south-1")
    response = client.describe_volumes(
            Filters=[
                {
                    "Name": "volume-id",
                    "Values": [volume_id],
                }
            ]
    )
    attachments = response['Volumes'][0]['Attachments']
    if len(attachments) == 0:
        return "attaching"
    return attachments[0]['State']
```
Lets mount the volume now. We will automate what is described in [this blog post](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)


## Finding the device name to mount

This is not very trivial, it required me to try some things out in the beginning.  
The basic mount command is very simple, you just run `mount "<device-name>" /mnt/cache` and it should just work. The problem here is that `/dev/sdh` did not work for device name for me. You need to use the NVMe variant in AWS (AWS suggests running `lsblk` to find the NVMe name that you would mount, its of the form `/dev/nvme<digit>`).  
You can try using an available `/dev/nvme<digit>` the first time you attach a volume above, but that was not working when I tried it out (I had to resort to using `/dev/sd` variant). I'm not sure what the reason is.  

Anyways, back to the implementation, we simply need to find the device the volume mounted on. Run this command
```bash
lsblk --json -o NAME,MOUNTPOINT,SERIAL
```

This gives an output like this
```json
{
   "blockdevices": [
      {"name":"nvme0n1", "mountpoint":null, "serial":"vol03ab0252469a225f2",
         "children": [
            {"name":"nvme0n1p1", "mountpoint":"/", "serial":null},
            {"name":"nvme0n1p14", "mountpoint":null, "serial":null},
            {"name":"nvme0n1p15", "mountpoint":"/boot/efi", "serial":null}
         ]
      },
   ]
}
```

We use the JSON output to make it easily parse-able from code. Note the attribute `blockdevices[0][serial]`, that looks a lot like a Volume ID.  
It actually is, simply remove the hyphen  
This basically gives us the disk we need to mount on. Lets get to the code  
```python
# mount_volume.py
def get_mounted_device_name(volume_id):
    import subprocess
    import json
    # run the command described above
    process = subprocess.run(
        "/usr/bin/lsblk --json -o NAME,MOUNTPOINT,SERIAL".split(),
        capture_output=True, text=True
    )
    blockdevices = json.loads(process.stdout)
    # vids in SERIAL do not have `-`, they just take it out
    normalized_vid = volume_id.replace("-", "")
    for device in blockdevices['blockdevices']:
        print(normalized_vid, device)
        if device['serial'] == normalized_vid:
            return f'/dev/{device["name"]}'
    print("could not find device")
```

## Mounting 
Finally here! The volume has attached to a device. You need to now `mount` it to `/mnt/cache` and instantiate an FS if its the first time its mounted


```bash
# mount_volume.sh
set -e
set +x

device_name="$1"

if [ "$(file -b -s $device_name)" == "data" ]; then
    # no FS already attached, attach it to the device
    echo "attached filesystem to $device_name"
    mkfs -t ext4 "$device_name"
fi
mkdir -p /mnt/cache || true
echo "mounting device to /mnt/cache"
mount "$device_name" /mnt/cache
```
We simply call this bash script from our python script with the device name extracted before  

```python
# mount_volume.py
def mount_volume(volume_id, device):
    cmd = f"/bin/bash /home/ubuntu/mount_volume.sh {device}"
    print("running", cmd)
    process = subprocess.run(
        cmd.split(), capture_output=True, text=True
    )
    print(process.stdout, getattr(process, 'stderr', "no stderr"))
    if process.returncode != 0:
        raise Exception("failed in mounting", volume_id, "at device", device)
    print("mounted volume_id", volume_id, "at device", device)
```



# Wrapping up

And thats it folks, we have attached the volume dynamically to an autoscaling agent on startup.   
You would need to add basic cleanup (if disk > 80% used, wipe it out) and some basic graceful degradation (in case of mount failure, simply create an empty directory `/mnt/cache`, this simply results in all cache misses, but the user jobs work as usual)  

The implications are amazing. You can use any location inside `/mnt/cache` as a normal file system.  

All pre-configured infra software also assumes this cache exists.   
Examples:  
- Its trivial to change the location of your docker daemon's data root in your user data (hint: set `data-root` property inside `/etc/docker/daemon.json` in your startup script)   
- Set repository cache location in `~/.bazelrc`
- Any of your builds also assume the location exists, in case you want to use it for other work. 