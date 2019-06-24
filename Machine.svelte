<script>
import { NO_OUTPUT } from "kingly"
import emitonoff from "emitonoff";

// props
export let commandHandlers;
export let effectHandlers;
export let fsm;
export let initEvent;

// Event emitter
const getEventEmitterAdapter = emitonoff => {
  const eventEmitter = emitonoff();
  const DUMMY_NAME_SPACE = "_";
  const subscribers = [];

  return {
      next: x => eventEmitter.emit(DUMMY_NAME_SPACE, x),
      complete: () => subscribers.forEach(f => eventEmitter.off(DUMMY_NAME_SPACE, f)),
      subscribe: ({ next: f, error: _, complete: __ }) => {
        return (subscribers.push(f), eventEmitter.on(DUMMY_NAME_SPACE, f));
      }
    }
};
const eventEmitter = getEventEmitterAdapter(emitonoff);
const next = eventEmitter.next.bind(eventEmitter);

// Subscribing to machine events
eventEmitter.subscribe({
	next: event => {
          // 1. Run the input on the machine to obtain the actions to perform
          const actions = fsm(event);

          // 2. Execute the actions, if any
          if (actions === NO_OUTPUT) {return void 0;}
          else {
            const filteredActions = actions.filter(action => action !== NO_OUTPUT);
            filteredActions.forEach(action => {
              const { command, params } = action;

              const commandHandler = commandHandlers[command];
              if (!commandHandler || typeof commandHandler !== "function") {
                throw new Error(
                  `Machine: Could not find command handler for command ${command}!`
                );
              }

              commandHandler(next, params, effectHandlers);
            });

            return void 0;
          }
        },
        error: error => {
          // We may get there for instance if there was a preprocessor throwing an exception
          console.error(
            `<Machine/>: an error in the event processing chain! The machine will not process
            any additional events. Remember that command handlers ought never throw, but should pass errors as events back to the mediator.`,
            error
          );
        },
        complete: () => {
        }
});

	initEvent && next(initEvent)

</script>

<slot dispatch={next}></slot>
