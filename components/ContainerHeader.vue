<template>
  <header :class="{ fixed: shouldFixHeader }">
    <div class="header-top-layer">
      <button
        type="button"
        class="menu-icon"
        aria-label="menu-icon"
        @click="handleClickMenuIcon"
      />

      <ClientOnly>
        <ContainerMembershipMemberIcon class="member-icon-mobile" />
      </ClientOnly>

      <div class="logo-wrapper">
        <a href="/" class="logo logo--site" @click="sendHeaderGa('logo')">
          <img :src="require(`~/assets/premium-logo.svg`)" :alt="SITE_TITLE" />
        </a>

        <ClientOnly>
          <UiEventLogo
            v-if="shouldOpenEventLogo"
            v-show="!shouldFixHeader"
            class="logo"
            :eventLogo="eventLogo"
            @sendGa="handleSendGa"
          />

          <ContainerGptAd
            v-show="shouldShowGptLogo"
            pageKey="global"
            adKey="RWD_LOGO"
            class="logo"
            @slotRenderEnded="handleLogoAdRenderEnded"
          />
        </ClientOnly>
      </div>

      <div class="header-search">
        <div class="header-search__and-magazine">
          <UiSearchBarWrapper
            class="header__search-bar-wrapper"
            :options="options"
            @sendGa="handleSendGa"
          />
          <UiSubscribeMagazineEntrance />
        </div>
        <ClientOnly>
          <ContainerMembershipMemberIcon class="member-icon-desktop" />
        </ClientOnly>

        <UiPromotionList
          class="header__promotion-list"
          :links="PROMOTION_LINKS"
          eventCategory="header"
          @sendGa="handleSendGa"
        />
      </div>
    </div>

    <nav class="header-nav">
      <UiHeaderNavSection
        :sections="sections"
        :currentSectionName="sectionName"
        :partners="partners"
        @sendGa="handleSendGa"
      />
      <UiHeaderNavTopic
        :topics="topics"
        :subBrands="SUB_BRAND_LINKS"
        @sendGa="handleSendGa"
      />
    </nav>

    <ClientOnly>
      <transition name="slide">
        <UiSidebar
          v-if="shouldOpenSidebar"
          :topics="topics"
          :sections="sections"
          :partners="partners"
          :subBrands="SUB_BRAND_LINKS"
          :promotions="PROMOTION_LINKS"
          :socialMedia="SOCIAL_MEDIA_LINKS"
          @close="handleSidebarClose"
          @sendGa="handleSendGa"
        />
      </transition>
    </ClientOnly>
  </header>
</template>

<script>
import { mapGetters } from 'vuex'

import UiEventLogo from './UiEventLogo.vue'
import UiSearchBarWrapper from './UiSearchBarWrapper.vue'
import UiPromotionList from './UiPromotionList.vue'
import UiHeaderNavSection from './UiHeaderNavSection.vue'
import UiHeaderNavTopic from './UiHeaderNavTopic.vue'
import UiSidebar from './UiSidebar.vue'
import ContainerGptAd from '~/components/ContainerGptAd.vue'
import ContainerMembershipMemberIcon from '~/components/ContainerMembershipMemberIcon.vue'
import UiSubscribeMagazineEntrance from '~/components/UiSubscribeMagazineEntrance.vue'

import {
  SUB_BRAND_LINKS,
  SOCIAL_MEDIA_LINKS,
  PROMOTION_LINKS,
  SITE_TITLE,
} from '~/constants/index'

let headerHight = 0

