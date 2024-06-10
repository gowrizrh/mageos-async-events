# Magento Asynchronous Events

[![Integration Test](https://github.com/aligent/magento-async-events/actions/workflows/integration-tests.yml/badge.svg)](https://github.com/aligent/magento-async-events/actions/workflows/integration-tests.yml)
[![REST](https://github.com/aligent/magento-async-events/actions/workflows/api-functional-tests.yml/badge.svg)](https://github.com/aligent/magento-async-events/actions/workflows/api-functional-tests.yml)

A framework for reliably handling asynchronous events with Magento.

* **Asynchronous**: The module uses RabbitMQ (or DB queues) to leverage asynchronous message delivery.
* **Flexible**: Decoupling events and dispatches provide greater flexibility in message modelling.
* **Scalable**: Handles back pressure and provides an asynchronous failover model automatically.

## Support

| Async Events | Magento 2.3.x      | >= Magento 2.4.0 <= Magento 2.4.3 | >= Magento 2.4.4   |
|--------------|--------------------|-----------------------------------|--------------------|
| 2.x          | :white_check_mark: | :white_check_mark:                | :x:                |
| 3.x          | :x:                | :x:                               | :white_check_mark: |

## Installation

```
composer config repositories.mageos-async-events git https://github.com/mage-os/mageos-async-events.git
composer require mage-os/mageos-async-events
```

If you run into an error like "Could not find a version of package mage-os/mageos-async-events matching your minimum-stability (stable).", run this command instead:
```
composer require mage-os/mageos-async-events @dev
```

Install the module:
```
bin/magento setup:upgrade
```

## Usage

### Define an asynchronous event

Create a new `async_events.xml` under a module's `etc/` directory.

```xml
<?xml version="1.0"?>
<config
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="urn:magento:module:MageOS_AsyncEvents:etc/async_events.xsd"
>
    <async_event name="sales.order.created">
        <service class="Magento\Sales\Api\OrderRepositoryInterface" method="get"/>
    </async_event>
</config>
```

### Create Subscription

#### HTTP Subscription
```shell
curl --location --request POST 'https://m2.dev.aligent.consulting:44356/rest/V1/async_event' \
--header 'Authorization: Bearer TOKEN' \
--header 'Content-Type: application/json' \
--data-raw '{
    "asyncEvent": {
        "event_name": "sales.order.created",
        "recipient_url": "https://example.com/order_created",
        "verification_token": "fD03@NpYbXYg",
        "metadata": "http"
    }
}'
```

#### Amazon EventBridge Subscription
Requires the [EventBridge Notifier](https://github.com/aligent/magento2-eventbridge-notifier)

```shell
curl --location --request POST 'https://m2.dev.aligent.consulting:44356/rest/V1/async_event' \
--header 'Authorization: Bearer TOKEN' \
--header 'Content-Type: application/json' \
--data-raw '{
    "asyncEvent": {
        "event_name": "sales.order.created",
        "recipient_url": "arn:aws:events:ap-southeast-2:005158166381:rule/Test.EventBridge.Rule",
        "verification_token": "aIW0G9n3*9wN",
        "metadata": "event_bridge"
    }
}'
```

### Dispatch an asynchronous event
```php
public function execute(Observer $observer): void
{
    /** @var Order $object */
    $object = $observer->getEvent()->getData('order');

    // arguments are the inputs required by the service class in the asynchronous
    // event definition in async_events.xml
    // e.g: Magento\Sales\Api\OrderRepositoryInterface::get
    $arguments = ['id' => $object->getId()];
    $data = ['sales.order.created', $this->json->serialize($arguments)];

    $this->publisher->publish(
        QueueMetadataInterface::EVENT_QUEUE,
        $data
    );
}
```

Ensure the following consumers are running

```shell
bin/magento queue:consumer:start event.trigger.consumer
bin/magento queue:consumer:start event.retry.consumer
```

## Advanced Usage
Refer to the [Wiki](https://github.com/aligent/magento-async-events/wiki)


# Features

## Trace

All events are logged at the individual subscription level with a UUID.

All information from the first delivery attempt to the latest attempt is presented as a trace table. The event payload
is also available to view for investigation purposes.

![Event Trace Page](docs/trace.png)

## Retries

Events are automatically retried with exponential back off. The default retry limit is 5. The maximum backoff is
60 seconds.

The exponential backoff is calculated as `min(60, pow($deathCount, 2));`

| Attempt | Backoff     |
|---------|-------------|
| 1       | 1 second    |
| 2       | 4 seconds   |
| 3       | 9 seconds   |
| 4       | 16 seconds  |
| 5       | 25 seconds  |

To change the default retry limit visit Admin > Stores > Settings > Configuration > Advanced > System > Async Events and update `Maximum Deaths`.

![Retry Limit Config Page](docs/retry_limit_config.png)

## Replays

An event can be replayed independent of its status. This is useful to debug or replay an event when all retries are
exhausted.

Replays start a new chain of delivery attempts and will respect the same retry mechanism if they fail again.

## Lucene Query Syntax

All events are indexed in Elasticsearch by default. This allows you to search through events including the event payload!

The module supports [Lucene Query Syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html) to query event data like attributes.

The following attributes are available across all asynchronous events.

```
log_id
uuid
event_name
success
created
```
The following attributes differ between asynchronous event types.
```
data
```

### Examples

Assuming you have the following events configured
```
customer.created
customer.updated
customer.deleted
sales.order.created
sales.invoice.created
shipment.created
shipment.updated
shipment.deleted
```
You can query all customer events by using a wildcard like `event_name: customer.*` which matches the following events
```
customer.created
customer.updated
customer.deleted
```

You can query all created events like `*.created` which matches the following events
```
customer.created
sales.order.created
sales.invoice.created
shipment.created
```

You can further narrow down using the other available attributes such as status or uuid.

The following query returns all customer events which have failed. `customer.* AND success: false`

You can combine complex lucene queries to fetch event history and then export them via the admin grid as a csv if you wish.

#### Searching inside event payloads
Searching an event payload depends on what event you are searching on.

For the following example event payload, four properties are indexed as attributes. Therefore, you can query on
`data.customer_email`, `data.customer_firstname`, `data.customer_lastname` and `data.increment_id`.
Properties inside array at any level are not searchable.

```json
{
    "data": {
        "customer_email": "roni_cost@example.com",
        "customer_firstname": "Veronica",
        "customer_lastname": "Costello",
        "increment_id": "CK00000001",
        "payment_additional_info": [
            {
                "key": "method_title",
                "value": "Check / Money order"
            }
        ]
    }
}
```

Search all events where the customer email is `roni_cost@example.com`

`data.data.customer_email: roni_cost@example.com`

![Lucene Example 1](docs/lucene_example_1.png)

Search all events with the order increment id starting with `CK` and status success

`data.data.increment_id: CK* AND success: true`

![Lucene Example 2](docs/lucene_example_2.png)

To turn off asynchronous event indexing visit Admin > Stores > Settings > Configuration > Advanced > System >
Async Events and disable `Enable Asynchronous Events Indexing`.

## Related Modules

### Common Events

[Mage-OS Common Asynchronous Events](https://github.com/mage-os/mageos-common-async-events) includes several
default events for this module:

| Event identifier         | Description                                    |
|--------------------------|------------------------------------------------|
| customer.created         | Whenever a customer is created                 |
| customer.updated         | Whenever a customer is saved, except it's new  |
| customer.address.created | Whenever a customer address is created                |
| customer.address.updated | Whenever a customer address is saved, except it's new |
| sales.order.created      | When a new order is created                    |
| sales.order.updated      | When the state of an existing order is changed |
| sales.order.paid         | When an order is fully paid                    |
| sales.order.shipped      | When an order is fully shipped                 |
| sales.order.holded       | When an order is set "on hold"                 |
| sales.order.unholded     | When an order is released from "on hold"       |
| sales.order.cancelled    | When an order is cancelled                     |
| sales.shipment.created   | When a new shipment is created                 |
| sales.invoice.created    | When a new invoice is created                  |
| sales.invoice.paid       | When an invoice is paid                        |
| sales.creditmemo.created | When a new creditmemo is created               |

These events work out of the box and can be used within subcribers.
The module can also be used as a template on how to implement custom events.

### Admin UI

[Mage-OS Asynchronous Events Admin Ui](https://github.com/mage-os/mageos-async-events-admin-ui) provides
a simple interface to create subscribers for this module in the Magento Admin area instead of via REST API.
It only supports HTTP subscribers at the moment.

![Admin UI Form](docs/admin_ui_form.png)

