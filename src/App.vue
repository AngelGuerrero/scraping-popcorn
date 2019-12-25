<template lang="pug">
  div
    list-scrapy(ppUrl="12-monos")
</template>

<script>
import ListScrapy from "@/components/ListScrapy";

export default {
  components: {
    ListScrapy
  }
};
</script>

<style lang="scss">
body {
  background-color: #2c2e35;
  color: rgba(5, 255, 255, 0.863);
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
}
</style>
