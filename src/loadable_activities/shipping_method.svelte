<script>
  import I from "../lib/interaction.js";
  const { Interaction, Prompt, Inputs, Feedback } = I;

  export let props;
  export let decision;

  const handler = function(event) {
    console.log(event);
    decision(true);
  };
</script>

<div class="activity">
  <label>How fast do you need this?</label>
  <input value="Next day +$15" />
  <input value="Standard 5-7 days +$0" />
</div>
