# Custom Producers

If you want to use Broadway but there is no existing Broadway producer
for the technology of your choice, you can integrate any existing GenStage
producer into the pipeline with relative ease.

## Example

In general, producers must generate `%Broadway.Message{}` structs in order
to be processed by Broadway. In case you need to use an existing GenStage
producer and you don't want to change its original implementation,
you'll have to set the producer's `:transformer` option to translate the
generated events into Broadway messages.

In the following example the producer is a regular GenStage, i.e., it
produces plain events that cannot be processed by Broadway directly:

    defmodule Counter do
      use GenStage

      def start_link(number) do
        GenStage.start_link(Counter, number)
      end

      def init(counter) do
        {:producer, counter}
      end

      def handle_demand(demand, counter) when demand > 0 do
        events = Enum.to_list(counter..counter+demand-1)
        {:noreply, events, counter + demand}
      end
    end

By using a transformer, you can tell Broadway to transform all events
generated by the producer into proper Broadway messages:

    defmodule MyBroadway do
      use Broadway

      @behaviour Broadway.Acknowledger

      alias Broadway.Message

      def start_link(_opts) do
        Broadway.start_link(__MODULE__,
          name: __MODULE__,
          producer: [
            module: {Counter, 1},
            transformer: {__MODULE__, :transform, []}
          ],
          processors: [
            default: [concurrency: 10]
          ],
          batchers: [
            default: [concurrency: 2, batch_size: 5],
          ]
        )
      end

      ...callbacks...

      def transform(event, _opts) do
        %Message{
          data: event,
          acknowledger: {__MODULE__, :ack_id, :ack_data}
        }
      end

      @impl Broadway.Acknowledger
      def ack(:ack_id, successful, failed) do
        # Write ack code here
        :ok
      end
    end

Notice that you need to pass two options to the producer:

  * `:module` - a tuple representing the GenStage producer as `{mod, arg}`.
    Where `mod` is module that implements the GenStage behaviour and `arg`
    the argument that will be given to the `init` callback of the GenStage.
    It is very important to note that Broadway **will not call** the
    `child_spec/1` or `start_link/1` function of the producer. That's
    because Broadway wraps the producer to augment it with extra features.

  * `:transformer` - a module-function-args tuple that will be invoked
    inside the producer, for every producer message, that should create
    a `Broadway.Message` struct with the `data` and `acknowledger` fields.

See the `Broadway.Acknowledger` module for more information on defining
and setting up acknowledgements.
