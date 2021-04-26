---
title: Notifications, Alerts, Webhooks
author: Alison Dowdney
patat:
  incrementalLists: true
...

# Notifications, Alerts and Webhooks

"note we're not going to be covering them in this order"

# the plan

- Why you want notifications with Flux
- Setting up webhook receiver with github
- Setting up discord notifications with Flux

# Why

-  When you're looking at the terminal only you know whats going on,
-  Surfacing alerts through messaging apps, and other services gives your team visibility, and saves you time, as you don't have to relay as many messages, flux is doing that for you
-  Automation is one of the key principles of GitOps, less manual ops === more time for building cool things
-  and thats why automation is one of the key business values of gitops
-  read all about gitops.community

# tl;dr Flux

- The most powerful tool to get the GitOps experience
- A set of Kubernetes controllers
- GitOps based continuous delivery system

# kubernet get

Create a kubernetes cluster using your tool of choice

I'm using ~~k3ds~~ eks because we're doing some loadbalancey stuff
and I don't really want to expose my dev station to the world

# Bootstrap your cluster

- Export tokens
  ```bash
  $ export GITHUB_TOKEN=<your-token>
  $ export GITHUB_USER=<your-username>
  ```
- Bootstrap your cluster
  ```bash
  $ export REPO_NAME="woug-demo"
  $ flux bootstrap github \
    --owner=$GITHUB_USER \
    --repository=$REPO_NAME \
    --branch=main \
    --personal
  ```

- Clone the cluster repository, then cd into it
  ```bash
  $ git clone git@github.com:$GITHUB_USER/$REPO_NAME
  $ cd naw-example
  ```

# Creating webhook receiver secret

```bash
$ TOKEN=$(head -c 12 /dev/urandom | shasum | cut -d ' ' -f1)
$ echo $TOKEN

$ kubectl -n flux-system create secret generic webhook-token \    
--from-literal=token=$TOKEN
```

# Exposing the webhook receiver

- create yaml for load balancer
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: receiver
    namespace: flux-system
  spec:
    type: LoadBalancer
    selector:
      app: notification-controller
    ports:
      - name: http
        port: 80
        protocol: TCP
        targetPort: 9292
  ```

# Creating the receiver
- create the receiver 
  ```bash
  $ flux create receiver flux-system \
  --type github \
  --event ping \
  --event push \
  --secret-ref webhook-token \
  --resource GitRepository/flux-system \
  --namespace flux-system \
  --export > naw-receiver.yaml
  ```

- It should produce the following file
  ```yaml
  # naw-receiver.yaml
  apiVersion: notification.toolkit.fluxcd.io/v1beta1
  kind: Receiver
  metadata:
    name: flux-system
    namespace: flux-system
  spec:
    type: github
    events:
      - "ping"
      - "push"
    secretRef:
      name: webhook-token
    resources:
      - kind: GitRepository
        name: flux-system
  ```

- commit and push this all

# Setup GitHub Webook

- Navigate to your repo's settings, and webhooks
- Create new webhook
- htto://url of loadbalancer , http insecure + path of hook flux get receiver flux-system /hook
- ```$echo $TOKEN``` is the secret

# Create a Discord Webhook

* assuming you have already created a discord server, if not [discord.new](https://discord.new)
* Create a text channel for your alerts
* Select the settings cog on the text channel you want to post alerts into

* Select Create Webhook
* Give your Webhook a name, Copy the Webhook url, set aside details for later
* Save changes

# Define a provider

- Create a secret with the webhook url
```bash
$ kubectl -n flux-system create secret generic discord-url \ 
--from-literal=address=https://discord.com/api/webhooks/YOUR_DISCORD/WEBHOOK
```

- Create a notification provider for Discord by referencing the above secret
```bash
export $D_CHANNEL=""
export $D_BOTUSR=""
$ flux create alert-provider naw-provider \
--type discord \
--secret-ref discord-url \
--channel $D_CHANNEL \
--username $D_BOTUSR \
--export > naw-provider.yaml
```

- it should produce the following file
  ```yaml
  # naw-provider.yaml
  apiVersion: notification.toolkit.fluxcd.io/v1beta1
  kind: Provider
  metadata:
    name: naw-provder
    namespace: flux-system
  spec:
    channel: $D_CHANNEL
    secretRef:
      name: discord-url
    type: discord
    username: $D_BOTUSR
  ```

# Define an alert

- Create an alert definition for all repositories and Kustomizations
  ```bash
  $ flux create alert naw-alert \
  --provider-ref naw-provider \
  --event-severity info \
  --event-source Kustomization/'*' \
  --event-source GitRepository/'*' \
  --namespace flux-system \
  --export > naw-alert.yaml
  ```
  
- yaml output
  ```yaml
  # naw-alert.yaml
  apiVersion: notification.toolkit.fluxcd.io/v1beta1
  kind: Alert
  metadata:
    name: naw-alert
    namespace: flux-system
  spec:
    eventSeverity: info
    eventSources:
    - kind: Kustomization
      name: '*'
    - kind: GitRepository
      name: '*'
    providerRef:
      name: naw-provider # The name of the Provider
  ---
  ```

# Deploying the alert definitions to our cluster

Commit the files to the repository

```bash
$ git add naw-alert.yaml
$ git add naw-provider.yaml
$ git commit -sm "Add alert, alert provider for discord"
$ git push
```

# Checking everything's in order

Use kubectl to get the status of the alert

```bash
$ kubectl -n flux-system get alerts

NAME             READY   STATUS        AGE
naw-alert        True    Initialized   1m
```

# Testing it out : Creating a new deployment

- Create a GitRepository source for podinfo
  ```bash
  $ flux create source git podinfo \
      --url=https://github.com/stefanprodan/podinfo \
      --branch=master \
      --interval=30s \
      --export > ./podinfo-source.yaml
  ```
- Create a Kustomization for podinfo
  ```bash
  $ flux create kustomization podinfo \
      --source=podinfo \
      --path="./kustomize" \
      --prune=true \
      --validation=client \
      --interval=5m \
      --export > ./podinfo-kustomization.yaml
  ```
- Commit them to git
  ```bash
  $ git add podinfo-source.yaml
  $ git add podinfo-kustomization.yaml
  $ git commit -m "Add podinfo deployment"
  $ git push
  ```

# Testing it out : Creating some failures

- Create a invalid GitRepository source
  ```bash
  $ flux create source git nonexist \
    --url=https://github.com/alisondy/nonexist \
    --branch=main \
    --interval=30s \
    --export > ./nonexist-source.yaml
  $ git add nonexist-source.yaml
  $ git commit -m "Add a faulty source"
  $ git push
  ``` 
- Delete source 
  ```bash
  $ git rm nonexist-source.yaml
  $ git commit -m "Get rid of the non existent source"
  $ git push
  ``` 

# What did we just do

- we setup a webhook receiver using flux
- configured gitup to send push notifications to it
- we setup flux so it alerted in discord

- and with those skills you will be able to adopt one of the key gitops principles we covered earlier
- AUTOMATION!

# Next steps!

## Go to the docs!
- Check out our core concepts guide
- Check out our notification guide

## Try this demo out for yourself
- [Demo repo](github.com/alisondy/flux-demos/main/notifications-alerts-webhooks)

## Reach out to us on GitHub discussions!
- [Flux GH Discussions](github.com/fluxcd)