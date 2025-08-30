a compact, production-lean Azure Messaging System project you can drop into a repo and run end-to-end. It uses **Azure Service Bus (queue + topic)**, **Terraform** for IaC, and **Python** producer/consumer apps (with DLQ handling & retries), plus basic **monitoring alerts**.

---

# 🎯 What you’ll build

* **Service Bus Namespace** (Standard)
* **Queue**: `orders` (dead-letter enabled)
* **Topic**: `events` with 2 subscriptions: `billing`, `analytics`
* **Auth rule** to fetch connection string (Manage)
* **Python Producer** sending to queue & topic
* **Python Consumers** for queue & subscriptions (+DLQ reprocessor)
* **Metric alert** when queue backlog grows
* **Makefile** for handy commands

---

# 🧱 Repo structure

```
azure-messaging-system/
├─ infra/
│  ├─ main.tf
│  ├─ variables.tf
│  ├─ outputs.tf
│  └─ terraform.tfvars.example
├─ app/
│  ├─ producer.py
│  ├─ consume_queue.py
│  ├─ consume_subscription.py
│  ├─ reprocess_dlq.py
│  ├─ requirements.txt
│  └─ .env.example
├─ Makefile
└─ README.md
```

---

## 1) Terraform IaC (Azure Service Bus + Monitoring)

**infra/variables.tf**

```hcl
variable "project"        { type = string  }
variable "location"       { type = string  default = "eastus" }
variable "sb_sku"         { type = string  default = "Standard" }
variable "alert_email"    { type = string  }
variable "queue_name"     { type = string  default = "orders" }
variable "topic_name"     { type = string  default = "events" }
variable "tags"           { type = map(string) default = {} }
```

**infra/main.tf**

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.114"
    }
  }
}

provider "azurerm" {
  features {}
}

locals {
  rg_name       = "${var.project}-rg"
  sb_namespace  = "${var.project}-sb"
}

resource "azurerm_resource_group" "rg" {
  name     = local.rg_name
  location = var.location
  tags     = var.tags
}

resource "azurerm_servicebus_namespace" "sb" {
  name                = local.sb_namespace
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = var.sb_sku
  tags                = var.tags
}

resource "azurerm_servicebus_queue" "orders" {
  name                = var.queue_name
  namespace_id        = azurerm_servicebus_namespace.sb.id
  enable_partitioning = true
  dead_lettering_on_message_expiration = true
  lock_duration       = "PT45S"
  max_delivery_count  = 10
}

resource "azurerm_servicebus_topic" "events" {
  name         = var.topic_name
  namespace_id = azurerm_servicebus_namespace.sb.id
  enable_partitioning = true
}

resource "azurerm_servicebus_subscription" "billing" {
  name               = "billing"
  topic_id           = azurerm_servicebus_topic.events.id
  max_delivery_count = 10
}

resource "azurerm_servicebus_subscription" "analytics" {
  name               = "analytics"
  topic_id           = azurerm_servicebus_topic.events.id
  max_delivery_count = 10
}

# Namespace-level Shared Access Policy for connection string
resource "azurerm_servicebus_namespace_authorization_rule" "manage" {
  name                = "manage-policy"
  namespace_id        = azurerm_servicebus_namespace.sb.id
  manage              = true
  listen              = true
  send                = true
}

# Action group (email) for alerts
resource "azurerm_monitor_action_group" "email" {
  name                = "${var.project}-ag"
  resource_group_name = azurerm_resource_group.rg.name
  short_name          = "msgag"
  email_receiver {
    name          = "email"
    email_address = var.alert_email
  }
  tags = var.tags
}

# Metric alert when queue backlog grows
resource "azurerm_monitor_metric_alert" "queue_backlog" {
  name                = "${var.project}-queue-backlog"
  resource_group_name = azurerm_resource_group.rg.name
  scopes              = [azurerm_servicebus_queue.orders.id]
  description         = "Active messages in orders queue exceed threshold"
  severity            = 2
  frequency           = "PT1M"
  window_size         = "PT5M"
  target_resource_type = "Microsoft.ServiceBus/namespaces/queues"
  criteria {
    metric_namespace = "Microsoft.ServiceBus/namespaces"
    metric_name      = "ActiveMessages"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 100
  }
  action {
    action_group_id = azurerm_monitor_action_group.email.id
  }
  tags = var.tags
}
```

**infra/outputs.tf**

```hcl
output "resource_group" { value = azurerm_resource_group.rg.name }
output "namespace_name" { value = azurerm_servicebus_namespace.sb.name }

# Connection strings
output "namespace_primary_connection_string" {
  value     = azurerm_servicebus_namespace_authorization_rule.manage.primary_connection_string
  sensitive = true
}

