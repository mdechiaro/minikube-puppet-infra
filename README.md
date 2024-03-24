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
* NATS.io queuing service
* Ad hoc certs for easy, secure communication between applications

## TODO (in no order)
* Add github actions-runner-controller to replace current setup
* Add topology diagram

## Get Started

The `RUNNER_TOKEN` secret env variable is required for control repo r10k
syncs with github repositories. It is the API key given when setting up
self-hosted runners in Github.

```
kubectl create secret generic runner-token \
  --from-literal=RUNNER_TOKEN=<token>
```

Run command to initialize.

```
rake setup_stack
```

## Ad hoc certs

Some applications, nats, and puppetboard for example, require Puppet
certificates to communicate securely. This requires PuppetCA to generate
a certificate. A cert can be created with specific rake task. It outputs
a cert bundle in `secrets/` that can be placed inside secrets. The name
should match the `Service` metadata name.

```
rake generate_puppet_cert[service_name]
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
kubectl apply -f tools
kubectl exec -i -t dnsutils -- nslookup <service>
```

## Workaround for iptables

If you have docker and lxd / lxc installed on same machine.

```
# https://discuss.linuxcontainers.org/t/lxd-and-docker-firewall-redux-how-to-deal-with-forward-policy-set-to-drop/9953
sudo iptables -I DOCKER-USER  -j ACCEPT
sudo iptables-save
```
