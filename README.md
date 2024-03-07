# Minikube Puppet Stack

This started out as weekend project to learn Kubernetes using minikube.
This project creates a multi-node Puppet stack. The configs are managed
with r10k and https://github.com/mdechiaro/puppet-control-repo.

This is not for production use.

## Features

* Puppet stack for config management inside minikube
* Centralized logging with fluentd
* Dashboards to monitor catalogs
* Github runners to keep r10k and puppetservers in sync with codebase

## TODO (in no order)
* Fix centralized logging to use something modern
* Add github actions-runner-controller to replace current setup
* Add lifecycle hooks to cleanup certs on terminated nodes

## Get Started

The `RUNNER_TOKEN` secret env variable is required for control repo r10k
syncs with github repositories. It is the API key given when setting up
self-hosted runners in Github.

```
kubectl create secret generic runner-token \
  --from-literal=RUNNER_TOKEN=<token>
```

The `POSTGRES_PASSWORD` secret env variable is required and can be set
with a rake task.

```
kubectl create secret generic postgres-password \
  --from-literal=POSTGRES_PASSWORD="$(rake generate_urandom_key)"
```

Run these commands to initialize.

```
rake minikube_config
minikube start
rake minikube_load_images[images.list]
kubectl apply -f puppet
```

## Puppetboard

The application requires a puppet cert in order to communicate with
PuppetDB.

```
rake generate_puppet_cert[puppetboard]
```

The application requires a secret key in its config.

```
kubectl create secret generic puppetboard-secret-key \
  --from-literal=PUPPETBOARD_SECRET_KEY="$(rake generate_urandom_key)"
```

## PuppetDB certs

Some applications, e.g. puppetboard, require Puppet certificates to
communicate with PuppetDB. This requires PuppetCA to generate a
certificate. A cert can be created with specific rake task. It outputs a
base64 encoded string that can be placed inside secret environment
variable.

```
rake generate_puppet_cert[app_name]
```

## External services

Use `minikube tunnel` in second terminal to test services with puppet
outside of minikube. Use `kubectl get services -o wide` to get those
external IPs. If external IPs are `<pending>`, then the tunnel isn't
setup yet.

Add `EXTERNAL-IP` output for puppet, puppetca services to `/etc/hosts`
on the external agent you want to register to this stack. IPs can
change, so check output of `kubectl get services -A -o wide` to get
correct IPs.

```
# /etc/hosts
10.99.237.81 puppet.default.svc.cluster.local
10.102.73.228 puppetca.default.svc.cluster.local
10.105.68.78 puppetboard.default.svc.cluster.local
```

You can verify dns is working with:

```
kubectl exec -i -t dnsutils -- nslookup <service>
```

## Workaround for iptables

If you have docker and lxd / lxc installed on same machine.

```
# https://discuss.linuxcontainers.org/t/lxd-and-docker-firewall-redux-how-to-deal-with-forward-policy-set-to-drop/9953
sudo iptables -I DOCKER-USER  -j ACCEPT
sudo iptables-save
```