output "queue_name" { value = azurerm_servicebus_queue.orders.name }
output "topic_name" { value = azurerm_servicebus_topic.events.name }
```

**infra/terraform.tfvars.example**

```hcl
project     = "ams-demo"
location    = "eastus"
alert_email = "your-email@example.com"
tags = { owner = "Atul", env = "dev" }
```

---

## 2) Python apps (producer/consumers)

**app/requirements.txt**

```
azure-servicebus==7.13.2
python-dotenv==1.0.1
```

**app/.env.example**

```
SERVICEBUS_CONNECTION_STRING=Endpoint=sb://...;SharedAccessKeyName=manage-policy;SharedAccessKey=...
QUEUE_NAME=orders
TOPIC_NAME=events
```

**app/producer.py**

```python
import os, json, uuid, time
from datetime import datetime
from dotenv import load_dotenv
from azure.servicebus import ServiceBusClient, ServiceBusMessage

load_dotenv()
CONN = os.environ["SERVICEBUS_CONNECTION_STRING"]
QUEUE = os.environ["QUEUE_NAME"]
TOPIC = os.environ["TOPIC_NAME"]

def send_queue_messages(sb: ServiceBusClient, n=10):
  with sb.get_queue_sender(queue_name=QUEUE) as sender:
    for i in range(n):
      order = {
        "order_id": str(uuid.uuid4()),
        "ts": datetime.utcnow().isoformat() + "Z",
        "amount": 49.99 + i
      }
      msg = ServiceBusMessage(json.dumps(order), application_properties={"type": "order"})
      sender.send_messages(msg)
      print(f"[queue] sent {order['order_id']}")

def send_topic_messages(sb: ServiceBusClient, n=5):
  with sb.get_topic_sender(topic_name=TOPIC) as sender:
    for i in range(n):
      evt = {
        "event_id": str(uuid.uuid4()),
        "type": "user.signup" if i % 2 == 0 else "purchase",
        "ts": datetime.utcnow().isoformat() + "Z"
      }
      msg = ServiceBusMessage(json.dumps(evt), subject=evt["type"])
      sender.send_messages(msg)
      print(f"[topic] sent {evt['event_id']} ({evt['type']})")

if __name__ == "__main__":
  sb = ServiceBusClient.from_connection_string(conn_str=CONN, logging_enable=True)
  send_queue_messages(sb, n=10)
  time.sleep(1)
  send_topic_messages(sb, n=6)
  print("Done.")
```

**app/consume\_queue.py**

```python
import os, json, time
from dotenv import load_dotenv
from azure.servicebus import ServiceBusClient, ServiceBusReceiveMode

load_dotenv()
CONN = os.environ["SERVICEBUS_CONNECTION_STRING"]
QUEUE = os.environ["QUEUE_NAME"]

if __name__ == "__main__":
  sb = ServiceBusClient.from_connection_string(CONN, logging_enable=True)
  with sb.get_queue_receiver(queue_name=QUEUE, receive_mode=ServiceBusReceiveMode.PEEK_LOCK) as receiver:
    while True:
      msgs = receiver.receive_messages(max_message_count=10, max_wait_time=5)
      if not msgs:
        time.sleep(2)
        continue
      for msg in msgs:
        try:
          body = json.loads(str(msg))
          print(f"[queue] processing order {body.get('order_id')} amount={body.get('amount')}")
          receiver.complete_message(msg)   # ACK
        except Exception as e:
          print("error:", e)
          receiver.abandon_message(msg)   # retry later
```

**app/consume\_subscription.py**

```python
import os, json, time, sys
from dotenv import load_dotenv
from azure.servicebus import ServiceBusClient, ServiceBusReceiveMode

load_dotenv()
CONN = os.environ["SERVICEBUS_CONNECTION_STRING"]
TOPIC = os.environ["TOPIC_NAME"]
SUB  = os.environ.get("SUBSCRIPTION_NAME", "billing")  # pass env to switch: billing/analytics

if __name__ == "__main__":
  sb = ServiceBusClient.from_connection_string(CONN, logging_enable=True)
  with sb.get_subscription_receiver(topic_name=TOPIC, subscription_name=SUB,
                                    receive_mode=ServiceBusReceiveMode.PEEK_LOCK) as receiver:
    while True:
      msgs = receiver.receive_messages(max_message_count=10, max_wait_time=5)
      if not msgs:
        time.sleep(2)
        continue
      for msg in msgs:
        try:
          body = json.loads(str(msg))
          print(f"[{SUB}] event type={msg.subject} payload={body}")
          receiver.complete_message(msg)
        except Exception as e:
          print("error:", e, file=sys.stderr)
          receiver.dead_letter_message(msg, reason="processing-failed", error_description=str(e))
```

**app/reprocess\_dlq.py**

```python
import os, json
from dotenv import load_dotenv
from azure.servicebus import ServiceBusClient

load_dotenv()
CONN = os.environ["SERVICEBUS_CONNECTION_STRING"]
TOPIC = os.environ["TOPIC_NAME"]
SUB   = os.environ.get("SUBSCRIPTION_NAME", "billing")

