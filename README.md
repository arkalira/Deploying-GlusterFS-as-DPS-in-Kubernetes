# Deploying GlusterFS as DPS in Kubernetes
## Basic settings

 - 5 nodes
 - Debian 9
 - 1 GB RAM and 1 vCPU
 - 1 network interface

## Installation
### GlusterFS Server and client

 - Update, install dependencies and GlusterFS (server and client are needed in all nodes)

```
apt install thin-provisioning-tools glusterfs-server glusterfs-client
```
 - Check glusterfs-server status in all nodes

```
systemctl status glusterfs-server.service
```

### Install HEKETI

 - Install Heketi on one of the GlusterFS nodes, the project does not provide .deb packages (only rpm) so we need to install via tarball:
  - Link: https://github.com/heketi/heketi/releases/

```
wget https://github.com/heketi/heketi/releases/download/v6.0.0/heketi-v6.0.0.linux.amd64.tar.gz
tar xzvf heketi-v6.0.0.linux.amd64.tar.gz
cd heketi
cp heketi heketi-cli /usr/local/bin/
heketi -v
```
#### Create user

 - We also need to manually create the heketi user and the directory structures for the configuration:

```
groupadd -r -g 515 heketi
useradd -r -c "Heketi user" -d /var/lib/heketi -s /bin/false -m -u 515 -g heketi heketi
mkdir -p /var/lib/heketi && chown -R heketi:heketi /var/lib/heketi
mkdir -p /var/log/heketi && chown -R heketi:heketi /var/log/heketi
mkdir -p /etc/heketi
```
  - Now, we need to setup password-less ssh login between gluster nodes to let heketi execute commands as root

  - In the first node where Heketi is installed (glusterfs-micro01), execute the following:

```
ssh-keygen -f /etc/heketi/heketi_key -t rsa -N ''
chown heketi:heketi /etc/heketi/heketi_key*
ssh-copy-id -i /etc/heketi/heketi_key.pub root@[glusterfs-micro01|glusterfs-micro02|glusterfs-micro03|glusterfs-micro04|glusterfs-micro05]
```
  - You need to setup SSH to accept root password for login correctly or paste the pub_key of the source server in the auth_keys file of the destination host.

 - Test SSH access password-less

```
ssh -i /etc/heketi/heketi_key root@glusterfs-micro01
ssh -i /etc/heketi/heketi_key root@glusterfs-micro02
ssh -i /etc/heketi/heketi_key root@glusterfs-micro03
ssh -i /etc/heketi/heketi_key root@glusterfs-micro04
ssh -i /etc/heketi/heketi_key root@glusterfs-micro05
```

###  Edit heketi.json to configure access to API

```
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8888",

  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": true,

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "SUPERPASSWORD"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "PASSWORD"
    }
  },

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"http://K8S.HOST",
      "cert" : "PATH.TO.CERT.FILE",
      "insecure": false,
      "user": "admin",
      "password": "SUPERPASSWORD",
      "namespace": "k8s namespace",
      "fstab": "/etc/fstab"
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",
    "brick_max_size_gb" : 1024,
    "brick_min_size_gb" : 1,
    "max_bricks_per_volume" : 33,

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug"
  }
}
```

 - Now, lets create a systemctl unit to manage heketi: /etc/systemd/system/heketi.service

 ```
 [Unit]
Description=Heketi Server
Requires=network-online.target
After=network-online.target

[Service]
Type=simple
User=heketi
Group=heketi
PermissionsStartOnly=true
PIDFile=/run/heketi/heketi.pid
Restart=on-failure
RestartSec=10
WorkingDirectory=/var/lib/heketi
RuntimeDirectory=heketi
RuntimeDirectoryMode=0755
ExecStartPre=[ -f "/run/heketi/heketi.pid" ] && /bin/rm -f /run/heketi/heketi.pid
ExecStart=/usr/local/bin/heketi --config=/etc/heketi/heketi.json
ExecReload=/bin/kill -s HUP $MAINPID
KillSignal=SIGINT
TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
 ```

 - Start the service and check its status and listening port:

