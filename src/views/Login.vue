<template>
  <div>
    <LoginBackground class="login-background"/>
    <div id="login" class="d-flex align-items-center justify-content-center">
      <div class="container">
        <b-row>
          <b-col lg="7">
            <b-card-group class="my-4">
              <!-- Login -->
              <b-card no-body class="p-4">
                <b-card-body>
                  <h2>{{ $t('widgetSignInTitle') }}</h2>
                  <SignInForm v-on:login="login" :enabled="enabled"/>
                  <p class="text-danger mt-3" v-if="response">{{ response }}</p>
                </b-card-body>
              </b-card>

              <!-- Registration -->
              <b-card
                no-body
                class="text-white bg-primary py-5"
                v-if="storeServerSettings && storeServerSettings.registrationEnabled"
              >
                <b-card-body class="text-center">
                  <div>
                    <h2>{{ registerTitle }}</h2>

                    <!-- HTML-capable blurb from locale -->
                    <div class="mb-2" v-if="registerBlurbHtml" v-html="registerBlurbHtml"></div>
                    <p class="mb-0" v-else>{{ $t('widgetRegisterText') }}</p>

                    <!-- Exactly one Register control -->
                    <b-button
                      v-if="externalRegisterUrl"
                      :href="externalRegisterUrl"
                      target="_blank"
                      rel="noopener"
                      variant="primary"
                      class="active mt-3"
                    >
                      {{ registerButtonLabel }}
                    </b-button>
                    <b-button
                      v-else
                      variant="primary"
                      class="active mt-3"
                      @click="$refs.registrationModal.show()"
                    >
                      {{ $t('buttonRegister') }}
                    </b-button>
                  </div>
                </b-card-body>
              </b-card>
            </b-card-group>
          </b-col>

          <!-- Spacing -->
          <b-col lg="1" class="d-none d-lg-block"></b-col>

          <!-- Germinate logo -->
          <b-col lg="4">
            <div id="svg-logo-container" class="d-flex justify-content-center align-items-center h-100 py-3">
              <router-link id="svg-logo" :to="{ name: 'home' }" v-if="isNotFullAuth">
                <b-img src="./img/germinate-square-name.svg" fluid />
              </router-link>
              <b-img id="svg-logo" src="./img/germinate-square-name.svg" fluid v-else />
            </div>
          </b-col>

          <!-- Horizontal logos below, same width as login+registration -->
          <b-col lg="7">
            <b-card no-body class="p-4 mt-3">
              <b-img-lazy :src="storeBaseUrl + 'image/src-svg/logo-horizontal.svg'" id="logo-horizontal" onerror="this.onerror=null;this.src='null';" alt="Project partner logo" />
            </b-card>
          </b-col>
        </b-row>
      </div>

      <!-- Registration modal (fallback path when no external URL is set) -->
      <RegistrationModal ref="registrationModal" v-if="storeServerSettings && storeServerSettings.registrationEnabled"/>
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import LoginBackground from '@/components/util/LoginBackground'
import SignInForm from '@/components/util/SignInForm'
import RegistrationModal from '@/components/modals/RegistrationModal'
import { apiPostToken } from '@/mixins/api/auth'
import { Pages } from '@/mixins/pages'

const emitter = require('tiny-emitter/instance')

export default {
  name: 'Login',
  data () {
    return {
      response: null,
      enabled: true,
      prevRoute: null
    }
  },
  components: {
    LoginBackground,
    RegistrationModal,
    SignInForm
  },
  computed: {
    ...mapGetters([
      'storeBaseUrl',
      'storeServerSettings'
    ]),
    isNotFullAuth () {
      if (this.storeServerSettings) {
        return this.storeServerSettings.authMode !== 'FULL'
      } else {
        return false
      }
    },
    // Title from locale (fallback to existing)
    registerTitle () {
      return this.resolveI18n('loginRegisterNoticeTitle') || this.$t('widgetRegisterTitle')
    },
    // Optional HTML-enabled blurb (renders with v-html)
    registerBlurbHtml () {
      return this.resolveI18n('loginRegisterNoticeHtml')
    },
    // External URL from locale; if absent, we fall back to modal
    externalRegisterUrl () {
      return this.resolveI18n('loginRegisterExternalUrl')
    },
    // Button label (fallback to default Register label)
    registerButtonLabel () {
      return this.resolveI18n('buttonRegisterExternal') || this.$t('buttonRegister')
    }
  },
  beforeRouteEnter (to, from, next) {
    next(vm => {
      if (from) {
        vm.prevRoute = from
      }
    })
  },
  methods: {
    resolveI18n (key) {
      const val = this.$t(key)
      // If vue-i18n returns the key itself, treat as "missing"
      return val && val !== key ? String(val) : null
    },
    login (user) {
      this.enabled = false
      apiPostToken(user, result => {
        this.enabled = true
        const originalTarget = this.originalTarget
        if (originalTarget) {
          const path = originalTarget
          this.$store.commit('ON_ORIGINAL_TARGET_CHANGED_MUTATION', null)
          this.$store.commit('ON_TOKEN_CHANGED_MUTATION', result)
          this.$router.push(path)
        } else {
          this.$store.commit('ON_TOKEN_CHANGED_MUTATION', result)
          if (this.prevRoute) {
            this.$router.push(this.prevRoute)
          } else {
            this.$router.push({ name: Pages.home })
          }
        }
        emitter.emit('update-sidebar-menu')
      }, {
        codes: [],
        callback: error => {
          if (error.status === 403 || error.status === 400) {
            this.response = this.$t('errorMessageInvalidUsernamePassword')
          } else {
            this.response = this.$t('errorMessageServerUnavailable')
          }
          this.enabled = true
          this.$store.dispatch('setToken', null)
          emitter.emit('update-sidebar-menu')
        }
      })
    }
  }
}
</script>

<style>
#login { min-height: 100vh; }
#svg-logo-container > #svg-logo { max-width: 300px; max-height: 300px; }
#logo-horizontal { width: 100%; height: auto; }
</style>
