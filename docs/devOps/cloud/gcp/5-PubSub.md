---
tags:
  - GCP
  - pubSub
---

```shell
# create topic

gcloud pubsub topics create myTopic
gcloud pubsub topics create Test1

gcloud pubsub topics list

gcloud pubsub topics delete Test1

gcloud pubsub subscriptions create --topic myTopic mySubscription

gcloud pubsub subscriptions create --topic myTopic Test1

gcloud pubsub topics list-subscriptions myTopic

gcloud subscriptions delete Test1

gcloud pubsub topics publish myTopic --message "hello"

gcloud pubsub subscriptions pull mySubscription --auto-ack --limit=3

```

