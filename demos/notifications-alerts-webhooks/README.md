# Notifications, Alerts, Webhooks Demo

Setting up Flux notifications in Discord.

## Prerequisites

To follow this guide you'll need 
  - a Kubernetes cluster
  - Flux CLI
  - Discord Server

## Create a Discord Webhoook

Select the settings cog on the text channel you want to post alerts into
![Hit the settings cog on the text channel](../../assets/Screen%20Shot%202021-04-15%20at%2013.03.17.png)

Select Create webhook
![Create a webhook](../../assets/Screen%20Shot%202021-04-15%20at%2011.20.31.png)

Give your webhook a name, Copy the webhook url, and set aside for later
![Webhook Details](../../assets/Screen%20Shot%202021-04-15%20at%2011.24.33.png)

Save changes
![Save webhook](../../assets/Screen%20Shot%202021-04-15%20at%2011.26.12.png)

## Bootstrap your cluster
```bash
$ export GITHUB_TOKEN=<your-token>
$ export GITHUB_USER=<your-username>

$ flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=naw-example \
  --branch=main \
  --personal

$ git clone git@github.com:$GITHUB_USER/naw-example
$ cd naw-example

```
## Define a provider

Create a secret with your Discord bot webhook

```bash
$ kubectl -n flux-system create secret generic discord-url \ 
  --from-literal=address=https://discord.com/api/webhooks/YOUR_DISCORD/WEBHOOK
```

Create a notification provider for Discord by referencing the above secret

```bash
$ flux create alert-provider naw-provider \
  --type discord \
  --secret-ref discord-url \
  --channel notifications-alerts-webhooks-example \
  --username notifications-alerts-webhooks-example-bot \
  --export > naw-provider.yaml
```

```yaml
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Provider
metadata:
  name: naw-provder
  namespace: flux-system
spec:
  channel: notifications-alerts-webhooks-example
  secretRef:
    name: discord-url
  type: discord
  username: notifications-alerts-webhooks-example-bot
```

## Define an alert

Create an alert definition for all repositories and Kustomizations

```bash
$ flux create alert naw-alert \
--provider-ref naw-provider \
--event-severity info \
--event-source Kustomization/'*' \
--event-source GitRepository/'*' \
--namespace flux-system \
--export > naw-alert.yaml
```

```yaml
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

With the files that you just created either :
  Apply them to your cluster
    or 
  Commit them to the repository you're using for flux

To verify that the alert has been acknowledged by the notification controller do:

```console
$ kubectl -n flux-system get alerts

NAME             READY   STATUS        AGE
on-call-webapp   True    Initialized   1m
```


## That's cool but theres' no alerts posting yet

To test things out, and see our alerts in action, we will deploy a test application.

```bash
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ./podinfo-source.yaml
```

```bash
flux create kustomization podinfo \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./podinfo-kustomization.yaml
```