<script>
import BrandImage from '@shell/components/BrandImage';
import ClusterIconMenu from '@shell/components/ClusterIconMenu';
import IconOrSvg from '../IconOrSvg';
import { BLANK_CLUSTER } from '@shell/store/store-types.js';
import { mapGetters } from 'vuex';
import { CAPI, COUNT, MANAGEMENT } from '@shell/config/types';
import { MENU_MAX_CLUSTERS, PINNED_CLUSTERS } from '@shell/store/prefs';
import { sortBy } from '@shell/utils/sort';
import { ucFirst } from '@shell/utils/string';
import { KEY } from '@shell/utils/platform';
import { getVersionInfo } from '@shell/utils/version';
import { SETTING } from '@shell/config/settings';
import { getProductFromRoute } from '@shell/utils/router';
import { isRancherPrime } from '@shell/config/version';
import Pinned from '@shell/components/nav/Pinned';
import { TopLevelMenuHelperPagination, TopLevelMenuHelperLegacy } from '@shell/components/nav/TopLevelMenu.helper';
import { debounce } from 'lodash';
import { sameContents } from '@shell/utils/array';
import {
  RcDropdown,
  RcDropdownItem,
  RcDropdownSeparator,
  RcDropdownTrigger
} from '@components/RcDropdown';

