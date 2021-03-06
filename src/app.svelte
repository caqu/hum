<script>
  import { onMount, setContext } from "svelte";
  import { writable, readable, derived } from "svelte/store";
  import parseq from "./lib/parseq.js";
  import bundled_activities from "./bundled_activities"; // find them at index.js

  import activity_list from "./activity_list.js";
  import MenuButton from "./lib/menu_button.svelte";
  import program from "./program.js";

  import Editor from "./editor.svelte";

  const log = console.log;
  const loaded_activities = writable({});
  let activity_index = writable(-1); // TODO initial value should be 0!
  const debugging = writable(true);
  const { initial_state, initial_rule_set, initial_control_flow } = program;
  // TODO here reload from sessionStorage
  const state = writable(initial_state);
  const rule_set = writable(initial_rule_set);

  // Custom store
  // main_sequence.add_next()
  // main_sequence.add_next_once()
  // main_sequence.add_last()
  // main_sequence.add_last_once()
  // main_sequence.remove()
  const main_sequence = writable(initial_control_flow);


  // Pre-loaded version
  // const ComponentRef = writable(bundled_activities["Brand intro"]);
  // Unloaded version
  const ComponentRef = writable();

  log("activity_list", activity_list);

  /**
   * When an activity completes (including "reducing" the state),
   * it should call-back this function that will:
   * 1- process the rules to update the control flow
   * 2- load the next activity (if needed)
   * 3- show the next activity
   */
  const decision = function decision(value, reason) {
    // if value is a rule_set
    if (is_rule_set(value)) {
      // Concat rules
      rule_set.update(function(rules) {
        const concatenated = [...rules, ...value];
        return concatenated;
      });
    }

    process_rules();

    next_activity();
  };

  const is_rule_set = function is_rule_set(rule_set) {
    return (
      Array.isArray(rule_set) &&
      rule_set[0] !== undefined &&
      typeof rule_set[0].predicate === "function"
    );
  };

  const process_rules = function process_rules() {
    $rule_set.forEach(function process_rule(rule) {
      if (rule.predicate($state)) {
        rule.consequence(main_sequence, $activity_index);
      }
    });
  };

  const next_activity = function next_activity() {
    const next_index = $activity_index + 1;
    const name = $main_sequence[next_index];
    const activity = bundled_activities[name];
    // If the activity is bundled
    if (activity !== undefined) {
      add_loaded_activity(name, activity);
      mount_activity(activity);
    }
    console.log("loaded_activities", $loaded_activities);
    // If the activity needs to be loaded
    if ($loaded_activities[name] === undefined) {
      load_activity(function handle_load(name, activity) {
        add_loaded_activity(name, activity);
        mount_activity(activity);
      }, name);
    }
  };

  function add_loaded_activity(name, activity) {
    loaded_activities.update(function(ws) {
      const newObj = {};
      newObj[name] = activity;
      const r = Object.assign({}, ws, newObj);
      return r;
    });
  }

  const mount_activity = function mount_activity(activity) {
    // TODO Mounting moves to the next activity (doesn't look right...)
    activity_index.set($activity_index + 1);
    ComponentRef.set(activity);
  };

  function load_activity(callback, bundle_name) {
    const file_name = activity_list[bundle_name];
    const bundle_address = `/bundles/${file_name}.js`;
    log("Will attempt to load ", file_name, "at", bundle_address);
    if (file_name === undefined) {
      throw new Error(
        'Please configure "' + bundle_name + '" in activity_list.js'
      );
    }
    // For example /bundles/brand_intro.js
    import(bundle_address)
      .then(function(m) {
        callback(bundle_name, m.default);
      })
      .catch(function(message) {
        console.log("Failed to fetch activity", message);
      });
  }

  onMount(function handle_mount() {
    console.log("On mount loading ", $main_sequence);
    console.log("ComponentRef ", $ComponentRef);
    next_activity();
    return function handle_dismount() {};
  });

  // For programmers
  let editor_readables = {
    state: readable(undefined, function(set) {
      return state.subscribe(set);
    }),
    main_sequence: readable(undefined, function(set) {
      return main_sequence.subscribe(set);
    }),
    activity_index: readable(undefined, function(set) {
      return activity_index.subscribe(set);
    })
  };
  let editor_writables = {
    debugging,
    rule_set
  };
</script>

<style>
  .container {
    display: flex;
    height: 100vh;
    flex-direction: column;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
      Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif;
    font-size: 16px;
    line-height: 1.5;
  }
  .app {
    min-height: 50vh;
    /* max-height: 66vh; */
    overflow: auto;
    margin: 0;
    padding: 8px;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    align-items: center;
    /* TODO this creates a problem on shallow screens */
    justify-content: center;
    flex: 100 1 auto;
  }
  .introspector {
    position: relative;
    padding: 10px;
    flex: 1 0 auto;
    max-height: 50vh;
    background: #333;
    color: #ccc;
    overflow: auto;
    /* resize: both; doesn't work all that well...*/
  }
  .toggle {
    position: absolute;
    top: 10px;
    right: 10px;
  }
  .toggle-on {
    position: absolute;
    bottom: 10px;
    right: 10px;
  }
</style>

<div class="container">
  <div class="app">
    <MenuButton
      decision={function() {
        alert('TODO show menu');
      }} />

    {#if $ComponentRef !== undefined}
      <svelte:component this={$ComponentRef} {decision} props={state} />
    {:else}
      <!--
        TODO Loading vs error
        <div style="color:red;padding:1rem;background:white">
        Please configure component "{$main_sequence[0]}"
      </div> -->
    {/if}
  </div>
  {#if $debugging}
    <div class="introspector">
      <Editor readables={editor_readables} writables={editor_writables} />
      <input type="checkbox" bind:checked={$debugging} class="toggle" />
    </div>
  {:else}
    <input type="checkbox" bind:checked={$debugging} class="toggle-on" />
  {/if}
</div>