```
systemctl daemon-reload
systemctl start heketi.service
systemctl status heketi.service
systemctl enable heketi.service
netstat -putan | grep 8888
```

- Test the service from local node and remote nodes

 - Local node
```
root@glusterfs-micro01:~|⇒  curl http://glusterfs-micro01:8888/hello
Hello from Heketi#
```

 - Remote node

 ```
 root@glusterfs-micro03:~|⇒  curl http://glusterfs-micro01:8888/hello
Hello from Heketi#
 ```

 - And check the auth configured in heketi.json

 ```
 root@glusterfs-micro01:~|⇒  heketi-cli --server http://glusterfs-micro01:8888 --user admin --secret "SUPERPASSWORD" cluster list
Clusters:
 ```

 - All working, great.  

## Heketi Topology

### Configure topology of the glusterfs cluster

 - We have 5 hosts with glusterfs
```
root@glusterfs-micro01:~|⇒  host glusterfs-micro01.ironshared.com
glusterfs-micro01.ironshared.com has address 10.200.1.121
root@glusterfs-micro01:~|⇒  host glusterfs-micro02.ironshared.com
glusterfs-micro02.ironshared.com has address 10.200.1.122
root@glusterfs-micro01:~|⇒  host glusterfs-micro03.ironshared.com
glusterfs-micro03.ironshared.com has address 10.200.1.123
root@glusterfs-micro01:~|⇒  host glusterfs-micro04.ironshared.com
glusterfs-micro04.ironshared.com has address 10.200.1.124
root@glusterfs-micro01:~|⇒  host glusterfs-micro05.ironshared.com
glusterfs-micro05.ironshared.com has address 10.200.1.125
```

- Create topology.json FILE

```
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.200.1.121"
              ],
              "storage": [
                "10.200.1.121"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/vdb"
          ]
        },
        {
        "node": {
          "hostnames": {
            "manage": [
              "10.200.1.122"
            ],
            "storage": [
              "10.200.1.122"
            ]
          },
          "zone": 1
        },
        "devices": [
          "/dev/vdb"
        ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.200.1.123"
              ],
              "storage": [
                "10.200.1.123"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/vdb"
          ]
        },
        {
         "node": {
           "hostnames": {
             "manage": [
               "10.200.1.124"
             ],
             "storage": [
               "10.200.1.124"
             ]
           },
           "zone": 1
        },
        "devices": [
          "/dev/vdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "10.200.1.125"
              ],
              "storage": [
                "10.200.1.125"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/vdb"
          ]
        }
      ]
    }
  ]
}
```

 - Where /dev/vdb must be a RAW block device attached to each gluster node in topology.json file

 - Now we have to load the FILE

 ```
export HEKETI_CLI_SERVER=http://glusterfs-micro01:88888
export HEKETI_CLI_USER=admin
export HEKETI_CLI_KEY=SUPERPASSWORD
heketi-cli topology load --json=/opt/heketi/topology.json
Creating cluster ... ID: 87fcab5a999fd7a27d84444cea12e765
	Allowing file volumes on cluster.
	Allowing block volumes on cluster.
	Creating node 10.200.1.121 ... Unable to create node: New Node doesn't have glusterd running
	Creating node 10.200.1.122 ... Unable to create node: New Node doesn't have glusterd running
	Creating node 10.200.1.123 ... Unable to create node: New Node doesn't have glusterd running
	Creating node 10.200.1.124 ... Unable to create node: New Node doesn't have glusterd running
	Creating node 10.200.1.125 ... Unable to create node: New Node doesn't have glusterd running
 ```

 - And then it fails. :(
 - Then we forget about env variables and try to do it in one liner style

 ```
 heketi-cli --server http://glusterfs-micro01:8888 --user admin --secret "SUPERPASSWORD" topology load --json=/etc/heketi/topology.json
 ```
 - But fails anyway. Then check the following: dns resolution, chage hostnames in topology.json to IPs, ssh access, gluster peer probe "IP" test, netstat looking for glusterd... all is ok.

 - To continue...