export default {
  components: {
    BrandImage,
    ClusterIconMenu,
    IconOrSvg,
    Pinned,
    RcDropdown,
    RcDropdownItem,
    RcDropdownSeparator,
    RcDropdownTrigger
  },

  data() {
    const { displayVersion, fullVersion } = getVersionInfo(this.$store);
    const hasProvCluster = this.$store.getters[`management/schemaFor`](CAPI.RANCHER_CLUSTER);

    const canPagination = this.$store.getters[`management/paginationEnabled`]({
      id: MANAGEMENT.CLUSTER,
      context: 'side-bar',
    }) && this.$store.getters[`management/paginationEnabled`]({
      id: CAPI.RANCHER_CLUSTER,
      context: 'side-bar',
    });
    const helper = canPagination ? new TopLevelMenuHelperPagination({ $store: this.$store }) : new TopLevelMenuHelperLegacy({ $store: this.$store });
    const provClusters = !canPagination && hasProvCluster ? this.$store.getters[`management/all`](CAPI.RANCHER_CLUSTER) : [];
    const mgmtClusters = !canPagination ? this.$store.getters[`management/all`](MANAGEMENT.CLUSTER) : [];

    if (!canPagination) {
      // Reduce the impact of the initial load, but only if we're not making a request
      const args = {
        pinnedIds: this.pinnedIds,
        searchTerm: this.search,
        unPinnedMax: this.maxClustersToShow
      };

      helper.update(args);
    }

    return {
      shown: false,
      displayVersion,
      fullVersion,
      clusterFilter: '',
      hasProvCluster,
      maxClustersToShow: MENU_MAX_CLUSTERS,
      emptyCluster: BLANK_CLUSTER,
      routeCombo: false,

      canPagination,
      helper,
      debouncedHelperUpdateSlow: debounce((...args) => this.helper.update(...args), 1000),
      debouncedHelperUpdateMedium: debounce((...args) => this.helper.update(...args), 750),
      debouncedHelperUpdateQuick: debounce((...args) => this.helper.update(...args), 200),
      provClusters,
      mgmtClusters,
    };
  },

  computed: {
    ...mapGetters(['clusterId']),
    ...mapGetters(['clusterReady', 'isRancher', 'currentCluster', 'currentProduct', 'isRancherInHarvester']),
    ...mapGetters({ features: 'features/get' }),

    pinnedIds() {
      return this.$store.getters['prefs/get'](PINNED_CLUSTERS);
    },

    showClusterSearch() {
      return this.allClustersCount > this.maxClustersToShow;
    },

    allClustersCount() {
      const counts = this.$store.getters[`management/all`](COUNT)?.[0]?.counts || {};
      const count = counts[MANAGEMENT.CLUSTER] || {};

      return count?.summary.count;
    },

    // New
    search() {
      return (this.clusterFilter || '').toLowerCase();
    },

    // New
    showPinClusters() {
      return !this.clusterFilter;
    },

    // New
    searchActive() {
      return !!this.search;
    },

    /**
     * Only Clusters that are pinned
     *
     * (see description of helper.clustersPinned for more details)
     */
    pinFiltered() {
      return this.hasProvCluster ? this.helper.clustersPinned : [];
    },

    /**
     * Used to shown unpinned clusters OR results of text search
     *
     * (see description of helper.clustersOthers for more details)
     */
    clustersFiltered() {
      return this.hasProvCluster ? this.helper.clustersOthers : [];
    },

    pinnedClustersHeight() {
      const pinCount = this.pinFiltered.length;
      const height = pinCount > 2 ? (pinCount * 43) : 90;

      return `min-height: ${height}px`;
    },
    clusterFilterCount() {
      return this.clusterFilter ? this.clustersFiltered.length : this.allClustersCount;
    },

    multiClusterApps() {
      const options = this.options;

      return options.filter((opt) => {
        const filterApps = (opt.inStore === 'management' || opt.isMultiClusterApp) && opt.category !== 'configuration' && opt.category !== 'legacy';

        if (this.isRancherInHarvester) {
          return filterApps && opt.category !== 'hci';
        } else {
          // We expect the location of Virtualization Management to remain the same when rancher-manage-support is not enabled
          return filterApps;
        }
      });
    },

    configurationApps() {
      const options = this.options;

      return options.filter((opt) => opt.category === 'configuration');
    },

    hciApps() {
      const options = this.options;

      return options.filter((opt) => this.isRancherInHarvester && opt.category === 'hci');
    },

    options() {
      const cluster = this.clusterId || this.$store.getters['defaultClusterId'];

      // TODO plugin routes
      const entries = this.$store.getters['type-map/activeProducts']?.map((p) => {
        // Try product-specific index first
        const to = p.to || {
          name: `c-cluster-${p.name}`,
          params: { cluster }
        };

        const matched = this.$router.getRoutes().filter((route) => route.name === to.name);

        if (!matched.length) {
          to.name = 'c-cluster-product';
          to.params.product = p.name;
        }

        return {
          label: this.$store.getters['i18n/withFallback'](`product."${p.name}"`, null, ucFirst(p.name)),
          icon: `icon-${p.icon || 'copy'}`,
          svg: p.svg,
          value: p.name,
          removable: p.removable !== false,
          inStore: p.inStore || 'cluster',
          weight: p.weight || 1,
          category: p.category || 'none',
          to,
          isMultiClusterApp: p.isMultiClusterApp,
        };
      });

      return sortBy(entries, ['weight']);
    },

    canEditSettings() {
      return (this.$store.getters['management/schemaFor'](MANAGEMENT.SETTING)?.resourceMethods || []).includes('PUT');
    },

    hasSupport() {
      return isRancherPrime() || this.$store.getters['management/byId'](MANAGEMENT.SETTING, SETTING.SUPPORTED)?.value === 'true';
    },

    isCurrRouteClusterExplorer() {
      return this.$route?.name?.startsWith('c-cluster');
    },

    productFromRoute() {
      return getProductFromRoute(this.$route);
    },

    aboutText() {
      // If a version number (starts with 'v') then use that
      if (this.displayVersion.startsWith('v')) {
        // Don't show the '.0' for a minor release (e.g. 2.8.0, 2.9.0 etc)
        return !this.displayVersion.endsWith('.0') ? this.displayVersion : this.displayVersion.substr(0, this.displayVersion.length - 2);
      }

      // Default fallback to 'About'
      return this.t('about.title');
    },

    largeAboutText() {
      return this.aboutText.length > 6;
    },

    appBar() {
      let activeFound = false;

      // order is important for the object keys here
      // since we want to check last pinFiltered and clustersFiltered
      const appBar = {
        hciApps: this.hciApps,
        multiClusterApps: this.multiClusterApps,
        configurationApps: this.configurationApps,
        pinFiltered: this.pinFiltered,
        clustersFiltered: this.clustersFiltered,
      };

      Object.keys(appBar).forEach((menuSection) => {
        const menuSectionItems = appBar[menuSection];
        const isClusterCheck = menuSection === 'pinFiltered' || menuSection === 'clustersFiltered';

        // need to reset active state on other menu items
        menuSectionItems.forEach((item) => {
          item.isMenuActive = false;

          if (!activeFound && this.checkActiveRoute(item, isClusterCheck)) {
            activeFound = true;
            item.isMenuActive = true;
          }
        });
      });

      return appBar;
    },

    hideLocalCluster() {
      const hideLocalSetting = this.$store.getters['management/byId'](MANAGEMENT.SETTING, SETTING.HIDE_LOCAL_CLUSTER) || {};
      const value = hideLocalSetting.value || hideLocalSetting.default || 'false';

      return value === 'true';
    }
  },

  // See https://github.com/rancher/dashboard/issues/12831 for outstanding performance related work
  watch: {
    $route() {
      this.shown = false;
    },

    // Before SSP world all of these changes were kicked off given Vue change detection to properties in a computed method.
    // Changes could come from two scenarios
    // 1. Changes made by the user (pin / search). Could be tens per second
    // 2. Changes made by rancher to clusters (state, label, etc change). Could be hundreds a second
    // They can be restricted to help the churn caused from above
    // 1. When SSP enabled reduce http spam
    // 2. When SSP is disabled (legacy) reduce fn churn (this was a known performance customer issue)

    pinnedIds: {
      immediate: true,
      handler(neu, old) {
        if (sameContents(neu, old)) {
          return;
        }

        // Low throughput (user click). Changes should be shown quickly
        this.updateClusters(neu, 'quick');
      }
    },

    search() {
      // Medium throughput. Changes should be shown quickly, unless we want to reduce http spam in SSP world
      this.updateClusters(this.pinnedIds, this.canPagination ? 'medium' : 'quick');
    },

    provClusters: {
      handler(neu, old) {
        // Potentially incredibly high throughput. Changes should be at least limited (slow if state change, quick if added/removed). Shouldn't get here if SSP
        this.updateClusters(this.pinnedIds, neu?.length === old?.length ? 'slow' : 'quick');
      },
      deep: true,
      immediate: true,
    },

    mgmtClusters: {
      handler(neu, old) {
        // Potentially incredibly high throughput. Changes should be at least limited (slow if state change, quick if added/removed). Shouldn't get here if SSP
        this.updateClusters(this.pinnedIds, neu?.length === old?.length ? 'slow' : 'quick');
      },
      deep: true,
      immediate: true,
    },

    hideLocalCluster() {
      this.updateClusters(this.pinnedIds, 'slow');
    }
  },

  mounted() {
    document.addEventListener('keyup', this.handler);
  },

  beforeUnmount() {
    document.removeEventListener('keyup', this.handler);
    this.helper?.destroy();
  },

  methods: {
    checkActiveRoute(obj, isClusterRoute) {
      // for Cluster links in main nav: check if route is a cluster explorer one + check if route cluster matches cluster obj id + check if curr product matches route product
      if (isClusterRoute) {
        return this.isCurrRouteClusterExplorer && this.$route?.params?.cluster === obj?.id && this.productFromRoute === this.currentProduct?.name;
      }

      // for remaining main nav items, check if curr product matches route product is enough
      return this.productFromRoute === obj?.value;
    },

    handleKeyComboClick() {
      this.routeCombo = !this.routeCombo;
    },

    clusterMenuClick(ev, cluster) {
      if (this.routeCombo) {
        ev.preventDefault();

        if (this.isCurrRouteClusterExplorer && this.productFromRoute === this.currentProduct?.name) {
          const clusterRoute = {
            name: this.$route.name,
            params: { ...this.$route.params },
            query: { ...this.$route.query }
          };

          clusterRoute.params.cluster = cluster.id;

          return this.$router.push(clusterRoute);
        }
      }

      return this.$router.push(cluster.clusterRoute);
    },

    handler(e) {
      if (e.keyCode === KEY.ESCAPE) {
        this.hide();
      }
    },

    hide() {
      this.shown = false;
      if (this.clustersFiltered === 0) {
        this.clusterFilter = '';
      }
    },

    toggle() {
      this.shown = !this.shown;
    },

    async goToHarvesterCluster() {
      const localCluster = this.$store.getters['management/all'](CAPI.RANCHER_CLUSTER).find((C) => C.id === 'fleet-local/local');

      try {
        await localCluster.goToHarvesterCluster();
      } catch {
      }
    },

    getTooltipConfig(item, showWhenClosed = false) {
      if (!item) {
        return;
      }

      let contentText = '';
      let content;
      let popperClass = '';

      // this is the normal tooltip scenario where we are just passing a string
      if (typeof item === 'string') {
        contentText = item;
        content = this.shown ? null : contentText;

        // if key combo is pressed, then we update the tooltip as well
      } else if (this.routeCombo &&
        typeof item === 'object' &&
        !Array.isArray(item) &&
        item !== null &&
        item.ready) {
        contentText = this.t('nav.keyComboTooltip');

        if (showWhenClosed) {
          content = !this.shown ? contentText : null;
        } else {
          content = this.shown ? contentText : null;
        }

        // this is scenario where we show a tooltip when we are on the expanded menu to show full description
      } else {
        contentText = item.label;
        // this adds a class to the tooltip container so that we can control the max width
        popperClass = 'menu-description-tooltip';

        if (item.description) {
          contentText += `<br><br>${item.description}`;
        }

        if (showWhenClosed) {
          content = !this.shown ? contentText : null;
        } else {
          content = this.shown ? contentText : null;

          // this adds a class to adjust tooltip position so it doesn't overlap the cluster pinning action
          popperClass += ' description-tooltip-pos-adjustment';
        }
      }

      return {
        content,
        placement: 'right',
        popperClass
      };
    },

    updateClusters(pinnedIds, speed = 'slow' | 'medium' | 'quick') {
      const args = {
        pinnedIds,
        searchTerm: this.search,
        unPinnedMax: this.maxClustersToShow
      };

      switch (speed) {
        case 'slow':
          this.debouncedHelperUpdateSlow(args);
          break;
        case 'medium':
          this.debouncedHelperUpdateMedium(args);
          break;
        case 'quick':
          this.debouncedHelperUpdateQuick(args);
          break;
      }
    }
  }
};
</script>

