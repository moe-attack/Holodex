<template>
  <v-container fluid style="height: 100%">
    <v-card class="ma-auto mt-6" elevation="12" max-width="500px">
      <user-card v-if="userdata.user" />
      <v-divider />
      <v-card-subtitle class="justify-center">
        {{ userdata.user ? $t("views.login.linkAcc") : $t("component.mainNav.login") }}
      </v-card-subtitle>
      <v-card-text class="d-flex flex-column">
        <v-btn
          v-if="!userdata.user || !userdata.user.google_id"
          class="my-3"
          color="red accent-2"
          @click.prevent="loginGoogle"
        >
          <v-icon left>
            {{ icons.mdiGoogle }}
          </v-icon>
          {{ $t("views.login.with.0") }}
        </v-btn>
        <v-btn
          v-if="!userdata.user || !userdata.user.discord_id"
          class="my-3"
          color="indigo"
          @click.prevent="loginDiscord"
        >
          <v-icon left>
            {{ icons.mdiDiscord }}
          </v-icon>

          {{ $t("views.login.with.1") }}
        </v-btn>
        <v-btn
          v-if="!userdata.user || !userdata.user.twitter_id"
          class="my-3"
          color="blue lighten-1"
          @click.prevent="loginTwitter"
        >
          <v-icon left>
            {{ icons.mdiTwitter }}
          </v-icon>

          {{ $t("views.login.with.2") }}
        </v-btn>
      </v-card-text>
      <v-divider />
      <v-card-text v-if="userdata.user">
        <span class="text-subtitle-2 mb-1 d-inline-block">{{ $t("views.login.ownedYtChannel") }}</span>
        <v-text-field
          readonly
          rounded
          filled
          dense
          hide-details
          :value="userdata.user.yt_channel_key || 'None on file'"
        />
        <span class="text-caption">
          {{ $t("views.login.futureYtcOwnerMessage") }}
        </span>
        <br>
        <br>
        <span class="text-subtitle-2 mb-1 d-inline-block">API Key</span>
        <v-text-field
          readonly
          rounded
          outlined
          dense
          hide-details
          :class="doneCopy ? 'green lighten-2' : ''"
          :value="userdata.user.api_key || 'None on file'"
          :append-icon="icons.mdiClipboardPlusOutline"
          @click:append="copyToClipboard(userdata.user.api_key)"
        />
        <br>
        <v-btn
          small
          block
          color="warning"
          @click="resetKey"
        >
          {{ $t("views.login.apikeyNew") }}
        </v-btn>
        <span class="text-caption">
          {{ $t("views.login.apikeyMsg") }}
        </span>
        <v-btn
          small
          block
          color="info"
          href="https://holodex.stoplight.io/"
          target="_blank"
        >
          API Documentation
          <v-icon small right>
            {{ icons.mdiOpenInNew }}
          </v-icon>
        </v-btn>
      </v-card-text>
    </v-card>
  </v-container>
</template>

<script lang="ts">
import GAuth from "vue-google-oauth2";
import open from "oauth-open";
import api from "@/utils/backend-api";
import Vue from "vue";
import UserCard from "@/components/user/UserCard.vue";
import copyToClipboard from "@/mixins/copyToClipboard";

const gauthOption = {
    clientId: "275540829388-87s7f9v2ht3ih51ah0tjkqng8pd8bqo2.apps.googleusercontent.com",
    scope: "https://www.googleapis.com/auth/userinfo.email",
    // scope: "openid",
    prompt: "select_account",
    fetch_basic_profile: false,
};
Vue.use(GAuth, gauthOption);

const apiURI = "/api";
// the fact this URI is invalid doesn't matter,
// all it matters is we own the domain so oauth-open can grab the URI from the window

export default {
    name: "Login",
    metaInfo() {
        const vm = this;
        return {
            get title() {
                return `${vm.$t("component.mainNav.login")} - Holodex`;
            },
        };
    },
    components: { UserCard },
    mixins: [copyToClipboard],
    data() {
        return {};
    },
    computed: {
        userdata() {
            return this.$store.state.userdata;
        },
    },
    mounted() {},
    methods: {
        async loginGoogle() {
            const authCode = await this.$gAuth.getAuthCode();
            const resp = await api.login(this.$store.state.userdata.jwt, authCode, "google");
            // console.log(resp);
            this.$store.commit("setUser", resp.data);
            this.$gtag.event("login", {
                event_label: "google",
            });

            this.$store.dispatch("favorites/resetFavorites");
        },
        async loginDiscord() {
            // redirect location:
            const redirectUri = `${window.location.protocol}//${window.location.host}/discord`;

            // out:
            // {token_type: "Bearer", access_token: "<SOMEACCESSTOKEN>", expires_in: "604800", scope: "identify"}
            open(
                `https://discord.com/api/oauth2/authorize?client_id=793619250115379262&redirect_uri=${encodeURIComponent(
                    redirectUri,
                )}&response_type=token&scope=identify`,
                async (err, out) => {
                    const resp = await api.login(this.$store.state.userdata.jwt, out.access_token, "discord");
                    // console.log(resp);
                    this.$store.commit("setUser", resp.data);
                    this.$gtag.event("login", {
                        event_label: "discord",
                    });

                    this.$store.dispatch("favorites/resetFavorites");
                },
            );
        },
        async loginTwitter() {
            open(`${apiURI}/v2/user/login/twitter`, async (err, out) => {
                const twitterTempJWT = out.jwt;
                const resp = await api.login(this.$store.state.userdata.jwt, twitterTempJWT, "twitter");
                // console.log(resp);
                this.$store.commit("setUser", resp.data);
                this.$gtag.event("login", {
                    event_label: "twitter",
                });
                this.$store.dispatch("favorites/resetFavorites");
            });
        },
        async forceUserUpdate() {
            const check = await api.loginIsValid(this.userdata.jwt);
            if (check === false) {
                this.$store.dispatch("logout");
            } else if (check.data && check.data.id) {
                this.$store.commit("setUser", { user: check.data, jwt: this.userdata.jwt });
            }
        },
        async resetKey() {
            /* eslint-disable no-restricted-globals, no-alert */
            if (this.userdata.user.api_key) {
                const confirm1 = confirm(this.$t("views.login.apikeyResetConfirm1"));
                if (!confirm1) {
                    alert(this.$t("views.login.apikeyResetNvm"));
                    return;
                }
                const confirm2 = confirm(this.$t("views.login.apikeyResetConfirm2"));
                if (!confirm2) {
                    alert(this.$t("views.login.apikeyResetNvm"));
                    return;
                }
            }
            await api.resetAPIKey(this.userdata.jwt);
            this.forceUserUpdate();
            /* eslint-enable no-restricted-globals, no-alert */
        },
    },
};
</script>
