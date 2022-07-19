<script lang="ts">
  export let value = '';
  export let redacted = false;
  export let trailing = '';

</script>
  {#if redacted}
    {#if trailing}
      <span class="wrapper">
        <span class="redacted" style="width: {value.length * 16}px;">{'▀'.repeat(value.length)}</span>
        <span>{trailing}</span>
      </span>
    {:else}
      <span class="redacted">{'▀'.repeat(value.length)}</span>
    {/if}
  {:else}
    <span>{value}{trailing}</span>
  {/if}
<style>
.wrapper {
  display: inline-flex;
  align-items: center;
}
.redacted {
  background-color: #ced4da;
  color: #ced4da;
  height: 100%;
  padding: 0 1px;
}
</style>