export default {
  name: 'ContainerHeader',
  components: {
    UiEventLogo,
    UiSearchBarWrapper,
    UiPromotionList,
    UiHeaderNavSection,
    UiHeaderNavTopic,
    UiSidebar,
    ContainerGptAd,
    ContainerMembershipMemberIcon,
    UiSubscribeMagazineEntrance,
  },

  props: {
    currentSectionName: {
      type: String,
      default: undefined,
    },
  },

  data() {
    return {
      shouldFixHeader: false,
      SITE_TITLE,

      eventLogo: {},
      now: new Date(),
      intervalIdOfUpdateNow: undefined,
      hasGptLogo: false,

      defaultOption: { title: '全部類別' },

      shouldOpenSidebar: false,
      PROMOTION_LINKS,
      SOCIAL_MEDIA_LINKS,
      SUB_BRAND_LINKS,
    }
  },
  computed: {
    ...mapGetters({
      sections: 'sections/displayedSections',
      searchedSections: 'sections/sections',
      getSectionByCategoryName: 'sections/getSectionByCategoryName',
      partners: 'partners/displayedPartners',
      topics: 'topics/displayedTopics',
      isDesktopWidth: 'viewport/isViewportWidthUpXl',
    }),

    sectionName() {
      let sectionName

      {
        const { path } = this.$route

        if (path.startsWith('/story/')) {
          return this.currentSectionName
        }

        {
          const sec = '/section/'
          const cat = '/category/'

          if (path === '/') {
            sectionName = 'home'
          } else if (path.startsWith(sec)) {
            sectionName = removePrefix(path, sec)

            if (sectionName === 'topic') {
              return
            }
          } else if (path.startsWith(cat)) {
            const categoryName = removePrefix(path, cat)

            sectionName = this.getSectionByCategoryName(categoryName).name
          }
        }
      }

      return sectionName

      function removePrefix(str = '', prefix = '') {
        return str.slice(prefix.length)
      }
    },

    shouldOpenEventLogo() {
      // 當有 GPT Logo 時不應該出現 Event Logo
      if (!this.hasEventLogo || this.hasGptLogo) {
        return false
      }

      return this.isInThePeriodBetween(
        new Date(this.eventLogo.startDate),
        new Date(this.eventLogo.endDate)
      )
    },
    hasEventLogo() {
      return Object.keys(this.eventLogo).length > 0
    },
    shouldShowGptLogo() {
      return !this.isPremiumMember && this.hasGptLogo && !this.shouldFixHeader
    },

    options() {
      const sections = this.searchedSections.filter(
        (section) => section.name !== 'videohub' && section.name !== 'member'
      )
      return [this.defaultOption, ...sections]
    },

    isPremiumMember() {
      return this.$store?.getters?.['membership-subscribe/isPremiumMember']
    },
  },
  watch: {
    isDesktopWidth() {
      if (!this.isDesktopWidth) {
        this.listenScrollToFixHeader()
      } else {
        this.cleanFixedHeader()
      }
    },
    '$route.fullPath'() {
      this.shouldOpenSidebar = false
    },
  },
  beforeMount() {
    this.fetchEventLogo()

    this.intervalIdOfUpdateNow = setInterval(this.updateNow, 1000)
  },

  mounted() {
    if (!this.isDesktopWidth) {
      this.listenScrollToFixHeader()
    }
  },

  beforeDestroy() {
    window.removeEventListener('scroll', this.handleFixHeader)

    clearInterval(this.intervalIdOfUpdateNow)
  },
  methods: {
    listenScrollToFixHeader() {
      headerHight = this.$el.offsetHeight
      this.handleFixHeader()
      window.addEventListener('scroll', this.handleFixHeader)
    },
    handleFixHeader() {
      this.shouldFixHeader = window.pageYOffset >= headerHight

      document.body.style.paddingTop = this.shouldFixHeader
        ? `${headerHight}px`
        : ''
    },
    cleanFixedHeader() {
      window.removeEventListener('scroll', this.handleFixHeader)

      this.shouldFixHeader = false
      document.body.style.paddingTop = ''
    },

    async fetchEventLogo() {
      const eventLogoResponse = await this.$fetchEvent({
        isFeatured: true,
        eventType: 'logo',
        maxResults: 1,
      })
      this.eventLogo = eventLogoResponse.items?.[0] ?? {}
    },
    isInThePeriodBetween(startTime, endTime) {
      return this.now >= startTime && this.now < endTime
    },
    updateNow() {
      this.now = new Date()
    },

    handleLogoAdRenderEnded(event) {
      this.hasGptLogo = !event.isEmpty
    },

    handleClickMenuIcon() {
      this.openSidebar()
      this.sendHeaderGa('menu open')
    },
    handleSidebarClose() {
      this.closeSidebar()
      this.sendHeaderGa('menu close')
    },
    openSidebar() {
      this.shouldOpenSidebar = true
    },
    closeSidebar() {
      this.shouldOpenSidebar = false
    },

    sendHeaderGa(eventLabel, eventAction = 'click') {
      this.$ga.event({
        eventCategory: 'header',
        eventAction,
        eventLabel,
      })
    },
    handleSendGa(param = {}) {
      this.$ga.event(param)
    },
  },
}
</script>

<style lang="scss" scoped>
$header-top-layer-width: 90%;
$header-top-layer-padding-x: math.div(100% - $header-top-layer-width, 2);
$menu-icon-width: 16px;
$logo-wrapper-margin-x: 8px;
$header-search-margin-right: 20px;
$share-wrapper-width: 70px;
$search-icon-width: 18px;
$search-field-arrow-width: 11px;

header {
  background: #204f74;
  z-index: 519;
  padding-bottom: 5px;
  position: relative;
  @include media-breakpoint-up(xl) {
    height: 160px;
  }

  &.fixed {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;

    .header-nav {
      display: none;
    }

    .header-search {
      margin-right: $header-search-margin-right;
    }

    .header__search-bar-wrapper::v-deep .search-bar .field {
      top: 76px;

      &::before {
        right: calc(
          #{$header-top-layer-padding-x} + #{$share-wrapper-width} + #{$header-search-margin-right} +
            #{math.div($search-icon-width - $search-field-arrow-width, 2)}
        );
      }
    }
  }
}

