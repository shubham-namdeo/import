<template>
  <IonApp>
    <IonSplitPane content-id="main-content" when="lg">
      <Menu />
      <ion-router-outlet id="main-content"></ion-router-outlet>
    </IonSplitPane>
  </IonApp>
</template>

<script lang="ts">
import Menu from '@/components/Menu.vue';
import { createAnimation, IonApp, IonRouterOutlet, IonSplitPane, loadingController } from '@ionic/vue';
import { defineComponent } from 'vue';
import emitter from "@/event-bus"
import { mapGetters, useStore } from 'vuex';
import { initialise, resetConfig } from '@/adapter'
import { showToast } from "@/utils";
import { translate, useProductIdentificationStore, useUserStore } from "@hotwax/dxp-components";
import { useRouter } from 'vue-router';
import { Settings } from 'luxon'
import logger from '@/logger';

export default defineComponent({
  name: 'App',
  components: {
    IonApp,
    IonRouterOutlet,
    IonSplitPane,
    Menu
  },
  data() {
    return {
      loader: null as any,
      maxAge: process.env.VUE_APP_CACHE_MAX_AGE ? parseInt(process.env.VUE_APP_CACHE_MAX_AGE) : 0,
      refreshing: false,
    }
  },
  methods: {
    async presentLoader(options = { message: '', backdropDismiss: true }) {
      // When having a custom message remove already existing loader
      if(options.message && this.loader) this.dismissLoader();

      if (!this.loader) {
        this.loader = await loadingController
          .create({
            message: options.message ? translate(options.message) : translate("Click the backdrop to dismiss."),
            translucent: true,
            backdropDismiss: options.backdropDismiss
          });
      }
      this.loader.present();
    },
    dismissLoader() {
      if (this.loader) {
        this.loader.dismiss();
        this.loader = null as any;
      }
    },
    playAnimation() {
      const aside = document.querySelector('aside') as Element
      const main = document.querySelector('main') as Element

      const revealAnimation = createAnimation()
        .addElement(aside)
        .duration(1500)
        .easing('ease')
        .keyframes([
          { offset: 0, flex: '0', opacity: '0' },
          { offset: 0.5, flex: '1', opacity: '0' },
          { offset: 1, flex: '1', opacity: '1' }
        ])

      const gapAnimation = createAnimation()
        .addElement(main)
        .duration(500)
        .fromTo('gap', '0', 'var(--spacer-2xl)');

      createAnimation()
        .addAnimation([gapAnimation, revealAnimation])
        .play();
    },
    updateAvailable($event: any) {
      const registration = $event.detail;
      const updateExists = true;
      this.store.dispatch('user/updatePwaState', { registration, updateExists });
      showToast(translate("New version available, please update the app."));
    },
    async unauthorized() {
      // Mark the user as unauthorised, this will help in not making the logout api call in actions
      this.store.dispatch("user/logout", { isUserUnauthorised: true });
      const redirectUrl = window.location.origin + '/login';
      window.location.href = `${process.env.VUE_APP_LOGIN_URL}?redirectUrl=${redirectUrl}`;
    }
  },
  async mounted() {
    this.loader = await loadingController
      .create({
        message: translate("Click the backdrop to dismiss."),
        translucent: true,
        backdropDismiss: true
      });
    emitter.on('presentLoader', this.presentLoader);
    emitter.on('dismissLoader', this.dismissLoader);
    emitter.on('playAnimation', this.playAnimation);
    
    if(this.userToken) {
      const currentEComStore : any = useUserStore().getCurrentEComStore;
      await useProductIdentificationStore().getIdentificationPref(currentEComStore.productStoreId)
        .catch((error) => logger.error(error));
    }
    if(this.userProfile && this.userProfile.userTimeZone) {
      // Luxon timezone should be set with the user's selected timezone
      Settings.defaultZone = this.userProfile.userTimeZone
    }
  },
  created() {
    initialise({
      token: this.userToken,
      instanceUrl: this.instanceUrl,
      cacheMaxAge: this.maxAge,
      events: {
        unauthorised: this.unauthorized,
        responseError: () => {
          setTimeout(() => this.dismissLoader(), 100);
        },
        queueTask: (payload: any) => {
          emitter.emit("queueTask", payload);
        }
      }
    })
    document.addEventListener('swUpdated', this.updateAvailable, { once: true })
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (this.refreshing) return
      this.refreshing = true
      window.location.reload()
    })
  },
  unmounted() {
    emitter.off('presentLoader', this.presentLoader);
    emitter.off('dismissLoader', this.dismissLoader);
    emitter.off('playAnimation', this.playAnimation);
    resetConfig()
  },
  computed: {
    ...mapGetters({
      userProfile: 'user/getUserProfile',
      userToken: 'user/getUserToken',
      instanceUrl: 'user/getInstanceUrl'
    })
  },
  setup(){
    const store = useStore();
    const router = useRouter();
    return {
      router,
      store
    }
  },
});
</script>
