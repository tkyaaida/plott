<template>
  <div>
    <div :class="{tabs: true, 'is-boxed': isBoxed, 'is-toggle': isToggle}">
      <ul>
        <li v-for="tab in tabs" :class="{ 'is-active': tab.isActive }">
          <a :href="tab.href" @click="selectTab(tab)">{{ tab.name }}</a>
        </li>
      </ul>
    </div>
    <div class="tabs-details">
      <slot></slot>
    </div>
  </div>
</template>

<script>
export default {
  name: 'tabs',
  props: {
    isBoxed: {default: false},
    isToggle: {default: false},
  },
  data() {
    return {
      tabs: [],
    }
  },
  created() {
    this.tabs = this.$children;
  },
  methods: {
    selectTab(selectedTab) {
      this.tabs.forEach(tab => {
        tab.isActive = (tab.href === selectedTab.href);
      });
      this.$emit('select-tab', selectedTab.name);
    },
  }
}
</script>

<style>

</style>