# Notifications, Alerts, Webhooks Demo

Setting up Flux notifications in Discord.

flux bootstrap github \
--personal=true \
--private=false \
--owner "${GITHUB_USER}" \
--repository "flux-demos" \
--path "demos/notifications-alerts-webhooks/"


## Prerequisites

To follow this guide you'll need a Kubernetes cluster with Flux installed on it.
Please see the [get started guide](https://toolkit.fluxcd.io/get-started)
or the [installation guide](ttps://toolkit.fluxcd.io/installation).

The Flux controllers emit Kubernetes events whenever a resource status changes.
You can use the [notification-controller](https://toolkit.fluxcd.io/components/notification/controller) to forward these events to Discord.
The notification controller is part of the default Flux installation.

## Create a Discord Webhoook

![Create a webhook](../../assets/Screen%20Shot%202021-04-15%20at%2011.20.31.png)

![Webhook Details](../../assets/Screen%20Shot%202021-04-15%20at%2011.24.33.png)

![Save webhook](../../assets/Screen%20Shot%202021-04-15%20at%2011.26.12.png)

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
  channel: notificaitons-alerts-webhooks-example
  secretRef:
    name: discord-url
  type: discord
  username: notififcations-alerts-webhooks-example-bot
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

Apply the above files or commit them to the repository you're using for flux

To verify that the alert has been acknowledged by the notification controller do:

```console
$ kubectl -n flux-system get alerts

NAME             READY   STATUS        AGE
on-call-webapp   True    Initialized   1m
```