Enhanced Dead Letter Queue feature provides additional functionalities and warranties to the regular [Dead Letter Queue](Dead-Letter-Queue) feature. It aims to complement it with additional dispatch warranties and additional messages metadata information.

This documentation only covers extra functionalities enhancing the Dead Letter Queue feature.

Please refer to the [Dead Letter Queue](Dead-Letter-Queue) documentation for more details on its core principles.

## Using Enhanced Dead Letter Queue

There are no extra steps needed. If you are using Karafka Pro, Enhanced Dead Letter Queue is configured the same way as the regular one:

```ruby
class KarafkaApp < Karafka::App
  routes.draw do
    topic :orders_states do
      consumer OrdersStatesConsumer

      dead_letter_queue(
        topic: 'dead_messages',
        max_retries: 2
      )
    end
  end
end
```

## Disabling dispatch

For some use cases, you may want to skip messages after retries without dispatching them to an alternative topic.

To do this, you need to set the DLQ `topic` attribute value to `false`:

```ruby
class KarafkaApp < Karafka::App
  routes.draw do
    topic :orders_states do
      consumer OrdersStatesConsumer

      dead_letter_queue(
        topic: false,
        max_retries: 2
      )
    end
  end
end
```

When that happens, Karafka will retry two times and continue processing despite errors.

## Dispatch warranties

Enhanced Dead Letter Queue ensures that messages moved to the DLQ topic will always reach the same partition and in order, even when the DLQ topic has a different number of partitions. This means that you can implement pipelines for processing broken messages and rely on the ordering warranties from the original topic.

<p align="center">
  <img src="https://raw.githubusercontent.com/karafka/misc/master/charts/enhanced_dlq_flow.png" />
</p>
<p align="center">
  <small>*This example illustrates how Enhanced DLQ preserves order of messages from different partitions.
  </small>
</p>

**Note**: The DLQ topic does not have to have the same number of partitions as the topics from which the broken messages come. Karafka will ensure that all the messages from the same origin partition will end up in the same DLQ topic partition.

## Additional headers for increased traceability

Karafka Pro, upon transferring the message to the DLQ topic, aside from preserving the `payload`, and the `headers` will add a few additional headers that allow for increased traceability of broken messages:

- `original_topic` - topic from which the message came
- `original_partition` - partition from which the message came
- `original_offset` - offset of the transferred message

**Note**: Karafka headers values are **always** strings.

This can be used for debugging or for example when you want to have a single DLQ topic with per topic strategies:

```ruby
class DlqConsumer
  def consume
    messages.each do |broken_message|
      original_topic = broken_message.headers['original_topic']
      original_partition = broken_message.headers['original_partition'].to_i
      original_offset = broken_message.headers['original_offset'].to_i
      payload = broken_message.raw_payload

      case original_topic
      when 'orders_events'
        BrokenOrders.create!(
          payload: payload,
          source_partition: original_partition,
          source_offset: original_offset
        )
      when 'users_events'
        NotifyDevTeam.call(
          payload: payload,
          source_partition: original_partition,
          source_offset: original_offset
        )
      else
        raise StandardError, "Unsupported original topic: #{original_topic}"
      end

      mark_as_consumed(broken_message)
    end
  end
end
```