.header-top-layer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: $header-top-layer-width;
  max-width: 1024px;
  height: 71px;
  margin-left: auto;
  margin-right: auto;
  padding-top: 33px;
  padding-bottom: 13px;
  @include media-breakpoint-up(md) {
    padding-top: 0px;
    padding-bottom: 0;
  }
  @include media-breakpoint-up(xl) {
    height: 70px;
  }
}
.menu-icon {
  flex-shrink: 0;
  width: $menu-icon-width;
  height: 10px;
  background-image: url(~assets/hamburger-white@2x.png);
  // background-size: 16px;
  background-position: center;
  background-repeat: no-repeat;
  cursor: pointer;
  user-select: none;
  @include media-breakpoint-up(xl) {
    display: none;
  }
}
.member-icon-mobile::v-deep {
  width: 26px;
  margin: 0 0 0 20px;
  @include media-breakpoint-up(xl) {
    display: none;
  }
  .logged-in-icon__icon path {
    fill: #fff;
  }
}
.member-icon-desktop {
  display: none;
  @include media-breakpoint-up(xl) {
    display: initial;
    width: 33px;
    height: 36px;
    margin: 0 10px 0 20px;
  }
}
.logo-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  width: calc(
    100% -
      (
        #{$menu-icon-width} + #{$search-icon-width} + #{$logo-wrapper-margin-x} *
          2
      )
  );
  @include media-breakpoint-up(xl) {
    justify-content: flex-start;
    width: auto;
  }
}
.logo {
  cursor: pointer;
  user-select: none;
  &::v-deep img {
    width: 74px;
    @include media-breakpoint-up(xl) {
      width: auto;
      height: 50px;
    }
  }
  + .logo {
    margin-left: 5px;
  }
}
.header-search {
  display: flex;
  flex-shrink: 0;
  align-items: center;
  z-index: 529;
  &__and-magazine::v-deep {
    display: flex;
    align-items: center;
    flex-direction: row-reverse;
    @include media-breakpoint-up(xl) {
      flex-direction: row;
    }

    .subscribe-magazine-entrance {
      background: #000000;
      color: #fff;

      @include media-breakpoint-up(xl) {
        display: block;
      }
    }

    .field {
      background-color: #d8d8d8;
      top: 76px;
      padding: 16px 24px;
      &::after {
        content: '';
        display: block;
        width: 0;
        height: 0;
        border-style: solid;
        border-width: 0 8px 16px 8px;
        border-color: transparent transparent #d8d8d8 transparent;
        position: absolute;
        top: -16px;
        right: calc(5vw);
        @include media-breakpoint-up(md) {
          right: calc(5vw + 20px);
        }
      }

      .search-bar-select {
        height: 32px;
      }

      .search-bar-input {
        height: 32px;
        margin-top: 16px;
        input {
          border-radius: 8px;
        }
      }
    }
  }
}

.header-search::v-deep {
  path {
    fill: #fff;
  }

  .more-icon {
    background-image: url(~assets/more_white@2x.png);
    background-position: center;
    background-repeat: no-repeat;
    width: 5px;
    height: 20px;
    background-size: inherit;
  }
}

.share-wrapper {
  display: flex;

  a {
    width: 30px;

    + a {
      margin-left: 10px;
    }
  }
}

.header__promotion-list {
  display: none;
  @include media-breakpoint-up(xl) {
    display: block;
  }
}
.header-nav::v-deep {
  height: 24px;
  .section {
    color: #fff !important;
    padding-top: 0;
    &::after {
      content: none !important;
    }
    @each $name, $color in $sections-color {
      &--#{$name}.active {
        color: $color !important;
      }
    }

    @include media-breakpoint-up(md) {
      padding: 0;
      border-top-color: #000 !important;
      @each $name, $color in $sections-color {
        &--#{$name} {
          &:hover {
            background: inherit !important;
            h2 {
              color: $color !important;
            }
          }
          & .section__dropdown a:hover {
            background-color: $color !important;
          }
        }
      }
    }
    &--member {
      background: inherit !important;
    }
    &__dropdown {
      z-index: 20;
      color: white;
    }
  }

  .topic-item {
    border-bottom: 3px solid #000 !important;
    position: relative;
    &::before {
      content: '';
      height: 3px;
      width: 100px;
      background-color: #000;
      display: block;
      position: absolute;
      bottom: 0;
      left: 0;
      transform: translate(-100%, 100%);
    }
    &--normal {
      &::before {
        display: none !important;
      }
      &:hover {
        background-color: #004dbc !important;
      }
    }
  }
}
.slide-enter-active,
.slide-leave-active {
  transition: transform 0.45s;
}
.slide-enter,
.slide-leave-to {
  transform: translateX(-100%);
}
</style>