if __name__ == "__main__":
  sb = ServiceBusClient.from_connection_string(CONN)
  with sb.get_subscription_receiver(topic_name=TOPIC, subscription_name=SUB, sub_queue="deadletter") as dlq, \
       sb.get_topic_sender(topic_name=TOPIC) as sender:
    msgs = dlq.receive_messages(max_message_count=50, max_wait_time=5)
    for msg in msgs:
      try:
        body = json.loads(str(msg))
        print(f"[DLQ->{SUB}] re-publishing {body}")
        sender.send_messages(msg)  # simple replay
        dlq.complete_message(msg)
      except Exception as e:
        print("failed to reprocess:", e)
        dlq.abandon_message(msg)
```

---

## 3) Makefile (quality-of-life)

**Makefile**

```make
PROJECT ?= ams-demo
ENVFILE ?= app/.env

init:
	cd infra && terraform init

plan:
	cd infra && terraform plan -var-file=terraform.tfvars

apply:
	cd infra && terraform apply -auto-approve -var-file=terraform.tfvars

output:
	cd infra && terraform output -json | jq

venv:
	python3 -m venv .venv && . .venv/bin/activate && pip install -U pip && pip install -r app/requirements.txt

env:
	cp -n app/.env.example app/.env || true
	@echo "Fill SERVICEBUS_CONNECTION_STRING in app/.env"

produce:
	. .venv/bin/activate && export $$(grep -v '^#' $(ENVFILE) | xargs) && python app/producer.py

consume-queue:
	. .venv/bin/activate && export $$(grep -v '^#' $(ENVFILE) | xargs) && python app/consume_queue.py

consume-billing:
	. .venv/bin/activate && export $$(grep -v '^#' $(ENVFILE) | xargs) && SUBSCRIPTION_NAME=billing python app/consume_subscription.py

consume-analytics:
	. .venv/bin/activate && export $$(grep -v '^#' $(ENVFILE) | xargs) && SUBSCRIPTION_NAME=analytics python app/consume_subscription.py

reprocess-dlq:
	. .venv/bin/activate && export $$(grep -v '^#' $(ENVFILE) | xargs) && SUBSCRIPTION_NAME=billing python app/reprocess_dlq.py
```

---

## 4) Deploy & run — step by step

1. **Prereqs**

* Azure subscription + `az login`
* Terraform `>= 1.5`
* Python 3.9+
* `jq` (optional)

2. **Set up infra**

```bash
cd infra
cp terraform.tfvars.example terraform.tfvars
# update project, alert_email if needed
terraform init
terraform apply -auto-approve
```

3. **Grab connection string**

* Terraform output (sensitive):

  ```bash
  terraform output namespace_primary_connection_string
  ```
* Or via CLI:

  ```bash
  az servicebus namespace authorization-rule keys list \
    -g <your-rg> --namespace-name <your-ns> \
    --name manage-policy --query primaryConnectionString -o tsv
  ```

4. **Configure app env & venv**

```bash
make venv
make env   # then paste the connection string into app/.env
```

5. **Run producer**

```bash
make produce
```

6. **Run consumers (in separate terminals)**

```bash
make consume-queue
make consume-billing
make consume-analytics
```

7. **Test DLQ/reprocessing**

* Temporarily force an exception in `consume_subscription.py` (e.g., raise RuntimeError) to push messages to DLQ, then:

```bash
make reprocess-dlq
```

---

## 5) Operational notes & options

* **Scale-out consumers**: run multiple `consume_queue.py` instances; PEEK\_LOCK + `complete_message` ensures each message is processed once (with retries).
* **Retry & backoff**: add exponential delays on `abandon_message`; consider poison message handling.
* **Scheduled messages**: Service Bus supports scheduled delivery; you can use `ServiceBusMessage(..., scheduled_enqueue_time_utc=...)`.
* **Sessions**: for strict FIFO per key, create a session-enabled queue/topic and set `session_id` on messages; use session receivers in consumers.
* **Observability**: use Azure Portal → Service Bus → Metrics (Active/Dead-lettered messages, Incoming/Outgoing). Alerts already set for queue backlog.
* **Costs**: Standard SKU is usually fine; move to **Premium** for isolation/throughput.

---

## 6) README starter (drop into `README.md`)

````md
# Azure Messaging System (Service Bus)

## Architecture
- Service Bus Namespace (Standard)
- Queue: `orders`
- Topic: `events` with `billing` and `analytics` subscriptions
- Python producer/consumers
- Azure Monitor alert for queue backlog

## Quickstart
```bash
cd infra && terraform init && terraform apply -auto-approve
cd .. && make venv && make env
make produce
make consume-queue   # terminal 1
make consume-billing # terminal 2
make consume-analytics # terminal 3
````

```

---