<template>

<!-- <rc-dropdown-item class="clusters-parent">
          

            <template #dropdownCollection>
            
              <rc-dropdown-item v-for="(c, index) in appBar.pinFiltered" :key="`pin-${index}`">
                <button v-if="c.ready" class="cluster selector option" :class="{ 'active-menu-link': c.isMenuActive }"
                  :to="c.clusterRoute" @click.prevent="clusterMenuClick($event, c)">
                  <ClusterIconMenu :cluster="c" />
                  <div class="cluster-name">
                    <p>{{ c.label }}</p>
                    <p v-if="c.description" class="description">{{ c.description }}</p>
                  </div>
                  <Pinned :cluster="c" />
                </button>
                <span v-else class="option cluster selector disabled">
                  <ClusterIconMenu :cluster="c" />
                  <div class="cluster-name">
                    <p>{{ c.label }}</p>
                    <p v-if="c.description" class="description">{{ c.description }}</p>
                  </div>
                  <Pinned :cluster="c" />
                </span>
              </rc-dropdown-item>

              
              <rc-dropdown-item v-for="(c, index) in appBar.clustersFiltered" :key="`c-${index}`">
                <button v-if="c.ready" class="cluster selector option" :class="{ 'active-menu-link': c.isMenuActive }"
                  :to="c.clusterRoute" @click.prevent="clusterMenuClick($event, c)">
                  <ClusterIconMenu :cluster="c" />
                  <div class="cluster-name">
                    <p>{{ c.label }}</p>
                    <p v-if="c.description" class="description">{{ c.description }}</p>
                  </div>
                  <Pinned :class="{ 'showPin': c.pinned }" :cluster="c" />
                </button>
                <span v-else class="option cluster selector disabled">
                  <ClusterIconMenu :cluster="c" />
                  <div class="cluster-name">
                    <p>{{ c.label }}</p>
                    <p v-if="c.description" class="description">{{ c.description }}</p>
                  </div>
                  <Pinned :class="{ 'showPin': c.pinned }" :cluster="c" />
                </span>
              </rc-dropdown-item>

           
              <rc-dropdown-item v-if="allClustersCount > maxClustersToShow">
                <router-link class="clusters-all" :to="{
                  name: 'c-cluster-product-resource',
                  params: { cluster: emptyCluster, product: 'manager', resource: 'provisioning.cattle.io.cluster' }
                }">
                  {{ t('nav.seeAllClusters') }}
                </router-link>
              </rc-dropdown-item>
            </template>
          

        </rc-dropdown-item> -->

  <div>

    <rc-dropdown :close-on-click-outside="true">
      <rc-dropdown-trigger class="main-trigger">
          <i class="icon icon-chevron-down" />
      </rc-dropdown-trigger>
      <template #dropdownCollection>
        <rc-dropdown-item>
          <div>
            <!-- Home button -->
            <div @click="hide()">
              <router-link class="option cluster selector home" :to="{ name: 'home' }" role="link"
                :aria-label="t('nav.ariaLabel.homePage')">
                <svg v-clean-tooltip="getTooltipConfig(t('nav.home'))" class="top-menu-icon"
                  xmlns="http://www.w3.org/2000/svg" height="24" viewBox="0 0 24 24" width="24">
                  <path d="M0 0h24v24H0z" fill="none" />
                  <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z" />
                </svg>
                <div class="home-text">
                  Home
                </div>
              </router-link>
            </div>
            <!-- Search bar -->
            <div v-if="showClusterSearch" class="clusters-search">
              <div class="clusters-search-count">
                <span>{{ clusterFilterCount }}</span>
                {{ t('nav.search.clusters') }}
                <i v-if="clusterFilter" class="icon icon-filter_alt" />
              </div>

              <div class="search">
                <input ref="clusterFilter" v-model="clusterFilter" :placeholder="t('nav.search.placeholder')"
                  :tabindex="!shown ? -1 : 0" :aria-label="t('nav.search.ariaLabel')">
                <i class="magnifier icon icon-search" :class="{ active: clusterFilter }" />
                <i v-if="clusterFilter" class="icon icon-close" @click="clusterFilter = ''" />
              </div>
            </div>
          </div>
        </rc-dropdown-item>
        
        <rc-dropdown-item v-if="hciApps.length">
          <template v-if="hciApps.length">
            <div class="category" />
            <div>
              <a v-if="isRancherInHarvester" class="option" tabindex="0" @click="goToHarvesterCluster()">
                <i class="icon icon-dashboard app-icon" />
                <div>
                  {{ t('nav.harvesterDashboard') }}
                </div>
              </a>
            </div>
            <div v-for="(a, i) in appBar.hciApps" :key="i" @click="hide()">
              <router-link class="option" :to="a.to" :class="{ 'active-menu-link': a.isMenuActive }" role="link"
                :aria-label="`${t('nav.ariaLabel.harvesterCluster')} ${a.label}`">
                <IconOrSvg class="app-icon" :icon="a.icon" :src="a.svg" />
                <div>{{ a.label }}</div>
              </router-link>
            </div>
          </template>
        </rc-dropdown-item>

        

        <!-- Multi-cluster category -->
        <div class="category-title">
          <span>{{ t('nav.categories.multiCluster') }}</span>
        </div>
        <rc-dropdown-item v-for="(a, i) in appBar.multiClusterApps" :key="`mc-${i}`" @click="hide()">
          <router-link class="option" :class="{ 'active-menu-link': a.isMenuActive }" :to="a.to" role="link"
            :aria-label="`${t('nav.ariaLabel.multiClusterApps')} ${a.label}`">
            <!-- <IconOrSvg v-clean-tooltip="getTooltipConfig(a.label)" class="app-icon" :icon="a.icon" :src="a.svg" /> -->
            <span class="option-link">{{ a.label }}</span>
          </router-link>
        </rc-dropdown-item>
        <rc-dropdown-separator />
        <!-- Configuration category -->
        <div class="category-title">
          <span>{{ t('nav.categories.configuration') }}</span>
        </div>
        <rc-dropdown-item v-for="(a, i) in appBar.configurationApps" :key="`cfg-${i}`">
          <router-link class="option" :class="{ 'active-menu-link': a.isMenuActive }" :to="a.to" role="link"
            :aria-label="`${t('nav.ariaLabel.configurationApps')} ${a.label}`">
            <!-- <IconOrSvg v-clean-tooltip="getTooltipConfig(a.label)" class="app-icon" :icon="a.icon" :src="a.svg" /> -->
            <div>{{ a.label }}</div>
          </router-link>
        </rc-dropdown-item>

      </template>
    </rc-dropdown>

  </div>

  <!-- Footer -->
  <!-- <div
          class="footer"
        >
          <div
            v-if="canEditSettings"
            class="support"
            @click="hide()"
          >
            <router-link
              :to="{name: 'support'}"
              role="link"
              :aria-label="t('nav.ariaLabel.support')"
            >
              {{ t('nav.support', {hasSupport}) }}
            </router-link>
          </div>
          <div
            class="version"
            :class="{'version-small': largeAboutText}"
            @click="hide()"
          >
            <router-link
              :to="{ name: 'about' }"
              role="link"
              :aria-label="t('nav.ariaLabel.about')"
            >
              {{ aboutText }}
            </router-link>
          </div>
        </div> -->


