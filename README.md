There's never been a better time to learn about Kubernetes, whether you are using it at work, want to better your skills or are starting a new project. This tutorial will help you set up Kubernetes on a Civo Instance using k3s. You'll also learn how to secure k3s and how to get access locally with `kubectl` through the new [k3sup](https://github.com/alexellis/k3sup) tool.

Once you have Kubernetes deployed you can start to explore the [Cloud Native Landscape](https://landscape.cncf.io) and the plethora of free, open-source tooling which is available.

## Pre-reqs

* You will need the Kubernetes CLI which is also called `kubectl`.
  Download it from the [Kubernetes Docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
	Follow the instructions for your own Operating System, then run: `kubectl version --client` to check it worked.

* MacOS, Linux or WSL on Windows 10
  Whilst any Operating System is compatible, it is recommended that you use MacOS or Linux so that you can benefit from using bash. Many blog posts and guides will assume this tool is available on your system.

## The Civo part

This part of the tutorial will help you create an Instance with Civo's CLI. There are several steps for the first time, you do this, but the second time you run through the instructions you can skip most of them such as installing the CLI or creating an SSH key.

There are two options, you can create the Civo Instance with the CLI (recommended) or with the UI.

### Option 1: Create your Civo Instance with the CLI

You'll need Ruby to use the Civo CLI, so first of all make sure you have Ruby installed.

If not get it from [here](https://www.ruby-lang.org/)

The Civo CLI comes in `gem` format, so you can download that with the following:

```sh
sudo gem install civo_cli
sudo gem install civo
```

If you run into any problems, please don't hesitate to reach out on Intercom or the community forum.

### Make sure you have an SSH key

We'll be using ssh to log in, so you'll need to make sure you have one. Then you'll have to add it to your Civo dashboard if you don't.

```sh
ls ~/.ssh/id_rsa.pub
```

If you see nothing come back then generate a new key and "hit enter" to every prompt.

```sh
ssh-keygen
```

Now run the following and copy it to the clipboard:

```sh
cat ~/.ssh/id_rsa.pub
```

* Click "SSH Keys" then "Add SSH Key"
* Enter a value for Name, then paste into "Public key"

Whenever you create a new Instance, you should click "SSH key" and then the name you entered above.

### Add your Civo API key to the CLI

Get your API key from https://civo.com/api/

```sh
# export KEY="DAb75oyqVeaE7BI6Aa74FaRSP0E2tMZXkDWLC9wNQdcpGfH51r"
# civo apikey add production $KEY
Saved the API Key DAb75oyqVeaE7BI6Aa74FaRSP0E2tMZXkDWLC9wNQdcpGfH51r as production
```

Now activate the key:

```sh
# civo api current production
The current API Key is now production
```

### Find your SSH key

You should only have one SSH key at this stage, but you may have more. Let's find it so that Civo can add it to our Instance automatically and allow us to log in without any password when using k3sup.

```sh
civo ssh ls

+--------------------------------------+-----------------------+----------------------------------------------------+
| ID                                   | Name                  | Fingerprint                                        |
+--------------------------------------+-----------------------+----------------------------------------------------+

```

### Create a Instance using the CLI

Find the template for Ubuntu, which is one of the most popular OSes we can use:

```sh
civo template list |grep "Ubuntu 18"
| 811a8dfb-8202-49ad-b1ef-1e6320b20497 | Ubuntu 18.04         |
```

We are looking for "18.04"

Now let's choose our own hostname for the server and provision the Instance:

```
# We can set our own hostname
export HOST="k3sup-1"

# Taken from the earlier step
export TEMPLATE="811a8dfb-8202-49ad-b1ef-1e6320b20497"

# Taken from the earlier step
export SSH_KEY_ID="123"

# civo instance create \
  --name ${HOST} \
	--ssh-key ${SSH_KEY_ID} \
  --template=${TEMPLATE} \
  --wait

Created instance k3sup-1
```

### Option 2: Create your Civo Instance with the UI

Pick the following options:

* Setup a new *Medium VM* (20 USD / mo)

This comes with 2CPU 4GB RAM and 50GB disk. The Small (10 USD / mo) with 2GB RAM is also workable if you want to stretch your credit out.

* For the Image, pick *Ubuntu 18.04 LTS*

* Initial user: *civo*

* Public IP Address: *Create*

* Select your SSH Key

Now create the instance and continue.

## Install Kubernetes with k3s

Civo has announced plans to launch a managed k3s service, which will mean that you can skip this whole guide and just type in: `civo kubernetes create`, but for the time being the team feel it's important to provide an alternative option.

This is where k3sup comes in. k3sup can take any VM, Instance, Raspberry Pi or laptop and install k3s, then bring back a correctly formatted `kubeconfig` file for your Kubernetes CLI `kubectl`.

The Github page describes the tool as enabling developers to get "from zero to KUBECONFIG in < 1 min".

Get `k3sup` by running the following on your computer:

```sh
curl -sLSf https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

Check it worked:

```sh
k3sup version
```

Get your IP for the instance you just created

```sh
export IP=$(civo instance public_ip -q ${HOST})
echo "The IP is: ${IP}"
```

Now run the tool and start your timers:

```sh
k3sup install --ip $IP --user civo
```

You will now see a `kubeconfig` file appear in your current directory.

```
$ ls
kubeconfig
```

Try connecting to the Kubernetes cluster directly from your computer:

```sh
export KUBECONFIG=`pwd`/kubeconfig
kubectl get node -o wide
```

You've now got a single-node Kubernetes cluster and can start running Pods. The documentation for Rancher states that k3s uses around 500MB of RAM for each server, so you should be able to run this in one of the cheapest Instances and make your money go even further than before.

If you ever lose your kubeconfig, you can get it back at any time with this command:

```sh
export HOST="k3sup-1"
export SERVER_IP=$(civo instance public_ip -q ${HOST})

k3sup install --ip $SERVER_IP --user civo --skip-install
```

Did it work? Let us know if you ran into any issues and feel free to tweet a screenshot if you liked it to [@CivoCloud](https://twitter.com/civocloud/).

### Add additional hosts (optional)

If you'd like to add some additional capacity to your cluster, you can add more Instances just like we did in the previous step and then run the `k3sup join` command.

> k3s will only use 75MB of RAM on each additional agent, which means you may even be able to use the smallest Instance types to save on costs. 

Let's use bash to automate adding in an additional 3 nodes.

We'll use a bash `for` loop which will run with the number `2`, `3` and `4`.

Save the following file as `add-agents.sh`:

```sh
#!/bin/bash

export USER="civo"
export HOST="k3sup-1"
# Taken from the earlier step
export SSH_KEY_ID="123"

export SERVER_IP=$(civo instance public_ip -q ${HOST})
echo "The IP of the server is: ${SERVER_IP}"

echo "Adding nodes 2, 3, and 4"

for i in {2..4};
do 
  export HOST="k3sup-agent-$i"

  # Taken from the earlier step
  export TEMPLATE="811a8dfb-8202-49ad-b1ef-1e6320b20497"

  civo instance create \
    --name ${HOST} \
    --ssh-key ${SSH_KEY_ID} \
    --template=${TEMPLATE} \
    --wait
done

echo Adding each new host into the cluster

for i in {2..4};
do 
  export HOST="k3sup-agent-$i"
  export IP=$(civo instance public_ip -q ${HOST})

  k3sup join --server-ip=${SERVER_IP} --ip=${IP} --user=${USER}
done
```

You may need to edit the lines for:

* `SSH_KEY_ID`
* `USER`
* `HOST`

Then execute it:

```sh
chmod +x add-agents.sh
./add-agents.sh
```

Now check that each node has come up and is listed as `Ready`:

```sh
kubectl get node -o wide
```

### Security & firewalls (recommended)

Now that you've created your single-node cluster and have access to Kubernetes, it's time to tighten up the security.

* Create a firewall
  Firewalls in Civo are whitelist-based, so allow:

  - `22` (SSH)
  - `80` (HTTP)
  - `443` (HTTPS)
  - `6443` (Kubernetes API)
  
  Now go into your Civo dashboard and select the firewall for your Instance

## Wrapping up

You now have full access to Kubernetes on your Instance. The k3sup tool will work with any VM you create whether on premises or in a cloud such as Civo. You can even [follow a micro-tutorial on the GitHub repository for your Raspberry Pi](https://github.com/alexellis/k3sup).

### Tear-down

If you'd like to tear things down run the following:

```sh
# Delete the server
civo instance delete k3sup-1

# If you created agents / additional nodes, then run:
civo instance delete k3sup-agent-2
civo instance delete k3sup-agent-3
civo instance delete k3sup-agent-4
```

To check everything is gone, type in: `civo instance ls`

### Keep on learning

Try out the [OpenFaaS on Civo guide](https://www.civo.com/learn/deploy-openfaas-with-k3s-on-civo) to get started with deploying applications to Kubernetes with ease.

[Learn about helm](https://helm.sh) the package manager for Kubernetes hosted by the CNCF, the home of Kubernetes.