</template>

<style lang="scss">
.menu-description-tooltip {
  max-width: 200px;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.description-tooltip-pos-adjustment {
  // needs !important so that we can
  // offset the tooltip a bit so it doesn't
  // overlap the pin icon and cause bad UX
  left: 48px !important;
}

.localeSelector,
.footer-tooltip {
  z-index: 1000;
}

.localeSelector {
  .v-popper__inner {
    padding: 10px 0;
  }

  .v-popper__arrow-container {
    display: none;
  }

  .v-popper:focus {
    outline: 0;
  }
}

.theme-dark .cluster-name .description {
  color: var(--input-label) !important;
}

.theme-dark .body .option {

  &:hover .cluster-name .description,
  &.router-link-active .cluster-name .description,
  &.active-menu-link .cluster-name .description {
    color: var(--side-menu-desc) !important;
  }
}
</style>

<style lang="scss" scoped>
$clear-search-size: 20px;
$icon-size: 25px;
$option-padding: 9px;
$option-padding-left: 14px;
$option-height: $icon-size + $option-padding + $option-padding;




.home {
  display: flex;
  align-items: center;
  justify-content: space-around;

  svg {
    width: 25px;
    height: 25px;
    margin-left: 9px;
  }
}

.home-text {
  margin-left: $option-padding-left - 7;
}

.body {
  background: orange;
  display: flex;
  flex: 1;
  justify-content: space-around;
  flex-direction: row;
  margin: 10px 0;


  .option {
    align-items: center;
    cursor: pointer;
    display: flex;
    color: var(--link);
    font-size: 14px;
    height: $option-height;
    white-space: nowrap;
    background-color: transparent;

    border-radius: 0;
    border: none;

    .cluster-badge-logo-text {
      color: var(--default-active-text);
      font-weight: 500;
    }

    .pin {
      font-size: 16px;
      margin-left: auto;
      display: none;
      color: var(--body-text);

      &.showPin {
        display: block;
      }
    }

    .cluster-name {
      flex: 1;
      /* takes remaining space */
      min-width: 0;
      /* required for ellipsis */
      display: flex;
      flex-direction: column;
      /* stack label + description */
      align-items: flex-start;
      /* align text to the left */

      &>p {
        margin: 0;
        /* remove default margins */
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        width: 100%;

        &.description {
          font-size: 0.85em;
          color: var(--text-muted);
        }
      }
    }

    &:hover {
      text-decoration: none;

      .pin {
        display: block;
        color: var(--darker-text);
      }
    }

    &.disabled {
      background: transparent;
      cursor: not-allowed;

      .rancher-provider-icon,
      .cluster-name p {
        filter: grayscale(1);
        color: var(--muted) !important;
      }

      .pin {
        cursor: pointer;
      }
    }

    &:focus {
      outline: 0;
      box-shadow: none;
    }

    >i,
    >img {
      display: block;
      font-size: $icon-size;
      margin-right: 14px;

      &:not(.pin) {
        width: 42px;
      }
    }

    .rancher-provider-icon,
    svg {
      margin-right: 16px;
      fill: var(--link);
    }

    .top-menu-icon {
      outline-offset: 4px;
    }

    &.router-link-active,
    &.active-menu-link {
      &:focus-visible {

        .top-menu-icon,
        .app-icon {
          @include focus-outline;
        }
      }

      &:focus-visible .rancher-provider-icon {
        @include focus-outline;
        outline-offset: -4px;
      }

      background: var(--primary-hover-bg);
      color: var(--primary-hover-text);

      svg {
        fill: var(--primary-hover-text);
      }

      i {
        color: var(--primary-hover-text);
      }

      div .description {
        color: var(--default);
      }
    }

    &:focus-visible {

      .top-menu-icon,
      .rancher-provider-icon,
      .app-icon {
        @include focus-outline;
      }
    }

    &:hover {
      color: var(--primary-hover-text);
      background: var(--primary-hover-bg);

      >div {
        color: var(--primary-hover-text);

        .description {
          color: var(--default);
        }
      }

      svg {
        fill: var(--primary-hover-text);
      }

      div {
        color: var(--primary-hover-text);
      }

      &.disabled {
        background: transparent;
        color: var(--muted);

        >.pin {
          color: var(--default-text);
          display: block;
        }
      }
    }
  }

  .option,
  .option-disabled {
    padding: $option-padding 0 $option-padding $option-padding-left;
  }

  .search {
    position: relative;

    >input {
      background-color: transparent;
      padding-right: 35px;
      padding-left: 25px;
      height: 32px;
    }

    >.magnifier {
      position: absolute;
      top: 12px;
      left: 8px;
      width: 12px;
      height: 12px;
      font-size: 12px;
      opacity: 0.4;

      &.active {
        opacity: 1;

        &:hover {
          color: var(--body-text);
        }
      }
    }

    >i {
      position: absolute;
      font-size: 12px;
      top: 12px;
      right: 8px;
      opacity: 0.7;
      cursor: pointer;

      &:hover {
        color: var(--disabled-bg);
      }
    }
  }

  .clusters-all {
    display: flex;
    flex-direction: row-reverse;
    margin-right: 16px;
    margin-top: 10px;

    &:focus-visible {
      outline: none;
    }

    span {
      display: flex;
      align-items: center;
    }

    &:focus-visible span {
      @include focus-outline;
      outline-offset: 4px;
    }
  }


  .clusters {
    margin-left: 10px;
    /* replace `width: 150px;` with: */
    flex: 0 0 220px;
    /* preferred column width (won't grow) */
    max-width: 300px;
    /* cap so it never expands too wide */
    min-width: 120px;
    /* allow shrinking a bit */
    box-sizing: border-box;
    overflow: hidden;
    /* ensure inner content can't push it wider */

    a,
    span {
      margin: 0;
    }

    &-search {
      display: flex;
      align-items: center;
      gap: 14px;
      margin: 16px 0;
      height: 42px;

      .search {
        transition: all 0.25s ease-in-out;
        transition-delay: 2s;
        width: 0;
        height: 36px;

        input {
          height: 100%;
        }
      }

      &-count {
        position: relative;
        display: flex;
        flex-direction: column;
        width: 42px;
        height: 42px;
        display: flex;
        align-items: center;
        justify-content: center;
        color: var(--default-active-text);
        margin-left: $option-padding-left;
        border-radius: 5px;
        font-size: 10px;
        font-weight: bold;

        span {
          font-size: 14px;
        }

        .router-link-active {
          &:hover {
            text-decoration: none;
          }
        }

        i {
          font-size: 12px;
          position: absolute;
          right: -3px;
          top: 2px;
        }
      }
    }
  }

  .none-matching {
    width: 50%;
    text-align: center;
    padding: 8px
  }

  .clustersPinned {
    width: 50%;

    .category {
      &-title {
        margin: 8px 0;
        margin-left: 16px;
      }
    }
  }

}

.main-trigger {
  width: 30px;
  display: flex;
  justify-content: center;
  align-items: center;
}
/* ensure wrapper item has no extra padding and is full width */
.clusters-parent {
  width: 100%;
  padding: 0;
  box-sizing: border-box;
}

/* make the trigger element fill the parent */
.clusters-trigger {
  display: block;
  width: 100%;
}

/* the visible trigger button: full width, left-aligned label, chevron at right */
.clusters-trigger__btn {
  display: flex;
  align-items: center;
  justify-content: space-between; /* label left, chevron right */
  width: 100%;
  background: transparent;
  border: none;
  padding: $option-padding 14px;
  cursor: pointer;
  text-align: left;
  box-sizing: border-box;
}

/* label: take remaining space and truncate if too long */
.clusters-trigger__label {
  flex: 1 1 auto;
  min-width: 0; /* required for ellipsis inside flex items */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  text-align: left;
}

/* keep the chevron from squeezing the label */
.clusters-trigger__btn .icon {
  margin-left: 12px;
  flex: 0 0 auto;
}

/* optional: when dropdown opens you might want the trigger to show active state */
.rc-dropdown[aria-expanded="true"] .clusters-trigger__btn,
.clusters-trigger__btn:focus {
  background: var(--primary-hover-bg);
  color: var(--primary-hover-text);
}

.secondary-trigger {
  width: 100%;
  text-align: left;
  padding: 9px 14px;
  font-size: 14px;
  background: transparent;
  border: none;
  color: var(--link);
  cursor: pointer;

  &:hover {
    background: var(--primary-hover-bg);
    color: var(--primary-hover-text);
    text-decoration: none;
  }

  &:focus-visible {
    @include focus-outline;
    outline-offset: -4px;
  }
}

button {
  background: transparent;
  width: 100%;
}

.cluster.selector.option {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  gap: 8px;
  background: transparent;
  /* spacing between icon, name, and pin */
  box-sizing: border-box;
  padding: 6px 8px;
}

.category {
  display: flex;
  flex-direction: row;
  justify-content: space-between;

  &-title {
    display: flex;
    flex-direction: row;
    align-items: flex-start;
    align-items: center;
    margin: 15px 0;
    margin-left: 16px;
    font-size: 14px;
    text-transform: uppercase;

    span {
      transition: all 0.25s ease-in-out;
      display: flex;
      max-height: 16px;
    }


  }

  i {
    padding-left: $option-padding-left - 5;
  }
}


.menu-open {
  .option {

    &.router-link-active,
    &.active-menu-link {
      &:focus-visible {
        @include focus-outline;
        border-radius: 0;
        outline-offset: -4px;

        .top-menu-icon,
        .app-icon,
        .rancher-provider-icon {
          outline: none;
          border-radius: 0;
        }
      }
    }

    &:focus-visible {
      @include focus-outline;
      outline-offset: -4px;

      .top-menu-icon,
      .app-icon,
      .rancher-provider-icon {
        outline: none;
        border-radius: 0;
      }
    }
  }
}

.menu-close {
  .side-menu-logo {
    opacity: 0;
  }

  .category {
    &-title {
      span {
        opacity: 0;
      }


    }
  }

  .clusters-all {
    flex-direction: row;
    margin-left: $option-padding-left + 2;

    span {
      i {
        display: none;
      }
    }
  }

  .clustersPinned {
    .category {
      &-title {
        hr {
          width: 40px;
        }
      }
    }
  }

  .footer {
    margin: 20px 10px;
    width: 50px;

    .support {
      display: none;
    }

    .version {
      text-align: center;

      &.version-small {
        font-size: 12px;
      }
    }
  }
}

.footer {
  margin: 20px;
  width: 240px;
  display: flex;
  flex: 0;
  flex-direction: row;

  >* {
    flex: 1;
    color: var(--link);

    &:first-child {
      text-align: left;
    }

    &:last-child {
      text-align: right;
    }

    text-align: center;
  }

  .support a:focus-visible {
    @include focus-outline;
    outline-offset: 4px;
  }

  .version {
    cursor: pointer;

    a:focus-visible {
      @include focus-outline;
      outline-offset: 4px;
    }
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: all 0.25s;
  transition-timing-function: ease;
}

.fade-leave-active {
  transition: all 0.25s;
}

.fade-leave-to {
  left: -300px;
}

.fade-enter {
  left: -300px;
}

.locale-chooser {
  cursor: pointer;
}

.localeSelector {
  :deep() .v-popper__inner {
    padding: 50px 0;
  }

  :deep() .v-popper__arrow-container {
    display: none;
  }

  :deep() .v-popper:focus {
    outline: 0;
  }

  li {
    padding: 8px 20px;

    &:hover {
      background-color: var(--primary-hover-bg);
      color: var(--primary-hover-text);
      text-decoration: none;
    }
  }
}
</style>
